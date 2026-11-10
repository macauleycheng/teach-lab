# SONiC 功能开发教育手册

> 对象：新进研发人员  
> 目的：以实际范例 `lbd-ec202012_anos`（LBD，Loop Back Detection）教你在 SONiC 上「从零开发一个功能」：**怎幺切模块、文档放哪里、怎幺跟系统沟通、debian 怎幺打包、`.mk` 怎幺改、怎幺进 docker image**。  
> 范例位置：`/home/manager/sonic-buildimage/lbd-ec202012_anos/`  
> 官方对照范例：`src/sonic-stp/` + `rules/sonic-stp.mk` + `rules/docker-stp.mk`（STP 与 LBD 同为 L2 daemon，结构几乎一对一对应，开发时可直接参考）。

---

## 目录

1. [先创建正确的心智模型](#1-先创建正确的心智模型)
2. [范例功能 LBD 的架构拆解](#2-范例功能-lbd-的架构拆解)
3. [一个功能该怎幺切模块](#3-一个功能该怎幺切模块)
4. [文档应该放在哪个目录](#4-文档应该放在哪个目录)
5. [功能如何与 SONiC 沟通（DB 访问）](#5-功能如何与-sonic-沟通db-访问)
6. [debian 打包](#6-debian-打包)
7. [相关 mk 文档怎幺修改](#7-相关-mk-文档怎幺修改)
8. [如何进入 docker image](#8-如何进入-docker-image)
9. [完整开发流程（逐步）](#9-完整开发流程逐步)
10. [开发注意事项与检查清单](#10-开发注意事项与检查清单)
11. [新进人员练习任务](#11-新进人员练习任务)

---

## 1. 先创建正确的心智模型

在 SONiC 上开发功能，和在传统 monolithic 交换器 OS（如早期的 ANOS/EC code）上开发，最大差异是：

| 项目 | 传统交换器 OS | SONiC |
|------|---------------|-------|
| 进程形态 | 同一个大进程内的模块 | **独立 daemon，跑在 Docker 容器里** |
| 模块间沟通 | 直接函数调用 / 共享内存 | **通过 Redis 中央数据库（DB）传递状态** |
| 下硬件 | 直接调用 SDK | 写 ASIC_DB → `syncd` → SAI → ASIC |
| 配置来源 | 私有 config | CONFIG_DB（由 config_db.json / CLI 填入） |
| 打包交付 | 整包 firmware | **一颗 `.deb`，装进某个 docker image** |

所以在 SONiC 开发一个功能，本质上就是：**写一个（或一组）daemon，通过 swss-common 读写 Redis DB，打包成 deb，装进一个 docker 容器，再由 build 系统组进 image。**

LBD 正是这个模式的教科书级范例——它原本是传统 OS 的 L2 功能（`liblbd`），被改写成 SONiC 形态。

---

## 2. 范例功能 LBD 的架构拆解

LBD（Loop Back Detection）侦测端口上的循环，侦测到就把端口 err-disable（shutdown）。它在 SONiC 上的组成：

```
lbd-ec202012_anos/
├── configure.ac          # autotools 项目配置（产生 configure）
├── autogen.sh            # 产生 build 系统并注入版本
├── getversion.sh         # 把 version.txt 的版本注入编译旗标
├── Makefile.am           # 顶层：SUBDIRS = lib lbdd lbdctrl
├── version.txt
├── lbdd/                 # ★ daemon 主进程（C）
│   ├── main.c            #   进程进入点
│   ├── lbd_mgr.c         #   内核逻辑（状态机/封包处理调度）
│   ├── lbd_om.c          #   object manager（数据对象）
│   ├── lbd_pkt.c/.h      #   LBD 封包收送
│   ├── lbd_backdoor.c/.h #   debug backdoor
│   └── include/
├── lbdctrl/              # ★ CLI 工具（C）
│   ├── lbd_cli.c         #   CLI 解析
│   ├── lbd_ctl.c         #   控制指令
│   ├── lbd_pmgr.c        #   与 daemon 沟通的 proxy
│   └── include/
├── lib/
│   ├── lbdsync/          # ★ C++ 桥接层：连接 SONiC Redis DB
│   │   ├── lbd_sync.cpp  #   用 swss-common 读写 DB，extern "C" 包给 C 调用
│   │   ├── lbd_sync.h
│   │   └── lbd_dbsync.h
│   └── stublib/          # 兼容旧 OS 的 stub（移植用的垫片）
└── debian/               # ★ deb 打包描述
    ├── control           #   套件定义/相依
    ├── rules             #   debhelper build 脚本
    ├── changelog
    ├── compat
    ├── lbd.install       #   哪些产物装到 deb 的哪个路径
    └── lbd.links         #   产生 .so symlink
```

三个「星号」区块就是 SONiC 功能的典型三件套：

1. **daemon（lbdd）** — 跑功能逻辑。
2. **桥接层（lbdsync）** — 功能逻辑 ↔ SONiC DB 的翻译层。
3. **CLI（lbdctrl）** — 操作/查找接口。

> 移植洞见：`lib/stublib/` 与 `-I/usr/include/sysinclude/...`、`-lcmnlib` 这些，是 LBD 从旧 OS 搬过来时保留的兼容层。**新功能若从头开发，不需要 stublib / cmnlib**，直接用 swss-common 即可。手册第 10 章会特别提醒。

---

## 3. 一个功能该怎幺切模块

参考 LBD，建议把功能切成以下几块（视复杂度可合并）：

| 模块 | 语言 | 职责 | LBD 对应 |
|------|------|------|----------|
| daemon 主体 | C/C++ | 功能逻辑、状态机、计时、封包收送 | `lbdd/` |
| DB 桥接层 | C++ | 用 swss-common 读 CONFIG_DB、写 APPL_DB/STATE_DB | `lib/lbdsync/` |
| CLI | C/Python | 查状态、下指令 | `lbdctrl/` |
| 打包 | debian/ | 编译产物打成 deb | `debian/` |

切模块的原则：

- **逻辑与 DB 访问分离**。LBD 把所有 swss-common 的调用集中在 `lbd_sync.cpp`，daemon 本身（C）只调用 `lbdsync_*()` 这组 `extern "C"` 接口。好处是 daemon 不必混 C/C++，DB schema 变动只改桥接层。
- **一个 daemon 专注一件事**。SONiC 偏好小而专的 daemon（像 `orchagent`、`teamd`、`stpd` 各管一块）。

---

## 4. 文档应该放在哪个目录

这是新人最常问的问题。对照 SONiC 惯例与 `sonic-stp` 的做法：

### 4.1 原代码放哪里

| 情况 | 放置位置 | 说明 |
|------|----------|------|
| 自研、平台无关的功能原代码 | `src/<你的功能>/` | 与 `src/sonic-stp`、`src/sonic-swss` 同级。**LBD 若要正式进 SONiC，应放这里**，例如 `src/sonic-lbd/` |
| 由外部 git 管理的组件 | 用 git submodule 挂在 `src/` 下 | 像 `src/sonic-swss` 就是 submodule |
| 厂商/平台专属 | `platform/<vendor>/` | 只有特定 ASIC 平台用到才放这 |

> 范例目前 `lbd-ec202012_anos/` 放在 repo 根目录，那是「开发暂存」的位置。**正式集成时请搬进 `src/`**（例如 `src/sonic-lbd/`），命名用小写、以 `-` 分隔，与既有惯例一致。

### 4.2 recipe（`.mk`）放哪里

| 文档 | 位置 | 作用 |
|------|------|------|
| package recipe | `rules/<功能>.mk` | 定义怎幺把 `src/<功能>/` build 成 deb。对照 `rules/sonic-stp.mk` |
| docker recipe | `rules/docker-<功能>.mk` | 若功能要自己的容器，定义 docker。对照 `rules/docker-stp.mk` |

### 4.3 容器内运行期文档放哪里

| 内容 | 容器内路径 | 来源（deb install 规则） |
|------|-----------|--------------------------|
| daemon 运行档 | `/usr/bin/lbdd` | `debian/lbd.install` |
| CLI 运行档 | `/usr/bin/lbdctrl`（或通过 `BASE_IMAGE_FILES` 暴露到 host） | `debian/lbd.install` / docker recipe |
| 共用库 | `/usr/lib/liblbdsync.so*` | `debian/lbd.install` + `debian/lbd.links` |

---

## 5. 功能如何与 SONiC 沟通（DB 访问）

这是 SONiC 开发的内核技能。LBD 的 `lib/lbdsync/lbd_sync.cpp` 是最好的教材。

### 5.1 用到的 DB 与表

SONiC 用多个 Redis DB 分工：

| DB | 用途 | LBD 的使用 |
|----|------|------------|
| `CONFIG_DB` | 用户配置 | 读 `VLAN_MEMBER`、`PORT`、`PORTCHANNEL` 配置 |
| `APPL_DB` | 应用层状态（下行给 orchagent） | 写 `PORT_TABLE`（err-disable、ingress filter）、STP 相关表 |
| `STATE_DB` | 系统运行状态 | 连接备用 |

### 5.2 swss-common 的用法（节录自 `lbd_sync.cpp`）

```cpp
#include "dbconnector.h"
#include "producerstatetable.h"
using namespace swss;

// 1) 连接 DB（以 DB 名称连，不用记 port/index）
DBConnector db("APPL_DB", 0);
DBConnector cfgDb("CONFIG_DB", 0);
DBConnector stateDb("STATE_DB", 0);

// 2) 绑定要操作的表
Table m_appPortTable(db, APP_PORT_TABLE_NAME);
Table m_cfgVlanMemberTable(cfgDb, CFG_VLAN_MEMBER_TABLE_NAME);

// 3) 写入：把 err-disable 状态写进 APPL_DB 的 PORT_TABLE
std::vector<FieldValueTuple> fv;
fv.emplace_back("err_disabled_reason", enabled ? "lbd" : "none");
m_appPortTable.set(if_name, fv);

// 4) 读取：查某端口是不是某 VLAN 的 member
std::vector<FieldValueTuple> fvs;
bool ok = m_cfgVlanMemberTable.get(key, fvs);
```

### 5.3 C daemon 怎幺调用 C++ 桥接层

daemon 是 C，桥接层是 C++。LBD 用 `extern "C"` 把 C++ 函数包成 C 可调用的接口：

```cpp
// lbd_sync.cpp
extern "C" {
    void lbdsync_set_ingress_filtering(char *ifName, bool enabled) {
        sync.setIngressFiltering(ifName, enabled);
    }
    bool lbdsync_is_vlan_member(uint32_t vlanId, char *ifName, bool *isTaggedMode) {
        return sync.isVlanMember(vlanId, ifName, isTaggedMode);
    }
}
```

daemon（C）只要 include 对应标头，直接调用 `lbdsync_set_ingress_filtering(...)` 即可，完全不碰 C++。

### 5.4 日志

统一用 swss-common 的 logger，输出会进 syslog，方便 `show logging` 查：

```cpp
#include "logger.h"
SWSS_LOG_NOTICE("LBD update appDB for %s port", if_name);
```

---

## 6. debian 打包

功能要进 SONiC，最终得是一颗 deb。LBD 的 `debian/` 展示了 autotools 项目的标准打包。

### 6.1 `debian/control` — 套件定义

```
Source: lbd
Build-Depends: dh-exec (>=0.3), debhelper (>= 9), autotools-dev, libtool, pkg-config,
 libswsscommon-dev, libhiredis-dev, libnl-3-dev, libnl-genl-3-dev
Package: lbd
Architecture: any
Depends: ${shlibs:Depends}
Description: This package contains AOS lbd daemon suits.

Package: lbd-dbg          # debug symbols 子套件
Architecture: any
Section: debug
Depends: lbd (=${binary:Version})
```

重点：

- **一个 source 可产出多个 binary package**（这里 `lbd` + `lbd-dbg`）。SONiC 惯例会额外产一个 `-dbg` 包放 debug symbols。
- `Build-Depends` 要列出编译期相依。LBD 在干净环境实测需要的最小集合为 `libswsscommon-dev`（桥接层 include `dbconnector.h`/`producerstatetable.h`/`logger.h`）、`libhiredis-dev`（`-lhiredis`）、`libnl-3-dev`/`libnl-genl-3-dev`（`-lnl-3`/`-lnl-genl-3`），以及 `libtool`/`pkg-config`（autogen 用）。范例原始的 `control` 只列了 `dh-exec/debhelper/autotools-dev`，正式集成时已补齐上述相依（见第 10 章注意事项）。

### 6.2 `debian/rules` — build 脚本

```makefile
#!/usr/bin/make -f
%:
	dh $@  --with autotools-dev        # 用 debhelper 跑 autotools 标准流程

override_dh_auto_install:
	dh_auto_install --destdir=debian/lbd

override_dh_clean:
	dh_clean --dbg-package=lbd-dbg
```

`dh $@ --with autotools-dev` 会自动依序跑 `./configure` → `make` → `make install`。

### 6.3 `debian/lbd.install` — 产物安装位置

左边是 build 出来的文档，右边是装进 deb 的目录：

```
lbdd/lbdd                            usr/bin
lbdctrl/lbdctrl                      usr/bin
lib/stublib/.libs/libstublib.so.0.0.0  usr/lib
lib/lbdsync/.libs/liblbdsync.so.0.0.0  usr/lib
```

### 6.4 `debian/lbd.links` — symlink（dh-exec）

产生共用库惯用的版本 symlink：

```
#! /usr/bin/dh-exec
/usr/lib/liblbdsync.so.0.0.0 /usr/lib/liblbdsync.so
/usr/lib/liblbdsync.so.0.0.0 /usr/lib/liblbdsync.so.0
```

---

## 7. 相关 mk 文档怎幺修改

这是题目的重点。要让 build 系统认得你的功能，需添加/修改两类 recipe。以 LBD 正式化为 `src/sonic-lbd/` 为例，对照官方 `sonic-stp`。

### 7.1 添加 package recipe：`rules/sonic-lbd.mk`

仿 `rules/sonic-stp.mk`：

```makefile
# LBD package

LBD_VERSION = 1.0.0
export LBD_VERSION

LBD = lbd_$(LBD_VERSION)_$(CONFIGURED_ARCH).deb
$(LBD)_SRC_PATH = $(SRC_PATH)/sonic-lbd
# 编译期相依：用到 swss-common 的开发标头/库
$(LBD)_DEPENDS += $(LIBSWSSCOMMON_DEV)
# 运行期相依：容器内要有 runtime 库
$(LBD)_RDEPENDS += $(LIBSWSSCOMMON)
SONIC_DPKG_DEBS += $(LBD)

# debug symbols 子套件
LBD_DBG = lbd-dbg_$(LBD_VERSION)_$(CONFIGURED_ARCH).deb
$(LBD_DBG)_DEPENDS += $(LBD)
$(LBD_DBG)_RDEPENDS += $(LBD)
$(eval $(call add_derived_package,$(LBD),$(LBD_DBG)))

export LBD
```

逐行说明：

- `SONIC_DPKG_DEBS += $(LBD)`：告诉 build 系统「用 `dpkg-buildpackage` 从 `$(LBD)_SRC_PATH` 建这颗 deb」。
- `_DEPENDS` 是**编译期**相依（build 这包前要先装好的其他 deb）；`_RDEPENDS` 是**运行期**相依（装进 docker 时要一起装的 deb）。两者别搞混。
- `add_derived_package` 让 `-dbg` 共用主套件的 metadata，不用重写一遍。

### 7.2 添加 docker recipe：`rules/docker-lbd.mk`（若功能需要自己的容器）

仿 `rules/docker-stp.mk`：

```makefile
# Docker image for LBD

DOCKER_LBD_STEM = docker-lbd
DOCKER_LBD = $(DOCKER_LBD_STEM).gz
DOCKER_LBD_DBG = $(DOCKER_LBD_STEM)-$(DBG_IMAGE_MARK).gz

$(DOCKER_LBD)_PATH = $(DOCKERS_PATH)/$(DOCKER_LBD_STEM)

# 要装进这个容器的 deb
$(DOCKER_LBD)_DEPENDS += $(LBD) $(SWSS)
$(DOCKER_LBD)_DBG_DEPENDS += $(LBD)

# 以哪个 base image 叠上去
$(DOCKER_LBD)_LOAD_DOCKERS = $(DOCKER_CONFIG_ENGINE_TRIXIE)

$(DOCKER_LBD)_VERSION = 1.0.0
$(DOCKER_LBD)_PACKAGE_NAME = lbd
$(DOCKER_LBD)_CONTAINER_NAME = lbd
$(DOCKER_LBD)_RUN_OPT += -t --cap-add=NET_ADMIN --cap-add=SYS_ADMIN
$(DOCKER_LBD)_RUN_OPT += -v /etc/sonic:/etc/sonic:ro

# 把 CLI 暴露到 host（在 host 上打 lbdctl 会转进容器）
$(DOCKER_LBD)_BASE_IMAGE_FILES += lbdctrl:/usr/bin/lbdctl

# 用开关控制是否纳入 image
ifeq ($(INCLUDE_LBD), y)
SONIC_DOCKER_IMAGES += $(DOCKER_LBD)
SONIC_INSTALL_DOCKER_IMAGES += $(DOCKER_LBD)
SONIC_DOCKER_DBG_IMAGES += $(DOCKER_LBD_DBG)
SONIC_INSTALL_DOCKER_DBG_IMAGES += $(DOCKER_LBD_DBG)
endif

SONIC_TRIXIE_DOCKERS += $(DOCKER_LBD)
SONIC_TRIXIE_DBG_DOCKERS += $(DOCKER_LBD_DBG)
```

> 若功能很轻、不需要独立容器，也可以**不建 docker recipe**，改在 7.3 把 `$(LBD)` 加进既有容器（例如 `docker-stp` 或 `swss`）的 `_DEPENDS`。
>
> **本次 LBD 正式集成的最终决定：采轻量做法，不新建独立容器，将 `$(LBD)` 挂进 `docker-stp`。** 理由：LBD 与 STP 同为 L2 功能，且 `lbd_sync.cpp` 直接参照 STP 的 APP table（`APP_STP_PORT_TABLE_NAME` 等），数据面高度耦合，共用容器最自然。落地方式见 7.3（在 deb 能成功 build 之后再加）。

### 7.3 把 deb 加进某个既有容器（替代 7.2 的轻量做法）

例如要把 LBD 放进 STP 容器，改 `rules/docker-stp.mk`：

```makefile
$(DOCKER_STP)_DEPENDS += $(STP) $(SWSS) $(LBD)   # ← 加 $(LBD)
```

### 7.4 加入 build 开关（可选但建议）

仿 STP 的 `INCLUDE_STP`，在 `rules/config` 加默认值、在 `slave.mk` 创建对应关系，让用户能用 `INCLUDE_LBD=y/n` 选装。这让功能可被不同客户 image 选配。

### 7.5 recipe 之间的关系图

```
rules/config            → 定义 INCLUDE_LBD 缺省
   │
rules/sonic-lbd.mk      → 怎幺把 src/sonic-lbd build 成 lbd_*.deb   (SONIC_DPKG_DEBS)
   │  产出 $(LBD)
   ▼
rules/docker-lbd.mk     → 把 $(LBD) 装进 docker-lbd 容器            (SONIC_DOCKER_IMAGES)
   │  产出 $(DOCKER_LBD)
   ▼
platform/*/one-image.mk → 把 $(DOCKER_LBD) 组进最终 .bin image
```

---

## 8. 如何进入 docker image

一颗 deb 要真的跑在交换器上，流程是：

1. `rules/sonic-lbd.mk` 把原代码 build 成 `lbd_*.deb`（进 `target/debs/`）。
2. `rules/docker-lbd.mk` 的 `_DEPENDS += $(LBD)` 让这颗 deb 在建 docker image 时被 `dpkg -i` 装进容器。
3. 容器 image 被 `one-image.mk` 收进最终 `.bin`。
4. 启动时由 `supervisord`（容器内）拉起 `/usr/bin/lbdd`。

`BASE_IMAGE_FILES` 的特殊作用：把容器内的工具「投影」到 host OS，用户在 host shell 直接打 CLI（例如 `stpctl` / 范例中的 `lbdctl`）就会被导进容器运行，不必手动 `docker exec`。

---

## 9. 完整开发流程（逐步）

以「开发 LBD 并正式集成进 SONiC」为例：

1. **放原代码**：把进程放进 `src/sonic-lbd/`（小写、`-` 分隔）。
2. **写 DB 桥接层**：集中所有 swss-common 访问在一个 `.cpp`，用 `extern "C"` 对 daemon 开接口（参考 `lbd_sync.cpp`）。
3. **写打包描述**：补 `debian/control`（记得 `Build-Depends` 加 `libswsscommon-dev`）、`rules`、`*.install`、`*.links`、`changelog`、`compat`。
4. **写 package recipe**：添加 `rules/sonic-lbd.mk`（第 7.1 节）。
5. **决定容器归属**：建 `rules/docker-lbd.mk`（7.2）或挂进既有容器（7.3）。
6. **（可选）加开关**：`INCLUDE_LBD`（7.4）。
7. **配置平台**：

   ```shell
   cd /home/manager/sonic-buildimage
   make init
   make configure PLATFORM=broadcom
   ```

8. **先单独 build 这颗 deb 验证**（比 build 整包 image 快很多）：

   ```shell
   make target/debs/trixie/lbd_1.0.0_amd64.deb
   # 本 repo 实测 BLDENV = trixie（顶层 Makefile 的 % 目标缺省以 BLDENV=trixie 调用
   # Makefile.work，且 sonic-slave-trixie 容器已存在）。其他分支可能是 bookworm 等，
   # 依 Makefile 内各目标配置而定。首次运行会连带把相依（含 libswsscommon）一并 build。
   ```

9. **验证 deb 内容**：

   ```shell
   dpkg-deb -c target/debs/<BLDENV>/lbd_1.0.0_amd64.deb   # 看装了哪些档、路径对不对
   dpkg-deb -I target/debs/<BLDENV>/lbd_1.0.0_amd64.deb   # 看相依与 metadata
   ```

10. **build 容器 / 整包 image**：

    ```shell
    make SONIC_BUILD_JOBS=4 INCLUDE_LBD=y PLATFORM=broadcom target/sonic-broadcom.bin
    ```

11. **上机验证**：装 image 后 `docker ps` 看容器起来、`show logging` 看 `SWSS_LOG` 输出、用 `redis-cli` 确认 DB 有被正确写入。

---

## 10. 开发注意事项与检查清单

### 10.1 架构类

- **不要在 daemon 里直接碰硬件 SDK**。要下 ASIC 一律通过 DB → orchagent/syncd → SAI。LBD 的做法（写 APPL_DB 的 `PORT_TABLE`）就是正解。
- **DB 访问集中管理**。所有 swss-common 调用放一个桥接层，别散落各处。
- **用既有 DB 表/字段**。写入前先确认 schema（参考 `sonic-swss-common` 的 `schema.h` 与 YANG model），不要自创会与既有 orchagent 冲突的字段。
- **一个 daemon 一件事**，别把多种功能塞进同一进程。

### 10.2 移植类（LBD 这种从旧 OS 搬过来的情况特别注意）

- 范例里的 `lib/stublib/`、`-lcmnlib`、`-I/usr/include/sysinclude/...` 都是**旧 OS 兼容垫片**。**全新功能不要沿用**，直接用 swss-common。移植既有 code 时，目标应是逐步把 stub 换成原生 SONiC API。
- **已知阻碍（实测）**：把范例原封不动搬进 `src/sonic-lbd/` 并用 `make target/debs/trixie/lbd_1.0.0_amd64.deb` build 时，相依（含 `libswsscommon`）都能成功，但 lbd 本身会在**第一个编译单元 `lib/stublib/stub.c` 就 fatal error**：

  ```
  include/stub.h:4:10: fatal error: sys_type.h: No such file or directory
  ```

  根因是 `stublib` 的标头（`sys_type.h`、`port.h`）与 `-lcmnlib` 都来自旧 OS 的 `cmnlib` 树，SONiC sonic-slave 容器没有它。`-lcmnlib` 的 link 失败是同一根因的下一阶段（compile 比 link 早发生，所以先卡在缺标头）。要让它编得过，必须先处理 cmnlib 垫片（择一）：(1) 把所需的最小 cmnlib 标头/库子集自带进 `src/sonic-lbd/` 并改 `Makefile.am` 的 include 路径；(2) 用精简的本地兼容标头取代 `sys_type.h`/`port.h`，并以本地实作取代 `-lcmnlib` 被参照的符号。这两者都属**移植工作**，不在「让它进 build 系统」的骨架集成范围内，且须确认无 NDA/机密才能放进 repo。
- `getversion.sh` 用 `sed` 动态改 `Makefile.am` 注入版本是旧习惯，脆弱且难维护；新项目建议用 `configure.ac` 的 `AC_INIT` 版本或编译旗标。

### 10.3 打包/相依类

- `control` 的 `Build-Depends` 要**诚实列出编译相依**（用到 swss-common 就加 `libswsscommon-dev`），否则在干净环境 build 会失败。
- recipe 的 `_DEPENDS`（编译期）vs `_RDEPENDS`（运行期）不要混。
- 一定要产 `-dbg` 套件，现场调试会需要 symbols。

### 10.4 代码规范类（本项目 C 开发守则）

本范例是 C/C++ 项目，提交前请对齐团队 C 编码规范（见全域 coding standard），重点提醒：

- 缩进 4 空格、不用 tab；LF 换行；行长 < 120。
- 所有 `.h` 要有 `#ifndef/#define/#endif` include guard。
- 全域变量禁用，改用 `static` 文档层级变量 + getter/setter。
- `if/while/switch` 一律加大括号，Allman 风格（`{` 独立一行）。
- `switch` 必含 `default`；刻意 fall-through 要注解。
- 检查所有函数回传值（尤其 `malloc`）；`free` 后把指针设 NULL。
- 用 `strncpy` 并以**目的缓冲区大小**为界；用 `strtok_r` 取代 `strtok`。
- 清掉所有编译警告才能 check-in。

> 注意：范例的 `lbd_sync.cpp` 等文档是既有移植 code，其风格不一定完全符合上述规范；请以规范为准，不要照抄旧风格。

### 10.5 提交前检查清单

- [ ] 原代码放在 `src/sonic-<功能>/`，命名小写 `-` 分隔
- [ ] DB 访问集中在单一桥接层，`extern "C"` 接口清楚
- [ ] `debian/control` 相依完整（含 `libswsscommon-dev`）
- [ ] `rules/sonic-<功能>.mk` 的 `SRC_PATH`、`_DEPENDS`、`_RDEPENDS` 正确
- [ ] 决定容器归属（独立 docker 或挂既有容器）并改好 docker recipe
- [ ] 单独 build deb 成功、`dpkg-deb -c/-I` 检查无误
- [ ] 整包 image build 成功、上机容器能起、DB 写入正确、syslog 有预期输出
- [ ] 无编译警告、符合 C 编码规范
- [ ] 不含 NDA / 机密内容；旧 OS 垫片已评估是否移除

---

## 11. 新进人员练习任务

依序完成，每步都有可验收产出：

1. **读懂范例（0.5 天）**：读 `lbd_sync.cpp` 与本文第 2、5 章。验收：能口述 LBD 侦测到循环后，状态如何经 APPL_DB → orchagent 让端口 err-disable。
2. **对照官方范例（0.5 天）**：把 `rules/sonic-stp.mk` / `rules/docker-stp.mk` 与本文第 7 章逐行对照。验收：能指出 STP recipe 中每一行的作用。
3. **把 LBD 正式化（1 天）**：将 `lbd-ec202012_anos/` 搬进 `src/sonic-lbd/`，补 `rules/sonic-lbd.mk`。验收：`make target/debs/.../lbd_*.deb` 成功，`dpkg-deb -c` 看到 `lbdd`/`lbdctrl` 在 `usr/bin`。
4. **进容器（0.5 天）**：把 `$(LBD)` 挂进一个容器并 build。验收：image 内容器含 `lbdd`。
5. **动手改一个小功能（1 天）**：例如在桥接层多写一个 STATE_DB 字段记录最后侦测时间。验收：`redis-cli` 查得到新字段。

> 完成 1～4 即具备「在 SONiC 上添加一个功能 daemon 并集成进 image」的独立能力。

---

## 附录：关键文档速查

| 路径 | 作用 |
|------|------|
| `lbd-ec202012_anos/lbdd/` | 范例：daemon 主进程 |
| `lbd-ec202012_anos/lib/lbdsync/lbd_sync.cpp` | 范例：swss-common DB 桥接层（必读） |
| `lbd-ec202012_anos/debian/` | 范例：deb 打包描述 |
| `src/sonic-stp/` | 官方对照：L2 daemon 原代码 |
| `rules/sonic-stp.mk` | 官方对照：package recipe 范本 |
| `rules/docker-stp.mk` | 官方对照：docker recipe 范本 |
| `rules/swss.mk` | 官方对照：DPKG recipe + `_DEPENDS/_RDEPENDS` 用法 |
| `README.buildsystem.md` | build 系统（target group）权威说明 |
| `src/sonic-swss-common/` | swss-common 原代码（DBConnector/Table/ProducerStateTable API） |

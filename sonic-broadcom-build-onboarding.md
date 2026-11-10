# SONiC Broadcom Image Build 教学文档

> 对象：新进研发人员  
> 目的：提供一套正式、可依循的 SONiC（Broadcom 平台）build 教学，取代「每个人自己去啃 README」的学习方式。  
> 范围：本文聚焦于 **Broadcom 平台**的 image build，特别是 **Broadcom 专属 debian 套件（受 NDA 保护的 SAI/SDK）的取得与替换流程**，以及 **SONiC 整体架构**的教育说明。

---

## 目录

1. [文档使用方式](#1-文档使用方式)
2. [SONiC 架构总览](#2-sonic-架构总览)
3. [sonic-buildimage build 系统架构](#3-sonic-buildimage-build-系统架构)
4. [Broadcom 平台的套件组成](#4-broadcom-平台的套件组成)
5. [Broadcom debian 的取得（NDA 议题）](#5-broadcom-debian-的取得nda-议题)
6. [替换 Broadcom debian 来 build 不同 image](#6-替换-broadcom-debian-来-build-不同-image)
7. [客户二次开发 debian 的集成流程](#7-客户二次开发-debian-的集成流程)
8. [完整 build 操作流程](#8-完整-build-操作流程)
9. [常见问题排查](#9-常见问题排查)
10. [新进人员学习路径](#10-新进人员学习路径)

---

## 1. 文档使用方式

本文与项目内既有文档互补，并非取代。既有文档的角色：

| 文档 | 角色 |
|------|------|
| `README.md` | 原始官方指引，硬件需求、环境准备、build 指令 |
| `README.buildsystem.md` | build 系统（make backend/frontend、target group）的权威说明 |
| `BUILD_CONFIG.md` | build 可调整的配置旗标 |
| **本文 `docs/sonic-broadcom-build-onboarding.md`** | **新进人员正式流程 + Broadcom debian 取得/替换的 how-to** |

阅读建议：先读第 2、3 章创建心智模型，再依任务跳到第 5～8 章实作。

---

## 2. SONiC 架构总览

SONiC（Software for Open Networking in the Cloud）是一套以 **Linux（Debian）+ Docker 容器**为基础的交换器网络操作系统。内核设计理念是：**将交换器功能拆成多个独立的 Docker 容器，通过中央数据库（Redis）交换状态**。

### 2.1 运行期架构（交换器上运作的样子）

```
┌─────────────────────────────────────────────────────────────┐
│                     SONiC (Debian Host OS)                    │
│                                                               │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────┐   │
│  │  bgp     │  │ swss     │  │ syncd    │  │ database     │   │
│  │ (FRR)    │  │(orchestr)│  │(SAI→ASIC)│  │ (Redis)      │   │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └──────┬───────┘   │
│       │             │             │               │           │
│       └─────────────┴─────────────┴───────────────┘           │
│                 通过 Redis(APPL/ASIC_DB) 沟通                  │
│                                   │                           │
│                          ┌────────▼─────────┐                 │
│                          │   SAI (厂商库)    │  ← Broadcom     │
│                          │   libsaibcm      │     专属 debian  │
│                          └────────┬─────────┘                 │
└───────────────────────────────────┼───────────────────────────┘
                                     │ (内核模块 linux_bcm_knet 等)
                              ┌──────▼───────┐
                              │  ASIC 芯片    │
                              │ (Broadcom)   │
                              └──────────────┘
```

### 2.2 关键容器与角色

| 容器 | 角色 |
|------|------|
| `database` | Redis，所有状态的中央保存（CONFIG_DB / APPL_DB / ASIC_DB / STATE_DB …） |
| `swss` | Switch State Service，各种 orchagent（路由、ACL、port 等）将配置写入 ASIC_DB |
| `syncd` | 读取 ASIC_DB，通过 **SAI** 下发到 ASIC。**这是 Broadcom SAI debian 被安装的容器** |
| `bgp` | FRR routing stack |
| `teamd` / `lldp` / `snmp` / `pmon` 等 | 各功能性服务 |

### 2.3 SAI 的关键地位

**SAI（Switch Abstraction Interface）** 是 SONiC 能够支持多厂商 ASIC 的关键。它是一层标准化 API：

- 上层（syncd）只调用 SAI 标准 API，不管底下是哪家芯片。
- 下层由各 ASIC 厂商提供 SAI 实作（`.deb` 套件）对接自家 SDK/芯片。

对 Broadcom 而言，SAI 实作就是 **`libsaibcm`**（以及开发用的 `libsaibcm-dev`）。这个套件由 Broadcom 提供、受 NDA 保护，正是本文第 5～7 章的主角。

---

## 3. sonic-buildimage build 系统架构

（权威细节见 `README.buildsystem.md`，此处为新进人员导读）

### 3.1 两层结构

- **Backend（后端）**：`Makefile`、`slave.mk`、`sonic-slave-*/Dockerfile`。
  - 所有 build 都在一个叫 `sonic-slave` 的 Docker 容器内运行，环境一致。
  - `slave.mk` 定义了各种「target group」的通用建置规则。
- **Frontend（前端）**：`rules/`（平台无关 recipe）与 `platform/[VENDOR]/`（厂商专属 recipe）。
  - 每个 recipe 是一个小的 `.mk`，描述「要 build 什幺、相依什幺、怎幺取得」。

### 3.2 build 产物流向

```
src/ + platform/      target/debs/        最终 image
(原代码/recipe)  ──►  (.deb 套件)   ──►   target/sonic-broadcom.bin
                                          │
                                          └─ 安装进各 docker → 打包成 ONIE installer
```

### 3.3 必懂的 target group（取得套件的四种方式）

这是理解「Broadcom debian 怎幺进来」的关键。来源见 `README.buildsystem.md`，内容为求教学已改写：

| Target group | 意义 | 使用时机 |
|--------------|------|----------|
| `SONIC_DPKG_DEBS` | 从原代码用 `dpkg-buildpackage` 建置 | 自家开源套件 |
| `SONIC_MAKE_DEBS` | 用自订 Makefile 建置（可套 patch） | 需要特殊建置流程 |
| `SONIC_ONLINE_DEBS` | 从网络 URL 下载现成 `.deb` | **缺省的 Broadcom SAI 取得方式** |
| `SONIC_COPY_DEBS` | 从本机指定路径拷贝现成 `.deb` | **替换成本地 Broadcom/客制 debian** |

> 关键认知：Broadcom SAI 缺省走 `SONIC_ONLINE_DEBS`（在线下载）。要替换成你自己拿到或二次开发的 debian，内核手法就是改走 `SONIC_COPY_DEBS`（本地拷贝）。

build 系统对这两者的实际处理（摘自 `slave.mk`）：

- `SONIC_ONLINE_DEBS`：对每个套件运行 `curl -L -f -o <目标> <URL>` 下载，并写入 DPKG cache。
- `SONIC_COPY_DEBS`：直接从 `$(套件)_PATH` 指定的本机路径拷贝到 `target/debs/`。

---

## 4. Broadcom 平台的套件组成

Broadcom 平台 recipe 都在 `platform/broadcom/`。`platform/broadcom/rules.mk` 是入口，会 `include` 以下与 SAI 相关的文档：

| 文档 | 内容 | 对应 ASIC 家族 |
|------|------|----------------|
| `sai-xgs.mk` | `libsaibcm`（XGS：Tomahawk/Trident 等） | XGS 系列 |
| `sai-dnx.mk` | `libsaibcm_dnx`（DNX：Jericho/Qumran 等） | DNX 系列 |
| `sai-legacy-th.mk` | 旧版 Tomahawk 兼容 SAI | Legacy TH |
| `sai-modules.mk` | SAI 内核驱动模块（`saibcm-modules`） | 通用 |

以 `sai-xgs.mk` 为例，定义 Broadcom SAI 套件的方式如下（节录实际内容）：

```makefile
LIBSAIBCM_XGS_VERSION = 15.2.0.0.0.0.3.1
LIBSAIBCM_XGS_BRANCH_NAME = SAI_15.2.0_GA

LIBSAIBCM_XGS_URL_PREFIX = "$(BUILD_PUBLIC_URL)/sai/sai-broadcom/$(LIBSAIBCM_XGS_BRANCH_NAME)/$(LIBSAIBCM_XGS_VERSION)/xgs"

# Runtime 套件
BRCM_XGS_SAI = libsaibcm_$(LIBSAIBCM_XGS_VERSION)_amd64.deb
$(BRCM_XGS_SAI)_URL = "$(LIBSAIBCM_XGS_URL_PREFIX)/$(BRCM_XGS_SAI)"

# Dev 套件（由 runtime 衍生）
BRCM_XGS_SAI_DEV = libsaibcm-dev_$(LIBSAIBCM_XGS_VERSION)_amd64.deb
$(eval $(call add_derived_package,$(BRCM_XGS_SAI),$(BRCM_XGS_SAI_DEV)))
$(BRCM_XGS_SAI_DEV)_URL = "$(LIBSAIBCM_XGS_URL_PREFIX)/$(BRCM_XGS_SAI_DEV)"

# 注册成在线下载套件
SONIC_ONLINE_DEBS += $(BRCM_XGS_SAI)
```

重点拆解：

- `BUILD_PUBLIC_URL` 缺省是官方公开 mirror（`rules/config` 内定义，缺省 `https://packages.trafficmanager.net/public`）。
- `libsaibcm`（runtime）是真正装进 `syncd` 容器的 SAI 库。
- `libsaibcm-dev` 是 build `syncd` 时需要的开发标头/库，用 `add_derived_package` 由 runtime 衍生（共用 metadata）。
- SAI 如何注入 syncd 容器，见 `platform/broadcom/docker-syncd-brcm.mk`：

```makefile
$(DOCKER_SYNCD_BASE)_DEPENDS += $(SYNCD)
$(DOCKER_SYNCD_BASE)_DEPENDS += $(BRCM_XGS_SAI)   # ← SAI 被加为 syncd 容器相依
```

以及 `platform/broadcom/rules.mk`：

```makefile
# 把 brcm sai 注入 syncd build
$(SYNCD)_DEPENDS += $(BRCM_XGS_SAI) $(BRCM_XGS_SAI_DEV)
```

---

## 5. Broadcom debian 的取得（NDA 议题）

### 5.1 为什幺需要特别处理

Broadcom 的 SAI/SDK 二进位（`libsaibcm*.deb`）属于**受 NDA 保护的厂商资产**：

- 公开 mirror 上的版本可能受限、或非你客户需要的特定版本。
- 客户若有自家 SDK patch、或 Broadcom 直接交付的特殊版本，**无法放到公开来源**。
- 因此实务上常见需求是：**用本机/内部来源的 `.deb` 取代缺省的在线下载**。

> 合规提醒：Broadcom SAI/SDK debian 的散布受 NDA 约束。请勿将这些 `.deb`、或其 URL/帐密，提交进公开 repository、公开 CI log 或外部可访问的位置。内部传递请走公司核可的受控管道。

### 5.2 合法取得管道（择一）

1. **Broadcom 官方交付**：签署 NDA 后，由 Broadcom 通过其客户入口 / 业务窗口提供对应 SAI 版本。
2. **公司内部 artifact server**：公司在内部 mirror（如 Artifactory / 内部 HTTP server）存放已授权的 `.deb`。
3. **客户提供**：客户基于其与 Broadcom 的 NDA，提供自家二次开发后的 `.deb`。

### 5.3 确认你手上的套件版本

命名规则为 `libsaibcm_<版本>_amd64.deb`。build 要成功，`.mk` 内的 `LIBSAIBCM_*_VERSION` 必须与实际文档名版本一致，否则 `curl` 或拷贝会找不到文档。

---

## 6. 替换 Broadcom debian 来 build 不同 image

本章是本文内核。目标：**在不改动官方缺省来源的前提下，用你自己的 Broadcom debian build 出 image**。

有三种做法，依「侵入程度」由低到高排列。**建议新手从做法 A 开始。**

### 做法 A：改走本地拷贝（`SONIC_COPY_DEBS`）— 推荐

概念：把 SAI 套件从「在线下载」改成「从本机路径拷贝」。这正是 `platform/nephos/sai.mk` 既有的官方范例所用的模式（以环境变量切换本地/在线）。

**步骤**

1. 把你取得的 `.deb` 放到项目可访问的本机路径，例如：

   ```
   platform/broadcom/sai-local/libsaibcm_15.2.0.0.0.0.3.1_amd64.deb
   platform/broadcom/sai-local/libsaibcm-dev_15.2.0.0.0.0.3.1_amd64.deb
   ```

2. 修改 `platform/broadcom/sai-xgs.mk`，将 `_URL` 改为 `_PATH`，并把注册从 `SONIC_ONLINE_DEBS` 改为 `SONIC_COPY_DEBS`：

   ```makefile
   LIBSAIBCM_XGS_VERSION = 15.2.0.0.0.0.3.1

   # Runtime 套件：改用本地路径
   BRCM_XGS_SAI = libsaibcm_$(LIBSAIBCM_XGS_VERSION)_amd64.deb
   $(BRCM_XGS_SAI)_PATH = $(PLATFORM_PATH)/sai-local/$(BRCM_XGS_SAI)

   # Dev 套件
   BRCM_XGS_SAI_DEV = libsaibcm-dev_$(LIBSAIBCM_XGS_VERSION)_amd64.deb
   $(eval $(call add_derived_package,$(BRCM_XGS_SAI),$(BRCM_XGS_SAI_DEV)))
   $(BRCM_XGS_SAI_DEV)_PATH = $(PLATFORM_PATH)/sai-local/$(BRCM_XGS_SAI_DEV)

   # 注册改为本地拷贝
   SONIC_COPY_DEBS += $(BRCM_XGS_SAI)

   $(BRCM_XGS_SAI_DEV)_DEPENDS += $(BRCM_XGS_SAI)
   $(BRCM_XGS_SAI)_SKIP_VERSION = y
   $(BRCM_XGS_SAI_DEV)_SKIP_VERSION = y
   $(eval $(call add_conflict_package,$(BRCM_XGS_SAI_DEV),$(LIBSAIVS_DEV)))
   ```

   > `$(PLATFORM_PATH)` 在 Broadcom build 时即 `platform/broadcom`。

### 做法 B：用环境变量切换（更干净、可多客户共用）— 建议团队采用

仿照 `platform/nephos/sai.mk` 的官方写法，用一个环境变量决定要「本地拷贝」还是「在线下载」。好处是**不同客户 build 时不需改码，只改环境变量**。

在 `platform/broadcom/sai-xgs.mk` 改成：

```makefile
LIBSAIBCM_XGS_VERSION = 15.2.0.0.0.0.3.1
LIBSAIBCM_XGS_BRANCH_NAME = SAI_15.2.0_GA
LIBSAIBCM_XGS_URL_PREFIX = "$(BUILD_PUBLIC_URL)/sai/sai-broadcom/$(LIBSAIBCM_XGS_BRANCH_NAME)/$(LIBSAIBCM_XGS_VERSION)/xgs"

# 在此指定本地 SAI 存放目录；留空则走在线下载
BRCM_XGS_SAI_LOCAL_PATH ?=
export BRCM_XGS_SAI_LOCAL_PATH

BRCM_XGS_SAI = libsaibcm_$(LIBSAIBCM_XGS_VERSION)_amd64.deb
BRCM_XGS_SAI_DEV = libsaibcm-dev_$(LIBSAIBCM_XGS_VERSION)_amd64.deb
$(eval $(call add_derived_package,$(BRCM_XGS_SAI),$(BRCM_XGS_SAI_DEV)))

ifneq ($(BRCM_XGS_SAI_LOCAL_PATH), )
# ── 本地来源 ──
$(BRCM_XGS_SAI)_PATH     = $(BRCM_XGS_SAI_LOCAL_PATH)/$(BRCM_XGS_SAI)
$(BRCM_XGS_SAI_DEV)_PATH = $(BRCM_XGS_SAI_LOCAL_PATH)/$(BRCM_XGS_SAI_DEV)
SONIC_COPY_DEBS += $(BRCM_XGS_SAI)
else
# ── 在线来源（官方缺省）──
$(BRCM_XGS_SAI)_URL     = "$(LIBSAIBCM_XGS_URL_PREFIX)/$(BRCM_XGS_SAI)"
$(BRCM_XGS_SAI_DEV)_URL = "$(LIBSAIBCM_XGS_URL_PREFIX)/$(BRCM_XGS_SAI_DEV)"
SONIC_ONLINE_DEBS += $(BRCM_XGS_SAI)
endif

$(BRCM_XGS_SAI_DEV)_DEPENDS += $(BRCM_XGS_SAI)
$(BRCM_XGS_SAI)_SKIP_VERSION = y
$(BRCM_XGS_SAI_DEV)_SKIP_VERSION = y
$(eval $(call add_conflict_package,$(BRCM_XGS_SAI_DEV),$(LIBSAIVS_DEV)))
```

build 时：

```shell
# 用客户 A 的本地 SAI
make BRCM_XGS_SAI_LOCAL_PATH=/data/sai/customerA PLATFORM=broadcom target/sonic-broadcom.bin

# 不带变量 → 走官方在线版本（回到缺省行为）
make PLATFORM=broadcom target/sonic-broadcom.bin
```

> 要让 `make` 把这个变量带进 build 容器，需确认它被 `export`（上方已 `export`）；若使用 `rules/config.user` 也可持久化配置。

### 做法 C：改内部 mirror URL（整个团队统一改来源）

若公司有内部 artifact server，最省事的是只改 `BUILD_PUBLIC_URL` 指向内部 mirror，套件结构沿用官方版面：

```shell
make BUILD_PUBLIC_URL=https://artifactory.内部网域/sonic-public PLATFORM=broadcom target/sonic-broadcom.bin
```

前提：内部 mirror 上要有与官方相同的目录结构与文档名（`sai/sai-broadcom/<branch>/<version>/xgs/libsaibcm_*.deb`）。

### 三种做法比较

| 做法 | 侵入性 | 适合情境 | 多客户友善度 |
|------|--------|----------|--------------|
| A 本地拷贝（硬改） | 中（改 recipe） | 单次、单客户快速验证 | 低 |
| B 环境变量切换 | 低（改一次 recipe，之后只改变量） | **多客户、CI 常态化** | 高 |
| C 内部 mirror | 最低（完全不改 recipe） | 全团队统一内部来源 | 中 |

> 另外请记得 DNX / legacy-TH 客户：若你的平台用的是 DNX 或 legacy-TH 芯片，相同手法要套到 `sai-dnx.mk` / `sai-legacy-th.mk`，而非只改 `sai-xgs.mk`。

---

## 7. 客户二次开发 debian 的集成流程

不同客户可能对 Broadcom SAI/SDK 做自家二次开发，产出自己的 `.deb`。建议用**做法 B（环境变量切换）**搭配以下管理惯例：

### 7.1 目录与版本管理建议

```
/data/sai/
  customerA/
    libsaibcm_<verA>_amd64.deb
    libsaibcm-dev_<verA>_amd64.deb
  customerB/
    libsaibcm_<verB>_amd64.deb
    libsaibcm-dev_<verB>_amd64.deb
```

- **不要**把客户 `.deb` 提交进 git（NDA + 体积）。改用内部 artifact server 或受控目录。
- 每个客户的 SAI 版本号若不同，需同步调整对应 `.mk` 的 `LIBSAIBCM_*_VERSION`，或让版本也由环境变量带入。

### 7.2 版本号也参数化（高端）

若客户版本号各异，可把版本也做成变量：

```makefile
LIBSAIBCM_XGS_VERSION ?= 15.2.0.0.0.0.3.1
```

build 时覆写：

```shell
make LIBSAIBCM_XGS_VERSION=15.2.0.0.0.0.9.9 \
     BRCM_XGS_SAI_LOCAL_PATH=/data/sai/customerB \
     PLATFORM=broadcom target/sonic-broadcom.bin
```

### 7.3 每客户一份配置档（最可维护）

为每个客户准备一份 `config.user` 片段，避免 build 指令过长、也便于 CI：

```makefile
# rules/config.user  （此档被 gitignore，不会进版控）
BRCM_XGS_SAI_LOCAL_PATH = /data/sai/customerA
LIBSAIBCM_XGS_VERSION = 15.2.0.0.0.0.3.1
SONIC_CONFIG_BUILD_JOBS = 4
```

### 7.4 验证替换是否生效

build 后，从产出的 deb 目录确认版本与来源：

```shell
ls -l target/debs/*/libsaibcm*_amd64.deb
dpkg-deb -I target/debs/<BLDENV>/libsaibcm_<version>_amd64.deb   # 看 metadata 确认是客户版本
```

> 小技巧：客户二次开发版本建议在套件 changelog / maintainer 字段加上可辨识字符串，方便事后确认「这颗 image 用的是哪个客户的 SAI」。

---

## 8. 完整 build 操作流程

以下是从零到产出 Broadcom image 的正式流程（集成 `README.md` 并补上 Broadcom 替换步骤）。

### 8.1 环境准备（一次性）

硬件建议：多内核、RAM ≥ 8 GiB（建议更多）、磁盘 ≥ 300G、支持 KVM。OS 建议 Ubuntu 22.04 / 24.04。

```shell
# 安装 pip 与 jinjanator
sudo apt install -y python3-pip
pip3 install --user jinjanator

# 安装 Docker，并让目前用户免 sudo
sudo gpasswd -a ${USER} docker
# 注销再登录让群组生效
```

### 8.2 取得原代码

```shell
git clone --recurse-submodules https://github.com/sonic-net/sonic-buildimage.git
cd sonic-buildimage
# 若需特定分支：git checkout <branch_name>
```

### 8.3 配置 Broadcom SAI 来源（本文重点）

依第 6 章择一做法。以做法 B 为例，先备妥本地 `.deb`，再通过变量指定（见 8.5）。

### 8.4 初始化与配置平台

```shell
sudo modprobe overlay     # 确保 overlay 模块加载

make init                 # clone 后或更新 submodule 后运行一次
make configure PLATFORM=broadcom
```

### 8.5 建置 image

```shell
# 官方在线 SAI
make SONIC_BUILD_JOBS=4 PLATFORM=broadcom target/sonic-broadcom.bin

# 或：使用客户本地 SAI（做法 B）
make SONIC_BUILD_JOBS=4 \
     BRCM_XGS_SAI_LOCAL_PATH=/data/sai/customerA \
     PLATFORM=broadcom target/sonic-broadcom.bin
```

内存对照（来自 `README.md`，供排 JOBS 参考）：

| JOBS | 约需 RAM |
|------|----------|
| 1 | ~10 GB |
| 4 | ~28 GB |
| 8 | ~52 GB |

加速/省内存选项：

```shell
# 跳过测试加速迭代
make SONIC_BUILD_JOBS=4 BUILD_SKIP_TEST=y PLATFORM=broadcom target/sonic-broadcom.bin

# 限制 build 容器内存（写进 rules/config.user）
# SONIC_BUILD_MEMORY = 24g
```

### 8.6 产物

- 最终安装档：`target/sonic-broadcom.bin`（ONIE installer）。
- 中间 deb：`target/debs/<BLDENV>/`，可在此确认 SAI 版本是否为预期。

---

## 9. 常见问题排查

| 症状 | 可能原因 | 处置 |
|------|----------|------|
| `curl: ... 404` 抓 `libsaibcm` 失败 | `LIBSAIBCM_*_VERSION` 与来源实际文档名不符 | 对齐版本字符串；或改走本地拷贝（做法 A/B） |
| 改成 `SONIC_COPY_DEBS` 后说找不到档 | `_PATH` 路径错、或文档不在 build 容器可见范围 | 确认路径相对 repo 根目录、文档确实存在 |
| 版本没变但 image 还是旧 SAI | DPKG cache 命中旧套件 | 清掉该 deb target 后重 build，或确认 `SKIP_VERSION` 配置 |
| 环境变量没带进 build 容器 | 变量未 `export` | 在 `.mk` 加 `export`，或写进 `rules/config.user` |
| DNX/legacy 平台改了 xgs 却没效果 | 改错文档 | DNX 改 `sai-dnx.mk`、legacy 改 `sai-legacy-th.mk` |
| 不确定最终用的是哪颗 SAI | 无法辨识来源 | `dpkg-deb -I` 看 metadata；建议替换版加辨识字符串 |

清除特定套件重建（示意）：

```shell
# 以 target group 的 clean 目标清掉，再重新 build
make target/debs/<BLDENV>/libsaibcm_<version>_amd64.deb-clean
make SONIC_BUILD_JOBS=4 PLATFORM=broadcom target/sonic-broadcom.bin
```

---

## 10. 新进人员学习路径

建议依序完成，每一步都有可验收的产出：

1. **读懂架构（约 0.5 天）**
   - 读本文第 2、3 章。
   - 验收：能画出「swss → syncd → SAI → ASIC」的数据流，并说明 SAI 的作用。

2. **跑通官方 build（约 1 天，含编译等待）**
   - 依第 8 章用官方在线 SAI build 出 `target/sonic-broadcom.bin`。
   - 验收：成功产出 `.bin`，并在 `target/debs/` 找到 `libsaibcm`。

3. **做一次本地 SAI 替换（约 0.5 天）**
   - 用做法 A 把 SAI 改成本地拷贝（可先用官方抓下来的同一颗 deb 当作练习素材）。
   - 验收：build 日志显示 copy 而非 curl，image 内 SAI 版本正确。

4. **理解多客户流程（约 0.5 天）**
   - 用做法 B 的环境变量切换，仿真两个客户来源。
   - 验收：同一份代码，靠不同变量 build 出不同 SAI 的 image。

5. **熟悉排查（持续）**
   - 刻意制造版本不符、路径错误，照第 9 章排除。

> 完成 1～4 即具备独立处理「Broadcom image build + 客户 SAI 替换」的能力。

---

## 附录 A：关键文档速查

| 路径 | 作用 |
|------|------|
| `platform/broadcom/rules.mk` | Broadcom 平台 recipe 入口，include 各 SAI/docker recipe |
| `platform/broadcom/sai-xgs.mk` | XGS SAI（`libsaibcm`）定义 — **替换主要改这里** |
| `platform/broadcom/sai-dnx.mk` | DNX SAI 定义 |
| `platform/broadcom/sai-legacy-th.mk` | Legacy-TH SAI 定义 |
| `platform/broadcom/docker-syncd-brcm.mk` | 将 SAI 注入 `syncd` 容器 |
| `platform/nephos/sai.mk` | 官方「本地/在线切换」写法范本（做法 B 参考） |
| `slave.mk` | build 后端，定义 `SONIC_ONLINE_DEBS` / `SONIC_COPY_DEBS` 等规则 |
| `rules/config` | build 全域配置（含 `BUILD_PUBLIC_URL` 默认值） |
| `rules/config.user` | 个人/客户持久化配置（gitignore，不进版控） |
| `README.buildsystem.md` | build 系统权威说明 |

## 附录 B：合规备忘

- Broadcom SAI/SDK debian 受 NDA 保护，禁止提交进公开 repo、公开 CI log 或外部可访问位置。
- 客户二次开发的 `.deb` 视同机密资产，仅于公司核可的受控管道传递与存放。
- 内部 mirror / artifact server 的帐密不得写进版控文档；使用 CI secret 或环境变量管理。

# SONiC 功能開發教育手冊

> 對象：新進研發人員  
> 目的：以實際範例 `lbd-ec202012_anos`（LBD，Loop Back Detection）教你在 SONiC 上「從零開發一個功能」：**怎麼切模組、檔案放哪裡、怎麼跟系統溝通、debian 怎麼打包、`.mk` 怎麼改、怎麼進 docker image**。  
> 範例位置：`/home/manager/sonic-buildimage/lbd-ec202012_anos/`  
> 官方對照範例：`src/sonic-stp/` + `rules/sonic-stp.mk` + `rules/docker-stp.mk`（STP 與 LBD 同為 L2 daemon，結構幾乎一對一對應，開發時可直接參考）。

---

## 目錄

1. [先建立正確的心智模型](#1-先建立正確的心智模型)
2. [範例功能 LBD 的架構拆解](#2-範例功能-lbd-的架構拆解)
3. [一個功能該怎麼切模組](#3-一個功能該怎麼切模組)
4. [檔案應該放在哪個目錄](#4-檔案應該放在哪個目錄)
5. [功能如何與 SONiC 溝通（DB 存取）](#5-功能如何與-sonic-溝通db-存取)
6. [debian 打包](#6-debian-打包)
7. [相關 mk 檔案怎麼修改](#7-相關-mk-檔案怎麼修改)
8. [如何進入 docker image](#8-如何進入-docker-image)
9. [完整開發流程（逐步）](#9-完整開發流程逐步)
10. [開發注意事項與檢查清單](#10-開發注意事項與檢查清單)
11. [新進人員練習任務](#11-新進人員練習任務)

---

## 1. 先建立正確的心智模型

在 SONiC 上開發功能，和在傳統 monolithic 交換器 OS（如早期的 ANOS/EC code）上開發，最大差異是：

| 項目 | 傳統交換器 OS | SONiC |
|------|---------------|-------|
| 程式形態 | 同一個大進程內的模組 | **獨立 daemon，跑在 Docker 容器裡** |
| 模組間溝通 | 直接函式呼叫 / 共享記憶體 | **透過 Redis 中央資料庫（DB）傳遞狀態** |
| 下硬體 | 直接呼叫 SDK | 寫 ASIC_DB → `syncd` → SAI → ASIC |
| 設定來源 | 私有 config | CONFIG_DB（由 config_db.json / CLI 填入） |
| 打包交付 | 整包 firmware | **一顆 `.deb`，裝進某個 docker image** |

所以在 SONiC 開發一個功能，本質上就是：**寫一個（或一組）daemon，透過 swss-common 讀寫 Redis DB，打包成 deb，裝進一個 docker 容器，再由 build 系統組進 image。**

LBD 正是這個模式的教科書級範例——它原本是傳統 OS 的 L2 功能（`liblbd`），被改寫成 SONiC 形態。

---

## 2. 範例功能 LBD 的架構拆解

LBD（Loop Back Detection）偵測埠上的迴圈，偵測到就把埠 err-disable（shutdown）。它在 SONiC 上的組成：

```
lbd-ec202012_anos/
├── configure.ac          # autotools 專案設定（產生 configure）
├── autogen.sh            # 產生 build 系統並注入版本
├── getversion.sh         # 把 version.txt 的版本注入編譯旗標
├── Makefile.am           # 頂層：SUBDIRS = lib lbdd lbdctrl
├── version.txt
├── lbdd/                 # ★ daemon 主程式（C）
│   ├── main.c            #   進程進入點
│   ├── lbd_mgr.c         #   核心邏輯（狀態機/封包處理調度）
│   ├── lbd_om.c          #   object manager（資料物件）
│   ├── lbd_pkt.c/.h      #   LBD 封包收送
│   ├── lbd_backdoor.c/.h #   debug backdoor
│   └── include/
├── lbdctrl/              # ★ CLI 工具（C）
│   ├── lbd_cli.c         #   CLI 解析
│   ├── lbd_ctl.c         #   控制指令
│   ├── lbd_pmgr.c        #   與 daemon 溝通的 proxy
│   └── include/
├── lib/
│   ├── lbdsync/          # ★ C++ 橋接層：連接 SONiC Redis DB
│   │   ├── lbd_sync.cpp  #   用 swss-common 讀寫 DB，extern "C" 包給 C 呼叫
│   │   ├── lbd_sync.h
│   │   └── lbd_dbsync.h
│   └── stublib/          # 相容舊 OS 的 stub（移植用的墊片）
└── debian/               # ★ deb 打包描述
    ├── control           #   套件定義/相依
    ├── rules             #   debhelper build 腳本
    ├── changelog
    ├── compat
    ├── lbd.install       #   哪些產物裝到 deb 的哪個路徑
    └── lbd.links         #   產生 .so symlink
```

三個「星號」區塊就是 SONiC 功能的典型三件套：

1. **daemon（lbdd）** — 跑功能邏輯。
2. **橋接層（lbdsync）** — 功能邏輯 ↔ SONiC DB 的翻譯層。
3. **CLI（lbdctrl）** — 操作/查詢介面。

> 移植洞見：`lib/stublib/` 與 `-I/usr/include/sysinclude/...`、`-lcmnlib` 這些，是 LBD 從舊 OS 搬過來時保留的相容層。**新功能若從頭開發，不需要 stublib / cmnlib**，直接用 swss-common 即可。手冊第 10 章會特別提醒。

---

## 3. 一個功能該怎麼切模組

參考 LBD，建議把功能切成以下幾塊（視複雜度可合併）：

| 模組 | 語言 | 職責 | LBD 對應 |
|------|------|------|----------|
| daemon 主體 | C/C++ | 功能邏輯、狀態機、計時、封包收送 | `lbdd/` |
| DB 橋接層 | C++ | 用 swss-common 讀 CONFIG_DB、寫 APPL_DB/STATE_DB | `lib/lbdsync/` |
| CLI | C/Python | 查狀態、下指令 | `lbdctrl/` |
| 打包 | debian/ | 編譯產物打成 deb | `debian/` |

切模組的原則：

- **邏輯與 DB 存取分離**。LBD 把所有 swss-common 的呼叫集中在 `lbd_sync.cpp`，daemon 本身（C）只呼叫 `lbdsync_*()` 這組 `extern "C"` 介面。好處是 daemon 不必混 C/C++，DB schema 變動只改橋接層。
- **一個 daemon 專注一件事**。SONiC 偏好小而專的 daemon（像 `orchagent`、`teamd`、`stpd` 各管一塊）。

---

## 4. 檔案應該放在哪個目錄

這是新人最常問的問題。對照 SONiC 慣例與 `sonic-stp` 的做法：

### 4.1 原始碼放哪裡

| 情況 | 放置位置 | 說明 |
|------|----------|------|
| 自研、平台無關的功能原始碼 | `src/<你的功能>/` | 與 `src/sonic-stp`、`src/sonic-swss` 同級。**LBD 若要正式進 SONiC，應放這裡**，例如 `src/sonic-lbd/` |
| 由外部 git 管理的元件 | 用 git submodule 掛在 `src/` 下 | 像 `src/sonic-swss` 就是 submodule |
| 廠商/平台專屬 | `platform/<vendor>/` | 只有特定 ASIC 平台用到才放這 |

> 範例目前 `lbd-ec202012_anos/` 放在 repo 根目錄，那是「開發暫存」的位置。**正式整合時請搬進 `src/`**（例如 `src/sonic-lbd/`），命名用小寫、以 `-` 分隔，與既有慣例一致。

### 4.2 recipe（`.mk`）放哪裡

| 檔案 | 位置 | 作用 |
|------|------|------|
| package recipe | `rules/<功能>.mk` | 定義怎麼把 `src/<功能>/` build 成 deb。對照 `rules/sonic-stp.mk` |
| docker recipe | `rules/docker-<功能>.mk` | 若功能要自己的容器，定義 docker。對照 `rules/docker-stp.mk` |

### 4.3 容器內執行期檔案放哪裡

| 內容 | 容器內路徑 | 來源（deb install 規則） |
|------|-----------|--------------------------|
| daemon 執行檔 | `/usr/bin/lbdd` | `debian/lbd.install` |
| CLI 執行檔 | `/usr/bin/lbdctrl`（或透過 `BASE_IMAGE_FILES` 暴露到 host） | `debian/lbd.install` / docker recipe |
| 共用庫 | `/usr/lib/liblbdsync.so*` | `debian/lbd.install` + `debian/lbd.links` |

---

## 5. 功能如何與 SONiC 溝通（DB 存取）

這是 SONiC 開發的核心技能。LBD 的 `lib/lbdsync/lbd_sync.cpp` 是最好的教材。

### 5.1 用到的 DB 與表

SONiC 用多個 Redis DB 分工：

| DB | 用途 | LBD 的使用 |
|----|------|------------|
| `CONFIG_DB` | 使用者設定 | 讀 `VLAN_MEMBER`、`PORT`、`PORTCHANNEL` 設定 |
| `APPL_DB` | 應用層狀態（下行給 orchagent） | 寫 `PORT_TABLE`（err-disable、ingress filter）、STP 相關表 |
| `STATE_DB` | 系統運行狀態 | 連線備用 |

### 5.2 swss-common 的用法（節錄自 `lbd_sync.cpp`）

```cpp
#include "dbconnector.h"
#include "producerstatetable.h"
using namespace swss;

// 1) 連接 DB（以 DB 名稱連，不用記 port/index）
DBConnector db("APPL_DB", 0);
DBConnector cfgDb("CONFIG_DB", 0);
DBConnector stateDb("STATE_DB", 0);

// 2) 綁定要操作的表
Table m_appPortTable(db, APP_PORT_TABLE_NAME);
Table m_cfgVlanMemberTable(cfgDb, CFG_VLAN_MEMBER_TABLE_NAME);

// 3) 寫入：把 err-disable 狀態寫進 APPL_DB 的 PORT_TABLE
std::vector<FieldValueTuple> fv;
fv.emplace_back("err_disabled_reason", enabled ? "lbd" : "none");
m_appPortTable.set(if_name, fv);

// 4) 讀取：查某埠是不是某 VLAN 的 member
std::vector<FieldValueTuple> fvs;
bool ok = m_cfgVlanMemberTable.get(key, fvs);
```

### 5.3 C daemon 怎麼呼叫 C++ 橋接層

daemon 是 C，橋接層是 C++。LBD 用 `extern "C"` 把 C++ 函式包成 C 可呼叫的介面：

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

daemon（C）只要 include 對應標頭，直接呼叫 `lbdsync_set_ingress_filtering(...)` 即可，完全不碰 C++。

### 5.4 日誌

統一用 swss-common 的 logger，輸出會進 syslog，方便 `show logging` 查：

```cpp
#include "logger.h"
SWSS_LOG_NOTICE("LBD update appDB for %s port", if_name);
```

---

## 6. debian 打包

功能要進 SONiC，最終得是一顆 deb。LBD 的 `debian/` 展示了 autotools 專案的標準打包。

### 6.1 `debian/control` — 套件定義

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

重點：

- **一個 source 可產出多個 binary package**（這裡 `lbd` + `lbd-dbg`）。SONiC 慣例會額外產一個 `-dbg` 包放 debug symbols。
- `Build-Depends` 要列出編譯期相依。LBD 在乾淨環境實測需要的最小集合為 `libswsscommon-dev`（橋接層 include `dbconnector.h`/`producerstatetable.h`/`logger.h`）、`libhiredis-dev`（`-lhiredis`）、`libnl-3-dev`/`libnl-genl-3-dev`（`-lnl-3`/`-lnl-genl-3`），以及 `libtool`/`pkg-config`（autogen 用）。範例原始的 `control` 只列了 `dh-exec/debhelper/autotools-dev`，正式整合時已補齊上述相依（見第 10 章注意事項）。

### 6.2 `debian/rules` — build 腳本

```makefile
#!/usr/bin/make -f
%:
	dh $@  --with autotools-dev        # 用 debhelper 跑 autotools 標準流程

override_dh_auto_install:
	dh_auto_install --destdir=debian/lbd

override_dh_clean:
	dh_clean --dbg-package=lbd-dbg
```

`dh $@ --with autotools-dev` 會自動依序跑 `./configure` → `make` → `make install`。

### 6.3 `debian/lbd.install` — 產物安裝位置

左邊是 build 出來的檔案，右邊是裝進 deb 的目錄：

```
lbdd/lbdd                            usr/bin
lbdctrl/lbdctrl                      usr/bin
lib/stublib/.libs/libstublib.so.0.0.0  usr/lib
lib/lbdsync/.libs/liblbdsync.so.0.0.0  usr/lib
```

### 6.4 `debian/lbd.links` — symlink（dh-exec）

產生共用庫慣用的版本 symlink：

```
#! /usr/bin/dh-exec
/usr/lib/liblbdsync.so.0.0.0 /usr/lib/liblbdsync.so
/usr/lib/liblbdsync.so.0.0.0 /usr/lib/liblbdsync.so.0
```

---

## 7. 相關 mk 檔案怎麼修改

這是題目的重點。要讓 build 系統認得你的功能，需新增/修改兩類 recipe。以 LBD 正式化為 `src/sonic-lbd/` 為例，對照官方 `sonic-stp`。

### 7.1 新增 package recipe：`rules/sonic-lbd.mk`

仿 `rules/sonic-stp.mk`：

```makefile
# LBD package

LBD_VERSION = 1.0.0
export LBD_VERSION

LBD = lbd_$(LBD_VERSION)_$(CONFIGURED_ARCH).deb
$(LBD)_SRC_PATH = $(SRC_PATH)/sonic-lbd
# 編譯期相依：用到 swss-common 的開發標頭/庫
$(LBD)_DEPENDS += $(LIBSWSSCOMMON_DEV)
# 執行期相依：容器內要有 runtime 庫
$(LBD)_RDEPENDS += $(LIBSWSSCOMMON)
SONIC_DPKG_DEBS += $(LBD)

# debug symbols 子套件
LBD_DBG = lbd-dbg_$(LBD_VERSION)_$(CONFIGURED_ARCH).deb
$(LBD_DBG)_DEPENDS += $(LBD)
$(LBD_DBG)_RDEPENDS += $(LBD)
$(eval $(call add_derived_package,$(LBD),$(LBD_DBG)))

export LBD
```

逐行說明：

- `SONIC_DPKG_DEBS += $(LBD)`：告訴 build 系統「用 `dpkg-buildpackage` 從 `$(LBD)_SRC_PATH` 建這顆 deb」。
- `_DEPENDS` 是**編譯期**相依（build 這包前要先裝好的其他 deb）；`_RDEPENDS` 是**執行期**相依（裝進 docker 時要一起裝的 deb）。兩者別搞混。
- `add_derived_package` 讓 `-dbg` 共用主套件的 metadata，不用重寫一遍。

### 7.2 新增 docker recipe：`rules/docker-lbd.mk`（若功能需要自己的容器）

仿 `rules/docker-stp.mk`：

```makefile
# Docker image for LBD

DOCKER_LBD_STEM = docker-lbd
DOCKER_LBD = $(DOCKER_LBD_STEM).gz
DOCKER_LBD_DBG = $(DOCKER_LBD_STEM)-$(DBG_IMAGE_MARK).gz

$(DOCKER_LBD)_PATH = $(DOCKERS_PATH)/$(DOCKER_LBD_STEM)

# 要裝進這個容器的 deb
$(DOCKER_LBD)_DEPENDS += $(LBD) $(SWSS)
$(DOCKER_LBD)_DBG_DEPENDS += $(LBD)

# 以哪個 base image 疊上去
$(DOCKER_LBD)_LOAD_DOCKERS = $(DOCKER_CONFIG_ENGINE_TRIXIE)

$(DOCKER_LBD)_VERSION = 1.0.0
$(DOCKER_LBD)_PACKAGE_NAME = lbd
$(DOCKER_LBD)_CONTAINER_NAME = lbd
$(DOCKER_LBD)_RUN_OPT += -t --cap-add=NET_ADMIN --cap-add=SYS_ADMIN
$(DOCKER_LBD)_RUN_OPT += -v /etc/sonic:/etc/sonic:ro

# 把 CLI 暴露到 host（在 host 上打 lbdctl 會轉進容器）
$(DOCKER_LBD)_BASE_IMAGE_FILES += lbdctrl:/usr/bin/lbdctl

# 用開關控制是否納入 image
ifeq ($(INCLUDE_LBD), y)
SONIC_DOCKER_IMAGES += $(DOCKER_LBD)
SONIC_INSTALL_DOCKER_IMAGES += $(DOCKER_LBD)
SONIC_DOCKER_DBG_IMAGES += $(DOCKER_LBD_DBG)
SONIC_INSTALL_DOCKER_DBG_IMAGES += $(DOCKER_LBD_DBG)
endif

SONIC_TRIXIE_DOCKERS += $(DOCKER_LBD)
SONIC_TRIXIE_DBG_DOCKERS += $(DOCKER_LBD_DBG)
```

> 若功能很輕、不需要獨立容器，也可以**不建 docker recipe**，改在 7.3 把 `$(LBD)` 加進既有容器（例如 `docker-stp` 或 `swss`）的 `_DEPENDS`。
>
> **本次 LBD 正式整合的最終決定：採輕量做法，不新建獨立容器，將 `$(LBD)` 掛進 `docker-stp`。** 理由：LBD 與 STP 同為 L2 功能，且 `lbd_sync.cpp` 直接參照 STP 的 APP table（`APP_STP_PORT_TABLE_NAME` 等），資料面高度耦合，共用容器最自然。落地方式見 7.3（在 deb 能成功 build 之後再加）。

### 7.3 把 deb 加進某個既有容器（替代 7.2 的輕量做法）

例如要把 LBD 放進 STP 容器，改 `rules/docker-stp.mk`：

```makefile
$(DOCKER_STP)_DEPENDS += $(STP) $(SWSS) $(LBD)   # ← 加 $(LBD)
```

### 7.4 加入 build 開關（可選但建議）

仿 STP 的 `INCLUDE_STP`，在 `rules/config` 加預設值、在 `slave.mk` 建立對應關係，讓使用者能用 `INCLUDE_LBD=y/n` 選裝。這讓功能可被不同客戶 image 選配。

### 7.5 recipe 之間的關係圖

```
rules/config            → 定義 INCLUDE_LBD 預設
   │
rules/sonic-lbd.mk      → 怎麼把 src/sonic-lbd build 成 lbd_*.deb   (SONIC_DPKG_DEBS)
   │  產出 $(LBD)
   ▼
rules/docker-lbd.mk     → 把 $(LBD) 裝進 docker-lbd 容器            (SONIC_DOCKER_IMAGES)
   │  產出 $(DOCKER_LBD)
   ▼
platform/*/one-image.mk → 把 $(DOCKER_LBD) 組進最終 .bin image
```

---

## 8. 如何進入 docker image

一顆 deb 要真的跑在交換器上，流程是：

1. `rules/sonic-lbd.mk` 把原始碼 build 成 `lbd_*.deb`（進 `target/debs/`）。
2. `rules/docker-lbd.mk` 的 `_DEPENDS += $(LBD)` 讓這顆 deb 在建 docker image 時被 `dpkg -i` 裝進容器。
3. 容器 image 被 `one-image.mk` 收進最終 `.bin`。
4. 啟動時由 `supervisord`（容器內）拉起 `/usr/bin/lbdd`。

`BASE_IMAGE_FILES` 的特殊作用：把容器內的工具「投影」到 host OS，使用者在 host shell 直接打 CLI（例如 `stpctl` / 範例中的 `lbdctl`）就會被導進容器執行，不必手動 `docker exec`。

---

## 9. 完整開發流程（逐步）

以「開發 LBD 並正式整合進 SONiC」為例：

1. **放原始碼**：把程式放進 `src/sonic-lbd/`（小寫、`-` 分隔）。
2. **寫 DB 橋接層**：集中所有 swss-common 存取在一個 `.cpp`，用 `extern "C"` 對 daemon 開介面（參考 `lbd_sync.cpp`）。
3. **寫打包描述**：補 `debian/control`（記得 `Build-Depends` 加 `libswsscommon-dev`）、`rules`、`*.install`、`*.links`、`changelog`、`compat`。
4. **寫 package recipe**：新增 `rules/sonic-lbd.mk`（第 7.1 節）。
5. **決定容器歸屬**：建 `rules/docker-lbd.mk`（7.2）或掛進既有容器（7.3）。
6. **（可選）加開關**：`INCLUDE_LBD`（7.4）。
7. **設定平台**：

   ```shell
   cd /home/manager/sonic-buildimage
   make init
   make configure PLATFORM=broadcom
   ```

8. **先單獨 build 這顆 deb 驗證**（比 build 整包 image 快很多）：

   ```shell
   make target/debs/trixie/lbd_1.0.0_amd64.deb
   # 本 repo 實測 BLDENV = trixie（頂層 Makefile 的 % 目標預設以 BLDENV=trixie 呼叫
   # Makefile.work，且 sonic-slave-trixie 容器已存在）。其他分支可能是 bookworm 等，
   # 依 Makefile 內各目標設定而定。首次執行會連帶把相依（含 libswsscommon）一併 build。
   ```

9. **驗證 deb 內容**：

   ```shell
   dpkg-deb -c target/debs/<BLDENV>/lbd_1.0.0_amd64.deb   # 看裝了哪些檔、路徑對不對
   dpkg-deb -I target/debs/<BLDENV>/lbd_1.0.0_amd64.deb   # 看相依與 metadata
   ```

10. **build 容器 / 整包 image**：

    ```shell
    make SONIC_BUILD_JOBS=4 INCLUDE_LBD=y PLATFORM=broadcom target/sonic-broadcom.bin
    ```

11. **上機驗證**：裝 image 後 `docker ps` 看容器起來、`show logging` 看 `SWSS_LOG` 輸出、用 `redis-cli` 確認 DB 有被正確寫入。

---

## 10. 開發注意事項與檢查清單

### 10.1 架構類

- **不要在 daemon 裡直接碰硬體 SDK**。要下 ASIC 一律透過 DB → orchagent/syncd → SAI。LBD 的做法（寫 APPL_DB 的 `PORT_TABLE`）就是正解。
- **DB 存取集中管理**。所有 swss-common 呼叫放一個橋接層，別散落各處。
- **用既有 DB 表/欄位**。寫入前先確認 schema（參考 `sonic-swss-common` 的 `schema.h` 與 YANG model），不要自創會與既有 orchagent 衝突的欄位。
- **一個 daemon 一件事**，別把多種功能塞進同一進程。

### 10.2 移植類（LBD 這種從舊 OS 搬過來的情況特別注意）

- 範例裡的 `lib/stublib/`、`-lcmnlib`、`-I/usr/include/sysinclude/...` 都是**舊 OS 相容墊片**。**全新功能不要沿用**，直接用 swss-common。移植既有 code 時，目標應是逐步把 stub 換成原生 SONiC API。
- **已知阻礙（實測）**：把範例原封不動搬進 `src/sonic-lbd/` 並用 `make target/debs/trixie/lbd_1.0.0_amd64.deb` build 時，相依（含 `libswsscommon`）都能成功，但 lbd 本身會在**第一個編譯單元 `lib/stublib/stub.c` 就 fatal error**：

  ```
  include/stub.h:4:10: fatal error: sys_type.h: No such file or directory
  ```

  根因是 `stublib` 的標頭（`sys_type.h`、`port.h`）與 `-lcmnlib` 都來自舊 OS 的 `cmnlib` 樹，SONiC sonic-slave 容器沒有它。`-lcmnlib` 的 link 失敗是同一根因的下一階段（compile 比 link 早發生，所以先卡在缺標頭）。要讓它編得過，必須先處理 cmnlib 墊片（擇一）：(1) 把所需的最小 cmnlib 標頭/庫子集自帶進 `src/sonic-lbd/` 並改 `Makefile.am` 的 include 路徑；(2) 用精簡的本地相容標頭取代 `sys_type.h`/`port.h`，並以本地實作取代 `-lcmnlib` 被參照的符號。這兩者都屬**移植工作**，不在「讓它進 build 系統」的骨架整合範圍內，且須確認無 NDA/機密才能放進 repo。
- `getversion.sh` 用 `sed` 動態改 `Makefile.am` 注入版本是舊習慣，脆弱且難維護；新專案建議用 `configure.ac` 的 `AC_INIT` 版本或編譯旗標。

### 10.3 打包/相依類

- `control` 的 `Build-Depends` 要**誠實列出編譯相依**（用到 swss-common 就加 `libswsscommon-dev`），否則在乾淨環境 build 會失敗。
- recipe 的 `_DEPENDS`（編譯期）vs `_RDEPENDS`（執行期）不要混。
- 一定要產 `-dbg` 套件，現場除錯會需要 symbols。

### 10.4 程式碼規範類（本專案 C 開發守則）

本範例是 C/C++ 專案，提交前請對齊團隊 C 編碼規範（見全域 coding standard），重點提醒：

- 縮排 4 空格、不用 tab；LF 換行；行長 < 120。
- 所有 `.h` 要有 `#ifndef/#define/#endif` include guard。
- 全域變數禁用，改用 `static` 檔案層級變數 + getter/setter。
- `if/while/switch` 一律加大括號，Allman 風格（`{` 獨立一行）。
- `switch` 必含 `default`；刻意 fall-through 要註解。
- 檢查所有函式回傳值（尤其 `malloc`）；`free` 後把指標設 NULL。
- 用 `strncpy` 並以**目的緩衝區大小**為界；用 `strtok_r` 取代 `strtok`。
- 清掉所有編譯警告才能 check-in。

> 注意：範例的 `lbd_sync.cpp` 等檔案是既有移植 code，其風格不一定完全符合上述規範；請以規範為準，不要照抄舊風格。

### 10.5 提交前檢查清單

- [ ] 原始碼放在 `src/sonic-<功能>/`，命名小寫 `-` 分隔
- [ ] DB 存取集中在單一橋接層，`extern "C"` 介面清楚
- [ ] `debian/control` 相依完整（含 `libswsscommon-dev`）
- [ ] `rules/sonic-<功能>.mk` 的 `SRC_PATH`、`_DEPENDS`、`_RDEPENDS` 正確
- [ ] 決定容器歸屬（獨立 docker 或掛既有容器）並改好 docker recipe
- [ ] 單獨 build deb 成功、`dpkg-deb -c/-I` 檢查無誤
- [ ] 整包 image build 成功、上機容器能起、DB 寫入正確、syslog 有預期輸出
- [ ] 無編譯警告、符合 C 編碼規範
- [ ] 不含 NDA / 機密內容；舊 OS 墊片已評估是否移除

---

## 11. 新進人員練習任務

依序完成，每步都有可驗收產出：

1. **讀懂範例（0.5 天）**：讀 `lbd_sync.cpp` 與本文第 2、5 章。驗收：能口述 LBD 偵測到迴圈後，狀態如何經 APPL_DB → orchagent 讓埠 err-disable。
2. **對照官方範例（0.5 天）**：把 `rules/sonic-stp.mk` / `rules/docker-stp.mk` 與本文第 7 章逐行對照。驗收：能指出 STP recipe 中每一行的作用。
3. **把 LBD 正式化（1 天）**：將 `lbd-ec202012_anos/` 搬進 `src/sonic-lbd/`，補 `rules/sonic-lbd.mk`。驗收：`make target/debs/.../lbd_*.deb` 成功，`dpkg-deb -c` 看到 `lbdd`/`lbdctrl` 在 `usr/bin`。
4. **進容器（0.5 天）**：把 `$(LBD)` 掛進一個容器並 build。驗收：image 內容器含 `lbdd`。
5. **動手改一個小功能（1 天）**：例如在橋接層多寫一個 STATE_DB 欄位記錄最後偵測時間。驗收：`redis-cli` 查得到新欄位。

> 完成 1～4 即具備「在 SONiC 上新增一個功能 daemon 並整合進 image」的獨立能力。

---

## 附錄：關鍵檔案速查

| 路徑 | 作用 |
|------|------|
| `lbd-ec202012_anos/lbdd/` | 範例：daemon 主程式 |
| `lbd-ec202012_anos/lib/lbdsync/lbd_sync.cpp` | 範例：swss-common DB 橋接層（必讀） |
| `lbd-ec202012_anos/debian/` | 範例：deb 打包描述 |
| `src/sonic-stp/` | 官方對照：L2 daemon 原始碼 |
| `rules/sonic-stp.mk` | 官方對照：package recipe 範本 |
| `rules/docker-stp.mk` | 官方對照：docker recipe 範本 |
| `rules/swss.mk` | 官方對照：DPKG recipe + `_DEPENDS/_RDEPENDS` 用法 |
| `README.buildsystem.md` | build 系統（target group）權威說明 |
| `src/sonic-swss-common/` | swss-common 原始碼（DBConnector/Table/ProducerStateTable API） |

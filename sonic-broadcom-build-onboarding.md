# SONiC Broadcom Image Build 教學文件

> 對象：新進研發人員  
> 目的：提供一套正式、可依循的 SONiC（Broadcom 平台）build 教學，取代「每個人自己去啃 README」的學習方式。  
> 範圍：本文聚焦於 **Broadcom 平台**的 image build，特別是 **Broadcom 專屬 debian 套件（受 NDA 保護的 SAI/SDK）的取得與替換流程**，以及 **SONiC 整體架構**的教育說明。

---

## 目錄

1. [文件使用方式](#1-文件使用方式)
2. [SONiC 架構總覽](#2-sonic-架構總覽)
3. [sonic-buildimage build 系統架構](#3-sonic-buildimage-build-系統架構)
4. [Broadcom 平台的套件組成](#4-broadcom-平台的套件組成)
5. [Broadcom debian 的取得（NDA 議題）](#5-broadcom-debian-的取得nda-議題)
6. [替換 Broadcom debian 來 build 不同 image](#6-替換-broadcom-debian-來-build-不同-image)
7. [客戶二次開發 debian 的整合流程](#7-客戶二次開發-debian-的整合流程)
8. [完整 build 操作流程](#8-完整-build-操作流程)
9. [常見問題排查](#9-常見問題排查)
10. [新進人員學習路徑](#10-新進人員學習路徑)

---

## 1. 文件使用方式

本文與專案內既有文件互補，並非取代。既有文件的角色：

| 文件 | 角色 |
|------|------|
| `README.md` | 原始官方指引，硬體需求、環境準備、build 指令 |
| `README.buildsystem.md` | build 系統（make backend/frontend、target group）的權威說明 |
| `BUILD_CONFIG.md` | build 可調整的設定旗標 |
| **本文 `docs/sonic-broadcom-build-onboarding.md`** | **新進人員正式流程 + Broadcom debian 取得/替換的 how-to** |

閱讀建議：先讀第 2、3 章建立心智模型，再依任務跳到第 5～8 章實作。

---

## 2. SONiC 架構總覽

SONiC（Software for Open Networking in the Cloud）是一套以 **Linux（Debian）+ Docker 容器**為基礎的交換器網路作業系統。核心設計理念是：**將交換器功能拆成多個獨立的 Docker 容器，透過中央資料庫（Redis）交換狀態**。

### 2.1 執行期架構（交換器上運作的樣子）

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
│                 透過 Redis(APPL/ASIC_DB) 溝通                  │
│                                   │                           │
│                          ┌────────▼─────────┐                 │
│                          │   SAI (廠商庫)    │  ← Broadcom     │
│                          │   libsaibcm      │     專屬 debian  │
│                          └────────┬─────────┘                 │
└───────────────────────────────────┼───────────────────────────┘
                                     │ (核心模組 linux_bcm_knet 等)
                              ┌──────▼───────┐
                              │  ASIC 晶片    │
                              │ (Broadcom)   │
                              └──────────────┘
```

### 2.2 關鍵容器與角色

| 容器 | 角色 |
|------|------|
| `database` | Redis，所有狀態的中央儲存（CONFIG_DB / APPL_DB / ASIC_DB / STATE_DB …） |
| `swss` | Switch State Service，各種 orchagent（路由、ACL、port 等）將設定寫入 ASIC_DB |
| `syncd` | 讀取 ASIC_DB，透過 **SAI** 下發到 ASIC。**這是 Broadcom SAI debian 被安裝的容器** |
| `bgp` | FRR routing stack |
| `teamd` / `lldp` / `snmp` / `pmon` 等 | 各功能性服務 |

### 2.3 SAI 的關鍵地位

**SAI（Switch Abstraction Interface）** 是 SONiC 能夠支援多廠商 ASIC 的關鍵。它是一層標準化 API：

- 上層（syncd）只呼叫 SAI 標準 API，不管底下是哪家晶片。
- 下層由各 ASIC 廠商提供 SAI 實作（`.deb` 套件）對接自家 SDK/晶片。

對 Broadcom 而言，SAI 實作就是 **`libsaibcm`**（以及開發用的 `libsaibcm-dev`）。這個套件由 Broadcom 提供、受 NDA 保護，正是本文第 5～7 章的主角。

---

## 3. sonic-buildimage build 系統架構

（權威細節見 `README.buildsystem.md`，此處為新進人員導讀）

### 3.1 兩層結構

- **Backend（後端）**：`Makefile`、`slave.mk`、`sonic-slave-*/Dockerfile`。
  - 所有 build 都在一個叫 `sonic-slave` 的 Docker 容器內執行，環境一致。
  - `slave.mk` 定義了各種「target group」的通用建置規則。
- **Frontend（前端）**：`rules/`（平台無關 recipe）與 `platform/[VENDOR]/`（廠商專屬 recipe）。
  - 每個 recipe 是一個小的 `.mk`，描述「要 build 什麼、相依什麼、怎麼取得」。

### 3.2 build 產物流向

```
src/ + platform/      target/debs/        最終 image
(原始碼/recipe)  ──►  (.deb 套件)   ──►   target/sonic-broadcom.bin
                                          │
                                          └─ 安裝進各 docker → 打包成 ONIE installer
```

### 3.3 必懂的 target group（取得套件的四種方式）

這是理解「Broadcom debian 怎麼進來」的關鍵。來源見 `README.buildsystem.md`，內容為求教學已改寫：

| Target group | 意義 | 使用時機 |
|--------------|------|----------|
| `SONIC_DPKG_DEBS` | 從原始碼用 `dpkg-buildpackage` 建置 | 自家開源套件 |
| `SONIC_MAKE_DEBS` | 用自訂 Makefile 建置（可套 patch） | 需要特殊建置流程 |
| `SONIC_ONLINE_DEBS` | 從網路 URL 下載現成 `.deb` | **預設的 Broadcom SAI 取得方式** |
| `SONIC_COPY_DEBS` | 從本機指定路徑複製現成 `.deb` | **替換成本地 Broadcom/客製 debian** |

> 關鍵認知：Broadcom SAI 預設走 `SONIC_ONLINE_DEBS`（線上下載）。要替換成你自己拿到或二次開發的 debian，核心手法就是改走 `SONIC_COPY_DEBS`（本地複製）。

build 系統對這兩者的實際處理（摘自 `slave.mk`）：

- `SONIC_ONLINE_DEBS`：對每個套件執行 `curl -L -f -o <目標> <URL>` 下載，並寫入 DPKG cache。
- `SONIC_COPY_DEBS`：直接從 `$(套件)_PATH` 指定的本機路徑複製到 `target/debs/`。

---

## 4. Broadcom 平台的套件組成

Broadcom 平台 recipe 都在 `platform/broadcom/`。`platform/broadcom/rules.mk` 是入口，會 `include` 以下與 SAI 相關的檔案：

| 檔案 | 內容 | 對應 ASIC 家族 |
|------|------|----------------|
| `sai-xgs.mk` | `libsaibcm`（XGS：Tomahawk/Trident 等） | XGS 系列 |
| `sai-dnx.mk` | `libsaibcm_dnx`（DNX：Jericho/Qumran 等） | DNX 系列 |
| `sai-legacy-th.mk` | 舊版 Tomahawk 相容 SAI | Legacy TH |
| `sai-modules.mk` | SAI 核心驅動模組（`saibcm-modules`） | 通用 |

以 `sai-xgs.mk` 為例，定義 Broadcom SAI 套件的方式如下（節錄實際內容）：

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

# 註冊成線上下載套件
SONIC_ONLINE_DEBS += $(BRCM_XGS_SAI)
```

重點拆解：

- `BUILD_PUBLIC_URL` 預設是官方公開 mirror（`rules/config` 內定義，預設 `https://packages.trafficmanager.net/public`）。
- `libsaibcm`（runtime）是真正裝進 `syncd` 容器的 SAI 庫。
- `libsaibcm-dev` 是 build `syncd` 時需要的開發標頭/庫，用 `add_derived_package` 由 runtime 衍生（共用 metadata）。
- SAI 如何注入 syncd 容器，見 `platform/broadcom/docker-syncd-brcm.mk`：

```makefile
$(DOCKER_SYNCD_BASE)_DEPENDS += $(SYNCD)
$(DOCKER_SYNCD_BASE)_DEPENDS += $(BRCM_XGS_SAI)   # ← SAI 被加為 syncd 容器相依
```

以及 `platform/broadcom/rules.mk`：

```makefile
# 把 brcm sai 注入 syncd build
$(SYNCD)_DEPENDS += $(BRCM_XGS_SAI) $(BRCM_XGS_SAI_DEV)
```

---

## 5. Broadcom debian 的取得（NDA 議題）

### 5.1 為什麼需要特別處理

Broadcom 的 SAI/SDK 二進位（`libsaibcm*.deb`）屬於**受 NDA 保護的廠商資產**：

- 公開 mirror 上的版本可能受限、或非你客戶需要的特定版本。
- 客戶若有自家 SDK patch、或 Broadcom 直接交付的特殊版本，**無法放到公開來源**。
- 因此實務上常見需求是：**用本機/內部來源的 `.deb` 取代預設的線上下載**。

> 合規提醒：Broadcom SAI/SDK debian 的散佈受 NDA 約束。請勿將這些 `.deb`、或其 URL/帳密，提交進公開 repository、公開 CI log 或外部可存取的位置。內部傳遞請走公司核可的受控管道。

### 5.2 合法取得管道（擇一）

1. **Broadcom 官方交付**：簽署 NDA 後，由 Broadcom 透過其客戶入口 / 業務窗口提供對應 SAI 版本。
2. **公司內部 artifact server**：公司在內部 mirror（如 Artifactory / 內部 HTTP server）存放已授權的 `.deb`。
3. **客戶提供**：客戶基於其與 Broadcom 的 NDA，提供自家二次開發後的 `.deb`。

### 5.3 確認你手上的套件版本

命名規則為 `libsaibcm_<版本>_amd64.deb`。build 要成功，`.mk` 內的 `LIBSAIBCM_*_VERSION` 必須與實際檔名版本一致，否則 `curl` 或複製會找不到檔案。

---

## 6. 替換 Broadcom debian 來 build 不同 image

本章是本文核心。目標：**在不改動官方預設來源的前提下，用你自己的 Broadcom debian build 出 image**。

有三種做法，依「侵入程度」由低到高排列。**建議新手從做法 A 開始。**

### 做法 A：改走本地複製（`SONIC_COPY_DEBS`）— 推薦

概念：把 SAI 套件從「線上下載」改成「從本機路徑複製」。這正是 `platform/nephos/sai.mk` 既有的官方範例所用的模式（以環境變數切換本地/線上）。

**步驟**

1. 把你取得的 `.deb` 放到專案可存取的本機路徑，例如：

   ```
   platform/broadcom/sai-local/libsaibcm_15.2.0.0.0.0.3.1_amd64.deb
   platform/broadcom/sai-local/libsaibcm-dev_15.2.0.0.0.0.3.1_amd64.deb
   ```

2. 修改 `platform/broadcom/sai-xgs.mk`，將 `_URL` 改為 `_PATH`，並把註冊從 `SONIC_ONLINE_DEBS` 改為 `SONIC_COPY_DEBS`：

   ```makefile
   LIBSAIBCM_XGS_VERSION = 15.2.0.0.0.0.3.1

   # Runtime 套件：改用本地路徑
   BRCM_XGS_SAI = libsaibcm_$(LIBSAIBCM_XGS_VERSION)_amd64.deb
   $(BRCM_XGS_SAI)_PATH = $(PLATFORM_PATH)/sai-local/$(BRCM_XGS_SAI)

   # Dev 套件
   BRCM_XGS_SAI_DEV = libsaibcm-dev_$(LIBSAIBCM_XGS_VERSION)_amd64.deb
   $(eval $(call add_derived_package,$(BRCM_XGS_SAI),$(BRCM_XGS_SAI_DEV)))
   $(BRCM_XGS_SAI_DEV)_PATH = $(PLATFORM_PATH)/sai-local/$(BRCM_XGS_SAI_DEV)

   # 註冊改為本地複製
   SONIC_COPY_DEBS += $(BRCM_XGS_SAI)

   $(BRCM_XGS_SAI_DEV)_DEPENDS += $(BRCM_XGS_SAI)
   $(BRCM_XGS_SAI)_SKIP_VERSION = y
   $(BRCM_XGS_SAI_DEV)_SKIP_VERSION = y
   $(eval $(call add_conflict_package,$(BRCM_XGS_SAI_DEV),$(LIBSAIVS_DEV)))
   ```

   > `$(PLATFORM_PATH)` 在 Broadcom build 時即 `platform/broadcom`。

### 做法 B：用環境變數切換（更乾淨、可多客戶共用）— 建議團隊採用

仿照 `platform/nephos/sai.mk` 的官方寫法，用一個環境變數決定要「本地複製」還是「線上下載」。好處是**不同客戶 build 時不需改碼，只改環境變數**。

在 `platform/broadcom/sai-xgs.mk` 改成：

```makefile
LIBSAIBCM_XGS_VERSION = 15.2.0.0.0.0.3.1
LIBSAIBCM_XGS_BRANCH_NAME = SAI_15.2.0_GA
LIBSAIBCM_XGS_URL_PREFIX = "$(BUILD_PUBLIC_URL)/sai/sai-broadcom/$(LIBSAIBCM_XGS_BRANCH_NAME)/$(LIBSAIBCM_XGS_VERSION)/xgs"

# 在此指定本地 SAI 存放目錄；留空則走線上下載
BRCM_XGS_SAI_LOCAL_PATH ?=
export BRCM_XGS_SAI_LOCAL_PATH

BRCM_XGS_SAI = libsaibcm_$(LIBSAIBCM_XGS_VERSION)_amd64.deb
BRCM_XGS_SAI_DEV = libsaibcm-dev_$(LIBSAIBCM_XGS_VERSION)_amd64.deb
$(eval $(call add_derived_package,$(BRCM_XGS_SAI),$(BRCM_XGS_SAI_DEV)))

ifneq ($(BRCM_XGS_SAI_LOCAL_PATH), )
# ── 本地來源 ──
$(BRCM_XGS_SAI)_PATH     = $(BRCM_XGS_SAI_LOCAL_PATH)/$(BRCM_XGS_SAI)
$(BRCM_XGS_SAI_DEV)_PATH = $(BRCM_XGS_SAI_LOCAL_PATH)/$(BRCM_XGS_SAI_DEV)
SONIC_COPY_DEBS += $(BRCM_XGS_SAI)
else
# ── 線上來源（官方預設）──
$(BRCM_XGS_SAI)_URL     = "$(LIBSAIBCM_XGS_URL_PREFIX)/$(BRCM_XGS_SAI)"
$(BRCM_XGS_SAI_DEV)_URL = "$(LIBSAIBCM_XGS_URL_PREFIX)/$(BRCM_XGS_SAI_DEV)"
SONIC_ONLINE_DEBS += $(BRCM_XGS_SAI)
endif

$(BRCM_XGS_SAI_DEV)_DEPENDS += $(BRCM_XGS_SAI)
$(BRCM_XGS_SAI)_SKIP_VERSION = y
$(BRCM_XGS_SAI_DEV)_SKIP_VERSION = y
$(eval $(call add_conflict_package,$(BRCM_XGS_SAI_DEV),$(LIBSAIVS_DEV)))
```

build 時：

```shell
# 用客戶 A 的本地 SAI
make BRCM_XGS_SAI_LOCAL_PATH=/data/sai/customerA PLATFORM=broadcom target/sonic-broadcom.bin

# 不帶變數 → 走官方線上版本（回到預設行為）
make PLATFORM=broadcom target/sonic-broadcom.bin
```

> 要讓 `make` 把這個變數帶進 build 容器，需確認它被 `export`（上方已 `export`）；若使用 `rules/config.user` 也可持久化設定。

### 做法 C：改內部 mirror URL（整個團隊統一改來源）

若公司有內部 artifact server，最省事的是只改 `BUILD_PUBLIC_URL` 指向內部 mirror，套件結構沿用官方版面：

```shell
make BUILD_PUBLIC_URL=https://artifactory.內部網域/sonic-public PLATFORM=broadcom target/sonic-broadcom.bin
```

前提：內部 mirror 上要有與官方相同的目錄結構與檔名（`sai/sai-broadcom/<branch>/<version>/xgs/libsaibcm_*.deb`）。

### 三種做法比較

| 做法 | 侵入性 | 適合情境 | 多客戶友善度 |
|------|--------|----------|--------------|
| A 本地複製（硬改） | 中（改 recipe） | 單次、單客戶快速驗證 | 低 |
| B 環境變數切換 | 低（改一次 recipe，之後只改變數） | **多客戶、CI 常態化** | 高 |
| C 內部 mirror | 最低（完全不改 recipe） | 全團隊統一內部來源 | 中 |

> 另外請記得 DNX / legacy-TH 客戶：若你的平台用的是 DNX 或 legacy-TH 晶片，相同手法要套到 `sai-dnx.mk` / `sai-legacy-th.mk`，而非只改 `sai-xgs.mk`。

---

## 7. 客戶二次開發 debian 的整合流程

不同客戶可能對 Broadcom SAI/SDK 做自家二次開發，產出自己的 `.deb`。建議用**做法 B（環境變數切換）**搭配以下管理慣例：

### 7.1 目錄與版本管理建議

```
/data/sai/
  customerA/
    libsaibcm_<verA>_amd64.deb
    libsaibcm-dev_<verA>_amd64.deb
  customerB/
    libsaibcm_<verB>_amd64.deb
    libsaibcm-dev_<verB>_amd64.deb
```

- **不要**把客戶 `.deb` 提交進 git（NDA + 體積）。改用內部 artifact server 或受控目錄。
- 每個客戶的 SAI 版本號若不同，需同步調整對應 `.mk` 的 `LIBSAIBCM_*_VERSION`，或讓版本也由環境變數帶入。

### 7.2 版本號也參數化（進階）

若客戶版本號各異，可把版本也做成變數：

```makefile
LIBSAIBCM_XGS_VERSION ?= 15.2.0.0.0.0.3.1
```

build 時覆寫：

```shell
make LIBSAIBCM_XGS_VERSION=15.2.0.0.0.0.9.9 \
     BRCM_XGS_SAI_LOCAL_PATH=/data/sai/customerB \
     PLATFORM=broadcom target/sonic-broadcom.bin
```

### 7.3 每客戶一份設定檔（最可維護）

為每個客戶準備一份 `config.user` 片段，避免 build 指令過長、也便於 CI：

```makefile
# rules/config.user  （此檔被 gitignore，不會進版控）
BRCM_XGS_SAI_LOCAL_PATH = /data/sai/customerA
LIBSAIBCM_XGS_VERSION = 15.2.0.0.0.0.3.1
SONIC_CONFIG_BUILD_JOBS = 4
```

### 7.4 驗證替換是否生效

build 後，從產出的 deb 目錄確認版本與來源：

```shell
ls -l target/debs/*/libsaibcm*_amd64.deb
dpkg-deb -I target/debs/<BLDENV>/libsaibcm_<version>_amd64.deb   # 看 metadata 確認是客戶版本
```

> 小技巧：客戶二次開發版本建議在套件 changelog / maintainer 欄位加上可辨識字串，方便事後確認「這顆 image 用的是哪個客戶的 SAI」。

---

## 8. 完整 build 操作流程

以下是從零到產出 Broadcom image 的正式流程（整合 `README.md` 並補上 Broadcom 替換步驟）。

### 8.1 環境準備（一次性）

硬體建議：多核心、RAM ≥ 8 GiB（建議更多）、磁碟 ≥ 300G、支援 KVM。OS 建議 Ubuntu 22.04 / 24.04。

```shell
# 安裝 pip 與 jinjanator
sudo apt install -y python3-pip
pip3 install --user jinjanator

# 安裝 Docker，並讓目前使用者免 sudo
sudo gpasswd -a ${USER} docker
# 登出再登入讓群組生效
```

### 8.2 取得原始碼

```shell
git clone --recurse-submodules https://github.com/sonic-net/sonic-buildimage.git
cd sonic-buildimage
# 若需特定分支：git checkout <branch_name>
```

### 8.3 設定 Broadcom SAI 來源（本文重點）

依第 6 章擇一做法。以做法 B 為例，先備妥本地 `.deb`，再透過變數指定（見 8.5）。

### 8.4 初始化與設定平台

```shell
sudo modprobe overlay     # 確保 overlay 模組載入

make init                 # clone 後或更新 submodule 後執行一次
make configure PLATFORM=broadcom
```

### 8.5 建置 image

```shell
# 官方線上 SAI
make SONIC_BUILD_JOBS=4 PLATFORM=broadcom target/sonic-broadcom.bin

# 或：使用客戶本地 SAI（做法 B）
make SONIC_BUILD_JOBS=4 \
     BRCM_XGS_SAI_LOCAL_PATH=/data/sai/customerA \
     PLATFORM=broadcom target/sonic-broadcom.bin
```

記憶體對照（來自 `README.md`，供排 JOBS 參考）：

| JOBS | 約需 RAM |
|------|----------|
| 1 | ~10 GB |
| 4 | ~28 GB |
| 8 | ~52 GB |

加速/省記憶體選項：

```shell
# 跳過測試加速迭代
make SONIC_BUILD_JOBS=4 BUILD_SKIP_TEST=y PLATFORM=broadcom target/sonic-broadcom.bin

# 限制 build 容器記憶體（寫進 rules/config.user）
# SONIC_BUILD_MEMORY = 24g
```

### 8.6 產物

- 最終安裝檔：`target/sonic-broadcom.bin`（ONIE installer）。
- 中間 deb：`target/debs/<BLDENV>/`，可在此確認 SAI 版本是否為預期。

---

## 9. 常見問題排查

| 症狀 | 可能原因 | 處置 |
|------|----------|------|
| `curl: ... 404` 抓 `libsaibcm` 失敗 | `LIBSAIBCM_*_VERSION` 與來源實際檔名不符 | 對齊版本字串；或改走本地複製（做法 A/B） |
| 改成 `SONIC_COPY_DEBS` 後說找不到檔 | `_PATH` 路徑錯、或檔案不在 build 容器可見範圍 | 確認路徑相對 repo 根目錄、檔案確實存在 |
| 版本沒變但 image 還是舊 SAI | DPKG cache 命中舊套件 | 清掉該 deb target 後重 build，或確認 `SKIP_VERSION` 設定 |
| 環境變數沒帶進 build 容器 | 變數未 `export` | 在 `.mk` 加 `export`，或寫進 `rules/config.user` |
| DNX/legacy 平台改了 xgs 卻沒效果 | 改錯檔案 | DNX 改 `sai-dnx.mk`、legacy 改 `sai-legacy-th.mk` |
| 不確定最終用的是哪顆 SAI | 無法辨識來源 | `dpkg-deb -I` 看 metadata；建議替換版加辨識字串 |

清除特定套件重建（示意）：

```shell
# 以 target group 的 clean 目標清掉，再重新 build
make target/debs/<BLDENV>/libsaibcm_<version>_amd64.deb-clean
make SONIC_BUILD_JOBS=4 PLATFORM=broadcom target/sonic-broadcom.bin
```

---

## 10. 新進人員學習路徑

建議依序完成，每一步都有可驗收的產出：

1. **讀懂架構（約 0.5 天）**
   - 讀本文第 2、3 章。
   - 驗收：能畫出「swss → syncd → SAI → ASIC」的資料流，並說明 SAI 的作用。

2. **跑通官方 build（約 1 天，含編譯等待）**
   - 依第 8 章用官方線上 SAI build 出 `target/sonic-broadcom.bin`。
   - 驗收：成功產出 `.bin`，並在 `target/debs/` 找到 `libsaibcm`。

3. **做一次本地 SAI 替換（約 0.5 天）**
   - 用做法 A 把 SAI 改成本地複製（可先用官方抓下來的同一顆 deb 當作練習素材）。
   - 驗收：build 日誌顯示 copy 而非 curl，image 內 SAI 版本正確。

4. **理解多客戶流程（約 0.5 天）**
   - 用做法 B 的環境變數切換，模擬兩個客戶來源。
   - 驗收：同一份程式碼，靠不同變數 build 出不同 SAI 的 image。

5. **熟悉排查（持續）**
   - 刻意製造版本不符、路徑錯誤，照第 9 章排除。

> 完成 1～4 即具備獨立處理「Broadcom image build + 客戶 SAI 替換」的能力。

---

## 附錄 A：關鍵檔案速查

| 路徑 | 作用 |
|------|------|
| `platform/broadcom/rules.mk` | Broadcom 平台 recipe 入口，include 各 SAI/docker recipe |
| `platform/broadcom/sai-xgs.mk` | XGS SAI（`libsaibcm`）定義 — **替換主要改這裡** |
| `platform/broadcom/sai-dnx.mk` | DNX SAI 定義 |
| `platform/broadcom/sai-legacy-th.mk` | Legacy-TH SAI 定義 |
| `platform/broadcom/docker-syncd-brcm.mk` | 將 SAI 注入 `syncd` 容器 |
| `platform/nephos/sai.mk` | 官方「本地/線上切換」寫法範本（做法 B 參考） |
| `slave.mk` | build 後端，定義 `SONIC_ONLINE_DEBS` / `SONIC_COPY_DEBS` 等規則 |
| `rules/config` | build 全域設定（含 `BUILD_PUBLIC_URL` 預設值） |
| `rules/config.user` | 個人/客戶持久化設定（gitignore，不進版控） |
| `README.buildsystem.md` | build 系統權威說明 |

## 附錄 B：合規備忘

- Broadcom SAI/SDK debian 受 NDA 保護，禁止提交進公開 repo、公開 CI log 或外部可存取位置。
- 客戶二次開發的 `.deb` 視同機密資產，僅於公司核可的受控管道傳遞與存放。
- 內部 mirror / artifact server 的帳密不得寫進版控檔案；使用 CI secret 或環境變數管理。

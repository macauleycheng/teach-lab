# SONiC 測試架構與新增測試案例教學

本文件說明 SONiC 的測試架構，並以實作範例帶你新增測試案例。

## 0. 本文的依據與限制

- 本機路徑以 `${HOME}/sonic-buildimage` 為根（下文簡寫為 `<BI>`），其中的子模組 `src/sonic-swss`、`src/sonic-utilities`、`src/sonic-swss-common`、`src/ptf-py3` 皆已存在，文中引用的本機路徑都已用 `ls` 確認過。
- **本機沒有 `sonic-mgmt` repo**（已搜尋 `${HOME}`、`.agents/tasks`、`.kiro`，只找到 `src/sonic-mgmt-framework`、`src/sonic-mgmt-common`，它們是管理介面程式，不是測試 repo）。因此 sonic-mgmt 的部分（第 3、4、8、9 章與 5.4、5.5 節）依據上游官方文件與原始碼，並附連結。
- 這台機器的 Python 環境缺少 `sonic_py_common` 等相依套件（執行 `src/sonic-utilities/tests` 時出現 `ModuleNotFoundError: No module named 'sonic_py_common'`），所以文中的新增範例**沒有在本機實際執行通過**，只做了語法檢查與對照既有測試的 API 驗證。請依第 6 章在 sonic-slave 容器或（另購伺服器上的）sonic-mgmt 容器內執行。
- **testbed 不會在這台機器（`${HOME}`）上搭建**，而是跑在另外採購的伺服器上。因此第 9 章整理了採購用的伺服器環境需求、規格分級與驗收指令；第 8 章的步驟是在那台新伺服器上執行。這台機器的狀態（無 `/dev/kvm`、Ubuntu 20.04）只在第 8.2.0 節留作簡短附註。第 8 章的指令來自上游文件與本機 Makefile 的查證，沒有在本機實跑。
- 上游 sonic-mgmt 的內容會隨 master 演進，數字與檔案清單以你 checkout 的版本為準。

## 1. SONiC 測試分層總覽

| 層級 | 位置 | 框架 | 需要的環境 | 驗證重點 |
|---|---|---|---|---|
| 單元測試（Python CLI） | `<BI>/src/sonic-utilities/tests` | pytest + click `CliRunner` + mock DB | 一般 Linux / slave 容器 | `show`、`config` 等 CLI 邏輯 |
| 單元測試（C++ 函式庫） | `<BI>/src/sonic-swss-common/tests` | gtest（`tests/Makefile.am`）與 pytest（`tests/test_*.py`） | slave 容器、Redis | swss-common 函式庫 |
| 單元測試（orchagent / cfgmgr） | `<BI>/src/sonic-swss/tests/mock_tests` | gtest + gmock（mock SAI） | slave 容器 | Orch 類別、cfgmgr 邏輯 |
| 虛擬交換機測試（vstest） | `<BI>/src/sonic-swss/tests/test_*.py` | pytest + `docker-sonic-vs` 容器 | Ubuntu 22.04、Docker | 寫入 CONFIG_DB / APPL_DB，檢查 ASIC_DB（控制面） |
| 系統測試 | 上游 `sonic-mgmt/tests` | pytest + Ansible + PTF | KVM 虛擬或實體 testbed | 整機功能、資料面、重開機、平台 |
| 資料面封包測試 | PTF（`<BI>/src/ptf-py3`，sonic-mgmt 內有 `tests/ptftests`） | PTF（unittest 為基礎） | PTF 容器或主機 | 送封包、驗證收到的封包 |
| CI | `<BI>/.azure-pipelines`、各 repo 的 `azure-pipelines.yml` | Azure Pipelines | 雲端 agent | 建置、單元測試、VS 映像上跑 sonic-mgmt |

選擇原則：能用單元測試驗證的邏輯就不要放到 testbed 上跑；單元測試快速、可在任何 Linux 上執行。需要真實進程互動與 Redis 資料流時用 vstest；需要多裝置、BGP 鄰居或封包收發時才用 sonic-mgmt。

## 2. 各 repo 內的測試（本機可查證）

### 2.1 sonic-utilities：pytest 單元測試

- 測試目錄：`<BI>/src/sonic-utilities/tests`，檔名慣例為 `*_test.py`（例如 `vlan_test.py`、`config_int_mtu_test.py`）。
- 根目錄的 `conftest.py` 說明是為 pytest-xdist 平行化而設；`pytest.ini` 設定了 `-n auto --dist loadfile`、覆蓋率與 `test-results.xml`，並定義 `serial` marker。
- `tests/mock_tables/` 內有模擬資料庫內容，例如 `config_db.json`、`appl_db.json`、`asic_db.json`、`counters_db.json`；`tests/mock_tables/dbconnector.py` 對 `swsssdk`／`SonicV2Connector` 做 monkey patch，讓測試讀寫的是 JSON 檔而不是真的 Redis。
- 例如 `mock_tables/config_db.json` 內有 `VLAN|Vlan1000`、`VLAN|Vlan2000`、`VLAN|Vlan3000`、`VLAN|Vlan4000`，`vlan_test.py` 的預期輸出就是依這份資料而來。
- `tests/conftest.py` 有 autouse fixture（如 `setup_db_config`、`setup_env`），以及 `setup_single_broadcom_asic`、`setup_t1_topo` 等情境 fixture。
- 慣用寫法是用 click 的 `CliRunner().invoke(...)` 呼叫指令物件，再檢查 `exit_code` 與 `output`；需要 DB 時用 `utilities_common.db.Db()` 並以 `obj=db` 傳入。
- 官方 README 的執行方式為在 sonic-slave 容器內執行 `python3 setup.py test`（見 `<BI>/src/sonic-utilities/README.md`）。

### 2.2 sonic-swss-common：gtest 與 pytest

- `<BI>/src/sonic-swss-common/tests/Makefile.am` 以 `tests_tests_SOURCES` 列出所有 `*_ut.cpp`，連結 `-lgtest -lgtest_main -lgmock -lgmock_main`，產生 `tests/tests` 執行檔。
- 同目錄也有 `test_*.py`（如 `test_redis_ut.py`、`test_logger.py`），搭配 `tests/conftest.py` 以 `SonicDBConfig.initialize(...)` 載入 `redis_multi_db_ut_config/database_config.json`。

### 2.3 sonic-swss：gtest（mock_tests）

- 位置：`<BI>/src/sonic-swss/tests/mock_tests`，例如 `bulker_ut.cpp`、`swssnet_ut.cpp`、`portsorch_ut.cpp`。
- `mock_tests/Makefile.am` 定義多個測試執行檔：`tests`（orchagent）、`tests_intfmgrd`、`tests_teammgrd`、`tests_portsyncd`、`tests_fpmsyncd`、`tests_fdbsyncd`、`tests_response_publisher`、`tests_nbrmgrd`、`tests_teamsyncd`，並登記在 `TESTS` 與 `noinst_PROGRAMS`。
- 使用 `-lsaivs -lsairedis -lsaimeta -lsaimetadata` 連結 SAI virtual switch 與 sairedis；`mock_sai_api.h`、`mock_*.cpp` 提供替身。
- 新增一個 `*_ut.cpp` 時，必須把檔名加進 `Makefile.am` 的 `tests_SOURCES`。
- `<BI>/src/sonic-swss/run-gtest-suite.py` 接受 `--test-name`、`--log-file`、`--trs-file` 等參數，推測為 automake 的測試驅動腳本（此點未實際驗證）。

### 2.4 sonic-swss vstest：docker-sonic-vs

`<BI>/src/sonic-swss/tests/README.md` 的重點如下：

- DVS（Docker Virtual Switch）讓開發者不需要實體交換機或 ASIC，就能驗證 SWSS 新功能的控制面行為。
- 測試流程：對 redis（通常是 CONFIG_DB 或 APPL_DB）寫入設定，檢查 SWSS 處理後的狀態（通常看 ASIC_DB）。
- SWSS、Redis 等元件都跑在 `docker-sonic-vs` 容器內。資料面不被模擬。
- 環境需求：Ubuntu 22.04、Python 3、Docker、`team` kernel module，以及 `libswsscommon`、`python3-swsscommon` 等套件。

程式結構：

- `tests/conftest.py`：定義 `pytest_addoption`（`--dvsname`、`--forcedvs`、`--keeptb`、`--imgname`、`--max_cpu`、`--num-ports` 等）、session 級的 `manage_dvs`、module 級的 `dvs`，以及 `testlog`。
- `dvs` fixture 會讀取測試模組內的 `DVS_ENV`、`NUM_PORTS` 變數來決定容器環境。
- `tests/dvslib/`：各功能的輔助類別，如 `dvs_vlan.py`、`dvs_acl.py`、`dvs_lag.py`、`dvs_route.py`、`dvs_database.py`（DB 存取與輪詢）、`dvs_common.py`（`PollingConfig`、`wait_for_result`）。
- `conftest.py` 以 class 級 fixture（如 `dvs_vlan_manager`、`dvs_lag_manager`）建立 `dvslib` 物件，並掛到 `request.cls`，所以測試類別內可直接用 `self.dvs_vlan`。

### 2.5 PTF（資料面）

- 本機有兩份原始碼：`<BI>/src/ptf` 與 `<BI>/src/ptf-py3`（後者為 Python 3 版本）。`platform/vs/docker-ptf.mk` 定義 PTF 容器映像。
- `<BI>/src/ptf-py3/README.md` 說明：PTF 是以 Python `unittest` 為基礎的資料面測試框架，預設使用 Scapy 組封包，需要 root 權限，並以 `--interface <port>@<netdev>` 指定要綁定的網卡，例如：

```bash
sudo ./ptf --test-dir mytests/ --pypath $PWD \
    --interface 0@veth1 --interface 1@veth3
```

- 範例測試放在 `<BI>/src/ptf-py3/example/mytests`。

## 3. sonic-mgmt：系統測試架構（上游文件）

本章依據 [sonic-mgmt 的 testbed 文件](https://github.com/sonic-net/sonic-mgmt/blob/master/docs/testbed/README.testbed.Overview.md)與[測試文件索引](https://github.com/sonic-net/sonic-mgmt/blob/master/docs/tests/README.md)。

### 3.1 實體連線與硬體需求

上游文件將 testbed 的實體元件分為：

- Test server（測試伺服器）
- Fanout switch（扇出交換機，分 leaf 與 root；root 非必要）
- SONiC DUT（Device Under Test，受測設備）

關鍵設計：每個 DUT 連接埠接到某個 leaf fanout，leaf fanout 為每個 DUT 連接埠指定唯一 VLAN tag，經 802.1Q trunk（必要時再經 root fanout）與測試伺服器相連。因此測試伺服器能以該 VLAN tag 存取 DUT 的任何連接埠。例如 testbed 1 使用 VLAN 100 至 131，testbed 2 使用 132 至 163（皆為 32 埠 DUT）。伺服器、fanout、DUT 的數量上游沒有固定。

測試伺服器上會建立：

- 一組 VM 或容器（vEOS、cEOS 或 SONiC）模擬 DUT 的鄰居。
- 一個 PTF 容器，用來送出與接收封包。
- 這些元件與 VLAN 介面之間以 Open vSwitch（OVS）bridge 互連。

硬體需求（出自[上游 Overview](https://github.com/sonic-net/sonic-mgmt/blob/master/docs/testbed/README.testbed.Overview.md)與 [FAQ](https://github.com/sonic-net/sonic-mgmt/blob/master/docs/testbed/README.testbed.FAQ.md)，此處為改寫整理）：

| 元件 | 需求 |
|---|---|
| DUT | 要測的 SONiC 實機。每個要測的連接埠都要接到 leaf fanout。 |
| Leaf fanout | 上游舉例 Arista 7260 或類似機型。埠數必須多於 DUT。需支援 LACP／LLDP 穿透與 802.1Q tunneling（QinQ）。 |
| Root fanout（選用） | 需求同 leaf。若只有一個 testbed，或測試伺服器只服務一個 testbed，可省略，leaf 的 trunk 直接接伺服器網卡。 |
| VLAN trunk | DUT 面向埠為 access（每埠唯一 VLAN），上行為 trunk 並允許全部這些 VLAN，直達測試伺服器網卡。 |
| 測試伺服器 | 上游的參考規格（明確寫「非強制」）：Dell 730、2 顆 CPU 各 18 核、192 GB RAM、2 顆 500 GB 磁碟、Mellanox MT27700（CX4）網卡。FAQ 的最低需求是一台 L2 fanout 交換機與至少 92 GB 記憶體的 Linux 伺服器（文件較舊）。 |
| 管理網路 | 伺服器、VM、PTF 與 DUT 的管理埠都在同一管理網段，伺服器端以 bridge `br1` 承載（範例網段 `10.250.0.0/24`）。建議伺服器另有一張網卡用於連到實驗室網路。Console server 與 PDU 為選用。 |

PTF 與鄰居的接法依拓撲不同。T0 的前 N 個 DUT 埠直接對應 PTF 容器。T1 與 T2 的 PTF 沒有直接接到 DUT 的埠，而是把 PTF 的 veth 「注入」到各 VM 連線的 OVS bridge。依上游 Overview，OVS flow 規則為：VM 到 DUT 只往 DUT；DUT 到 VM 與 PTF；PTF 到 DUT 只往 DUT。

### 3.2 邏輯拓撲

上游 Overview 寫明主要有 5 類拓撲：T0、T1、T2、PTF、PTP。每一類各有許多變體，細節定義在 sonic-mgmt 的 `ansible/vars/topo_*.yml`（目前超過 150 個檔，清單見 [ansible/vars](https://github.com/sonic-net/sonic-mgmt/tree/master/ansible/vars)，以你 checkout 為準）。testbed 項目的 `topo:` 欄位（例如 `t0`、`t1-lag`、`ptf32`）決定使用哪個檔案；測試內的 `pytest.mark.topology('t0')` 則是「這個測試允許在哪些拓撲跑」的選擇器。

先釐清兩個概念，避免混淆：

| 網路層級 | 下游 | 上游 |
|---|---|---|
| T0（ToR） | 伺服器 | T1 |
| T1 | T0 | T2 |
| T2 | T1 | Regional Gateway（T3） |

這張表只是資料中心網路的階層。下面的「五類邏輯拓撲」是測試環境怎麼接 DUT 的定義。

#### 五類邏輯拓撲總表（以各類代表變體計）

數字來源：VM 數為 `topo_*.yml` 中 `topology.VMs` 的項目數（我解析 yml 計算），DUT 埠數與角色對照 Overview 文字。

| 類型 | 代表變體 | DUT 埠數 | 鄰居 VM 數 | 埠角色 | 需要 VM | 上游檔案 |
|---|---|---|---|---|---|---|
| T0 | `t0` | 32 | 4（ARISTA01T1 至 04T1） | 前 28 埠接 PTF（模擬伺服器），後 4 埠各接 1 台 VM（扮演 T1，單一連線的 port-channel） | 是 | [topo_t0.yml](https://github.com/sonic-net/sonic-mgmt/blob/master/ansible/vars/topo_t0.yml) |
| T1 | `t1` | 32 | 32 | 16 埠各接 1 台 VM 扮演上游 T2，16 埠各接 1 台 VM 扮演下游 T0；無 port-channel；PTF 沒有直連埠，只有注入埠 | 是 | [topo_t1.yml](https://github.com/sonic-net/sonic-mgmt/blob/master/ansible/vars/topo_t1.yml) |
| T2 | `t2` | 96（3 張 linecard 各 32）；supervisor 無 DUT 埠 | 72 | linecard 1 的 24 台 VM 扮演 T3，linecard 2 與 3 各 24 台扮演 T1；`dut_num: 4`（3 LC + 1 SUP）；PTF 只有注入埠（0 至 71） | 是 | [topo_t2.yml](https://github.com/sonic-net/sonic-mgmt/blob/master/ansible/vars/topo_t2.yml) |
| PTF | `ptf32` | 32 | 0 | 全部 DUT 埠接 PTF 容器，無注入埠 | 否 | [topo_ptf32.yml](https://github.com/sonic-net/sonic-mgmt/blob/master/ansible/vars/topo_ptf32.yml) |
| PTP | `ptp-32` | 32 | 0 | 兩台 SONiC DUT 以多條連線互連（point-to-point），不接 PTF 與 VM | 否 | [topo_ptp-32.yml](https://github.com/sonic-net/sonic-mgmt/blob/master/ansible/vars/topo_ptp-32.yml) |

重要提醒：「T0 為 4 台 VM、T1 為 32 台 VM」只對代表變體 `t0`、`t1` 成立，換變體數字就會變。T2 不能一概視為機箱：`t2` 是 72 台 VM 的機箱拓撲，另有單節點版本。PTP 不是 Precision Time Protocol。

#### 變體清單與連結

| 類型 | 變體 | DUT 埠 | VM 數 | 說明 | 檔案 |
|---|---|---|---|---|---|
| T0 | `t0` | 32 | 4 | 28 埠接 PTF，4 埠接 4 台 VM | [yml](https://github.com/sonic-net/sonic-mgmt/blob/master/ansible/vars/topo_t0.yml) |
| T0 | `t0-56` | 56 | 8 | 48 埠接 PTF，8 埠接 8 台 VM | [yml](https://github.com/sonic-net/sonic-mgmt/blob/master/ansible/vars/topo_t0-56.yml) |
| T0 | `t0-64` | 64 | 4 | 56 個 host 介面加 8 個 VM 埠（每台 VM 2 條連線，由 yml 計算） | [yml](https://github.com/sonic-net/sonic-mgmt/blob/master/ansible/vars/topo_t0-64.yml) |
| T0 | `t0-80` | 80 | 8 | 64 埠接 PTF，16 埠接 8 台 VM（各 2 條連線的 port-channel） | Overview 描述 |
| T0 | `t0-116` | 以 yml 計算為 112 個 host 加 8 個 VM 埠 | 4 | 只在 yml 中確認，不把「116 埠」當事實 | [yml](https://github.com/sonic-net/sonic-mgmt/blob/master/ansible/vars/topo_t0-116.yml) |
| T1 | `t1` | 32 | 32 | 見上表 | [yml](https://github.com/sonic-net/sonic-mgmt/blob/master/ansible/vars/topo_t1.yml) |
| T1 | `t1-lag` | 32 | 24 | 16 埠接 8 台 T2 VM（各 2 條連線的 port-channel），16 埠接 16 台 T0 VM | [yml](https://github.com/sonic-net/sonic-mgmt/blob/master/ansible/vars/topo_t1-lag.yml) |
| T1 | `t1-64` | 64 | 64 | 每埠 1 台 VM（由 yml 計算） | [yml](https://github.com/sonic-net/sonic-mgmt/blob/master/ansible/vars/topo_t1-64.yml) |
| T1 | `t1-8-lag` | 多 ASIC 虛擬 DUT 用 | 2 | vtestbed 的 four-asic 項目使用 | [yml](https://github.com/sonic-net/sonic-mgmt/blob/master/ansible/vars/topo_t1-8-lag.yml) |
| T1 | `t1-isolated-d224u8` | 232 | 232 | 8 台 T2 在上、224 台 T0 在下 | Overview 描述 |
| T2 | `t2` | 96 | 72 | 機箱：3 linecard 加 1 supervisor | [yml](https://github.com/sonic-net/sonic-mgmt/blob/master/ansible/vars/topo_t2.yml) |
| T2 | `t2_single_node_min` | 單節點（非機箱） | 5 | vtestbed 項目 `vms-kvm-t2-single-node-min`（DUT `vlab-ut2`） | [yml](https://github.com/sonic-net/sonic-mgmt/blob/master/ansible/vars/topo_t2_single_node_min.yml) |
| T2 | `t2_2lc_min_ports-masic` | 多 ASIC 機箱 | 未逐一統計 | vtestbed 項目 `vms-kvm-t2`，3 台 DUT（`vlab-t2-1-1`、`vlab-t2-1-2`、`vlab-t2-1-sup`） | [vtestbed.yaml](https://github.com/sonic-net/sonic-mgmt/blob/master/ansible/vtestbed.yaml) |
| PTF | `ptf32` | 32 | 0 | 全部接 PTF | [yml](https://github.com/sonic-net/sonic-mgmt/blob/master/ansible/vars/topo_ptf32.yml) |
| PTF | `ptf64` | 64 | 0 | 全部接 PTF | [yml](https://github.com/sonic-net/sonic-mgmt/blob/master/ansible/vars/topo_ptf64.yml) |
| PTF | `ptf8` | 8 | 0 | `dut_type` 為 SonicHost | [yml](https://github.com/sonic-net/sonic-mgmt/blob/master/ansible/vars/topo_ptf8.yml) |
| PTP | `ptp-32` | 32 | 0 | 名稱中的數字即埠數；另有 `ptp-56`、`ptp-64`、`ptp-256` 等 | [yml](https://github.com/sonic-net/sonic-mgmt/blob/master/ansible/vars/topo_ptp-32.yml) |

上游另有 M0、MX、dualtor、wan、smartswitch 等其他家族（Overview 列有 M0：52 埠 6 台 VM；MX：48 埠 2 台 VM），本文不展開。

#### 各類型的 DUT、VM 與硬體需求

| 類型 | 要不要 VM | DUT 台數與埠角色 | 實體測試需要的硬體 |
|---|---|---|---|
| T0 | 要，`t0` 為 4 台（`t0-56`、`t0-80` 為 8 台） | 1 台 DUT，32 埠中 28 埠為伺服器側（PTF），4 埠為上行（VM 扮 T1） | 32 埠 DUT；leaf fanout 至少 32 個 DUT 面向埠加上行 trunk；每埠一個 VLAN；測試伺服器要跑 4 台鄰居與 PTF |
| T1 | 要，`t1` 為 32 台 | 1 台 DUT，16 埠上行、16 埠下行，全部接 VM | 32 埠 DUT；leaf fanout 32 個以上；`t1` 鄰居 VM 數多，伺服器資源需求高於 T0（推論：上游未給 T1 專屬規格，FAQ 的 92 GB 為通用最低需求，且文件較舊） |
| T2 | 要，`t2` 為 72 台 | 機箱 4 個 DUT 節點（3 linecard 加 1 supervisor），共 96 個 DUT 埠 | 需要機箱式 DUT；96 個 DUT 埠對應的 fanout 埠與 VLAN；伺服器可能要多台，資源遠大於 T0／T1（此點為推論，上游未給 T2 的伺服器規格） |
| PTF | 不要 | 1 台 DUT，全部埠接 PTF | 只需要 DUT、fanout（或直連）與測試伺服器，不需鄰居 VM，伺服器資源最小 |
| PTP | 不要 | 2 台 SONiC DUT，埠對埠互連 | 兩台實體 DUT 以光模組／DAC 直接互連，資料路徑不需要 fanout VLAN trunk；管理網路仍需要 |

PTP 的真正含義：point-to-point，不是 Precision Time Protocol。依 Overview，它用來驗證收發器（transceiver）及其在 LLDP 等 L2 控制流量下的連線穩定性，且不使用接到 PTF 或 VM 的埠，改由兩台 SONiC DUT 之間以多條連線互連。有一個不一致之處要提醒：`topo_ptp-32.yml` 本身是 `VMs: {}`、`host_interfaces` 為 0 至 31、`configuration: {}`，也就是說 yml 只是「無 VM」的定義，兩台 DUT 互連是實驗室實體接線。我在 `ansible/vtestbed.yaml` 找不到任何 PTP 項目，因此沒有虛擬 KVM 的 PTP 範例，PTP 偏向實體實驗室。

資料來源：[Overview 的 Logical topologies 章節](https://github.com/sonic-net/sonic-mgmt/blob/master/docs/testbed/README.testbed.Overview.md)與上表連結的 `topo_*.yml`。

### 3.3 Testbed 設定檔

`ansible/vtestbed.yaml` 是虛擬環境的 testbed 檔，實體環境使用 `ansible/testbed.yaml`（CSV 版已棄用），欄位相同。每筆記錄形如：

```yaml
- conf-name: vms-kvm-t0
  group-name: vms6-1
  topo: t0
  ptf_image_name: docker-ptf
  ptf: ptf-01
  ptf_ip: 10.250.0.102/24
  server: server_1
  vm_base: VM0100
  dut:
    - vlab-01
  inv_name: veos_vtb
```

（欄位取自[上游 vtestbed.yaml](https://github.com/sonic-net/sonic-mgmt/blob/master/ansible/vtestbed.yaml)，僅列出部分欄位；實際還有 `ptf_ipv6`、`auto_recover`、`use_converged_peers`、`comment`。）

- `conf-name`：testbed 名稱，即 `run_tests.sh -n` 與 pytest `--testbed` 的值，必須唯一。
- `group-name`：長度不超過 8 個字元；相同 `group-name` 的項目必須共用 `ptf_ip`、`server`、`vm_base`。
- `topo`：使用哪份 `topo_*.yml`，必須存在於 `ansible/vars/`。
- `vm_base`：此 testbed 第一台鄰居 VM 的名稱。`t0` 有 4 台 VM，所以佔用 `VM0100` 至 `VM0103`；不同 `group-name` 的 `vm_base` 範圍不可重疊（見[組態文件的 Consistency Rule](https://github.com/sonic-net/sonic-mgmt/blob/master/docs/testbed/README.new.testbed.Configuration.md)）。
- `dut`：DUT 主機名稱，即 `-d` 或 `--host-pattern`。
- `inv_name`：Ansible inventory 檔名（`veos_vtb`；實體環境為 `veos` 加上 `lab`）。

### 3.4 搭建流程總覽

搭建測試環境的完整步驟已移到第 8 章：8.1 為路線選擇表，8.2 為單機 KVM 虛擬 testbed（路線 A），8.3 為實體 testbed（路線 B）。要採購什麼規格的伺服器，見第 9 章。整體順序在兩條路線一致：準備主機與管理網路、取得映像、建立 sonic-mgmt 容器、改 inventory 與 testbed 檔、`add-topo`、`deploy-mg`、跑冒煙測試、清除。

### 3.5 程式目錄與重要元件

以下依 [sonic-mgmt master 目錄](https://github.com/sonic-net/sonic-mgmt/tree/master/tests)與[測試組織文件](https://github.com/sonic-net/sonic-mgmt/blob/master/docs/tests/pytest.org.md)整理：

| 路徑 | 用途 |
|---|---|
| `tests/<feature>/test_*.py` | 依功能分資料夾的測試，例如 `bgp/`、`arp/`、`acl/` |
| `tests/conftest.py` | 根 conftest，定義 `--testbed` 等命令列選項與核心 fixture |
| `tests/pytest.ini` | marker 定義（`topology`、`platform`、`disable_loganalyzer` 等）與 `python_files = test_*.py unit_test_*.py` |
| `tests/common/` | 可重用程式碼（helpers、fixtures、devices、plugins、utilities） |
| `tests/common/helpers/assertions.py` | `pytest_assert`、`pytest_require` |
| `tests/common/utilities.py` | `wait_until` 等工具 |
| `tests/common/devices/sonic.py` | `SonicHost`（`duthost`），含 `show_and_parse`、`get_running_config_facts` 等 |
| `tests/common/plugins/` | `ptfadapter`、`loganalyzer`、`sanity_check`、`conditional_mark`、`custom_markers` 等外掛 |
| `tests/ptftests/` | 在 PTF 容器內執行的封包測試 |
| `tests/ptf_runner.py` | 從 pytest 啟動 PTF 測試的 `ptf_runner()` |
| `tests/run_tests.sh` | 官方建議的執行入口 |
| `ansible/` | testbed 檔、inventory、拓撲、`testbed-cli.sh` |
| `.azure-pipelines/` | PR 檢查與 `pr_test_scripts.yaml` |

### 3.6 核心 fixture

下列 fixture 均可在上游 [tests/conftest.py](https://github.com/sonic-net/sonic-mgmt/blob/master/tests/conftest.py) 找到定義：

| fixture | scope | 說明 |
|---|---|---|
| `tbinfo` | session | 由 `--testbed`、`--testbed_file` 載入的 testbed 資訊（拓撲、PTF 等） |
| `duthosts` | session | 所有 DUT 的集合，以主機名稱取值：`duthosts[hostname]` |
| `rand_one_dut_hostname` | module | 隨機挑一台 DUT 的主機名稱 |
| `rand_selected_dut` | module | `duthosts[rand_one_dut_hostname]` |
| `duthost` | session | 單一 DUT 物件（在單 DUT testbed 方便使用） |
| `ptfhost` | session | PTF 容器的主機物件（向下相容單一 PTF） |
| `localhost` | session | 在 sonic-mgmt 容器本機執行 Ansible 模組 |
| `nbrhosts` | session | 鄰居裝置（EOS 等） |
| `creds` | session | 帳密資訊 |
| `enum_frontend_dut_hostname`、`enum_asic_index` 等 | 見原始碼 | 透過 `pytest_generate_tests` 對每個 DUT／ASIC 展開參數化 |
| `ptfadapter` | 見外掛 | 在 pytest 中直接收發封包，詳見[ptfadapter README](https://github.com/sonic-net/sonic-mgmt/blob/master/tests/common/plugins/ptfadapter/README.md) |
| `loganalyzer` | 自動啟用 | 測試前後在 DUT syslog 加標記並分析，詳見[loganalyzer README](https://github.com/sonic-net/sonic-mgmt/blob/master/tests/common/plugins/loganalyzer/README.md) |

`duthost` 物件會把未知的方法呼叫轉成 Ansible 模組（`tests/common/devices/base.py` 的 `__getattr__`），因此 `duthost.shell(...)`、`duthost.command(...)` 等都可直接使用。

### 3.7 Markers

依[測試組織文件](https://github.com/sonic-net/sonic-mgmt/blob/master/docs/tests/pytest.org.md)與[指南](https://github.com/sonic-net/sonic-mgmt/blob/master/docs/tests/guidelines.md)：

- 每個測試都需要 `topology` marker，用來宣告可執行的拓撲（`t0`、`t1`、`ptf`、`any` 等）。
- 測試模組以 `pytestmark = [...]` 設定模組層級 marker；case 層級 marker 會覆蓋模組層級。
- 沒有 topology marker 的測試，在指定拓撲時會被跳過。
- 其他常用 marker：`asic`、`feature`、`device_type('vs')`、`disable_loganalyzer`。實際可用清單以 `tests/pytest.ini` 為準。

例如上游 `tests/bgp/test_bgp_fact.py` 全文只有數行：

```python
import pytest
from tests.common.helpers.bgp import run_bgp_facts

pytestmark = [
    pytest.mark.topology('any', 't0-sonic', 't1-multi-asic'),
    pytest.mark.device_type('vs')
]


def test_bgp_facts(duthosts, enum_frontend_dut_hostname, enum_asic_index):
    run_bgp_facts(duthosts[enum_frontend_dut_hostname], enum_asic_index)
```

### 3.8 CI 如何串起來

- `<BI>/.azure-pipelines/azure-pipelines-build-vs-and-test.yml` 在 `resources.repositories` 宣告 `sonic-net/sonic-mgmt` 的 `master`，`Test` 階段（job `KVMTEST`，名稱 "kvmtest by Elastictest"）使用 `run-test-elastictest-template.yml@sonic-mgmt` 範本，並設定 `inventory: veos_vtb`、`testbed_file: vtestbed.yaml`。也就是說 buildimage 建完 VS 映像後，會在 KVM testbed 上跑 sonic-mgmt 測試。
- `<BI>/src/sonic-swss/.azure-pipelines/test-docker-sonic-vs-template.yml` 會載入 `docker-sonic-vs.gz`，用 `./run-tests.sh` 平行執行 vstest。
- sonic-mgmt 本身的 PR 檢查包含 `pre-commit`（flake8，`--max-line-length=120`）、markers 檢查（`.azure-pipelines/markers-check.yml`）與 `pytest-collect-only`。另有 `.azure-pipelines/pr_test_scripts.yaml`，依拓撲（如 `t0`）列出 PR 要跑的測試檔。

## 4. 測試風格要求（上游）

來自[上游 style guide](https://github.com/sonic-net/sonic-mgmt/blob/master/docs/tests/styleguide.md)：

- 遵循 PEP8；上游 style guide 寫明 E501 可忽略，但 pre-commit 的 flake8 目前設定為 `--max-line-length=120`，建議控制在 120 字元內。
- 對公開介面使用 Google 風格 docstring。
- 引入 fixture 但看起來沒被使用時，依專案做法加上 `# noqa: F401` 或 `# noqa: F811`（例如 `tests/ipfwd/test_dir_bcast.py` 的做法）。

## 5. 新增測試案例：逐步教學

### 5.1 範例 A：在 sonic-utilities 新增 CLI 單元測試

目標：驗證 `config vlan add` 對非法 VLAN ID 與已存在 VLAN 的錯誤訊息。依據：

- `<BI>/src/sonic-utilities/config/vlan.py` 的 `add_vlan`：VID 不在 1 到 4094 時以 `ctx.fail("Invalid VLAN ID {} (2-4094)")` 結束，VLAN 已存在時以 `"{} already exists."` 結束。
- `utilities_common/cli.py` 的 `is_vlanid_in_range` 以 `1 <= vid <= 4094` 判斷。
- `tests/mock_tables/config_db.json` 內已有 `Vlan1000`。
- 範例沿用 `tests/config_int_mtu_test.py` 的寫法：`Db()` 加 `obj=db`。

步驟：

1. 在 `src/sonic-utilities/tests/` 建立 `config_vlan_validation_test.py`（檔名以 `_test.py` 結尾，與既有慣例一致）。
2. 寫入下列內容：

```python
import pytest
import config.main as config
from click.testing import CliRunner
from utilities_common.db import Db


class TestConfigVlanValidation(object):
    def test_add_vlan_out_of_range(self):
        runner = CliRunner()
        db = Db()

        result = runner.invoke(
            config.config.commands["vlan"].commands["add"], ["4095"], obj=db)

        print(result.exit_code)
        print(result.output)
        assert result.exit_code != 0
        assert "Invalid VLAN ID 4095 (2-4094)" in result.output

    def test_add_vlan_already_exists(self):
        runner = CliRunner()
        db = Db()

        result = runner.invoke(
            config.config.commands["vlan"].commands["add"], ["1000"], obj=db)

        print(result.exit_code)
        print(result.output)
        assert result.exit_code != 0
        assert "Vlan1000 already exists" in result.output

    @pytest.mark.parametrize("vid", ["0", "4096"])
    def test_add_vlan_boundary(self, vid):
        runner = CliRunner()
        db = Db()

        result = runner.invoke(
            config.config.commands["vlan"].commands["add"], [vid], obj=db)

        assert result.exit_code != 0
        assert "Invalid VLAN ID" in result.output
```

3. 若要驗證 `show` 輸出，把預期字串放在測試檔內或 `tests/<name>_input/` 資料夾（如 `pfc_input/assert_show_output.py` 的做法），再比對 `result.output`。
4. 若需要新的 DB 內容，優先在測試內用 `db.cfgdb.set_entry(...)` 暫時寫入（見 `vlan_test.py` 的 `test_show_vlan_brief_explicit_proxy_arp_disable`）；只有多個測試共用時才修改 `mock_tables/*.json`，因為改動會影響所有既有測試的預期輸出。
5. 執行（在 sonic-slave 容器內）：

```bash
cd src/sonic-utilities
python3 -m pytest tests/config_vlan_validation_test.py -o addopts="" -v
```

`-o addopts=""` 會覆蓋 `pytest.ini` 的 `addopts`（含 xdist 與覆蓋率），方便單檔除錯。若要完整流程則用 `python3 setup.py test`。

註：`tests/vlan_test.py` 的 `TestVlan.setup_class` 另外做了 `multi_asic` 的 patch。若你的測試需要多 ASIC 或特定 namespace，請參考該檔與 `multi_asic_vlan_test.py`，不要自行重寫。

### 5.2 範例 B：在 sonic-swss 新增 vstest（控制面）

目標：建立 VLAN，驗證 ASIC_DB 出現對應的 VLAN 物件，再刪除並確認消失。這是 `tests/test_vlan.py::TestVlan::test_VlanAddRemove` 的精簡版，所用的方法都存在於 `tests/dvslib/dvs_vlan.py`。

1. 在 `src/sonic-swss/tests/` 建立 `test_vlan_example.py`。
2. 寫入：

```python
import pytest


@pytest.mark.usefixtures("testlog")
@pytest.mark.usefixtures("dvs_vlan_manager")
class TestVlanExample(object):
    def test_vlan_create_remove(self, dvs):
        vlan = "100"

        # 寫入 CONFIG_DB 的 VLAN 表
        self.dvs_vlan.create_vlan(vlan)

        # 等待 ASIC_DB 出現 1 個非預設 VLAN，並檢查其 VLAN ID
        vlan_oid = self.dvs_vlan.get_and_verify_vlan_ids(1)[0]
        self.dvs_vlan.verify_vlan(vlan_oid, vlan)

        # 刪除後，ASIC_DB 內不應再有非預設 VLAN
        self.dvs_vlan.remove_vlan(vlan)
        self.dvs_vlan.get_and_verify_vlan_ids(0)
```

說明：

- `dvs` 是 module 級 fixture，回傳 `DockerVirtualSwitch`；`dvs_vlan_manager` 是 class 級 fixture，把 `DVSVlan` 掛到 `self.dvs_vlan`。
- `get_and_verify_vlan_ids` 內部用 `wait_for_n_keys` 輪詢，不必自己 `sleep`。
- 需要特殊容器環境時，在模組層級定義 `DVS_ENV = [...]` 或 `NUM_PORTS = N`（`conftest.py` 的 `dvs` fixture 會讀取）。
- 若需要的操作還沒有 dvslib 封裝，優先在 `tests/dvslib/` 新增或擴充輔助類別，並在 `conftest.py` 加對應的 `dvs_xxx_manager` fixture；不要在每個測試重複寫低階 DB 操作。
- 檔名以 `test_` 開頭。若你的測試需要被 CI 單獨分派，請查看 `.azure-pipelines/test-docker-sonic-vs-template.yml` 中測試清單的產生方式。

執行：

```bash
cd src/sonic-swss/tests
sudo pytest -v --imgname=docker-sonic-vs:latest test_vlan_example.py
```

### 5.3 範例 C：在 sonic-swss 新增 gtest 單元測試

1. 在 `src/sonic-swss/tests/mock_tests/` 建立 `example_ut.cpp`，內容參考 `swssnet_ut.cpp`（`#include "ut_helper.h"`，在 namespace 中定義 `::testing::Test` 子類別與 `TEST_F`）：

```cpp
#include "ut_helper.h"
#include "swssnet.h"

namespace example_test
{
    struct ExampleTest : public ::testing::Test
    {
        ExampleTest()
        {
        }
    };

    TEST_F(ExampleTest, IpPrefixToString)
    {
        IpPrefix ip_prefix("10.0.0.1/24");

        ASSERT_EQ("10.0.0.1/24", ip_prefix.to_string());
    }
}
```

2. 把 `example_ut.cpp` 加到 `mock_tests/Makefile.am` 的 `tests_SOURCES`（與其它 `*_ut.cpp` 同一個清單，行尾保留 `\` 續行）。
3. 在 sonic-slave 容器內重新產生 build 並編譯。`tests` 是 automake 的 `noinst_PROGRAMS`，可用 `make check` 執行全部，或直接執行 `tests` 執行檔並以 gtest 篩選：

```bash
./tests --gtest_filter='ExampleTest.*'
```

本機沒有編譯過 sonic-swss 的 `mock_tests`，上述指令請以你的 build 目錄為準。

### 5.4 範例 D：在 sonic-mgmt 新增 pytest 系統測試

目標：在 T0／T1 拓撲上，驗證所有 admin up 的連接埠在一段時間內都會變成 oper up。使用的 API 皆已在上游原始碼確認存在：`duthosts`、`rand_one_dut_hostname`、`pytest_assert`（`tests/common/helpers/assertions.py`）、`wait_until`（`tests/common/utilities.py`）、`SonicHost.show_and_parse`（`tests/common/devices/sonic.py`，輸出為 list of dict，欄位 key 為小寫的欄名，如 `interface`、`oper`、`admin`）。

步驟：

1. 決定位置：依功能放在對應資料夾，新功能就建立新資料夾；可重用的函式放到 `tests/common`。本例假設新建 `tests/example_feature/test_port_oper_status.py`（`example_feature` 為示意名稱）。新資料夾是否需要 `__init__.py`，請參照你 checkout 版本中鄰近資料夾的做法。
2. 宣告 topology marker（必要）。
3. 寫入：

```python
"""
Verify that every admin-up front panel port becomes oper-up.

The test reads 'show interfaces status' on the DUT and waits until all
admin-up Ethernet ports report oper-up.
"""
import logging

import pytest

from tests.common.helpers.assertions import pytest_assert
from tests.common.utilities import wait_until

pytestmark = [
    pytest.mark.topology('t0', 't1'),
]

logger = logging.getLogger(__name__)


def _get_admin_up_ports(duthost):
    """Return a list of parsed rows for admin-up Ethernet ports."""
    rows = duthost.show_and_parse("show interfaces status")
    return [row for row in rows
            if row["interface"].startswith("Ethernet") and row["admin"] == "up"]


def _all_oper_up(duthost):
    """Return True when every admin-up Ethernet port is oper-up."""
    return all(row["oper"] == "up" for row in _get_admin_up_ports(duthost))


def test_admin_up_ports_are_oper_up(duthosts, rand_one_dut_hostname):
    """
    Args:
        duthosts: Fixture holding all DUT objects.
        rand_one_dut_hostname: Fixture returning the hostname of a random DUT.
    """
    duthost = duthosts[rand_one_dut_hostname]

    admin_up_ports = _get_admin_up_ports(duthost)
    pytest_assert(len(admin_up_ports) > 0, "No admin-up Ethernet port found on DUT")
    logger.info("admin-up ports: %s", [row["interface"] for row in admin_up_ports])

    # wait_until(timeout, interval, delay, condition, *args)
    pytest_assert(wait_until(120, 5, 0, _all_oper_up, duthost),
                  "Some admin-up ports did not become oper-up within 120s")
```

說明：

- `duthosts[rand_one_dut_hostname]` 是官方範例常見的取 DUT 方式，例如 `tests/ipfwd/test_dir_bcast.py` 的 `duthost = duthosts[rand_one_dut_hostname]`。
- `wait_until(timeout, interval, delay, condition, *args, **kwargs)` 的簽名來自上游 `tests/common/utilities.py`。
- 需要收發封包時，加入 `ptfadapter` fixture（見第 3.6 節連結）；需要執行較重的 PTF 腳本時，參考 `tests/ipfwd/test_dir_bcast.py`：以 `copy_ptftests_directory` fixture 把 `ptftests` 複製進 PTF 容器，再呼叫 `ptf_runner(ptfhost, 'ptftests', 'dir_bcast_test.BcastTest', '/root/ptftests', params=params, log_file=log_file, is_python3=True)`。
- 需要在測試中修改 DUT 設定時，務必在 fixture 的 `yield` 之後還原；上游的 `sanity_check` 外掛會在測試前後檢查 DUT 狀態，失敗的清理會連累後續測試。
- 會故意觸發錯誤 log 的測試，使用 `@pytest.mark.disable_loganalyzer`，或在測試內設定 loganalyzer 的 ignore／expect 規則（見 loganalyzer README）。

4. 若要讓 PR 的 CI 跑你的測試，需要把測試檔加到 `.azure-pipelines/pr_test_scripts.yaml` 對應拓撲的清單，細節請依 sonic-mgmt 當時的 PR 規範確認。

### 5.5 範例 E：撰寫 PTF 資料面測試的流程

1. 在 sonic-mgmt 的 `tests/ptftests/`（新的測試多放在 `py3` 子目錄，`ptf_runner()` 的 `test_subdir='py3'` 預設值可佐證）新增繼承 `BaseTest` 的類別，使用 `ptf.testutils` 的 `simple_tcp_packet`、`send`、`verify_packet` 等。
2. 在 pytest 側準備參數（例如路由器 MAC 用 `duthost.facts['router_mac']`），寫入 JSON 並以 `ptfhost.copy(...)` 傳到 PTF 容器。
3. 以 `ptf_runner()` 啟動，並將 `log_file` 指向 `/tmp` 下的檔案，便於事後查看。
4. 只在 pytest 內需要少量封包時，用 `ptfadapter` 即可，不必另寫 PTF 腳本（見上游 ptfadapter README）。

可參考的真實案例：上游 `tests/ipfwd/test_dir_bcast.py`（pytest 側）。

## 6. 如何執行與除錯

### 6.1 sonic-mgmt

依[上游執行文件](https://github.com/sonic-net/sonic-mgmt/blob/master/docs/tests/pytest.run.md)：

```bash
# 在 sonic-mgmt 容器內，於 tests 目錄下
./run_tests.sh -n vms-kvm-t0 -d vlab-01 -c example_feature/test_port_oper_status.py \
    -f vtestbed.yaml -i ../ansible/veos_vtb -u
```

- `-n` testbed 名稱、`-d` DUT 主機名稱、`-c` 測試檔或清單、`-f` testbed 檔、`-i` inventory、`-u` 略過 utility（pretest／posttest）測試、`-C` 以 `-k` 運算式篩選案例。
- 指定檔案清單時不需要 `-t`。
- 檔案樣式（如 `"snmp/test_*.py"`）必須用雙引號包住。

直接使用 pytest：

```bash
cd tests
pytest example_feature/test_port_oper_status.py \
    --inventory ../ansible/veos_vtb --host-pattern vlab-01 \
    --testbed vms-kvm-t0 --testbed_file vtestbed.yaml \
    --log-cli-level info --disable_loganalyzer --skip_sanity
```

重點：自訂選項必須放在測試檔路徑**之後**，否則 pytest 預解析時可能找不到子資料夾 `conftest.py` 定義的選項（上游文件有詳細說明）。`--inventory` 要寫 `../ansible/veos_vtb`，因為 group_vars 在 `ansible/` 底下。

除錯建議：

- `--log-cli-level debug` 查看即時日誌。
- 初期開發可加 `--disable_loganalyzer --skip_sanity` 縮短迭代時間，提交前務必拿掉再跑一次。
- 進入 DUT 檢查：`ssh admin@10.250.0.101`（KVM 範例 DUT IP 與帳號見上游 KVM 文件；預設密碼為 `password`）。
- 失敗時看 DUT 的 `/var/log/syslog`，以及 loganalyzer 在報告中列出的比對行。

### 6.2 sonic-swss vstest

```bash
cd src/sonic-swss/tests
sudo pytest -v test_vlan_example.py                 # 使用預設映像
sudo pytest --imgname=docker-sonic-vs:my-changes.333 test_vlan_example.py
sudo pytest --keeptb test_vlan_example.py            # 測試後保留容器供除錯
sudo pytest test_vlan.py::TestVlan::test_VlanAddRemove
```

以上選項皆出自 `<BI>/src/sonic-swss/tests/README.md`。如需持久化容器，使用 `--dvsname=vs`（可再加 `--forcedvs`）；失敗後可 `docker exec -it <容器> bash` 進去看 `redis-cli`、`/var/log/swss`、`/var/log/syslog`。

### 6.3 sonic-utilities

```bash
cd src/sonic-utilities
python3 -m pytest tests/config_vlan_validation_test.py -o addopts="" -v -s
```

`-s` 讓測試中的 `print()` 輸出顯示出來，配合 `result.output` 可快速對照實際訊息。

## 7. 慣例與檢查清單

提交前逐項確認：

- [ ] 選對層級：能用單元測試就不用 testbed 測試。
- [ ] 檔名：sonic-mgmt 與 sonic-swss 為 `test_*.py`；sonic-utilities 為 `*_test.py`；gtest 為 `*_ut.cpp`。
- [ ] sonic-mgmt：已加 `topology` marker，且拓撲宣告與實際需求一致。
- [ ] sonic-mgmt：通用輔助函式放在 `tests/common`，功能專屬輔助檔放在該功能資料夾。
- [ ] sonic-mgmt：使用 `pytest_assert` 並附上說明訊息；等待狀態用 `wait_until`，避免固定 `sleep`。
- [ ] sonic-mgmt：測試會改設定時，有還原機制（fixture 的 `yield` 後清理）。
- [ ] sonic-mgmt：通過 flake8（`--max-line-length=120`）與 pre-commit。
- [ ] sonic-mgmt：公開函式有 Google 風格 docstring。
- [ ] sonic-swss vstest：使用 `dvslib` 輔助類別與輪詢（`wait_for_*`），不寫死等待時間。
- [ ] sonic-swss gtest：新 `*_ut.cpp` 已加入 `Makefile.am`，測試之間不共享未還原的全域狀態（參考 `bulker_ut.cpp` 的 `SetUp`／`TearDown` 做法）。
- [ ] sonic-utilities：預期輸出放在測試檔或 `*_input` 資料夾；修改 `mock_tables/*.json` 前先確認不會破壞既有測試。
- [ ] 本機或 CI 實際跑過至少一次，並在 PR 說明記錄跑過哪些指令與結果。
- [ ] 若新增 C 程式碼的測試輔助，遵守團隊 C 編碼規範（Allman 大括號、4 空格縮排、只用 `/* */` 註解）。

## 8. 搭建測試環境（Testbed）

本章教你從零建出可以跑第 5.4 節範例 D 的環境。分兩條路線：A 為單機 KVM 虛擬 testbed，B 為實體 testbed。依據：[VsSetup](https://github.com/sonic-net/sonic-mgmt/blob/master/docs/testbed/README.testbed.VsSetup.md)、[Setup](https://github.com/sonic-net/sonic-mgmt/blob/master/docs/testbed/README.testbed.Setup.md)、[新 testbed 組態](https://github.com/sonic-net/sonic-mgmt/blob/master/docs/testbed/README.new.testbed.Configuration.md)、[Fanout](https://github.com/sonic-net/sonic-mgmt/blob/master/docs/testbed/README.testbed.Fanout.md)、[Minigraph](https://github.com/sonic-net/sonic-mgmt/blob/master/docs/testbed/README.testbed.Minigraph.md)、[testbed-cli](https://github.com/sonic-net/sonic-mgmt/blob/master/docs/testbed/README.testbed.Cli.md)。上游文件與腳本會演進，以下指令請以你 checkout 的版本為準。

### 8.1 選擇路線的決策表

| 考量 | 路線 A：單機 KVM 虛擬 testbed | 路線 B：實體 testbed |
|---|---|---|
| 目的 | 開發測試案例、驗證 SONiC 軟體行為、CI | 驗證真實 ASIC、光模組、平台、效能與資料面 |
| 需要的硬體 | 一台另購的 Linux 伺服器（Ubuntu 22.04 或 24.04，含 `/dev/kvm`；規格見第 9 章） | 實體 DUT、leaf（與 root）fanout、測試伺服器、管理交換機 |
| 可用拓撲 | `vtestbed.yaml` 內建的 t0、t1-lag、t2 等；PTP 沒有範例 | 全部五類，PTP 只能在實體環境 |
| 成本與時間 | 低，數小時可完成 | 高，需接線、VLAN 設定與 lab 檔維護 |
| 資料面真實度 | 虛擬交換機，不能代表 ASIC 行為 | 真實 |
| 建議 | 初學者與寫新測試案例先走這條 | 要驗證硬體或平台時才走這條 |

兩條路線的伺服器採購規格（CPU、記憶體、磁碟、網卡、軟體版本與驗收指令）統一整理在第 9 章。

### 8.2 路線 A：單機 KVM 虛擬 testbed

#### 8.2.0 附註：本機不是搭建目標

testbed 會在另外採購的伺服器上搭建，不在這台機器（`${HOME}`）。我對本機只做過唯讀檢查：它沒有 `/dev/kvm`（本機是沒開放巢狀虛擬化的 VM）、作業系統為 Ubuntu 20.04（上游要求 22.04 或 24.04），所以路線 A 無法在本機完整執行。本機的 Docker、`sonic-slave-*` 映像仍可用來建置 buildimage 的產物（如 `sonic-vs.img.gz`、`docker-ptf`），再複製到新伺服器。新伺服器的規格與驗收請見第 9 章，以下步驟都假設在那台新伺服器上執行。

#### 8.2.1 前置需求

- Ubuntu AMD64 22.04 或 24.04。
- 跑 T0 至少要有 20 GB 可用記憶體；T1 的鄰居更多，需要更多。
- 若主機本身是 VM，必須開啟巢狀虛擬化。DUT 是 libvirt／QEMU 的 KVM domain，所以一定要有 `/dev/kvm`。預設 DUT 為 6 GiB RAM 與 4 vCPU（見[sonic.xml.j2](https://github.com/sonic-net/sonic-mgmt/blob/master/ansible/roles/vm_set/templates/sonic.xml.j2)）；vEOS 鄰居每台 2 GB（`group_vars/vm_host/main.yml` 的 `memory: 2097152`，單位 KiB）。
- 這些需求的採購規格與驗收指令見第 9 章。檢查：

```bash
ls -l /dev/kvm
egrep -c '(vmx|svm)' /proc/cpuinfo
```

兩者都要有結果（檔案存在、計數大於 0）。

#### 8.2.2 步驟 1：取得 sonic-mgmt、建立管理網路

```bash
sudo apt install -y python3 python3-pip openssh-server
git clone https://github.com/sonic-net/sonic-mgmt
cd sonic-mgmt/ansible
sudo -H ./setup-management-network.sh
```

腳本會建立管理 bridge `br1`（文件中為 `10.250.0.1/24`），並安裝 `j2cli`、`bridge-utils`、`net-tools`、`ethtool` 等套件。我只讀過腳本的前幾步，執行前請先完整讀過；加 `-d` 會刪除既有 bridge。

#### 8.2.3 步驟 2：安裝 Docker

依 Docker 官方文件安裝 Docker CE，並做 post-install 讓一般使用者不用 `sudo` 就能執行 `docker`：

```bash
sudo usermod -aG docker $USER
# 登出再登入後驗證
docker info
```

`setup-container.sh` 需要 `docker info` 能正常執行。

#### 8.2.4 步驟 3：準備映像目錄與鄰居映像

```bash
mkdir -p ~/veos-vm/images ~/sonic-vm/images
```

`~/veos-vm` 來自 `ansible/group_vars/vm_host/main.yml` 的 `root_path`（預設 `veos-vm`），可自行修改。鄰居映像三選一：

- cEOS（推薦，較省記憶體）：自 Arista 網站（需帳號）下載 cEOS64-lab 的 `.tar.xz`，`unxz` 解壓後放進 `~/veos-vm/images`。版本必須與 `ansible/group_vars/all/ceos.yml` 一致：VsSetup 範例為 `cEOS64-lab-4.29.3M`，目前 `ceos.yml` 預設為 `cEOS64-lab-4.32.5M.tar`；上游說明高於測試過的版本可能失敗，最新的 4.35.0F 不能用。也可把映像放在 HTTP 伺服器，設定 `ceos_image_url` 並把 `skip_ceos_image_downloading` 設為 `false`。
- vEOS：把 `Aboot-veos-serial-8.0.0.iso` 與 `vEOS-lab-4.20.15M.vmdk` 放進 `~/veos-vm/images`（檔名定義在 `group_vars/vm_host/veos.yml`）。
- SONiC 鄰居：把 `sonic-vs.img` 放進 `~/veos-vm/images`。

#### 8.2.5 步驟 4：取得 sonic-vs.img（DUT 映像）

方式一，下載上游預建映像：

```bash
wget "https://sonic-build.azurewebsites.net/api/sonic/artifacts?branchName=master&platform=vs&target=target/sonic-vs.img.gz" -O sonic-vs.img.gz
gzip -d sonic-vs.img.gz
cp sonic-vs.img ~/sonic-vm/images
mv sonic-vs.img ~/veos-vm/images
```

方式二，自 `${HOME}/sonic-buildimage` 建置。已驗證的 make target 是 `target/sonic-vs.img.gz`：

- `platform/vs/kvm-image.mk` 定義 `SONIC_KVM_IMAGE = sonic-vs.img.gz`，`_MACHINE = vs`，`_IMAGE_TYPE = kvm`。
- `platform/vs/rules.mk` 把它加進 `SONIC_ALL`。
- CI 的 `.azure-pipelines/azure-pipelines-build.yml` 以 `make $BUILD_OPTIONS target/docker-sonic-vs.gz target/sonic-vs.img.gz` 建置。
- `README.md` 把 `PLATFORM=vs` 列為支援的平台，並說明用 `make configure PLATFORM=...` 設定。

```bash
cd ${HOME}/sonic-buildimage
make init
make configure PLATFORM=vs
make target/sonic-vs.img.gz
```

產物在 `target/sonic-vs.img.gz`，之後同上解壓並複製。目前 `target/` 內沒有這個檔案。我沒有實際執行建置（耗時長且佔用大量磁碟），所以這是依 Makefile、CI 與 README 驗證的指令，不是實跑結果。建置過程需要網路，且首次建置會花數小時。需要多 ASIC 或單台手動 VM 的作法可參考 `platform/vs/README.vsvm.md`。

#### 8.2.6 步驟 5：建立 sonic-mgmt 容器

```bash
cd sonic-mgmt
./setup-container.sh -n <container-name> -d /data
docker exec --user $USER -it <container-name> bash
```

選項：`-n` 容器名稱（必填）、`-i` 映像、`-d` 掛載目錄（腳本預設 `/var/src`，文件使用 `/data`）、`-m` 額外掛載、`-p` 連接埠、`-f`、`-v`、`-x`、`-h`、`--enable-debug`。映像預設使用本機的 `docker-sonic-mgmt`，沒有就從 `sonicdev-microsoft.azurecr.io:443/docker-sonic-mgmt:latest` 拉取（最多重試 3 次）。腳本會產生 `~/.ssh/id_ed25519_docker_sonic_mgmt` 金鑰，在容器內建立同名使用者，並掛載 `/var/run/docker.sock`。容器內 repo 位於 `/data/sonic-mgmt`。

若要自建 `docker-sonic-mgmt`：在 sonic-buildimage 執行 `make configure PLATFORM=generic` 後 `make target/docker-sonic-mgmt.gz`（規則在 `rules/docker-sonic-mgmt.mk`，Dockerfile 範本在 `dockers/docker-sonic-mgmt/Dockerfile.j2`）。

如需 IPv6 測試，編輯 `/etc/docker/daemon.json` 加入 `"ipv6": true`、`"fixed-cidr-v6": "fd00:1::1/64"`、`"experimental": true`、`"ip6tables": true`，然後重啟 docker。

#### 8.2.7 步驟 6：編輯 inventory，讓容器能 SSH 回主機

以下都在容器內的 `/data/sonic-mgmt/ansible` 編輯：

- `veos_vtb`：把 `STR-ACS-VSERV-01` 的 `ansible_user` 改成你的主機使用者，`ansible_host` 為 `172.17.0.1`（`docker0` 位址）。
- `group_vars/vm_host/creds.yml`：設定 `vm_host_user`、`vm_host_password`、`vm_host_become_password`。
- 建立任意內容的 `password.txt`（Ansible Vault 密碼檔）。
- 在主機執行 `sudo visudo`，加入 `<user> ALL=(ALL) NOPASSWD:ALL`。
- 在主機執行 `sudo chmod 755 /home/<user>`。
- 驗證：容器內 `ssh <user>@172.17.0.1` 不用密碼即可登入。若 `docker0` 是 `172.18.0.1`，需改 docker bridge 子網並重建容器。

#### 8.2.8 步驟 7：只在 vEOS 或 SONiC 鄰居時啟動 VM

cEOS 跳過此步。

```bash
./testbed-cli.sh -m veos_vtb -n 4 -k veos start-vms server_1 password.txt
ansible -m ping -i veos_vtb server_1 -u root -k
```

SONiC 鄰居將 `-k veos` 換成 `-k vsonic`。EOS 的 ping 密碼為 `123456`，SONiC 為 `admin`／`password`。

#### 8.2.9 步驟 8：add-topo

```bash
cd /data/sonic-mgmt/ansible
./testbed-cli.sh -t vtestbed.yaml -m veos_vtb add-topo vms-kvm-t0 password.txt
```

預設鄰居是 cEOS；vEOS 加 `-k veos`，SONiC 加 `-k vsonic`。對 vtestbed 而言，`add-topo` 同時會建立 KVM DUT `vlab-01`（若已存在會重建）。第一次執行會從 registry 拉取 `docker-ptf`，需要網路。看到 `cached_topologies_path file content is empty` 的訊息可以忽略。

驗證：

```bash
docker ps      # 應有 ceos_vms6-1_VM0100 等、net_vms6-1_*、ptf_vms6-1
virsh list     # 應有 vlab-01
```

#### 8.2.10 步驟 9：deploy-mg 與登入檢查

```bash
./testbed-cli.sh -t vtestbed.yaml -m veos_vtb deploy-mg vms-kvm-t0 veos_vtb password.txt
```

此步驟需要網路（如需 proxy，設定 `group_vars/all/env.yml`）。完成後：

```bash
ssh admin@10.250.0.101      # 密碼 password
show ip bgp sum              # 應看到 4 個鄰居 ARISTA01T1 至 ARISTA04T1（鄰居 AS 64600，DUT AS 65100）
```

#### 8.2.11 步驟 10：冒煙測試

```bash
cd /data/sonic-mgmt/tests
./run_tests.sh -n vms-kvm-t0 -d vlab-01 -c bgp/test_bgp_fact.py -f vtestbed.yaml -i ../ansible/veos_vtb
```

使用 SONiC 鄰居時加 `-e "--neighbor_type=sonic"`。上游文件預期該檔案會跑出 3 組測試並通過。通過後即可回到第 5.4 節跑自己的測試。

#### 8.2.12 步驟 11：清除

```bash
./testbed-cli.sh -t vtestbed.yaml -m veos_vtb -k ceos remove-topo vms-kvm-t0 password.txt
```

會移除 PTF、cEOS 與 KVM DUT；`-k` 要與建立時一致。其他常用動詞（出自 `testbed-cli.sh` 的 usage）：`start-vms`、`stop-vms`、`redeploy-topo`、`renumber-topo`、`connect-topo`、`gen-mg`、`test-mg`、`announce-routes`、`restart-ptf`、`refresh-dut`、`config-vm`、`install-image`、`cleanup-vmhost`。

#### 8.2.13 常見失敗與排除

| 症狀 | 可能原因與處理 |
|---|---|
| 容器內 SSH 到 `172.17.0.1` 失敗 | docker bridge 子網為 `172.18.x.x`（改子網並重建容器）；`visudo` 沒設；`/home/<user>` 權限不是 755；密碼錯誤 |
| `add-topo` 的 cEOS 啟動失敗 | `ceos.yml` 的版本與 `~/veos-vm/images` 內的檔案不一致；使用了不支援的新版（4.35.0F） |
| 找不到映像 | 映像沒放進 `root_path/images`（預設 `~/veos-vm/images`） |
| KVM DUT 起不來 | 沒有 `/dev/kvm`；主機是 VM 但沒開巢狀虛擬化 |
| IPv6 測試失敗 | `daemon.json` 沒設 IPv6 |
| 拉取 `docker-ptf` 或 deploy-mg 逾時 | 沒網路或需要 proxy（`group_vars/all/env.yml`） |
| 記憶體不足 | T0 至少 20 GB 可用；改用 cEOS 並減少同時存在的 testbed |

### 8.3 路線 B：實體 testbed

#### 8.3.1 硬體清單

| 項目 | 規格 | 說明 |
|---|---|---|
| DUT | 要測的 SONiC 設備 | T0／T1／PTF 常見為 32 埠；PTP 要兩台 |
| Leaf fanout | 埠數多於 DUT，支援 LACP／LLDP 穿透與 802.1Q tunneling | 上游舉例 Arista 7260 |
| Root fanout | 選用，規格同 leaf | 只有一個 testbed 時可省略 |
| 測試伺服器 | Ubuntu 22.04 或 24.04 amd64；參考規格 Dell 730、2×18 核、192 GB RAM、2×500 GB 磁碟、Mellanox MT27700 | 參考規格非強制；規模小的拓撲可大幅降低；規格分級見第 9.3 節 |
| 管理網路 | 一台管理交換機與網段 | 伺服器、VM、PTF、DUT 管理埠、fanout 管理埠都接入 |
| 選配 | Console server、PDU | 對應 `sonic_lab_console_links.csv`、`sonic_lab_pdu_links.csv` |

#### 8.3.2 ASCII 接線圖

```text
              管理網路 (br1, 10.250.0.0/24, 經管理交換機)
   +----------------+---------------+---------------+-----------+
   |                |               |               |           |
[測試伺服器]     [Root fanout]   [Leaf fanout]      [DUT]    [PDU/Console]
 mgmt NIC          mgmt            mgmt           mgmt port     (選用)
 trunk NIC ===== trunk ======= trunk (VLAN 100-131)
 (如 p4p1)       (選用)            |
                                  | DUT 面向埠 = access, 每埠唯一 VLAN
                                  +-- Eth1  VLAN 100 ---- DUT Ethernet0
                                  +-- Eth2  VLAN 101 ---- DUT Ethernet1
                                  +-- ...
                                  +-- Eth32 VLAN 131 ---- DUT Ethernet31
```

若沒有 root fanout，測試伺服器的 trunk NIC 直接接 leaf 的 trunk 埠。連線規則為：DUT 埠接 leaf 的 access 埠（每埠唯一 VLAN），leaf 的 trunk 接 root（或直接接伺服器），root 的 trunk 接伺服器 NIC。PTP 拓撲則是兩台 DUT 的資料埠彼此直連，不經 fanout。

#### 8.3.3 步驟 1：測試伺服器基礎環境

```bash
sudo apt install -y python3 python-is-python3 python3-pip curl git openssh-server make
sudo pip3 install j2cli
sudo usermod -aG docker $USER      # 先安裝 Docker CE，再登出登入
```

可選擇關閉防火牆：`sudo ufw disable`。

#### 8.3.4 步驟 2：準備映像

- cEOS（實體環境推薦，記憶體與核心互動較少）：先 `docker pull debian:jessie`（必要時把 `debian:latest` tag 成 `jessie`），取得 cEOS tar 後 `docker import <tar> ceosimage:<version>`。版本需與 `ansible/group_vars/all/ceos.yml` 一致（`ceos_image_filename`、`ceos_image_orig`、`ceos_image`），本地已有映像時設 `skip_ceos_image_downloading: true`。
- `docker-ptf`：`add-topo` 會自動從預設 registry `sonicdev-microsoft.azurecr.io:443` 拉取。也可在 sonic-buildimage 自建：`make configure PLATFORM=vs` 然後 `make target/docker-ptf.gz`；使用自建 registry 時修改 `ansible/vars/docker_registry.yml`。
- `docker-sonic-mgmt`：下載預建映像（建議），或 `make configure PLATFORM=generic` 加 `make target/docker-sonic-mgmt.gz`，然後依第 8.2.6 節執行 `setup-container.sh`。

#### 8.3.5 步驟 3：管理網路與 OVS（測試伺服器網路）

管理 bridge `br1` 承載伺服器、VM、PTF 與 DUT 的管理流量。設定方式二擇一：

- 在 netplan（或 ifupdown）將管理網卡橋接到 `br1`，範例位址為 `10.250.0.245/24`，gateway `10.250.0.1`，範例使用的網卡名為 `ma0`。
- 暫時性做法：`sudo -H ./ansible/setup-management-network.sh`（重開機後消失）。

因為 `br1` 持有 IP，建議伺服器另有第二張網卡連到實驗室網路。設定後重開機確認網路仍正常。

資料面的 OVS bridge 不用手動建立：`add-topo` 會依 `topo_*.yml` 與 `external_port`（trunk 網卡，如 `p4p1`）建立 VLAN 子介面並接上 OVS bridge、設定第 3.1 節所述的 flow 規則。所以你要做的是：在 `ansible/host_vars/STR-ACS-SERV-01.yml` 把 `external_port` 設為接 fanout trunk 的網卡，並確認伺服器上有 Open vSwitch（`ovs-vsctl`）。我沒有逐一確認哪個 role 會自動安裝 OVS，若 `add-topo` 回報缺少，請依 [Setup 文件](https://github.com/sonic-net/sonic-mgmt/blob/master/docs/testbed/README.testbed.Setup.md)補裝。

#### 8.3.6 步驟 4：設定 fanout 的 VLAN trunk

- Arista leaf：使用 `ansible/roles/fanout` playbook。先把 `roles/fanout/template/rc.eos` 以 `scp` 傳到交換機 flash，並備妥 eosadmin shell 的認證（`fanout_admin_user`、`fanout_admin_password`）。
- 其他廠商：手動設定。每個 DUT 面向埠設為 access，並指定唯一 VLAN；上行埠設為 trunk，允許所有這些 VLAN。VLAN 範圍為「最小 VLAN + 埠數 - 1」，例如 32 埠從 100 開始就是 100 至 131。或者直接部署一份一般的 L2 交換機設定。
- 注意：`testbed-cli.sh add-topo` 的最後一步會嘗試在 root fanout 重新部署 VLAN 範圍，而且只支援 Arista。使用其他廠商時，請把這一步註解掉並手動設定 VLAN。

#### 8.3.7 步驟 5：inventory、testbed.yaml、lab 檔與認證

在 sonic-mgmt 容器內編輯下列檔案（欄位說明見[組態文件](https://github.com/sonic-net/sonic-mgmt/blob/master/docs/testbed/README.new.testbed.Configuration.md)）：

| 檔案 | 內容 |
|---|---|
| `ansible/veos` | 伺服器管理 IP、VM 管理 IP（在管理網段內）、`vms_1` VM 清單與 ID |
| `ansible/group_vars/vm_host/creds.yml` | `vm_host_user`、`vm_host_password`、`vm_host_become_password` |
| `ansible/host_vars/STR-ACS-SERV-01.yml` | `external_port`（trunk 網卡）、`mgmt_gw`、`mgmt_prefixlen`、`mgmt_bridge` |
| `ansible/group_vars/eos/creds.yml` | `ansible_user: admin`、`ansible_password` |
| `ansible/group_vars/all/ceos.yml` | cEOS 映像檔名與名稱 |
| `ansible/lab` | DUT 群組（如 `sonic_sn2700_40`，含 `hwsku`、`iface_speed`、`ansible_host`、`model`、`serial`、`base_mac`、`syseeprom_info`）、fanout、pdu、server、`ptf` 群組 |
| `ansible/files/sonic_lab_devices.csv`、`sonic_lab_links.csv` | 設備清單與連線（DUT 埠到 fanout 埠與其 Access VLAN；fanout trunk 到伺服器網卡與 Trunk VLAN 範圍） |
| `sonic_lab_pdu_links.csv`、`sonic_lab_console_links.csv`、`sonic_lab_bmc_links.csv` | 選用的 PDU、console、BMC 連線 |
| `ansible/testbed.yaml` | testbed 項目（第 3.3 節欄位） |
| `ansible/password.txt` | Ansible Vault 密碼檔 |

要點：

- `ansible/lab` 的 `ptf` 群組 `ansible_host` 必須等於 testbed 項目的 `ptf_ip`；PTF 帳密為 root／root。
- 每個 DUT 要有 `iface_speed`，否則 `iface_namingmode` 測試會失敗。
- `hwsku` 必須存在於 `ansible/module_utils/port_utils.py`，連接埠速率需有對應的 `port_config.ini`。
- 連線檔填完後，用 `creategraph.py` 產生 `lab_connection_graph.xml`。
- 也可改用單一檔案 `testbed.yaml` 加 `TestbedProcessing.py` 的方式（組態文件列有 device_groups、devices、host_vars、veos_groups、veos、testbed、topology、docker_registry 等區段）。
- 驗證伺服器可被 Ansible 控制：`ansible -m ping -i veos vm_host_1`。

#### 8.3.8 步驟 6：部署拓撲與 minigraph

```bash
./testbed-cli.sh -t testbed.yaml -m veos -k ceos add-topo <conf-name> password.txt
docker ps      # 檢查 ceos_<group>_VM0100..、net_<group>_VM0100..、ptf_<group>
docker exec -it ceos_vms1-1_VM0101 Cli     # 進 cEOS，執行 show version
./testbed-cli.sh -t testbed.yaml -m veos deploy-mg <conf-name> lab password.txt
```

`start-vms`、`start-topo-vms` 只有使用 vEOS／vSONiC 才需要。`deploy-mg` 會讀取 testbed、`vars/topo_<topo>.yml` 與 veos VM 資訊，渲染 minigraph 並部署到 DUT、載入並儲存 config_db。

#### 8.3.9 步驟 7：連接真實 DUT 與驗證

1. 確認 DUT 已安裝 SONiC 映像，管理 IP、帳密與 `ansible/lab` 的設定一致，可由 sonic-mgmt 容器 SSH。
2. 確認 DUT 各埠 link up（`show interfaces status`），且 fanout 對應埠為正確的 access VLAN。
3. 檢查鄰居 BGP：

```bash
show ip bgp sum      # t0 應有 4 個鄰居 ARISTA01T1 至 04T1
```

4. 跑冒煙測試：

```bash
cd tests
./run_tests.sh -n <conf-name> -d <dut> -c bgp/test_bgp_fact.py -f testbed.yaml -i ../ansible/lab
```

5. 通過後再跑第 5.4 節的範例。
6. 清除：`./testbed-cli.sh -t testbed.yaml -m veos -k ceos remove-topo <conf-name> password.txt`。

本章尚未涵蓋或無法確認之處：PTP 沒有上游的虛擬或部署範例；單一 cEOS 容器的記憶體用量在我取得的檔案中找不到；T2 的伺服器規格上游未明列（採購相關的限制見第 9.8 節）。

## 9. 伺服器環境需求與採購指南

本章回答「要買什麼樣的伺服器來跑 sonic-mgmt testbed」。testbed 不會在 `${HOME}` 這台機器搭建，而是在另外採購的伺服器上執行第 8 章的步驟。

### 9.1 先讀這一節：數字的可信度標示

上游文件對伺服器規格給的數字很少，大部分規格要自己推算。為了避免把估計當成事實，本章每個數字都標示來源：

| 標示 | 意思 |
|---|---|
| 【上游】 | 直接出自 sonic-mgmt 上游文件或原始碼，附連結 |
| 【推導】 | 用上游數字（例如 VM 數量、每台 VM 的記憶體）做的算術，算式會寫出來 |
| 【估計】 | 作者的經驗估計，上游沒有對應數字，採購前請自行驗證 |

本章引用的上游來源：

- [README.testbed.VsSetup.md](https://github.com/sonic-net/sonic-mgmt/blob/master/docs/testbed/README.testbed.VsSetup.md)：KVM 虛擬 testbed 的主機需求（20 GB、巢狀虛擬化、Ubuntu 22.04／24.04）。
- [README.testbed.Setup.md](https://github.com/sonic-net/sonic-mgmt/blob/master/docs/testbed/README.testbed.Setup.md)：實體 testbed 的伺服器準備，建議 cEOS。
- [README.testbed.Overview.md](https://github.com/sonic-net/sonic-mgmt/blob/master/docs/testbed/README.testbed.Overview.md)：硬體需求（參考伺服器規格、fanout、實體連線）與五類拓撲。
- [README.testbed.FAQ.md](https://github.com/sonic-net/sonic-mgmt/blob/master/docs/testbed/README.testbed.FAQ.md)：最低硬體需求（92 GB 記憶體，文件較舊）。
- [ansible/group_vars/vm_host/main.yml](https://github.com/sonic-net/sonic-mgmt/blob/master/ansible/group_vars/vm_host/main.yml)：vEOS 每台 `memory: 2097152`（KiB，即 2 GiB）。
- [ansible/roles/vm_set/templates/arista.xml.j2](https://github.com/sonic-net/sonic-mgmt/blob/master/ansible/roles/vm_set/templates/arista.xml.j2)：vEOS 每台 `vcpu` 為 1。
- [ansible/roles/vm_set/templates/sonic.xml.j2](https://github.com/sonic-net/sonic-mgmt/blob/master/ansible/roles/vm_set/templates/sonic.xml.j2)：KVM DUT 預設 6 GiB、4 vCPU，其他 SKU 見 9.2.3。
- [ansible/roles/vm_set/tasks/host_setup.yml](https://github.com/sonic-net/sonic-mgmt/blob/master/ansible/roles/vm_set/tasks/host_setup.yml)：主機會安裝的套件（`openvswitch-switch`、`libvirt-*`、`qemu-system-x86` 等）。
- [ansible/roles/vm_set/tasks/docker.yml](https://github.com/sonic-net/sonic-mgmt/blob/master/ansible/roles/vm_set/tasks/docker.yml)：Docker CE 的 apt 來源（jammy 與 noble）。
- [ansible/vtestbed.yaml](https://github.com/sonic-net/sonic-mgmt/blob/master/ansible/vtestbed.yaml)：虛擬 testbed 項目與 IP。

### 9.2 單一台伺服器要扛多少資源

#### 9.2.1 先決定三件事

1. 路線：A（單機 KVM 虛擬 testbed）或 B（實體 testbed，見 8.1）。兩條路線都需要一台 Linux 伺服器，B 另外需要 DUT 與 fanout。
2. 要跑哪一類拓撲：鄰居 VM 數量決定記憶體與 CPU（見 9.2.2，數字與 3.2 節的拓撲總表一致）。
3. 鄰居種類：vEOS、cEOS 或 SONiC VM（見 9.2.3）。

#### 9.2.2 各拓撲的資源估算（以 vEOS 鄰居為上限）

算式：VM 記憶體 = 鄰居 VM 數 × 2 GiB（【上游】每台 vEOS `memory: 2097152` KiB）+ DUT 6 GiB（【上游】預設 KVM DUT）；vCPU = 鄰居 VM 數 × 1（【上游】vEOS `vcpu` 為 1）+ 4（【上游】預設 KVM DUT 4 vCPU）。這是 VM 本身的分配量，不含主機 OS、PTF 與 Docker 開銷（那部分為【估計】約 4 至 8 GiB）。

鄰居 VM 數量取自 3.2 節的拓撲總表與變體清單，未重新統計。

| 拓撲 | 鄰居 VM 數（見 3.2） | VM 記憶體【推導】 | vCPU 配置量【推導】 | 說明 |
|---|---|---|---|---|
| 單一 DUT、不需鄰居（`ptf32`、`ptf64`、`ptf8`） | 0 | 6 GiB（只有 DUT） | 4 | 我沒有確認 vtestbed 是否有現成的 PTF 項目，請以你 checkout 的 `vtestbed.yaml` 為準 |
| T0（`t0`） | 4 | 4×2+6 = 14 GiB | 4+4 = 8 | 上游 VsSetup 要求至少 20 GB 可用記憶體【上游】，與此推導相容（含主機與 PTF 開銷） |
| T0（`t0-56`、`t0-80`） | 8 | 8×2+6 = 22 GiB | 8+4 = 12 | 超過 20 GB，以此為準 |
| T1（`t1-lag`） | 24 | 24×2+6 = 54 GiB | 24+4 = 28 | |
| T1（`t1`） | 32 | 32×2+6 = 70 GiB | 32+4 = 36 | 上游沒有 T1 專屬的伺服器規格 |
| T1（`t1-64`） | 64 | 64×2+6 = 134 GiB | 64+4 = 68 | 單機以 vEOS 跑會很吃重，建議改 cEOS 或分散到多台 |
| T2（`t2`，機箱） | 72 | 72×2 = 144 GiB，再加 3 張 linecard 與 1 個 supervisor 的 DUT 記憶體 | 72 加 DUT | 4 個 DUT 節點在 vtestbed 的各自配置我沒有逐一驗證；DUT 部分【估計】再加 24 至 32 GiB，合計約 170 GiB 以上；上游未給 T2 伺服器規格 |
| T2 單節點（`t2_single_node_min`） | 5 | 5×2+6 = 16 GiB | 5+4 = 9 | 5 台 VM 取自 3.2 節；適合虛擬環境練習 |
| PTP | 0 | 不適用 | 不適用 | 兩台實體 DUT 互連，沒有虛擬化負載；伺服器只需管理網路（見 8.3.2） |

注意：

- 上表以 vEOS 為上限。cEOS 用容器，上游只說它比較省記憶體並減少與核心的互動（【上游】Setup 文件的說法），但**我在上游檔案中找不到每個 cEOS 容器的記憶體數字**，因此用 cEOS 能省多少是未知數。規劃時請先用 vEOS 的數字當保守上限，實測後再調整。
- vCPU 是配置量，不是實體核心需求。KVM 允許超配（overcommit），所以實體核心可以少於 vCPU 合計；但 BGP 收斂與路由載入時 VM 會同時吃 CPU，超配過高會讓測試逾時。核心數的建議見 9.3（【估計】）。

#### 9.2.3 鄰居與 DUT 種類的差異

| 種類 | CPU／記憶體（上游數字） | 資源相對大小 | 備註 |
|---|---|---|---|
| vEOS（KVM VM） | 每台 1 vCPU、2 GiB【上游】 | 大 | 需要 `/dev/kvm`；映像為 `Aboot-veos-serial-8.0.0.iso` 加 `vEOS-lab-4.20.15M.vmdk` |
| cEOS（Docker 容器） | 上游沒有數字 | 小於 vEOS（【上游】只有定性描述） | VsSetup 與 Setup 都建議使用（Setup 針對實體環境）；版本要與 `ceos.yml` 一致，最新的 4.35.0F 不能用 |
| SONiC VM 鄰居（`-k vsonic`） | 我沒有確認鄰居 SONiC VM 的配置；若與 DUT 的預設相同（6 GiB、4 vCPU，【上游】`sonic.xml.j2`），32 台就是 192 GiB（【推導】32×6） | 大（待驗證） | 這個假設未驗證；規劃時先假設比 vEOS 大，請以小拓撲為主，並在實測後修正 |
| KVM DUT 預設 | 6 GiB、4 vCPU【上游】 | | `sonic.xml.j2` 的其他 SKU：`msft_four_asic_vs` 8 GB／10 vCPU、`msft_multi_asic_vs` 8 GB／16 vCPU、`cisco-8101-p4` 8 GB／6 vCPU、`Nokia-IXR7250E-36x400G` 12 GB／5 vCPU、SUP-10 4 GB／3 vCPU |

多 ASIC 與機箱類的虛擬 DUT 會把 vCPU 與記憶體需求拉高。例如 `sonic.xml.j2` 中的 `msft_four_asic_vs` 一台 DUT 就要 8 GB／10 vCPU【上游】；vtestbed 的 `vms-kvm-four-asic-t1-lag` 是四 ASIC 項目，但它實際套用哪個 SKU 我沒有逐一驗證。

#### 9.2.4 同時跑多組 testbed

同一台伺服器上的每一組 testbed 各自有 `group-name`、PTF、鄰居 VM 與 DUT，資源需求是累加的，不共用：

- 總記憶體 ≈ 各組 VM 記憶體之和 + 主機開銷一次（【推導】）。例如 2 組 `t0` ≈ 2×14 = 28 GiB，加主機開銷。
- 每組的 `vm_base` 範圍不可重疊（上游 Consistency Rule，見 3.3 節），所以規劃 VM 名稱與管理 IP 時要預留空間。
- 實體環境中，一台測試伺服器可服務多個 testbed（每組 DUT 各用一段 VLAN，例如 100 至 131 與 132 至 163，見 3.1 節）。上游的參考伺服器（192 GB、2×18 核）設計目的就是這種共用情境，不過上游並未說明它同時服務幾組。

### 9.3 規格分級（最低與建議）

來源標示規則見 9.1。「最低」表示能跑起來的底線，「建議」表示留有餘裕、較不容易因資源不足而出現不穩定。

#### 9.3.1 路線 A：虛擬 KVM testbed（單一伺服器）

| 項目 | 最低 | 建議 | 依據 |
|---|---|---|---|
| 情境 | 單一 DUT，一組 `t0`（cEOS 或 vEOS 皆可） | 一組 `t1-lag`／`t1`，或同時 2 至 3 組 `t0` | |
| CPU 虛擬化 | 必須有 VT-x 或 AMD-V，且 `/dev/kvm` 可用 | 同左，建議直接用實體機（bare metal） | 【上游】VsSetup：主機若為 VM 必須支援巢狀虛擬化；DUT 是 libvirt／QEMU KVM domain（`sonic.xml.j2`） |
| 實體核心 | 8 核 | 16 核（`t0` 多組）；24 至 36 核（`t1`） | 【估計】；依據是 9.2.2 的 vCPU 配置量（`t0` 8、`t1` 36）與可超配的前提 |
| 記憶體 | 24 GB 以上（可用 20 GB 為上游底線） | `t0` 多組：64 GB；`t1-lag`／`t1`：128 GB | 20 GB 為【上游】VsSetup；`t1` 70 GiB 為【推導】；其餘為【估計】 |
| 磁碟 | 200 GB SSD | 500 GB NVMe SSD | 【估計】：上游沒有磁碟容量數字。依據：cEOS tar、`sonic-vs.img`、`docker-ptf`、`docker-sonic-mgmt` 與每台 VM 的磁碟映像；映像大小請以實際下載檔為準 |
| 網卡 | 1 張，連到管理網路 | 2 張（一張給 `br1` 管理，一張連實驗室網路） | 【上游】Setup 建議第二張網卡，因 `br1` 持有 IP；速率上游未規定，1 GbE 即可（【估計】） |
| 作業系統 | Ubuntu 22.04 或 24.04 amd64 | 同左（22.04 與 24.04 擇一，團隊內統一） | 【上游】VsSetup 與 Setup |

#### 9.3.2 路線 B：實體 testbed 的測試伺服器

| 項目 | 最低 | 建議 | 依據 |
|---|---|---|---|
| 情境 | 一組 DUT，`t0` 級以下，使用 cEOS | 多組 DUT 共用，或 `t1` 以上 | |
| CPU／記憶體 | 記憶體 92 GB（FAQ 的底線，文件較舊） | 上游參考規格：2 顆 CPU 各 18 核（共 36 核）、192 GB | 【上游】Overview（參考規格，明確寫「非強制」）與 FAQ |
| 磁碟 | 500 GB | 2×500 GB（上游參考規格） | 參考規格為【上游】Overview；最低值為【估計】 |
| 網卡 | 一張管理網卡，加一張 trunk 網卡（接 fanout） | 管理與 trunk 分開；trunk 網卡上游參考為 Mellanox MT27700（ConnectX-4 系列） | 網卡型號為【上游】Overview；速率上游沒有規定。trunk 要承載所有 DUT 埠的測試流量，速率至少不低於單一 DUT 埠的速率，建議 25 GbE 以上（【估計】） |
| 其他 | 一台管理交換機與 L2 fanout | 見 8.3.1 硬體清單 | 【上游】Overview 與 FAQ |

兩個數字（20 GB 與 92 GB）看起來差很多，原因是它們的情境不同：20 GB 是虛擬 `t0` 一組的底線（VsSetup），92 GB 是 FAQ 針對實體環境的通用最低需求（文件較舊，沒有說明它是針對哪個拓撲）。採購時請依你要跑的最大拓撲與同時組數來選，而不是取兩者之一。

#### 9.3.3 一眼看懂的採購建議（作者的建議，屬【估計】）

| 你的目標 | 建議採購 | 理由 |
|---|---|---|
| 學習、寫新測試案例（本文主要目標） | 路線 A，16 核、64 GB、500 GB NVMe、Ubuntu 22.04／24.04、bare metal | 夠跑 2 至 3 組 `t0` 或單組 `t1-lag`（54 GiB 加開銷），且不用擔心巢狀虛擬化 |
| 團隊共用、同時多組 `t1` | 路線 A 或 B，2 顆 18 核、192 GB 以上，2×500 GB 以上 SSD | 對齊上游參考規格 |
| 驗證真實硬體（光模組、平台） | 路線 B：上游參考伺服器等級，加 fanout 與管理交換機 | 見 8.3 |
| T2 機箱 | 另行評估，可能需要多台伺服器 | 上游沒有 T2 伺服器規格；72 台 VM 單機需 144 GiB 以上 |

### 9.4 作業系統與軟體版本

| 項目 | 要求 | 依據與備註 |
|---|---|---|
| 作業系統 | Ubuntu 22.04 或 24.04，amd64 | 【上游】VsSetup、Setup。20.04 不符合 |
| 核心 | 上游沒有指定版本；使用該 Ubuntu 版本的預設核心，並確認有 `kvm`、`kvm_intel` 或 `kvm_amd` 模組 | 【估計】；已知 sonic-swss 的 vstest 需要 `team` kernel module（見 2.4 節），若伺服器也要跑 vstest 請確認 `modinfo team` 有結果 |
| Docker | Docker CE（`docker-ce`、`docker-ce-cli`、`containerd.io`），一般使用者在 `docker` 群組內 | 【上游】Setup 與 `docker.yml`（apt 來源 jammy／noble）。上游沒有固定 Docker 版本，用官方 repo 的穩定版 |
| Open vSwitch | `openvswitch-switch` | 【上游】`host_setup.yml` 會安裝；VM 與 PTF 的資料面 bridge 都是 OVS（見 3.1 節） |
| libvirt／QEMU | `libvirt-clients`、`libvirt-daemon-system`、`python3-libvirt`、`python3-lxml`、`qemu-system-x86` | 【上游】`host_setup.yml`；KVM DUT 與 vEOS 需要 |
| 網路工具 | `net-tools`、`bridge-utils`、`ethtool` | 【上游】`host_setup.yml` 與 `setup-management-network.sh` |
| Python | 系統的 `python3`、`python3-pip`；`j2cli`（`setup-management-network.sh` 會裝 `j2cli==0.3.10`，Setup 文件用 `pip3 install j2cli`） | 【上游】VsSetup、Setup。`vm_set` role 另外會建立一個 venv（`ensure_python_env.yml`，使用 `uv`），其中的套件版本以 checkout 為準，這部分我只確認有 venv 這件事 |
| sysctl | `net.bridge.bridge-nf-call-*` 由 `host_setup.yml` 設定 | 【上游】`host_setup.yml` |
| Hugepages | 我在上游的 VsSetup、Setup 與上述 role 檔案中沒有找到 hugepages 的要求 | 【估計】：不需要事先設定；若 DUT 或平台映像另有要求，以該映像文件為準 |
| 防火牆 | 可選擇關閉：`sudo ufw disable` | 【上游】Setup（選用） |
| SSH 與 sudo | 開啟 `openssh-server`；容器要能免密碼 SSH 回主機，並設定 `NOPASSWD:ALL` | 【上游】VsSetup（見 8.2.7） |
| cEOS 映像 | 版本要與 `ansible/group_vars/all/ceos.yml` 一致；需要 Arista 帳號下載 | 【上游】VsSetup；`debian:jessie` 為 cEOS 實體流程的前置映像（Setup） |

### 9.5 網路需求

#### 9.5.1 管理網路與 IP 規劃

| 項目 | 虛擬 KVM（路線 A） | 實體（路線 B） |
|---|---|---|
| 管理 bridge | `br1`，由 `setup-management-network.sh` 建立，位址 `10.250.0.1/24`（【上游】VsSetup） | `br1`，管理網卡橋接進去，範例 `10.250.0.245/24`、gateway `10.250.0.1`（【上游】Setup） |
| DUT 管理 IP | 例如 `vlab-01` 為 `10.250.0.101`（【上游】`vtestbed.yaml`） | 依 `ansible/lab` 的 `ansible_host` |
| PTF 管理 IP | 例如 `vms-kvm-t0` 為 `10.250.0.102/24`（【上游】`vtestbed.yaml`） | 依 `testbed.yaml` 的 `ptf_ip`，且要等於 `lab` 的 `ptf` 群組 `ansible_host` |
| 鄰居 VM 管理 IP | 由 `veos_vtb` 定義 | 由 `ansible/veos` 定義，須在管理網段內（【上游】Setup） |
| 容器回連主機 | `172.17.0.1`（`docker0`）；若 `docker0` 是 `172.18.0.1` 要改 docker bridge 子網（【上游】VsSetup） | 同左，視 inventory 而定 |

規劃要點：

- 預留的位址：`10.250.0.0/24` 是上游範例，不是強制值，但改了要同步改 `veos_vtb`、`vtestbed.yaml`、`ansible/lab` 等檔案。若實驗室內已有網段與它重疊，請先換掉，並避免與 Docker 預設的 `172.17.0.0/16`（也可能用到 `172.18.0.0/16`）衝突（【估計】的建議，依據 8.2.7 的 docker0 子網問題）。
- 管理網路上要容納：伺服器、每台鄰居 VM、PTF、DUT，以及（實體）fanout 與 PDU／console。一組 `t1` 就有 32 台鄰居，IP 位址要夠用。
- IPv6：若要跑 IPv6 測試，需要在 `/etc/docker/daemon.json` 啟用（見 8.2.6）；IPv6-only 管理網路為選配（VsSetup）。

#### 9.5.2 資料面網路（只有實體 testbed 需要）

| 項目 | 需求 | 依據 |
|---|---|---|
| Trunk 網卡 | 測試伺服器的一張網卡接 root fanout（或無 root 時直接接 leaf）的 802.1Q trunk；在 `host_vars/STR-ACS-SERV-01.yml` 設為 `external_port` | 【上游】Overview、Setup |
| VLAN | 每個 DUT 埠一個唯一 VLAN，範圍 = 最小 VLAN + 埠數 − 1，例如 32 埠為 100 至 131 | 【上游】Overview、組態文件 |
| 伺服器端 bridge | OVS bridge 由 `add-topo` 建立 | 【上游】Overview（見 8.3.5） |
| 管理網卡與 trunk 網卡 | 分開兩張，不要共用 | 【估計】的建議：`br1` 是管理，trunk 要整張網卡交給 VLAN 子介面 |
| 第二張網卡 | 連到實驗室網路，因為 `br1` 持有管理 IP | 【上游】Setup |

虛擬 KVM（路線 A）完全在主機內部用 OVS 與 bridge 連接，不需要 trunk 網卡與 fanout。

#### 9.5.3 網際網路、registry 與離線鏡像

| 需要連到的資源 | 用途 | 依據 |
|---|---|---|
| Ubuntu 的 apt 來源 | 安裝套件與 `setup-management-network.sh` | 【上游】VsSetup |
| `download.docker.com` | Docker CE 的 apt 來源 | 【上游】`docker.yml`、Setup |
| `sonicdev-microsoft.azurecr.io:443` | 拉取 `docker-ptf`、`docker-sonic-mgmt` | 【上游】VsSetup、Setup、`setup-container.sh` |
| `sonic-build.azurewebsites.net` | 下載預建 `sonic-vs.img.gz` | 【上游】VsSetup |
| Arista 下載網站（需帳號） | cEOS 或 vEOS 映像 | 【上游】VsSetup |
| GitHub | `git clone sonic-mgmt` | 【上游】VsSetup |
| 外部網際網路（`deploy-mg` 期間） | VsSetup 註明 `deploy-mg` 需要網路，需要連到哪些站點上游沒有列出 | 【上游】VsSetup |
| Docker Hub（`debian:jessie`） | cEOS 實體流程的前置映像 | 【上游】Setup |

離線或受限網路的做法：

- 自建 Docker registry（`registry:latest`），並修改 `ansible/vars/docker_registry.yml`（【上游】Setup）。
- cEOS 映像放在內部 HTTP 伺服器，設定 `ceos_image_url` 並把 `skip_ceos_image_downloading` 設為 `false`，或預先放進 `images/` 並設為 `true`（【上游】VsSetup、Setup）。
- 需要 proxy 時設定 `ansible/group_vars/all/env.yml`（【上游】VsSetup）。
- apt、pip 的內部鏡像：上游沒有提供，這是常見的企業作法（【估計】的建議）。

磁碟上要留空間放：cEOS tar、vEOS 的 `.iso` 與 `.vmdk`、`sonic-vs.img`、`docker-ptf`、`docker-sonic-mgmt` 與多版本映像。`root_path`（預設 `~/veos-vm`）與 Docker 的資料目錄（預設 `/var/lib/docker`）最好放在容量最大的磁碟，必要時改 `root_path` 或搬動 Docker data-root（【估計】的建議）。

### 9.6 採購檢查清單

採購前請對供應商確認下列各項；勾選代表已確認。

- [ ] CPU 支援虛擬化：Intel VT-x 或 AMD-V，且 BIOS 預設已啟用（不是出貨後才發現沒開）。
- [ ] 建議買實體機（bare metal）。若是雲端或虛擬化平台的主機，確認平台支援巢狀虛擬化並提供 `/dev/kvm`（【上游】VsSetup 要求）。
- [ ] 核心數與記憶體符合 9.3 的等級（路線 A 建議 16 核／64 GB 起；多組 `t1` 以上看 192 GB 等級）。
- [ ] 磁碟為 SSD 或 NVMe，容量依 9.3 的估計；若可選，預留擴充槽。
- [ ] 網卡數量：至少一張管理網卡；實體 testbed 另加 trunk 網卡（參考 Mellanox MT27700）與連實驗室網路的第二張網卡。
- [ ] 交付的作業系統為 Ubuntu 22.04 或 24.04 amd64（或允許你自行重灌）。
- [ ] 取得 root 或 sudo 權限、可 SSH 登入。
- [ ] 網路：管理網段與 IP 範圍已規劃，不與 `10.250.0.0/24`、`172.17.0.0/16`、`172.18.0.0/16` 衝突（或你已決定要改）。
- [ ] 能連到第 9.5.3 節列出的 registry 與下載站，或已備妥離線鏡像與 proxy。
- [ ] 取得 Arista 帳號可下載 cEOS／vEOS，且已確認要用的版本與 `ceos.yml` 一致。
- [ ] 實體 testbed：fanout 交換機的埠數多於 DUT、支援 LACP／LLDP 穿透與 QinQ（上游舉例 Arista 7260）；有管理交換機。
- [ ] 機房條件：機架空間、電力、散熱、網路孔位已確認。
- [ ] 保固與到貨驗收時程已確認，便於發現規格不符時退換。

### 9.7 到貨後的驗收指令（皆為唯讀）

以下指令只讀取資訊，不會修改系統。請在**新採購的伺服器**上執行（不是 `${HOME}` 這台）。`kvm-ok` 需要 `cpu-checker` 套件，若沒有安裝可略過，用其他指令取代。網路連線檢查會對外送出 HTTP 請求，但不含任何專案資料。

```bash
# 1. 作業系統與核心
cat /etc/os-release
uname -r
systemd-detect-virt            # none 代表實體機；kvm、vmware 等代表本身是 VM

# 2. CPU 與虛擬化旗標
lscpu | egrep 'Model name|^CPU\(s\)|Thread|Core|Socket|Virtualization'
egrep -c '(vmx|svm)' /proc/cpuinfo     # 大於 0 才行
ls -l /dev/kvm                          # 必須存在
kvm-ok                                  # 需 cpu-checker，輸出 KVM acceleration can be used
cat /sys/module/kvm_intel/parameters/nested 2>/dev/null   # Intel 且本機是 VM 時看巢狀虛擬化是否為 Y 或 1
cat /sys/module/kvm_amd/parameters/nested 2>/dev/null     # AMD 同上

# 3. 記憶體與磁碟
free -h
swapon --show
df -h
lsblk -d -o NAME,SIZE,ROTA,MODEL        # ROTA 為 0 表示 SSD 或 NVMe

# 4. 網卡與網路
ip -br link
ip -br addr
lspci | grep -i ethernet
ethtool <nic> | egrep 'Speed|Link detected'     # <nic> 換成實際網卡名稱
ip route

# 5. 軟體版本（尚未安裝的項目會回報找不到，屬正常）
python3 --version
docker --version
docker info | egrep 'Server Version|Storage Driver|Docker Root Dir'
ovs-vsctl --version
virsh --version
qemu-system-x86_64 --version
ip -4 addr show docker0                 # 看 docker0 子網是否為 172.17.x.x

# 6. 對外連線（用於確認 registry 與下載站可達；離線環境請略過）
curl -sI --max-time 10 https://download.docker.com | head -1
curl -sI --max-time 10 https://sonicdev-microsoft.azurecr.io | head -1
curl -sI --max-time 10 https://sonic-build.azurewebsites.net | head -1
```

驗收標準：

| 檢查 | 通過條件 | 對應章節 |
|---|---|---|
| 作業系統 | `VERSION_ID` 為 `22.04` 或 `24.04`，架構 amd64 | 9.4 |
| 虛擬化旗標 | `egrep -c '(vmx\|svm)'` 大於 0；`/dev/kvm` 存在；`kvm-ok` 通過 | 9.3.1 |
| 若主機是 VM | `nested` 為 `Y` 或 `1`（Intel 或 AMD 對應的檔案） | 9.3.1 |
| 記憶體 | `free -h` 的 total 符合你購買的等級 | 9.3 |
| 磁碟 | `df -h` 掛載 `/`、`/var/lib/docker`、`root_path` 所在的磁碟都有足夠空間；`ROTA` 為 0 | 9.3 |
| 網卡 | 管理網卡 `Link detected: yes`；實體 testbed 的 trunk 網卡速率符合採購規格 | 9.5 |
| docker0 | 若有輸出，確認不是 `172.18.x.x`，否則依 8.2.7 調整 | 9.5.1 |
| 對外連線 | 第 6 組指令回傳 HTTP 狀態列（任何 HTTP 回應即代表可達）；失敗則改用離線鏡像 | 9.5.3 |

通過後就可以依第 8 章搭建。若任一項不通過，請在退換貨期限內回報供應商。

### 9.8 本章的限制

- 【估計】欄位沒有上游依據，請在小規模實測後修正。尤其是磁碟容量、實體核心數與 trunk 網卡速率。
- 我沒有在上游找到：每個 cEOS 容器的記憶體用量、T1／T2 專屬的伺服器規格、核心版本需求、hugepages 要求、每個映像的精確大小。
- 上游文件會演進（例如 Ubuntu 版本、cEOS 版本），採購前請再讀一次[VsSetup](https://github.com/sonic-net/sonic-mgmt/blob/master/docs/testbed/README.testbed.VsSetup.md)與[Setup](https://github.com/sonic-net/sonic-mgmt/blob/master/docs/testbed/README.testbed.Setup.md)。
- 本章是依上游文件整理改寫（Content was rephrased for compliance with licensing restrictions），所有指令都沒有在本機執行，因為本章的目標是另一台尚未採購的伺服器。

## 10. 參考資料

本機路徑：見第 11 章「本機引用路徑」清單。

外部連結：

- [sonic-mgmt docs/tests/README.md](https://github.com/sonic-net/sonic-mgmt/blob/master/docs/tests/README.md)
- [sonic-mgmt docs/tests/pytest.org.md](https://github.com/sonic-net/sonic-mgmt/blob/master/docs/tests/pytest.org.md)
- [sonic-mgmt docs/tests/guidelines.md](https://github.com/sonic-net/sonic-mgmt/blob/master/docs/tests/guidelines.md)
- [sonic-mgmt docs/tests/styleguide.md](https://github.com/sonic-net/sonic-mgmt/blob/master/docs/tests/styleguide.md)
- [sonic-mgmt docs/tests/pytest.run.md](https://github.com/sonic-net/sonic-mgmt/blob/master/docs/tests/pytest.run.md)
- [sonic-mgmt docs/testbed/README.testbed.Overview.md](https://github.com/sonic-net/sonic-mgmt/blob/master/docs/testbed/README.testbed.Overview.md)
- [sonic-mgmt docs/testbed/README.testbed.VsSetup.md](https://github.com/sonic-net/sonic-mgmt/blob/master/docs/testbed/README.testbed.VsSetup.md)
- [sonic-mgmt tests/conftest.py](https://github.com/sonic-net/sonic-mgmt/blob/master/tests/conftest.py)
- [sonic-mgmt tests/pytest.ini](https://github.com/sonic-net/sonic-mgmt/blob/master/tests/pytest.ini)
- [sonic-mgmt tests/bgp/test_bgp_fact.py](https://github.com/sonic-net/sonic-mgmt/blob/master/tests/bgp/test_bgp_fact.py)
- [sonic-mgmt tests/ipfwd/test_dir_bcast.py](https://github.com/sonic-net/sonic-mgmt/blob/master/tests/ipfwd/test_dir_bcast.py)
- [sonic-mgmt tests/common/helpers/assertions.py](https://github.com/sonic-net/sonic-mgmt/blob/master/tests/common/helpers/assertions.py)
- [sonic-mgmt tests/common/utilities.py](https://github.com/sonic-net/sonic-mgmt/blob/master/tests/common/utilities.py)
- [sonic-mgmt tests/common/devices/sonic.py](https://github.com/sonic-net/sonic-mgmt/blob/master/tests/common/devices/sonic.py)
- [sonic-mgmt tests/ptf_runner.py](https://github.com/sonic-net/sonic-mgmt/blob/master/tests/ptf_runner.py)
- [sonic-mgmt tests/common/plugins/ptfadapter/README.md](https://github.com/sonic-net/sonic-mgmt/blob/master/tests/common/plugins/ptfadapter/README.md)
- [sonic-mgmt tests/common/plugins/loganalyzer/README.md](https://github.com/sonic-net/sonic-mgmt/blob/master/tests/common/plugins/loganalyzer/README.md)
- [sonic-mgmt docs/testbed/README.testbed.Setup.md](https://github.com/sonic-net/sonic-mgmt/blob/master/docs/testbed/README.testbed.Setup.md)
- [sonic-mgmt docs/testbed/README.new.testbed.Configuration.md](https://github.com/sonic-net/sonic-mgmt/blob/master/docs/testbed/README.new.testbed.Configuration.md)
- [sonic-mgmt docs/testbed/README.testbed.Fanout.md](https://github.com/sonic-net/sonic-mgmt/blob/master/docs/testbed/README.testbed.Fanout.md)
- [sonic-mgmt docs/testbed/README.testbed.Minigraph.md](https://github.com/sonic-net/sonic-mgmt/blob/master/docs/testbed/README.testbed.Minigraph.md)
- [sonic-mgmt docs/testbed/README.testbed.Cli.md](https://github.com/sonic-net/sonic-mgmt/blob/master/docs/testbed/README.testbed.Cli.md)
- [sonic-mgmt docs/testbed/README.testbed.FAQ.md](https://github.com/sonic-net/sonic-mgmt/blob/master/docs/testbed/README.testbed.FAQ.md)
- [sonic-mgmt ansible/testbed-cli.sh](https://github.com/sonic-net/sonic-mgmt/blob/master/ansible/testbed-cli.sh)
- [sonic-mgmt setup-container.sh](https://github.com/sonic-net/sonic-mgmt/blob/master/setup-container.sh)
- [sonic-mgmt ansible/setup-management-network.sh](https://github.com/sonic-net/sonic-mgmt/blob/master/ansible/setup-management-network.sh)
- [sonic-mgmt ansible/vars/topo_t0.yml](https://github.com/sonic-net/sonic-mgmt/blob/master/ansible/vars/topo_t0.yml)
- [sonic-mgmt ansible/group_vars/vm_host/main.yml](https://github.com/sonic-net/sonic-mgmt/blob/master/ansible/group_vars/vm_host/main.yml)
- [sonic-mgmt ansible/roles/vm_set/templates/arista.xml.j2](https://github.com/sonic-net/sonic-mgmt/blob/master/ansible/roles/vm_set/templates/arista.xml.j2)
- [sonic-mgmt ansible/roles/vm_set/templates/sonic.xml.j2](https://github.com/sonic-net/sonic-mgmt/blob/master/ansible/roles/vm_set/templates/sonic.xml.j2)
- [sonic-mgmt ansible/roles/vm_set/tasks/host_setup.yml](https://github.com/sonic-net/sonic-mgmt/blob/master/ansible/roles/vm_set/tasks/host_setup.yml)
- [sonic-mgmt ansible/roles/vm_set/tasks/docker.yml](https://github.com/sonic-net/sonic-mgmt/blob/master/ansible/roles/vm_set/tasks/docker.yml)
- [sonic-mgmt ansible/vtestbed.yaml](https://github.com/sonic-net/sonic-mgmt/blob/master/ansible/vtestbed.yaml)
- [sonic-mgmt ansible/vars](https://github.com/sonic-net/sonic-mgmt/tree/master/ansible/vars)
- [sonic-mgmt .azure-pipelines](https://github.com/sonic-net/sonic-mgmt/tree/master/.azure-pipelines)
- [sonic-mgmt .pre-commit-config.yaml](https://github.com/sonic-net/sonic-mgmt/blob/master/.pre-commit-config.yaml)

文中內容為依上游文件整理改寫（Content was rephrased for compliance with licensing restrictions）。

## 11. 本機引用路徑

- `${HOME}/sonic-buildimage/.azure-pipelines/azure-pipelines-build-vs-and-test.yml`
- `${HOME}/sonic-buildimage/.azure-pipelines/azure-pipelines-build.yml`
- `${HOME}/sonic-buildimage/README.md`
- `${HOME}/sonic-buildimage/platform/vs/docker-ptf.mk`
- `${HOME}/sonic-buildimage/platform/vs/kvm-image.mk`
- `${HOME}/sonic-buildimage/platform/vs/rules.mk`
- `${HOME}/sonic-buildimage/platform/vs/README.vsvm.md`
- `${HOME}/sonic-buildimage/rules/docker-sonic-mgmt.mk`
- `${HOME}/sonic-buildimage/dockers/docker-sonic-mgmt/Dockerfile.j2`
- `${HOME}/sonic-buildimage/platform/vs/docker-sonic-vs.mk`
- `${HOME}/sonic-buildimage/src/ptf`
- `${HOME}/sonic-buildimage/src/ptf-py3` (`README.md`、`example/mytests`)
- `${HOME}/sonic-buildimage/src/sonic-swss-common/tests` (`Makefile.am`、`conftest.py`、`test_*.py`)
- `${HOME}/sonic-buildimage/src/sonic-swss/.azure-pipelines/test-docker-sonic-vs-template.yml`
- `${HOME}/sonic-buildimage/src/sonic-swss/run-gtest-suite.py`
- `${HOME}/sonic-buildimage/src/sonic-swss/tests` (`README.md`、`conftest.py`、`run-tests.sh`、`test_vlan.py`、`dvslib/dvs_vlan.py`、`dvslib/dvs_common.py`)
- `${HOME}/sonic-buildimage/src/sonic-swss/tests/mock_tests` (`Makefile.am`、`bulker_ut.cpp`、`swssnet_ut.cpp`)
- `${HOME}/sonic-buildimage/src/sonic-utilities` (`README.md`、`pytest.ini`、`conftest.py`、`config/vlan.py`、`utilities_common/cli.py`、`utilities_common/db.py`)
- `${HOME}/sonic-buildimage/src/sonic-utilities/tests` (`conftest.py`、`vlan_test.py`、`config_int_mtu_test.py`、`pfc_test.py`、`mock_tables/config_db.json`、`mock_tables/dbconnector.py`)

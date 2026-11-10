# SONiC 测试架构与添加测试案例教学

本文档说明 SONiC 的测试架构，并以实作范例带你添加测试案例。

## 0. 本文的依据与限制

- 本机路径以 `${HOME}/sonic-buildimage` 为根（下文简写为 `<BI>`），其中的子模块 `src/sonic-swss`、`src/sonic-utilities`、`src/sonic-swss-common`、`src/ptf-py3` 皆已存在，文中引用的本机路径都已用 `ls` 确认过。
- **本机没有 `sonic-mgmt` repo**（已搜索 `${HOME}`、`.agents/tasks`、`.kiro`，只找到 `src/sonic-mgmt-framework`、`src/sonic-mgmt-common`，它们是管理接口进程，不是测试 repo）。因此 sonic-mgmt 的部分（第 3、4、8、9 章与 5.4、5.5 节）依据上游官方文档与原代码，并附链接。
- 这台机器的 Python 环境缺少 `sonic_py_common` 等相依套件（运行 `src/sonic-utilities/tests` 时出现 `ModuleNotFoundError: No module named 'sonic_py_common'`），所以文中的添加范例**没有在本机实际运行通过**，只做了语法检查与对照既有测试的 API 验证。请依第 6 章在 sonic-slave 容器或（另购服务器上的）sonic-mgmt 容器内运行。
- **testbed 不会在这台机器（`${HOME}`）上搭建**，而是跑在另外采购的服务器上。因此第 9 章整理了采购用的服务器环境需求、规格分级与验收指令；第 8 章的步骤是在那台新服务器上运行。这台机器的状态（无 `/dev/kvm`、Ubuntu 20.04）只在第 8.2.0 节留作简短附注。第 8 章的指令来自上游文档与本机 Makefile 的查证，没有在本机实跑。
- 上游 sonic-mgmt 的内容会随 master 演进，数字与文档清单以你 checkout 的版本为准。

## 1. SONiC 测试分层总览

| 层级 | 位置 | 框架 | 需要的环境 | 验证重点 |
|---|---|---|---|---|
| 单元测试（Python CLI） | `<BI>/src/sonic-utilities/tests` | pytest + click `CliRunner` + mock DB | 一般 Linux / slave 容器 | `show`、`config` 等 CLI 逻辑 |
| 单元测试（C++ 函数库） | `<BI>/src/sonic-swss-common/tests` | gtest（`tests/Makefile.am`）与 pytest（`tests/test_*.py`） | slave 容器、Redis | swss-common 函数库 |
| 单元测试（orchagent / cfgmgr） | `<BI>/src/sonic-swss/tests/mock_tests` | gtest + gmock（mock SAI） | slave 容器 | Orch 类别、cfgmgr 逻辑 |
| 虚拟交换机测试（vstest） | `<BI>/src/sonic-swss/tests/test_*.py` | pytest + `docker-sonic-vs` 容器 | Ubuntu 22.04、Docker | 写入 CONFIG_DB / APPL_DB，检查 ASIC_DB（控制面） |
| 系统测试 | 上游 `sonic-mgmt/tests` | pytest + Ansible + PTF | KVM 虚拟或实体 testbed | 整机功能、数据面、重开机、平台 |
| 数据面封包测试 | PTF（`<BI>/src/ptf-py3`，sonic-mgmt 内有 `tests/ptftests`） | PTF（unittest 为基础） | PTF 容器或主机 | 送封包、验证收到的封包 |
| CI | `<BI>/.azure-pipelines`、各 repo 的 `azure-pipelines.yml` | Azure Pipelines | 云端 agent | 建置、单元测试、VS 映像上跑 sonic-mgmt |

选择原则：能用单元测试验证的逻辑就不要放到 testbed 上跑；单元测试快速、可在任何 Linux 上运行。需要真实进程交互与 Redis 数据流时用 vstest；需要多设备、BGP 邻居或封包收发时才用 sonic-mgmt。

## 2. 各 repo 内的测试（本机可查证）

### 2.1 sonic-utilities：pytest 单元测试

- 测试目录：`<BI>/src/sonic-utilities/tests`，文档名惯例为 `*_test.py`（例如 `vlan_test.py`、`config_int_mtu_test.py`）。
- 根目录的 `conftest.py` 说明是为 pytest-xdist 平行化而设；`pytest.ini` 配置了 `-n auto --dist loadfile`、覆盖率与 `test-results.xml`，并定义 `serial` marker。
- `tests/mock_tables/` 内有仿真数据库内容，例如 `config_db.json`、`appl_db.json`、`asic_db.json`、`counters_db.json`；`tests/mock_tables/dbconnector.py` 对 `swsssdk`／`SonicV2Connector` 做 monkey patch，让测试读写的是 JSON 档而不是真的 Redis。
- 例如 `mock_tables/config_db.json` 内有 `VLAN|Vlan1000`、`VLAN|Vlan2000`、`VLAN|Vlan3000`、`VLAN|Vlan4000`，`vlan_test.py` 的预期输出就是依这份数据而来。
- `tests/conftest.py` 有 autouse fixture（如 `setup_db_config`、`setup_env`），以及 `setup_single_broadcom_asic`、`setup_t1_topo` 等情境 fixture。
- 惯用写法是用 click 的 `CliRunner().invoke(...)` 调用指令对象，再检查 `exit_code` 与 `output`；需要 DB 时用 `utilities_common.db.Db()` 并以 `obj=db` 传入。
- 官方 README 的运行方式为在 sonic-slave 容器内运行 `python3 setup.py test`（见 `<BI>/src/sonic-utilities/README.md`）。

### 2.2 sonic-swss-common：gtest 与 pytest

- `<BI>/src/sonic-swss-common/tests/Makefile.am` 以 `tests_tests_SOURCES` 列出所有 `*_ut.cpp`，链接 `-lgtest -lgtest_main -lgmock -lgmock_main`，产生 `tests/tests` 运行档。
- 同目录也有 `test_*.py`（如 `test_redis_ut.py`、`test_logger.py`），搭配 `tests/conftest.py` 以 `SonicDBConfig.initialize(...)` 加载 `redis_multi_db_ut_config/database_config.json`。

### 2.3 sonic-swss：gtest（mock_tests）

- 位置：`<BI>/src/sonic-swss/tests/mock_tests`，例如 `bulker_ut.cpp`、`swssnet_ut.cpp`、`portsorch_ut.cpp`。
- `mock_tests/Makefile.am` 定义多个测试运行档：`tests`（orchagent）、`tests_intfmgrd`、`tests_teammgrd`、`tests_portsyncd`、`tests_fpmsyncd`、`tests_fdbsyncd`、`tests_response_publisher`、`tests_nbrmgrd`、`tests_teamsyncd`，并登记在 `TESTS` 与 `noinst_PROGRAMS`。
- 使用 `-lsaivs -lsairedis -lsaimeta -lsaimetadata` 链接 SAI virtual switch 与 sairedis；`mock_sai_api.h`、`mock_*.cpp` 提供替身。
- 添加一个 `*_ut.cpp` 时，必须把文档名加进 `Makefile.am` 的 `tests_SOURCES`。
- `<BI>/src/sonic-swss/run-gtest-suite.py` 接受 `--test-name`、`--log-file`、`--trs-file` 等参数，推测为 automake 的测试驱动脚本（此点未实际验证）。

### 2.4 sonic-swss vstest：docker-sonic-vs

`<BI>/src/sonic-swss/tests/README.md` 的重点如下：

- DVS（Docker Virtual Switch）让开发者不需要实体交换机或 ASIC，就能验证 SWSS 新功能的控制面行为。
- 测试流程：对 redis（通常是 CONFIG_DB 或 APPL_DB）写入配置，检查 SWSS 处理后的状态（通常看 ASIC_DB）。
- SWSS、Redis 等组件都跑在 `docker-sonic-vs` 容器内。数据面不被仿真。
- 环境需求：Ubuntu 22.04、Python 3、Docker、`team` kernel module，以及 `libswsscommon`、`python3-swsscommon` 等套件。

进程结构：

- `tests/conftest.py`：定义 `pytest_addoption`（`--dvsname`、`--forcedvs`、`--keeptb`、`--imgname`、`--max_cpu`、`--num-ports` 等）、session 级的 `manage_dvs`、module 级的 `dvs`，以及 `testlog`。
- `dvs` fixture 会读取测试模块内的 `DVS_ENV`、`NUM_PORTS` 变量来决定容器环境。
- `tests/dvslib/`：各功能的辅助类别，如 `dvs_vlan.py`、`dvs_acl.py`、`dvs_lag.py`、`dvs_route.py`、`dvs_database.py`（DB 访问与轮询）、`dvs_common.py`（`PollingConfig`、`wait_for_result`）。
- `conftest.py` 以 class 级 fixture（如 `dvs_vlan_manager`、`dvs_lag_manager`）创建 `dvslib` 对象，并挂到 `request.cls`，所以测试类别内可直接用 `self.dvs_vlan`。

### 2.5 PTF（数据面）

- 本机有两份原代码：`<BI>/src/ptf` 与 `<BI>/src/ptf-py3`（后者为 Python 3 版本）。`platform/vs/docker-ptf.mk` 定义 PTF 容器映像。
- `<BI>/src/ptf-py3/README.md` 说明：PTF 是以 Python `unittest` 为基础的数据面测试框架，缺省使用 Scapy 组封包，需要 root 权限，并以 `--interface <port>@<netdev>` 指定要绑定的网卡，例如：

```bash
sudo ./ptf --test-dir mytests/ --pypath $PWD \
    --interface 0@veth1 --interface 1@veth3
```

- 范例测试放在 `<BI>/src/ptf-py3/example/mytests`。

## 3. sonic-mgmt：系统测试架构（上游文档）

本章依据 [sonic-mgmt 的 testbed 文档](https://github.com/sonic-net/sonic-mgmt/blob/master/docs/testbed/README.testbed.Overview.md)与[测试文档索引](https://github.com/sonic-net/sonic-mgmt/blob/master/docs/tests/README.md)。

### 3.1 实体连接与硬件需求

上游文档将 testbed 的实体组件分为：

- Test server（测试服务器）
- Fanout switch（扇出交换机，分 leaf 与 root；root 非必要）
- SONiC DUT（Device Under Test，受测设备）

关键设计：每个 DUT 连接端口接到某个 leaf fanout，leaf fanout 为每个 DUT 连接端口指定唯一 VLAN tag，经 802.1Q trunk（必要时再经 root fanout）与测试服务器相连。因此测试服务器能以该 VLAN tag 访问 DUT 的任何连接端口。例如 testbed 1 使用 VLAN 100 至 131，testbed 2 使用 132 至 163（皆为 32 端口 DUT）。服务器、fanout、DUT 的数量上游没有固定。

测试服务器上会创建：

- 一组 VM 或容器（vEOS、cEOS 或 SONiC）仿真 DUT 的邻居。
- 一个 PTF 容器，用来送出与接收封包。
- 这些组件与 VLAN 接口之间以 Open vSwitch（OVS）bridge 互连。

硬件需求（出自[上游 Overview](https://github.com/sonic-net/sonic-mgmt/blob/master/docs/testbed/README.testbed.Overview.md)与 [FAQ](https://github.com/sonic-net/sonic-mgmt/blob/master/docs/testbed/README.testbed.FAQ.md)，此处为改写整理）：

| 组件 | 需求 |
|---|---|
| DUT | 要测的 SONiC 实机。每个要测的连接端口都要接到 leaf fanout。 |
| Leaf fanout | 上游举例 Arista 7260 或类似机型。端口数必须多于 DUT。需支持 LACP／LLDP 穿透与 802.1Q tunneling（QinQ）。 |
| Root fanout（选用） | 需求同 leaf。若只有一个 testbed，或测试服务器只服务一个 testbed，可省略，leaf 的 trunk 直接接服务器网卡。 |
| VLAN trunk | DUT 面向端口为 access（每端口唯一 VLAN），上行为 trunk 并允许全部这些 VLAN，直达测试服务器网卡。 |
| 测试服务器 | 上游的参考规格（明确写「非强制」）：Dell 730、2 颗 CPU 各 18 核、192 GB RAM、2 颗 500 GB 磁盘、Mellanox MT27700（CX4）网卡。FAQ 的最低需求是一台 L2 fanout 交换机与至少 92 GB 内存的 Linux 服务器（文档较旧）。 |
| 管理网络 | 服务器、VM、PTF 与 DUT 的管理端口都在同一管理网段，服务器端以 bridge `br1` 承载（范例网段 `10.250.0.0/24`）。建议服务器另有一张网卡用于连到实验室网络。Console server 与 PDU 为选用。 |

PTF 与邻居的接法依拓扑不同。T0 的前 N 个 DUT 端口直接对应 PTF 容器。T1 与 T2 的 PTF 没有直接接到 DUT 的端口，而是把 PTF 的 veth 「注入」到各 VM 连接的 OVS bridge。依上游 Overview，OVS flow 规则为：VM 到 DUT 只往 DUT；DUT 到 VM 与 PTF；PTF 到 DUT 只往 DUT。

### 3.2 逻辑拓扑

上游 Overview 写明主要有 5 类拓扑：T0、T1、T2、PTF、PTP。每一类各有许多变体，细节定义在 sonic-mgmt 的 `ansible/vars/topo_*.yml`（目前超过 150 个档，清单见 [ansible/vars](https://github.com/sonic-net/sonic-mgmt/tree/master/ansible/vars)，以你 checkout 为准）。testbed 项目的 `topo:` 字段（例如 `t0`、`t1-lag`、`ptf32`）决定使用哪个文档；测试内的 `pytest.mark.topology('t0')` 则是「这个测试允许在哪些拓扑跑」的选择器。

先厘清两个概念，避免混淆：

| 网络层级 | 下游 | 上游 |
|---|---|---|
| T0（ToR） | 服务器 | T1 |
| T1 | T0 | T2 |
| T2 | T1 | Regional Gateway（T3） |

这张表只是数据中心网络的阶层。下面的「五类逻辑拓扑」是测试环境怎幺接 DUT 的定义。

#### 五类逻辑拓扑总表（以各类代表变体计）

数字来源：VM 数为 `topo_*.yml` 中 `topology.VMs` 的项目数（我解析 yml 计算），DUT 端口数与角色对照 Overview 文本。

| 类型 | 代表变体 | DUT 端口数 | 邻居 VM 数 | 端口角色 | 需要 VM | 上游文档 |
|---|---|---|---|---|---|---|
| T0 | `t0` | 32 | 4（ARISTA01T1 至 04T1） | 前 28 端口接 PTF（仿真服务器），后 4 端口各接 1 台 VM（扮演 T1，单一连接的 port-channel） | 是 | [topo_t0.yml](https://github.com/sonic-net/sonic-mgmt/blob/master/ansible/vars/topo_t0.yml) |
| T1 | `t1` | 32 | 32 | 16 端口各接 1 台 VM 扮演上游 T2，16 端口各接 1 台 VM 扮演下游 T0；无 port-channel；PTF 没有直连端口，只有注入端口 | 是 | [topo_t1.yml](https://github.com/sonic-net/sonic-mgmt/blob/master/ansible/vars/topo_t1.yml) |
| T2 | `t2` | 96（3 张 linecard 各 32）；supervisor 无 DUT 端口 | 72 | linecard 1 的 24 台 VM 扮演 T3，linecard 2 与 3 各 24 台扮演 T1；`dut_num: 4`（3 LC + 1 SUP）；PTF 只有注入端口（0 至 71） | 是 | [topo_t2.yml](https://github.com/sonic-net/sonic-mgmt/blob/master/ansible/vars/topo_t2.yml) |
| PTF | `ptf32` | 32 | 0 | 全部 DUT 端口接 PTF 容器，无注入端口 | 否 | [topo_ptf32.yml](https://github.com/sonic-net/sonic-mgmt/blob/master/ansible/vars/topo_ptf32.yml) |
| PTP | `ptp-32` | 32 | 0 | 两台 SONiC DUT 以多条连接互连（point-to-point），不接 PTF 与 VM | 否 | [topo_ptp-32.yml](https://github.com/sonic-net/sonic-mgmt/blob/master/ansible/vars/topo_ptp-32.yml) |

重要提醒：「T0 为 4 台 VM、T1 为 32 台 VM」只对代表变体 `t0`、`t1` 成立，换变体数字就会变。T2 不能一概视为机箱：`t2` 是 72 台 VM 的机箱拓扑，另有单节点版本。PTP 不是 Precision Time Protocol。

#### 变体清单与链接

| 类型 | 变体 | DUT 端口 | VM 数 | 说明 | 文档 |
|---|---|---|---|---|---|
| T0 | `t0` | 32 | 4 | 28 端口接 PTF，4 端口接 4 台 VM | [yml](https://github.com/sonic-net/sonic-mgmt/blob/master/ansible/vars/topo_t0.yml) |
| T0 | `t0-56` | 56 | 8 | 48 端口接 PTF，8 端口接 8 台 VM | [yml](https://github.com/sonic-net/sonic-mgmt/blob/master/ansible/vars/topo_t0-56.yml) |
| T0 | `t0-64` | 64 | 4 | 56 个 host 接口加 8 个 VM 端口（每台 VM 2 条连接，由 yml 计算） | [yml](https://github.com/sonic-net/sonic-mgmt/blob/master/ansible/vars/topo_t0-64.yml) |
| T0 | `t0-80` | 80 | 8 | 64 端口接 PTF，16 端口接 8 台 VM（各 2 条连接的 port-channel） | Overview 描述 |
| T0 | `t0-116` | 以 yml 计算为 112 个 host 加 8 个 VM 端口 | 4 | 只在 yml 中确认，不把「116 端口」当事实 | [yml](https://github.com/sonic-net/sonic-mgmt/blob/master/ansible/vars/topo_t0-116.yml) |
| T1 | `t1` | 32 | 32 | 见上表 | [yml](https://github.com/sonic-net/sonic-mgmt/blob/master/ansible/vars/topo_t1.yml) |
| T1 | `t1-lag` | 32 | 24 | 16 端口接 8 台 T2 VM（各 2 条连接的 port-channel），16 端口接 16 台 T0 VM | [yml](https://github.com/sonic-net/sonic-mgmt/blob/master/ansible/vars/topo_t1-lag.yml) |
| T1 | `t1-64` | 64 | 64 | 每端口 1 台 VM（由 yml 计算） | [yml](https://github.com/sonic-net/sonic-mgmt/blob/master/ansible/vars/topo_t1-64.yml) |
| T1 | `t1-8-lag` | 多 ASIC 虚拟 DUT 用 | 2 | vtestbed 的 four-asic 项目使用 | [yml](https://github.com/sonic-net/sonic-mgmt/blob/master/ansible/vars/topo_t1-8-lag.yml) |
| T1 | `t1-isolated-d224u8` | 232 | 232 | 8 台 T2 在上、224 台 T0 在下 | Overview 描述 |
| T2 | `t2` | 96 | 72 | 机箱：3 linecard 加 1 supervisor | [yml](https://github.com/sonic-net/sonic-mgmt/blob/master/ansible/vars/topo_t2.yml) |
| T2 | `t2_single_node_min` | 单节点（非机箱） | 5 | vtestbed 项目 `vms-kvm-t2-single-node-min`（DUT `vlab-ut2`） | [yml](https://github.com/sonic-net/sonic-mgmt/blob/master/ansible/vars/topo_t2_single_node_min.yml) |
| T2 | `t2_2lc_min_ports-masic` | 多 ASIC 机箱 | 未逐一统计 | vtestbed 项目 `vms-kvm-t2`，3 台 DUT（`vlab-t2-1-1`、`vlab-t2-1-2`、`vlab-t2-1-sup`） | [vtestbed.yaml](https://github.com/sonic-net/sonic-mgmt/blob/master/ansible/vtestbed.yaml) |
| PTF | `ptf32` | 32 | 0 | 全部接 PTF | [yml](https://github.com/sonic-net/sonic-mgmt/blob/master/ansible/vars/topo_ptf32.yml) |
| PTF | `ptf64` | 64 | 0 | 全部接 PTF | [yml](https://github.com/sonic-net/sonic-mgmt/blob/master/ansible/vars/topo_ptf64.yml) |
| PTF | `ptf8` | 8 | 0 | `dut_type` 为 SonicHost | [yml](https://github.com/sonic-net/sonic-mgmt/blob/master/ansible/vars/topo_ptf8.yml) |
| PTP | `ptp-32` | 32 | 0 | 名称中的数字即端口数；另有 `ptp-56`、`ptp-64`、`ptp-256` 等 | [yml](https://github.com/sonic-net/sonic-mgmt/blob/master/ansible/vars/topo_ptp-32.yml) |

上游另有 M0、MX、dualtor、wan、smartswitch 等其他家族（Overview 列有 M0：52 端口 6 台 VM；MX：48 端口 2 台 VM），本文不展开。

#### 各类型的 DUT、VM 与硬件需求

| 类型 | 要不要 VM | DUT 台数与端口角色 | 实体测试需要的硬件 |
|---|---|---|---|
| T0 | 要，`t0` 为 4 台（`t0-56`、`t0-80` 为 8 台） | 1 台 DUT，32 端口中 28 端口为服务器侧（PTF），4 端口为上行（VM 扮 T1） | 32 端口 DUT；leaf fanout 至少 32 个 DUT 面向端口加上行 trunk；每端口一个 VLAN；测试服务器要跑 4 台邻居与 PTF |
| T1 | 要，`t1` 为 32 台 | 1 台 DUT，16 端口上行、16 端口下行，全部接 VM | 32 端口 DUT；leaf fanout 32 个以上；`t1` 邻居 VM 数多，服务器资源需求高于 T0（推论：上游未给 T1 专属规格，FAQ 的 92 GB 为通用最低需求，且文档较旧） |
| T2 | 要，`t2` 为 72 台 | 机箱 4 个 DUT 节点（3 linecard 加 1 supervisor），共 96 个 DUT 端口 | 需要机箱式 DUT；96 个 DUT 端口对应的 fanout 端口与 VLAN；服务器可能要多台，资源远大于 T0／T1（此点为推论，上游未给 T2 的服务器规格） |
| PTF | 不要 | 1 台 DUT，全部端口接 PTF | 只需要 DUT、fanout（或直连）与测试服务器，不需邻居 VM，服务器资源最小 |
| PTP | 不要 | 2 台 SONiC DUT，端口对端口互连 | 两台实体 DUT 以光模块／DAC 直接互连，数据路径不需要 fanout VLAN trunk；管理网络仍需要 |

PTP 的真正含义：point-to-point，不是 Precision Time Protocol。依 Overview，它用来验证收发器（transceiver）及其在 LLDP 等 L2 控制流量下的连接稳定性，且不使用接到 PTF 或 VM 的端口，改由两台 SONiC DUT 之间以多条连接互连。有一个不一致之处要提醒：`topo_ptp-32.yml` 本身是 `VMs: {}`、`host_interfaces` 为 0 至 31、`configuration: {}`，也就是说 yml 只是「无 VM」的定义，两台 DUT 互连是实验室实体接线。我在 `ansible/vtestbed.yaml` 找不到任何 PTP 项目，因此没有虚拟 KVM 的 PTP 范例，PTP 偏向实体实验室。

数据源：[Overview 的 Logical topologies 章节](https://github.com/sonic-net/sonic-mgmt/blob/master/docs/testbed/README.testbed.Overview.md)与上表链接的 `topo_*.yml`。

### 3.3 Testbed 配置档

`ansible/vtestbed.yaml` 是虚拟环境的 testbed 档，实体环境使用 `ansible/testbed.yaml`（CSV 版已弃用），字段相同。每笔记录形如：

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

（字段取自[上游 vtestbed.yaml](https://github.com/sonic-net/sonic-mgmt/blob/master/ansible/vtestbed.yaml)，仅列出部分字段；实际还有 `ptf_ipv6`、`auto_recover`、`use_converged_peers`、`comment`。）

- `conf-name`：testbed 名称，即 `run_tests.sh -n` 与 pytest `--testbed` 的值，必须唯一。
- `group-name`：长度不超过 8 个字符；相同 `group-name` 的项目必须共用 `ptf_ip`、`server`、`vm_base`。
- `topo`：使用哪份 `topo_*.yml`，必须存在于 `ansible/vars/`。
- `vm_base`：此 testbed 第一台邻居 VM 的名称。`t0` 有 4 台 VM，所以占用 `VM0100` 至 `VM0103`；不同 `group-name` 的 `vm_base` 范围不可重叠（见[组态文档的 Consistency Rule](https://github.com/sonic-net/sonic-mgmt/blob/master/docs/testbed/README.new.testbed.Configuration.md)）。
- `dut`：DUT 主机名称，即 `-d` 或 `--host-pattern`。
- `inv_name`：Ansible inventory 文档名（`veos_vtb`；实体环境为 `veos` 加上 `lab`）。

### 3.4 搭建流程总览

搭建测试环境的完整步骤已移到第 8 章：8.1 为路线选择表，8.2 为单机 KVM 虚拟 testbed（路线 A），8.3 为实体 testbed（路线 B）。要采购什幺规格的服务器，见第 9 章。整体顺序在两条路线一致：准备主机与管理网络、取得映像、创建 sonic-mgmt 容器、改 inventory 与 testbed 档、`add-topo`、`deploy-mg`、跑冒烟测试、清除。

### 3.5 进程目录与重要组件

以下依 [sonic-mgmt master 目录](https://github.com/sonic-net/sonic-mgmt/tree/master/tests)与[测试组织文档](https://github.com/sonic-net/sonic-mgmt/blob/master/docs/tests/pytest.org.md)整理：

| 路径 | 用途 |
|---|---|
| `tests/<feature>/test_*.py` | 依功能分文档夹的测试，例如 `bgp/`、`arp/`、`acl/` |
| `tests/conftest.py` | 根 conftest，定义 `--testbed` 等命令行选项与内核 fixture |
| `tests/pytest.ini` | marker 定义（`topology`、`platform`、`disable_loganalyzer` 等）与 `python_files = test_*.py unit_test_*.py` |
| `tests/common/` | 可重用代码（helpers、fixtures、devices、plugins、utilities） |
| `tests/common/helpers/assertions.py` | `pytest_assert`、`pytest_require` |
| `tests/common/utilities.py` | `wait_until` 等工具 |
| `tests/common/devices/sonic.py` | `SonicHost`（`duthost`），含 `show_and_parse`、`get_running_config_facts` 等 |
| `tests/common/plugins/` | `ptfadapter`、`loganalyzer`、`sanity_check`、`conditional_mark`、`custom_markers` 等插件 |
| `tests/ptftests/` | 在 PTF 容器内运行的封包测试 |
| `tests/ptf_runner.py` | 从 pytest 启动 PTF 测试的 `ptf_runner()` |
| `tests/run_tests.sh` | 官方建议的运行入口 |
| `ansible/` | testbed 档、inventory、拓扑、`testbed-cli.sh` |
| `.azure-pipelines/` | PR 检查与 `pr_test_scripts.yaml` |

### 3.6 内核 fixture

下列 fixture 均可在上游 [tests/conftest.py](https://github.com/sonic-net/sonic-mgmt/blob/master/tests/conftest.py) 找到定义：

| fixture | scope | 说明 |
|---|---|---|
| `tbinfo` | session | 由 `--testbed`、`--testbed_file` 加载的 testbed 信息（拓扑、PTF 等） |
| `duthosts` | session | 所有 DUT 的集合，以主机名称取值：`duthosts[hostname]` |
| `rand_one_dut_hostname` | module | 随机挑一台 DUT 的主机名称 |
| `rand_selected_dut` | module | `duthosts[rand_one_dut_hostname]` |
| `duthost` | session | 单一 DUT 对象（在单 DUT testbed 方便使用） |
| `ptfhost` | session | PTF 容器的主机对象（向下兼容单一 PTF） |
| `localhost` | session | 在 sonic-mgmt 容器本机运行 Ansible 模块 |
| `nbrhosts` | session | 邻居设备（EOS 等） |
| `creds` | session | 帐密信息 |
| `enum_frontend_dut_hostname`、`enum_asic_index` 等 | 见原代码 | 通过 `pytest_generate_tests` 对每个 DUT／ASIC 展开参数化 |
| `ptfadapter` | 见插件 | 在 pytest 中直接收发封包，详见[ptfadapter README](https://github.com/sonic-net/sonic-mgmt/blob/master/tests/common/plugins/ptfadapter/README.md) |
| `loganalyzer` | 自动激活 | 测试前后在 DUT syslog 加标记并分析，详见[loganalyzer README](https://github.com/sonic-net/sonic-mgmt/blob/master/tests/common/plugins/loganalyzer/README.md) |

`duthost` 对象会把未知的方法调用转成 Ansible 模块（`tests/common/devices/base.py` 的 `__getattr__`），因此 `duthost.shell(...)`、`duthost.command(...)` 等都可直接使用。

### 3.7 Markers

依[测试组织文档](https://github.com/sonic-net/sonic-mgmt/blob/master/docs/tests/pytest.org.md)与[指南](https://github.com/sonic-net/sonic-mgmt/blob/master/docs/tests/guidelines.md)：

- 每个测试都需要 `topology` marker，用来声明可运行的拓扑（`t0`、`t1`、`ptf`、`any` 等）。
- 测试模块以 `pytestmark = [...]` 配置模块层级 marker；case 层级 marker 会覆盖模块层级。
- 没有 topology marker 的测试，在指定拓扑时会被跳过。
- 其他常用 marker：`asic`、`feature`、`device_type('vs')`、`disable_loganalyzer`。实际可用清单以 `tests/pytest.ini` 为准。

例如上游 `tests/bgp/test_bgp_fact.py` 全文只有数行：

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

### 3.8 CI 如何串起来

- `<BI>/.azure-pipelines/azure-pipelines-build-vs-and-test.yml` 在 `resources.repositories` 声明 `sonic-net/sonic-mgmt` 的 `master`，`Test` 阶段（job `KVMTEST`，名称 "kvmtest by Elastictest"）使用 `run-test-elastictest-template.yml@sonic-mgmt` 范本，并配置 `inventory: veos_vtb`、`testbed_file: vtestbed.yaml`。也就是说 buildimage 建完 VS 映像后，会在 KVM testbed 上跑 sonic-mgmt 测试。
- `<BI>/src/sonic-swss/.azure-pipelines/test-docker-sonic-vs-template.yml` 会加载 `docker-sonic-vs.gz`，用 `./run-tests.sh` 平行运行 vstest。
- sonic-mgmt 本身的 PR 检查包含 `pre-commit`（flake8，`--max-line-length=120`）、markers 检查（`.azure-pipelines/markers-check.yml`）与 `pytest-collect-only`。另有 `.azure-pipelines/pr_test_scripts.yaml`，依拓扑（如 `t0`）列出 PR 要跑的测试档。

## 4. 测试风格要求（上游）

来自[上游 style guide](https://github.com/sonic-net/sonic-mgmt/blob/master/docs/tests/styleguide.md)：

- 遵循 PEP8；上游 style guide 写明 E501 可忽略，但 pre-commit 的 flake8 目前配置为 `--max-line-length=120`，建议控制在 120 字符内。
- 对公开接口使用 Google 风格 docstring。
- 引入 fixture 但看起来没被使用时，依项目做法加上 `# noqa: F401` 或 `# noqa: F811`（例如 `tests/ipfwd/test_dir_bcast.py` 的做法）。

## 5. 添加测试案例：逐步教学

### 5.1 范例 A：在 sonic-utilities 添加 CLI 单元测试

目标：验证 `config vlan add` 对非法 VLAN ID 与已存在 VLAN 的错误消息。依据：

- `<BI>/src/sonic-utilities/config/vlan.py` 的 `add_vlan`：VID 不在 1 到 4094 时以 `ctx.fail("Invalid VLAN ID {} (2-4094)")` 结束，VLAN 已存在时以 `"{} already exists."` 结束。
- `utilities_common/cli.py` 的 `is_vlanid_in_range` 以 `1 <= vid <= 4094` 判断。
- `tests/mock_tables/config_db.json` 内已有 `Vlan1000`。
- 范例沿用 `tests/config_int_mtu_test.py` 的写法：`Db()` 加 `obj=db`。

步骤：

1. 在 `src/sonic-utilities/tests/` 创建 `config_vlan_validation_test.py`（文档名以 `_test.py` 结尾，与既有惯例一致）。
2. 写入下列内容：

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

3. 若要验证 `show` 输出，把预期字符串放在测试档内或 `tests/<name>_input/` 文档夹（如 `pfc_input/assert_show_output.py` 的做法），再比对 `result.output`。
4. 若需要新的 DB 内容，优先在测试内用 `db.cfgdb.set_entry(...)` 暂时写入（见 `vlan_test.py` 的 `test_show_vlan_brief_explicit_proxy_arp_disable`）；只有多个测试共用时才修改 `mock_tables/*.json`，因为改动会影响所有既有测试的预期输出。
5. 运行（在 sonic-slave 容器内）：

```bash
cd src/sonic-utilities
python3 -m pytest tests/config_vlan_validation_test.py -o addopts="" -v
```

`-o addopts=""` 会覆盖 `pytest.ini` 的 `addopts`（含 xdist 与覆盖率），方便单档调试。若要完整流程则用 `python3 setup.py test`。

注：`tests/vlan_test.py` 的 `TestVlan.setup_class` 另外做了 `multi_asic` 的 patch。若你的测试需要多 ASIC 或特定 namespace，请参考该档与 `multi_asic_vlan_test.py`，不要自行重写。

### 5.2 范例 B：在 sonic-swss 添加 vstest（控制面）

目标：创建 VLAN，验证 ASIC_DB 出现对应的 VLAN 对象，再删除并确认消失。这是 `tests/test_vlan.py::TestVlan::test_VlanAddRemove` 的精简版，所用的方法都存在于 `tests/dvslib/dvs_vlan.py`。

1. 在 `src/sonic-swss/tests/` 创建 `test_vlan_example.py`。
2. 写入：

```python
import pytest


@pytest.mark.usefixtures("testlog")
@pytest.mark.usefixtures("dvs_vlan_manager")
class TestVlanExample(object):
    def test_vlan_create_remove(self, dvs):
        vlan = "100"

        # 写入 CONFIG_DB 的 VLAN 表
        self.dvs_vlan.create_vlan(vlan)

        # 等待 ASIC_DB 出现 1 个非缺省 VLAN，并检查其 VLAN ID
        vlan_oid = self.dvs_vlan.get_and_verify_vlan_ids(1)[0]
        self.dvs_vlan.verify_vlan(vlan_oid, vlan)

        # 删除后，ASIC_DB 内不应再有非缺省 VLAN
        self.dvs_vlan.remove_vlan(vlan)
        self.dvs_vlan.get_and_verify_vlan_ids(0)
```

说明：

- `dvs` 是 module 级 fixture，回传 `DockerVirtualSwitch`；`dvs_vlan_manager` 是 class 级 fixture，把 `DVSVlan` 挂到 `self.dvs_vlan`。
- `get_and_verify_vlan_ids` 内部用 `wait_for_n_keys` 轮询，不必自己 `sleep`。
- 需要特殊容器环境时，在模块层级定义 `DVS_ENV = [...]` 或 `NUM_PORTS = N`（`conftest.py` 的 `dvs` fixture 会读取）。
- 若需要的操作还没有 dvslib 封装，优先在 `tests/dvslib/` 添加或扩充辅助类别，并在 `conftest.py` 加对应的 `dvs_xxx_manager` fixture；不要在每个测试重复写低级 DB 操作。
- 文档名以 `test_` 开头。若你的测试需要被 CI 单独分派，请查看 `.azure-pipelines/test-docker-sonic-vs-template.yml` 中测试清单的产生方式。

运行：

```bash
cd src/sonic-swss/tests
sudo pytest -v --imgname=docker-sonic-vs:latest test_vlan_example.py
```

### 5.3 范例 C：在 sonic-swss 添加 gtest 单元测试

1. 在 `src/sonic-swss/tests/mock_tests/` 创建 `example_ut.cpp`，内容参考 `swssnet_ut.cpp`（`#include "ut_helper.h"`，在 namespace 中定义 `::testing::Test` 子类别与 `TEST_F`）：

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

2. 把 `example_ut.cpp` 加到 `mock_tests/Makefile.am` 的 `tests_SOURCES`（与其它 `*_ut.cpp` 同一个清单，行尾保留 `\` 续行）。
3. 在 sonic-slave 容器内重新产生 build 并编译。`tests` 是 automake 的 `noinst_PROGRAMS`，可用 `make check` 运行全部，或直接运行 `tests` 运行档并以 gtest 筛选：

```bash
./tests --gtest_filter='ExampleTest.*'
```

本机没有编译过 sonic-swss 的 `mock_tests`，上述指令请以你的 build 目录为准。

### 5.4 范例 D：在 sonic-mgmt 添加 pytest 系统测试

目标：在 T0／T1 拓扑上，验证所有 admin up 的连接端口在一段时间内都会变成 oper up。使用的 API 皆已在上游原代码确认存在：`duthosts`、`rand_one_dut_hostname`、`pytest_assert`（`tests/common/helpers/assertions.py`）、`wait_until`（`tests/common/utilities.py`）、`SonicHost.show_and_parse`（`tests/common/devices/sonic.py`，输出为 list of dict，字段 key 为小写的栏名，如 `interface`、`oper`、`admin`）。

步骤：

1. 决定位置：依功能放在对应文档夹，新功能就创建新文档夹；可重用的函数放到 `tests/common`。本例假设新建 `tests/example_feature/test_port_oper_status.py`（`example_feature` 为示意名称）。新文档夹是否需要 `__init__.py`，请参照你 checkout 版本中邻近文档夹的做法。
2. 声明 topology marker（必要）。
3. 写入：

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

说明：

- `duthosts[rand_one_dut_hostname]` 是官方范例常见的取 DUT 方式，例如 `tests/ipfwd/test_dir_bcast.py` 的 `duthost = duthosts[rand_one_dut_hostname]`。
- `wait_until(timeout, interval, delay, condition, *args, **kwargs)` 的签名来自上游 `tests/common/utilities.py`。
- 需要收发封包时，加入 `ptfadapter` fixture（见第 3.6 节链接）；需要运行较重的 PTF 脚本时，参考 `tests/ipfwd/test_dir_bcast.py`：以 `copy_ptftests_directory` fixture 把 `ptftests` 拷贝进 PTF 容器，再调用 `ptf_runner(ptfhost, 'ptftests', 'dir_bcast_test.BcastTest', '/root/ptftests', params=params, log_file=log_file, is_python3=True)`。
- 需要在测试中修改 DUT 配置时，务必在 fixture 的 `yield` 之后还原；上游的 `sanity_check` 插件会在测试前后检查 DUT 状态，失败的清理会连累后续测试。
- 会故意触发错误 log 的测试，使用 `@pytest.mark.disable_loganalyzer`，或在测试内配置 loganalyzer 的 ignore／expect 规则（见 loganalyzer README）。

4. 若要让 PR 的 CI 跑你的测试，需要把测试档加到 `.azure-pipelines/pr_test_scripts.yaml` 对应拓扑的清单，细节请依 sonic-mgmt 当时的 PR 规范确认。

### 5.5 范例 E：撰写 PTF 数据面测试的流程

1. 在 sonic-mgmt 的 `tests/ptftests/`（新的测试多放在 `py3` 子目录，`ptf_runner()` 的 `test_subdir='py3'` 默认值可佐证）添加继承 `BaseTest` 的类别，使用 `ptf.testutils` 的 `simple_tcp_packet`、`send`、`verify_packet` 等。
2. 在 pytest 侧准备参数（例如路由器 MAC 用 `duthost.facts['router_mac']`），写入 JSON 并以 `ptfhost.copy(...)` 传到 PTF 容器。
3. 以 `ptf_runner()` 启动，并将 `log_file` 指向 `/tmp` 下的文档，便于事后查看。
4. 只在 pytest 内需要少量封包时，用 `ptfadapter` 即可，不必另写 PTF 脚本（见上游 ptfadapter README）。

可参考的真实案例：上游 `tests/ipfwd/test_dir_bcast.py`（pytest 侧）。

## 6. 如何运行与调试

### 6.1 sonic-mgmt

依[上游运行文档](https://github.com/sonic-net/sonic-mgmt/blob/master/docs/tests/pytest.run.md)：

```bash
# 在 sonic-mgmt 容器内，于 tests 目录下
./run_tests.sh -n vms-kvm-t0 -d vlab-01 -c example_feature/test_port_oper_status.py \
    -f vtestbed.yaml -i ../ansible/veos_vtb -u
```

- `-n` testbed 名称、`-d` DUT 主机名称、`-c` 测试档或清单、`-f` testbed 档、`-i` inventory、`-u` 略过 utility（pretest／posttest）测试、`-C` 以 `-k` 表达式筛选案例。
- 指定文档清单时不需要 `-t`。
- 文档样式（如 `"snmp/test_*.py"`）必须用双引号包住。

直接使用 pytest：

```bash
cd tests
pytest example_feature/test_port_oper_status.py \
    --inventory ../ansible/veos_vtb --host-pattern vlab-01 \
    --testbed vms-kvm-t0 --testbed_file vtestbed.yaml \
    --log-cli-level info --disable_loganalyzer --skip_sanity
```

重点：自订选项必须放在测试档路径**之后**，否则 pytest 预解析时可能找不到子文档夹 `conftest.py` 定义的选项（上游文档有详细说明）。`--inventory` 要写 `../ansible/veos_vtb`，因为 group_vars 在 `ansible/` 底下。

调试建议：

- `--log-cli-level debug` 查看即时日志。
- 初期开发可加 `--disable_loganalyzer --skip_sanity` 缩短迭代时间，提交前务必拿掉再跑一次。
- 进入 DUT 检查：`ssh admin@10.250.0.101`（KVM 范例 DUT IP 与帐号见上游 KVM 文档；缺省密码为 `password`）。
- 失败时看 DUT 的 `/var/log/syslog`，以及 loganalyzer 在报告中列出的比对行。

### 6.2 sonic-swss vstest

```bash
cd src/sonic-swss/tests
sudo pytest -v test_vlan_example.py                 # 使用缺省映像
sudo pytest --imgname=docker-sonic-vs:my-changes.333 test_vlan_example.py
sudo pytest --keeptb test_vlan_example.py            # 测试后保留容器供调试
sudo pytest test_vlan.py::TestVlan::test_VlanAddRemove
```

以上选项皆出自 `<BI>/src/sonic-swss/tests/README.md`。如需持久化容器，使用 `--dvsname=vs`（可再加 `--forcedvs`）；失败后可 `docker exec -it <容器> bash` 进去看 `redis-cli`、`/var/log/swss`、`/var/log/syslog`。

### 6.3 sonic-utilities

```bash
cd src/sonic-utilities
python3 -m pytest tests/config_vlan_validation_test.py -o addopts="" -v -s
```

`-s` 让测试中的 `print()` 输出显示出来，配合 `result.output` 可快速对照实际消息。

## 7. 惯例与检查清单

提交前逐项确认：

- [ ] 选对层级：能用单元测试就不用 testbed 测试。
- [ ] 文档名：sonic-mgmt 与 sonic-swss 为 `test_*.py`；sonic-utilities 为 `*_test.py`；gtest 为 `*_ut.cpp`。
- [ ] sonic-mgmt：已加 `topology` marker，且拓扑声明与实际需求一致。
- [ ] sonic-mgmt：通用辅助函数放在 `tests/common`，功能专属辅助档放在该功能文档夹。
- [ ] sonic-mgmt：使用 `pytest_assert` 并附上说明消息；等待状态用 `wait_until`，避免固定 `sleep`。
- [ ] sonic-mgmt：测试会改配置时，有还原机制（fixture 的 `yield` 后清理）。
- [ ] sonic-mgmt：通过 flake8（`--max-line-length=120`）与 pre-commit。
- [ ] sonic-mgmt：公开函数有 Google 风格 docstring。
- [ ] sonic-swss vstest：使用 `dvslib` 辅助类别与轮询（`wait_for_*`），不写死等待时间。
- [ ] sonic-swss gtest：新 `*_ut.cpp` 已加入 `Makefile.am`，测试之间不共享未还原的全域状态（参考 `bulker_ut.cpp` 的 `SetUp`／`TearDown` 做法）。
- [ ] sonic-utilities：预期输出放在测试档或 `*_input` 文档夹；修改 `mock_tables/*.json` 前先确认不会破坏既有测试。
- [ ] 本机或 CI 实际跑过至少一次，并在 PR 说明记录跑过哪些指令与结果。
- [ ] 若添加 C 代码的测试辅助，遵守团队 C 编码规范（Allman 大括号、4 空格缩进、只用 `/* */` 注解）。

## 8. 搭建测试环境（Testbed）

本章教你从零建出可以跑第 5.4 节范例 D 的环境。分两条路线：A 为单机 KVM 虚拟 testbed，B 为实体 testbed。依据：[VsSetup](https://github.com/sonic-net/sonic-mgmt/blob/master/docs/testbed/README.testbed.VsSetup.md)、[Setup](https://github.com/sonic-net/sonic-mgmt/blob/master/docs/testbed/README.testbed.Setup.md)、[新 testbed 组态](https://github.com/sonic-net/sonic-mgmt/blob/master/docs/testbed/README.new.testbed.Configuration.md)、[Fanout](https://github.com/sonic-net/sonic-mgmt/blob/master/docs/testbed/README.testbed.Fanout.md)、[Minigraph](https://github.com/sonic-net/sonic-mgmt/blob/master/docs/testbed/README.testbed.Minigraph.md)、[testbed-cli](https://github.com/sonic-net/sonic-mgmt/blob/master/docs/testbed/README.testbed.Cli.md)。上游文档与脚本会演进，以下指令请以你 checkout 的版本为准。

### 8.1 选择路线的决策表

| 考量 | 路线 A：单机 KVM 虚拟 testbed | 路线 B：实体 testbed |
|---|---|---|
| 目的 | 开发测试案例、验证 SONiC 软件行为、CI | 验证真实 ASIC、光模块、平台、性能与数据面 |
| 需要的硬件 | 一台另购的 Linux 服务器（Ubuntu 22.04 或 24.04，含 `/dev/kvm`；规格见第 9 章） | 实体 DUT、leaf（与 root）fanout、测试服务器、管理交换机 |
| 可用拓扑 | `vtestbed.yaml` 内置的 t0、t1-lag、t2 等；PTP 没有范例 | 全部五类，PTP 只能在实体环境 |
| 成本与时间 | 低，数小时可完成 | 高，需接线、VLAN 配置与 lab 档维护 |
| 数据面真实度 | 虚拟交换机，不能代表 ASIC 行为 | 真实 |
| 建议 | 初学者与写新测试案例先走这条 | 要验证硬件或平台时才走这条 |

两条路线的服务器采购规格（CPU、内存、磁盘、网卡、软件版本与验收指令）统一整理在第 9 章。

### 8.2 路线 A：单机 KVM 虚拟 testbed

#### 8.2.0 附注：本机不是搭建目标

testbed 会在另外采购的服务器上搭建，不在这台机器（`${HOME}`）。我对本机只做过唯读检查：它没有 `/dev/kvm`（本机是没开放嵌套虚拟化的 VM）、操作系统为 Ubuntu 20.04（上游要求 22.04 或 24.04），所以路线 A 无法在本机完整运行。本机的 Docker、`sonic-slave-*` 映像仍可用来建置 buildimage 的产物（如 `sonic-vs.img.gz`、`docker-ptf`），再拷贝到新服务器。新服务器的规格与验收请见第 9 章，以下步骤都假设在那台新服务器上运行。

#### 8.2.1 前置需求

- Ubuntu AMD64 22.04 或 24.04。
- 跑 T0 至少要有 20 GB 可用内存；T1 的邻居更多，需要更多。
- 若主机本身是 VM，必须打开嵌套虚拟化。DUT 是 libvirt／QEMU 的 KVM domain，所以一定要有 `/dev/kvm`。缺省 DUT 为 6 GiB RAM 与 4 vCPU（见[sonic.xml.j2](https://github.com/sonic-net/sonic-mgmt/blob/master/ansible/roles/vm_set/templates/sonic.xml.j2)）；vEOS 邻居每台 2 GB（`group_vars/vm_host/main.yml` 的 `memory: 2097152`，单位 KiB）。
- 这些需求的采购规格与验收指令见第 9 章。检查：

```bash
ls -l /dev/kvm
egrep -c '(vmx|svm)' /proc/cpuinfo
```

两者都要有结果（文档存在、计数大于 0）。

#### 8.2.2 步骤 1：取得 sonic-mgmt、创建管理网络

```bash
sudo apt install -y python3 python3-pip openssh-server
git clone https://github.com/sonic-net/sonic-mgmt
cd sonic-mgmt/ansible
sudo -H ./setup-management-network.sh
```

脚本会创建管理 bridge `br1`（文档中为 `10.250.0.1/24`），并安装 `j2cli`、`bridge-utils`、`net-tools`、`ethtool` 等套件。我只读过脚本的前几步，运行前请先完整读过；加 `-d` 会删除既有 bridge。

#### 8.2.3 步骤 2：安装 Docker

依 Docker 官方文档安装 Docker CE，并做 post-install 让一般用户不用 `sudo` 就能运行 `docker`：

```bash
sudo usermod -aG docker $USER
# 注销再登录后验证
docker info
```

`setup-container.sh` 需要 `docker info` 能正常运行。

#### 8.2.4 步骤 3：准备映像目录与邻居映像

```bash
mkdir -p ~/veos-vm/images ~/sonic-vm/images
```

`~/veos-vm` 来自 `ansible/group_vars/vm_host/main.yml` 的 `root_path`（缺省 `veos-vm`），可自行修改。邻居映像三选一：

- cEOS（推荐，较省内存）：自 Arista 网站（需帐号）下载 cEOS64-lab 的 `.tar.xz`，`unxz` 解压后放进 `~/veos-vm/images`。版本必须与 `ansible/group_vars/all/ceos.yml` 一致：VsSetup 范例为 `cEOS64-lab-4.29.3M`，目前 `ceos.yml` 缺省为 `cEOS64-lab-4.32.5M.tar`；上游说明高于测试过的版本可能失败，最新的 4.35.0F 不能用。也可把映像放在 HTTP 服务器，配置 `ceos_image_url` 并把 `skip_ceos_image_downloading` 设为 `false`。
- vEOS：把 `Aboot-veos-serial-8.0.0.iso` 与 `vEOS-lab-4.20.15M.vmdk` 放进 `~/veos-vm/images`（文档名定义在 `group_vars/vm_host/veos.yml`）。
- SONiC 邻居：把 `sonic-vs.img` 放进 `~/veos-vm/images`。

#### 8.2.5 步骤 4：取得 sonic-vs.img（DUT 映像）

方式一，下载上游预建映像：

```bash
wget "https://sonic-build.azurewebsites.net/api/sonic/artifacts?branchName=master&platform=vs&target=target/sonic-vs.img.gz" -O sonic-vs.img.gz
gzip -d sonic-vs.img.gz
cp sonic-vs.img ~/sonic-vm/images
mv sonic-vs.img ~/veos-vm/images
```

方式二，自 `${HOME}/sonic-buildimage` 建置。已验证的 make target 是 `target/sonic-vs.img.gz`：

- `platform/vs/kvm-image.mk` 定义 `SONIC_KVM_IMAGE = sonic-vs.img.gz`，`_MACHINE = vs`，`_IMAGE_TYPE = kvm`。
- `platform/vs/rules.mk` 把它加进 `SONIC_ALL`。
- CI 的 `.azure-pipelines/azure-pipelines-build.yml` 以 `make $BUILD_OPTIONS target/docker-sonic-vs.gz target/sonic-vs.img.gz` 建置。
- `README.md` 把 `PLATFORM=vs` 列为支持的平台，并说明用 `make configure PLATFORM=...` 配置。

```bash
cd ${HOME}/sonic-buildimage
make init
make configure PLATFORM=vs
make target/sonic-vs.img.gz
```

产物在 `target/sonic-vs.img.gz`，之后同上解压并拷贝。目前 `target/` 内没有这个文档。我没有实际运行建置（耗时长且占用大量磁盘），所以这是依 Makefile、CI 与 README 验证的指令，不是实跑结果。建置过程需要网络，且首次建置会花数小时。需要多 ASIC 或单台手动 VM 的作法可参考 `platform/vs/README.vsvm.md`。

#### 8.2.6 步骤 5：创建 sonic-mgmt 容器

```bash
cd sonic-mgmt
./setup-container.sh -n <container-name> -d /data
docker exec --user $USER -it <container-name> bash
```

选项：`-n` 容器名称（必填）、`-i` 映像、`-d` 挂载目录（脚本缺省 `/var/src`，文档使用 `/data`）、`-m` 额插件载、`-p` 连接端口、`-f`、`-v`、`-x`、`-h`、`--enable-debug`。映像缺省使用本机的 `docker-sonic-mgmt`，没有就从 `sonicdev-microsoft.azurecr.io:443/docker-sonic-mgmt:latest` 拉取（最多重试 3 次）。脚本会产生 `~/.ssh/id_ed25519_docker_sonic_mgmt` 密钥，在容器内置立同名用户，并挂载 `/var/run/docker.sock`。容器内 repo 位于 `/data/sonic-mgmt`。

若要自建 `docker-sonic-mgmt`：在 sonic-buildimage 运行 `make configure PLATFORM=generic` 后 `make target/docker-sonic-mgmt.gz`（规则在 `rules/docker-sonic-mgmt.mk`，Dockerfile 范本在 `dockers/docker-sonic-mgmt/Dockerfile.j2`）。

如需 IPv6 测试，编辑 `/etc/docker/daemon.json` 加入 `"ipv6": true`、`"fixed-cidr-v6": "fd00:1::1/64"`、`"experimental": true`、`"ip6tables": true`，然后重启 docker。

#### 8.2.7 步骤 6：编辑 inventory，让容器能 SSH 回主机

以下都在容器内的 `/data/sonic-mgmt/ansible` 编辑：

- `veos_vtb`：把 `STR-ACS-VSERV-01` 的 `ansible_user` 改成你的主机用户，`ansible_host` 为 `172.17.0.1`（`docker0` 地址）。
- `group_vars/vm_host/creds.yml`：配置 `vm_host_user`、`vm_host_password`、`vm_host_become_password`。
- 创建任意内容的 `password.txt`（Ansible Vault 密码档）。
- 在主机运行 `sudo visudo`，加入 `<user> ALL=(ALL) NOPASSWD:ALL`。
- 在主机运行 `sudo chmod 755 /home/<user>`。
- 验证：容器内 `ssh <user>@172.17.0.1` 不用密码即可登录。若 `docker0` 是 `172.18.0.1`，需改 docker bridge 子网并重建容器。

#### 8.2.8 步骤 7：只在 vEOS 或 SONiC 邻居时启动 VM

cEOS 跳过此步。

```bash
./testbed-cli.sh -m veos_vtb -n 4 -k veos start-vms server_1 password.txt
ansible -m ping -i veos_vtb server_1 -u root -k
```

SONiC 邻居将 `-k veos` 换成 `-k vsonic`。EOS 的 ping 密码为 `123456`，SONiC 为 `admin`／`password`。

#### 8.2.9 步骤 8：add-topo

```bash
cd /data/sonic-mgmt/ansible
./testbed-cli.sh -t vtestbed.yaml -m veos_vtb add-topo vms-kvm-t0 password.txt
```

缺省邻居是 cEOS；vEOS 加 `-k veos`，SONiC 加 `-k vsonic`。对 vtestbed 而言，`add-topo` 同时会创建 KVM DUT `vlab-01`（若已存在会重建）。第一次运行会从 registry 拉取 `docker-ptf`，需要网络。看到 `cached_topologies_path file content is empty` 的消息可以忽略。

验证：

```bash
docker ps      # 应有 ceos_vms6-1_VM0100 等、net_vms6-1_*、ptf_vms6-1
virsh list     # 应有 vlab-01
```

#### 8.2.10 步骤 9：deploy-mg 与登录检查

```bash
./testbed-cli.sh -t vtestbed.yaml -m veos_vtb deploy-mg vms-kvm-t0 veos_vtb password.txt
```

此步骤需要网络（如需 proxy，配置 `group_vars/all/env.yml`）。完成后：

```bash
ssh admin@10.250.0.101      # 密码 password
show ip bgp sum              # 应看到 4 个邻居 ARISTA01T1 至 ARISTA04T1（邻居 AS 64600，DUT AS 65100）
```

#### 8.2.11 步骤 10：冒烟测试

```bash
cd /data/sonic-mgmt/tests
./run_tests.sh -n vms-kvm-t0 -d vlab-01 -c bgp/test_bgp_fact.py -f vtestbed.yaml -i ../ansible/veos_vtb
```

使用 SONiC 邻居时加 `-e "--neighbor_type=sonic"`。上游文档预期该文档会跑出 3 组测试并通过。通过后即可回到第 5.4 节跑自己的测试。

#### 8.2.12 步骤 11：清除

```bash
./testbed-cli.sh -t vtestbed.yaml -m veos_vtb -k ceos remove-topo vms-kvm-t0 password.txt
```

会移除 PTF、cEOS 与 KVM DUT；`-k` 要与创建时一致。其他常用动词（出自 `testbed-cli.sh` 的 usage）：`start-vms`、`stop-vms`、`redeploy-topo`、`renumber-topo`、`connect-topo`、`gen-mg`、`test-mg`、`announce-routes`、`restart-ptf`、`refresh-dut`、`config-vm`、`install-image`、`cleanup-vmhost`。

#### 8.2.13 常见失败与排除

| 症状 | 可能原因与处理 |
|---|---|
| 容器内 SSH 到 `172.17.0.1` 失败 | docker bridge 子网为 `172.18.x.x`（改子网并重建容器）；`visudo` 没设；`/home/<user>` 权限不是 755；密码错误 |
| `add-topo` 的 cEOS 启动失败 | `ceos.yml` 的版本与 `~/veos-vm/images` 内的文档不一致；使用了不支持的新版（4.35.0F） |
| 找不到映像 | 映像没放进 `root_path/images`（缺省 `~/veos-vm/images`） |
| KVM DUT 起不来 | 没有 `/dev/kvm`；主机是 VM 但没开嵌套虚拟化 |
| IPv6 测试失败 | `daemon.json` 没设 IPv6 |
| 拉取 `docker-ptf` 或 deploy-mg 逾时 | 没网络或需要 proxy（`group_vars/all/env.yml`） |
| 内存不足 | T0 至少 20 GB 可用；改用 cEOS 并减少同时存在的 testbed |

### 8.3 路线 B：实体 testbed

#### 8.3.1 硬件清单

| 项目 | 规格 | 说明 |
|---|---|---|
| DUT | 要测的 SONiC 设备 | T0／T1／PTF 常见为 32 端口；PTP 要两台 |
| Leaf fanout | 端口数多于 DUT，支持 LACP／LLDP 穿透与 802.1Q tunneling | 上游举例 Arista 7260 |
| Root fanout | 选用，规格同 leaf | 只有一个 testbed 时可省略 |
| 测试服务器 | Ubuntu 22.04 或 24.04 amd64；参考规格 Dell 730、2×18 核、192 GB RAM、2×500 GB 磁盘、Mellanox MT27700 | 参考规格非强制；规模小的拓扑可大幅降低；规格分级见第 9.3 节 |
| 管理网络 | 一台管理交换机与网段 | 服务器、VM、PTF、DUT 管理端口、fanout 管理端口都接入 |
| 选配 | Console server、PDU | 对应 `sonic_lab_console_links.csv`、`sonic_lab_pdu_links.csv` |

#### 8.3.2 ASCII 接线图

```text
              管理网络 (br1, 10.250.0.0/24, 经管理交换机)
   +----------------+---------------+---------------+-----------+
   |                |               |               |           |
[测试服务器]     [Root fanout]   [Leaf fanout]      [DUT]    [PDU/Console]
 mgmt NIC          mgmt            mgmt           mgmt port     (选用)
 trunk NIC ===== trunk ======= trunk (VLAN 100-131)
 (如 p4p1)       (选用)            |
                                  | DUT 面向端口 = access, 每端口唯一 VLAN
                                  +-- Eth1  VLAN 100 ---- DUT Ethernet0
                                  +-- Eth2  VLAN 101 ---- DUT Ethernet1
                                  +-- ...
                                  +-- Eth32 VLAN 131 ---- DUT Ethernet31
```

若没有 root fanout，测试服务器的 trunk NIC 直接接 leaf 的 trunk 端口。连接规则为：DUT 端口接 leaf 的 access 端口（每端口唯一 VLAN），leaf 的 trunk 接 root（或直接接服务器），root 的 trunk 接服务器 NIC。PTP 拓扑则是两台 DUT 的数据端口彼此直连，不经 fanout。

#### 8.3.3 步骤 1：测试服务器基础环境

```bash
sudo apt install -y python3 python-is-python3 python3-pip curl git openssh-server make
sudo pip3 install j2cli
sudo usermod -aG docker $USER      # 先安装 Docker CE，再注销登录
```

可选择关闭防火墙：`sudo ufw disable`。

#### 8.3.4 步骤 2：准备映像

- cEOS（实体环境推荐，内存与内核交互较少）：先 `docker pull debian:jessie`（必要时把 `debian:latest` tag 成 `jessie`），取得 cEOS tar 后 `docker import <tar> ceosimage:<version>`。版本需与 `ansible/group_vars/all/ceos.yml` 一致（`ceos_image_filename`、`ceos_image_orig`、`ceos_image`），本地已有映像时设 `skip_ceos_image_downloading: true`。
- `docker-ptf`：`add-topo` 会自动从缺省 registry `sonicdev-microsoft.azurecr.io:443` 拉取。也可在 sonic-buildimage 自建：`make configure PLATFORM=vs` 然后 `make target/docker-ptf.gz`；使用自建 registry 时修改 `ansible/vars/docker_registry.yml`。
- `docker-sonic-mgmt`：下载预建映像（建议），或 `make configure PLATFORM=generic` 加 `make target/docker-sonic-mgmt.gz`，然后依第 8.2.6 节运行 `setup-container.sh`。

#### 8.3.5 步骤 3：管理网络与 OVS（测试服务器网络）

管理 bridge `br1` 承载服务器、VM、PTF 与 DUT 的管理流量。配置方式二择一：

- 在 netplan（或 ifupdown）将管理网卡桥接到 `br1`，范例地址为 `10.250.0.245/24`，gateway `10.250.0.1`，范例使用的网卡名为 `ma0`。
- 暂时性做法：`sudo -H ./ansible/setup-management-network.sh`（重开机后消失）。

因为 `br1` 持有 IP，建议服务器另有第二张网卡连到实验室网络。配置后重开机确认网络仍正常。

数据面的 OVS bridge 不用手动创建：`add-topo` 会依 `topo_*.yml` 与 `external_port`（trunk 网卡，如 `p4p1`）创建 VLAN 子接口并接上 OVS bridge、配置第 3.1 节所述的 flow 规则。所以你要做的是：在 `ansible/host_vars/STR-ACS-SERV-01.yml` 把 `external_port` 设为接 fanout trunk 的网卡，并确认服务器上有 Open vSwitch（`ovs-vsctl`）。我没有逐一确认哪个 role 会自动安装 OVS，若 `add-topo` 回报缺少，请依 [Setup 文档](https://github.com/sonic-net/sonic-mgmt/blob/master/docs/testbed/README.testbed.Setup.md)补装。

#### 8.3.6 步骤 4：配置 fanout 的 VLAN trunk

- Arista leaf：使用 `ansible/roles/fanout` playbook。先把 `roles/fanout/template/rc.eos` 以 `scp` 传到交换机 flash，并备妥 eosadmin shell 的认证（`fanout_admin_user`、`fanout_admin_password`）。
- 其他厂商：手动配置。每个 DUT 面向端口设为 access，并指定唯一 VLAN；上行端口设为 trunk，允许所有这些 VLAN。VLAN 范围为「最小 VLAN + 端口数 - 1」，例如 32 端口从 100 开始就是 100 至 131。或者直接部署一份一般的 L2 交换机配置。
- 注意：`testbed-cli.sh add-topo` 的最后一步会尝试在 root fanout 重新部署 VLAN 范围，而且只支持 Arista。使用其他厂商时，请把这一步注解掉并手动配置 VLAN。

#### 8.3.7 步骤 5：inventory、testbed.yaml、lab 档与认证

在 sonic-mgmt 容器内编辑下列文档（字段说明见[组态文档](https://github.com/sonic-net/sonic-mgmt/blob/master/docs/testbed/README.new.testbed.Configuration.md)）：

| 文档 | 内容 |
|---|---|
| `ansible/veos` | 服务器管理 IP、VM 管理 IP（在管理网段内）、`vms_1` VM 清单与 ID |
| `ansible/group_vars/vm_host/creds.yml` | `vm_host_user`、`vm_host_password`、`vm_host_become_password` |
| `ansible/host_vars/STR-ACS-SERV-01.yml` | `external_port`（trunk 网卡）、`mgmt_gw`、`mgmt_prefixlen`、`mgmt_bridge` |
| `ansible/group_vars/eos/creds.yml` | `ansible_user: admin`、`ansible_password` |
| `ansible/group_vars/all/ceos.yml` | cEOS 映像文档名与名称 |
| `ansible/lab` | DUT 群组（如 `sonic_sn2700_40`，含 `hwsku`、`iface_speed`、`ansible_host`、`model`、`serial`、`base_mac`、`syseeprom_info`）、fanout、pdu、server、`ptf` 群组 |
| `ansible/files/sonic_lab_devices.csv`、`sonic_lab_links.csv` | 设备清单与连接（DUT 端口到 fanout 端口与其 Access VLAN；fanout trunk 到服务器网卡与 Trunk VLAN 范围） |
| `sonic_lab_pdu_links.csv`、`sonic_lab_console_links.csv`、`sonic_lab_bmc_links.csv` | 选用的 PDU、console、BMC 连接 |
| `ansible/testbed.yaml` | testbed 项目（第 3.3 节字段） |
| `ansible/password.txt` | Ansible Vault 密码档 |

要点：

- `ansible/lab` 的 `ptf` 群组 `ansible_host` 必须等于 testbed 项目的 `ptf_ip`；PTF 帐密为 root／root。
- 每个 DUT 要有 `iface_speed`，否则 `iface_namingmode` 测试会失败。
- `hwsku` 必须存在于 `ansible/module_utils/port_utils.py`，连接端口速率需有对应的 `port_config.ini`。
- 连接档填完后，用 `creategraph.py` 产生 `lab_connection_graph.xml`。
- 也可改用单一文档 `testbed.yaml` 加 `TestbedProcessing.py` 的方式（组态文档列有 device_groups、devices、host_vars、veos_groups、veos、testbed、topology、docker_registry 等区段）。
- 验证服务器可被 Ansible 控制：`ansible -m ping -i veos vm_host_1`。

#### 8.3.8 步骤 6：部署拓扑与 minigraph

```bash
./testbed-cli.sh -t testbed.yaml -m veos -k ceos add-topo <conf-name> password.txt
docker ps      # 检查 ceos_<group>_VM0100..、net_<group>_VM0100..、ptf_<group>
docker exec -it ceos_vms1-1_VM0101 Cli     # 进 cEOS，运行 show version
./testbed-cli.sh -t testbed.yaml -m veos deploy-mg <conf-name> lab password.txt
```

`start-vms`、`start-topo-vms` 只有使用 vEOS／vSONiC 才需要。`deploy-mg` 会读取 testbed、`vars/topo_<topo>.yml` 与 veos VM 信息，渲染 minigraph 并部署到 DUT、加载并保存 config_db。

#### 8.3.9 步骤 7：连接真实 DUT 与验证

1. 确认 DUT 已安装 SONiC 映像，管理 IP、帐密与 `ansible/lab` 的配置一致，可由 sonic-mgmt 容器 SSH。
2. 确认 DUT 各端口 link up（`show interfaces status`），且 fanout 对应端口为正确的 access VLAN。
3. 检查邻居 BGP：

```bash
show ip bgp sum      # t0 应有 4 个邻居 ARISTA01T1 至 04T1
```

4. 跑冒烟测试：

```bash
cd tests
./run_tests.sh -n <conf-name> -d <dut> -c bgp/test_bgp_fact.py -f testbed.yaml -i ../ansible/lab
```

5. 通过后再跑第 5.4 节的范例。
6. 清除：`./testbed-cli.sh -t testbed.yaml -m veos -k ceos remove-topo <conf-name> password.txt`。

本章尚未涵盖或无法确认之处：PTP 没有上游的虚拟或部署范例；单一 cEOS 容器的内存用量在我取得的文档中找不到；T2 的服务器规格上游未明列（采购相关的限制见第 9.8 节）。

## 9. 服务器环境需求与采购指南

本章回答「要买什幺样的服务器来跑 sonic-mgmt testbed」。testbed 不会在 `${HOME}` 这台机器搭建，而是在另外采购的服务器上运行第 8 章的步骤。

### 9.1 先读这一节：数字的可信度标示

上游文档对服务器规格给的数字很少，大部分规格要自己推算。为了避免把估计当成事实，本章每个数字都标示来源：

| 标示 | 意思 |
|---|---|
| 【上游】 | 直接出自 sonic-mgmt 上游文档或原代码，附链接 |
| 【推导】 | 用上游数字（例如 VM 数量、每台 VM 的内存）做的算术，算式会写出来 |
| 【估计】 | 作者的经验估计，上游没有对应数字，采购前请自行验证 |

本章引用的上游来源：

- [README.testbed.VsSetup.md](https://github.com/sonic-net/sonic-mgmt/blob/master/docs/testbed/README.testbed.VsSetup.md)：KVM 虚拟 testbed 的主机需求（20 GB、嵌套虚拟化、Ubuntu 22.04／24.04）。
- [README.testbed.Setup.md](https://github.com/sonic-net/sonic-mgmt/blob/master/docs/testbed/README.testbed.Setup.md)：实体 testbed 的服务器准备，建议 cEOS。
- [README.testbed.Overview.md](https://github.com/sonic-net/sonic-mgmt/blob/master/docs/testbed/README.testbed.Overview.md)：硬件需求（参考服务器规格、fanout、实体连接）与五类拓扑。
- [README.testbed.FAQ.md](https://github.com/sonic-net/sonic-mgmt/blob/master/docs/testbed/README.testbed.FAQ.md)：最低硬件需求（92 GB 内存，文档较旧）。
- [ansible/group_vars/vm_host/main.yml](https://github.com/sonic-net/sonic-mgmt/blob/master/ansible/group_vars/vm_host/main.yml)：vEOS 每台 `memory: 2097152`（KiB，即 2 GiB）。
- [ansible/roles/vm_set/templates/arista.xml.j2](https://github.com/sonic-net/sonic-mgmt/blob/master/ansible/roles/vm_set/templates/arista.xml.j2)：vEOS 每台 `vcpu` 为 1。
- [ansible/roles/vm_set/templates/sonic.xml.j2](https://github.com/sonic-net/sonic-mgmt/blob/master/ansible/roles/vm_set/templates/sonic.xml.j2)：KVM DUT 缺省 6 GiB、4 vCPU，其他 SKU 见 9.2.3。
- [ansible/roles/vm_set/tasks/host_setup.yml](https://github.com/sonic-net/sonic-mgmt/blob/master/ansible/roles/vm_set/tasks/host_setup.yml)：主机会安装的套件（`openvswitch-switch`、`libvirt-*`、`qemu-system-x86` 等）。
- [ansible/roles/vm_set/tasks/docker.yml](https://github.com/sonic-net/sonic-mgmt/blob/master/ansible/roles/vm_set/tasks/docker.yml)：Docker CE 的 apt 来源（jammy 与 noble）。
- [ansible/vtestbed.yaml](https://github.com/sonic-net/sonic-mgmt/blob/master/ansible/vtestbed.yaml)：虚拟 testbed 项目与 IP。

### 9.2 单一台服务器要扛多少资源

#### 9.2.1 先决定三件事

1. 路线：A（单机 KVM 虚拟 testbed）或 B（实体 testbed，见 8.1）。两条路线都需要一台 Linux 服务器，B 另外需要 DUT 与 fanout。
2. 要跑哪一类拓扑：邻居 VM 数量决定内存与 CPU（见 9.2.2，数字与 3.2 节的拓扑总表一致）。
3. 邻居种类：vEOS、cEOS 或 SONiC VM（见 9.2.3）。

#### 9.2.2 各拓扑的资源估算（以 vEOS 邻居为上限）

算式：VM 内存 = 邻居 VM 数 × 2 GiB（【上游】每台 vEOS `memory: 2097152` KiB）+ DUT 6 GiB（【上游】缺省 KVM DUT）；vCPU = 邻居 VM 数 × 1（【上游】vEOS `vcpu` 为 1）+ 4（【上游】缺省 KVM DUT 4 vCPU）。这是 VM 本身的分配量，不含主机 OS、PTF 与 Docker 开销（那部分为【估计】约 4 至 8 GiB）。

邻居 VM 数量取自 3.2 节的拓扑总表与变体清单，未重新统计。

| 拓扑 | 邻居 VM 数（见 3.2） | VM 内存【推导】 | vCPU 配置量【推导】 | 说明 |
|---|---|---|---|---|
| 单一 DUT、不需邻居（`ptf32`、`ptf64`、`ptf8`） | 0 | 6 GiB（只有 DUT） | 4 | 我没有确认 vtestbed 是否有现成的 PTF 项目，请以你 checkout 的 `vtestbed.yaml` 为准 |
| T0（`t0`） | 4 | 4×2+6 = 14 GiB | 4+4 = 8 | 上游 VsSetup 要求至少 20 GB 可用内存【上游】，与此推导兼容（含主机与 PTF 开销） |
| T0（`t0-56`、`t0-80`） | 8 | 8×2+6 = 22 GiB | 8+4 = 12 | 超过 20 GB，以此为准 |
| T1（`t1-lag`） | 24 | 24×2+6 = 54 GiB | 24+4 = 28 | |
| T1（`t1`） | 32 | 32×2+6 = 70 GiB | 32+4 = 36 | 上游没有 T1 专属的服务器规格 |
| T1（`t1-64`） | 64 | 64×2+6 = 134 GiB | 64+4 = 68 | 单机以 vEOS 跑会很吃重，建议改 cEOS 或分散到多台 |
| T2（`t2`，机箱） | 72 | 72×2 = 144 GiB，再加 3 张 linecard 与 1 个 supervisor 的 DUT 内存 | 72 加 DUT | 4 个 DUT 节点在 vtestbed 的各自配置我没有逐一验证；DUT 部分【估计】再加 24 至 32 GiB，合计约 170 GiB 以上；上游未给 T2 服务器规格 |
| T2 单节点（`t2_single_node_min`） | 5 | 5×2+6 = 16 GiB | 5+4 = 9 | 5 台 VM 取自 3.2 节；适合虚拟环境练习 |
| PTP | 0 | 不适用 | 不适用 | 两台实体 DUT 互连，没有虚拟化负载；服务器只需管理网络（见 8.3.2） |

注意：

- 上表以 vEOS 为上限。cEOS 用容器，上游只说它比较省内存并减少与内核的交互（【上游】Setup 文档的说法），但**我在上游文档中找不到每个 cEOS 容器的内存数字**，因此用 cEOS 能省多少是未知数。规划时请先用 vEOS 的数字当保守上限，实测后再调整。
- vCPU 是配置量，不是实体内核需求。KVM 允许超配（overcommit），所以实体内核可以少于 vCPU 合计；但 BGP 收敛与路由加载时 VM 会同时吃 CPU，超配过高会让测试逾时。内核数的建议见 9.3（【估计】）。

#### 9.2.3 邻居与 DUT 种类的差异

| 种类 | CPU／内存（上游数字） | 资源相对大小 | 备注 |
|---|---|---|---|
| vEOS（KVM VM） | 每台 1 vCPU、2 GiB【上游】 | 大 | 需要 `/dev/kvm`；映像为 `Aboot-veos-serial-8.0.0.iso` 加 `vEOS-lab-4.20.15M.vmdk` |
| cEOS（Docker 容器） | 上游没有数字 | 小于 vEOS（【上游】只有定性描述） | VsSetup 与 Setup 都建议使用（Setup 针对实体环境）；版本要与 `ceos.yml` 一致，最新的 4.35.0F 不能用 |
| SONiC VM 邻居（`-k vsonic`） | 我没有确认邻居 SONiC VM 的配置；若与 DUT 的缺省相同（6 GiB、4 vCPU，【上游】`sonic.xml.j2`），32 台就是 192 GiB（【推导】32×6） | 大（待验证） | 这个假设未验证；规划时先假设比 vEOS 大，请以小拓扑为主，并在实测后修正 |
| KVM DUT 缺省 | 6 GiB、4 vCPU【上游】 | | `sonic.xml.j2` 的其他 SKU：`msft_four_asic_vs` 8 GB／10 vCPU、`msft_multi_asic_vs` 8 GB／16 vCPU、`cisco-8101-p4` 8 GB／6 vCPU、`Nokia-IXR7250E-36x400G` 12 GB／5 vCPU、SUP-10 4 GB／3 vCPU |

多 ASIC 与机箱类的虚拟 DUT 会把 vCPU 与内存需求拉高。例如 `sonic.xml.j2` 中的 `msft_four_asic_vs` 一台 DUT 就要 8 GB／10 vCPU【上游】；vtestbed 的 `vms-kvm-four-asic-t1-lag` 是四 ASIC 项目，但它实际套用哪个 SKU 我没有逐一验证。

#### 9.2.4 同时跑多组 testbed

同一台服务器上的每一组 testbed 各自有 `group-name`、PTF、邻居 VM 与 DUT，资源需求是累加的，不共用：

- 总内存 ≈ 各组 VM 内存之和 + 主机开销一次（【推导】）。例如 2 组 `t0` ≈ 2×14 = 28 GiB，加主机开销。
- 每组的 `vm_base` 范围不可重叠（上游 Consistency Rule，见 3.3 节），所以规划 VM 名称与管理 IP 时要预留空间。
- 实体环境中，一台测试服务器可服务多个 testbed（每组 DUT 各用一段 VLAN，例如 100 至 131 与 132 至 163，见 3.1 节）。上游的参考服务器（192 GB、2×18 核）设计目的就是这种共用情境，不过上游并未说明它同时服务几组。

### 9.3 规格分级（最低与建议）

来源标示规则见 9.1。「最低」表示能跑起来的底线，「建议」表示留有余裕、较不容易因资源不足而出现不稳定。

#### 9.3.1 路线 A：虚拟 KVM testbed（单一服务器）

| 项目 | 最低 | 建议 | 依据 |
|---|---|---|---|
| 情境 | 单一 DUT，一组 `t0`（cEOS 或 vEOS 皆可） | 一组 `t1-lag`／`t1`，或同时 2 至 3 组 `t0` | |
| CPU 虚拟化 | 必须有 VT-x 或 AMD-V，且 `/dev/kvm` 可用 | 同左，建议直接用实体机（bare metal） | 【上游】VsSetup：主机若为 VM 必须支持嵌套虚拟化；DUT 是 libvirt／QEMU KVM domain（`sonic.xml.j2`） |
| 实体内核 | 8 核 | 16 核（`t0` 多组）；24 至 36 核（`t1`） | 【估计】；依据是 9.2.2 的 vCPU 配置量（`t0` 8、`t1` 36）与可超配的前提 |
| 内存 | 24 GB 以上（可用 20 GB 为上游底线） | `t0` 多组：64 GB；`t1-lag`／`t1`：128 GB | 20 GB 为【上游】VsSetup；`t1` 70 GiB 为【推导】；其余为【估计】 |
| 磁盘 | 200 GB SSD | 500 GB NVMe SSD | 【估计】：上游没有磁盘容量数字。依据：cEOS tar、`sonic-vs.img`、`docker-ptf`、`docker-sonic-mgmt` 与每台 VM 的磁盘映像；映像大小请以实际下载档为准 |
| 网卡 | 1 张，连到管理网络 | 2 张（一张给 `br1` 管理，一张连实验室网络） | 【上游】Setup 建议第二张网卡，因 `br1` 持有 IP；速率上游未规定，1 GbE 即可（【估计】） |
| 操作系统 | Ubuntu 22.04 或 24.04 amd64 | 同左（22.04 与 24.04 择一，团队内统一） | 【上游】VsSetup 与 Setup |

#### 9.3.2 路线 B：实体 testbed 的测试服务器

| 项目 | 最低 | 建议 | 依据 |
|---|---|---|---|
| 情境 | 一组 DUT，`t0` 级以下，使用 cEOS | 多组 DUT 共用，或 `t1` 以上 | |
| CPU／内存 | 内存 92 GB（FAQ 的底线，文档较旧） | 上游参考规格：2 颗 CPU 各 18 核（共 36 核）、192 GB | 【上游】Overview（参考规格，明确写「非强制」）与 FAQ |
| 磁盘 | 500 GB | 2×500 GB（上游参考规格） | 参考规格为【上游】Overview；最低值为【估计】 |
| 网卡 | 一张管理网卡，加一张 trunk 网卡（接 fanout） | 管理与 trunk 分开；trunk 网卡上游参考为 Mellanox MT27700（ConnectX-4 系列） | 网卡型号为【上游】Overview；速率上游没有规定。trunk 要承载所有 DUT 端口的测试流量，速率至少不低于单一 DUT 端口的速率，建议 25 GbE 以上（【估计】） |
| 其他 | 一台管理交换机与 L2 fanout | 见 8.3.1 硬件清单 | 【上游】Overview 与 FAQ |

两个数字（20 GB 与 92 GB）看起来差很多，原因是它们的情境不同：20 GB 是虚拟 `t0` 一组的底线（VsSetup），92 GB 是 FAQ 针对实体环境的通用最低需求（文档较旧，没有说明它是针对哪个拓扑）。采购时请依你要跑的最大拓扑与同时组数来选，而不是取两者之一。

#### 9.3.3 一眼看懂的采购建议（作者的建议，属【估计】）

| 你的目标 | 建议采购 | 理由 |
|---|---|---|
| 学习、写新测试案例（本文主要目标） | 路线 A，16 核、64 GB、500 GB NVMe、Ubuntu 22.04／24.04、bare metal | 够跑 2 至 3 组 `t0` 或单组 `t1-lag`（54 GiB 加开销），且不用担心嵌套虚拟化 |
| 团队共用、同时多组 `t1` | 路线 A 或 B，2 颗 18 核、192 GB 以上，2×500 GB 以上 SSD | 对齐上游参考规格 |
| 验证真实硬件（光模块、平台） | 路线 B：上游参考服务器等级，加 fanout 与管理交换机 | 见 8.3 |
| T2 机箱 | 另行评估，可能需要多台服务器 | 上游没有 T2 服务器规格；72 台 VM 单机需 144 GiB 以上 |

### 9.4 操作系统与软件版本

| 项目 | 要求 | 依据与备注 |
|---|---|---|
| 操作系统 | Ubuntu 22.04 或 24.04，amd64 | 【上游】VsSetup、Setup。20.04 不符合 |
| 内核 | 上游没有指定版本；使用该 Ubuntu 版本的缺省内核，并确认有 `kvm`、`kvm_intel` 或 `kvm_amd` 模块 | 【估计】；已知 sonic-swss 的 vstest 需要 `team` kernel module（见 2.4 节），若服务器也要跑 vstest 请确认 `modinfo team` 有结果 |
| Docker | Docker CE（`docker-ce`、`docker-ce-cli`、`containerd.io`），一般用户在 `docker` 群组内 | 【上游】Setup 与 `docker.yml`（apt 来源 jammy／noble）。上游没有固定 Docker 版本，用官方 repo 的稳定版 |
| Open vSwitch | `openvswitch-switch` | 【上游】`host_setup.yml` 会安装；VM 与 PTF 的数据面 bridge 都是 OVS（见 3.1 节） |
| libvirt／QEMU | `libvirt-clients`、`libvirt-daemon-system`、`python3-libvirt`、`python3-lxml`、`qemu-system-x86` | 【上游】`host_setup.yml`；KVM DUT 与 vEOS 需要 |
| 网络工具 | `net-tools`、`bridge-utils`、`ethtool` | 【上游】`host_setup.yml` 与 `setup-management-network.sh` |
| Python | 系统的 `python3`、`python3-pip`；`j2cli`（`setup-management-network.sh` 会装 `j2cli==0.3.10`，Setup 文档用 `pip3 install j2cli`） | 【上游】VsSetup、Setup。`vm_set` role 另外会创建一个 venv（`ensure_python_env.yml`，使用 `uv`），其中的套件版本以 checkout 为准，这部分我只确认有 venv 这件事 |
| sysctl | `net.bridge.bridge-nf-call-*` 由 `host_setup.yml` 配置 | 【上游】`host_setup.yml` |
| Hugepages | 我在上游的 VsSetup、Setup 与上述 role 文档中没有找到 hugepages 的要求 | 【估计】：不需要事先配置；若 DUT 或平台映像另有要求，以该映像文档为准 |
| 防火墙 | 可选择关闭：`sudo ufw disable` | 【上游】Setup（选用） |
| SSH 与 sudo | 打开 `openssh-server`；容器要能免密码 SSH 回主机，并配置 `NOPASSWD:ALL` | 【上游】VsSetup（见 8.2.7） |
| cEOS 映像 | 版本要与 `ansible/group_vars/all/ceos.yml` 一致；需要 Arista 帐号下载 | 【上游】VsSetup；`debian:jessie` 为 cEOS 实体流程的前置映像（Setup） |

### 9.5 网络需求

#### 9.5.1 管理网络与 IP 规划

| 项目 | 虚拟 KVM（路线 A） | 实体（路线 B） |
|---|---|---|
| 管理 bridge | `br1`，由 `setup-management-network.sh` 创建，地址 `10.250.0.1/24`（【上游】VsSetup） | `br1`，管理网卡桥接进去，范例 `10.250.0.245/24`、gateway `10.250.0.1`（【上游】Setup） |
| DUT 管理 IP | 例如 `vlab-01` 为 `10.250.0.101`（【上游】`vtestbed.yaml`） | 依 `ansible/lab` 的 `ansible_host` |
| PTF 管理 IP | 例如 `vms-kvm-t0` 为 `10.250.0.102/24`（【上游】`vtestbed.yaml`） | 依 `testbed.yaml` 的 `ptf_ip`，且要等于 `lab` 的 `ptf` 群组 `ansible_host` |
| 邻居 VM 管理 IP | 由 `veos_vtb` 定义 | 由 `ansible/veos` 定义，须在管理网段内（【上游】Setup） |
| 容器回连主机 | `172.17.0.1`（`docker0`）；若 `docker0` 是 `172.18.0.1` 要改 docker bridge 子网（【上游】VsSetup） | 同左，视 inventory 而定 |

规划要点：

- 预留的地址：`10.250.0.0/24` 是上游范例，不是强制值，但改了要同步改 `veos_vtb`、`vtestbed.yaml`、`ansible/lab` 等文档。若实验室内已有网段与它重叠，请先换掉，并避免与 Docker 缺省的 `172.17.0.0/16`（也可能用到 `172.18.0.0/16`）冲突（【估计】的建议，依据 8.2.7 的 docker0 子网问题）。
- 管理网络上要容纳：服务器、每台邻居 VM、PTF、DUT，以及（实体）fanout 与 PDU／console。一组 `t1` 就有 32 台邻居，IP 地址要够用。
- IPv6：若要跑 IPv6 测试，需要在 `/etc/docker/daemon.json` 激活（见 8.2.6）；IPv6-only 管理网络为选配（VsSetup）。

#### 9.5.2 数据面网络（只有实体 testbed 需要）

| 项目 | 需求 | 依据 |
|---|---|---|
| Trunk 网卡 | 测试服务器的一张网卡接 root fanout（或无 root 时直接接 leaf）的 802.1Q trunk；在 `host_vars/STR-ACS-SERV-01.yml` 设为 `external_port` | 【上游】Overview、Setup |
| VLAN | 每个 DUT 端口一个唯一 VLAN，范围 = 最小 VLAN + 端口数 − 1，例如 32 端口为 100 至 131 | 【上游】Overview、组态文档 |
| 服务器端 bridge | OVS bridge 由 `add-topo` 创建 | 【上游】Overview（见 8.3.5） |
| 管理网卡与 trunk 网卡 | 分开两张，不要共用 | 【估计】的建议：`br1` 是管理，trunk 要整张网卡交给 VLAN 子接口 |
| 第二张网卡 | 连到实验室网络，因为 `br1` 持有管理 IP | 【上游】Setup |

虚拟 KVM（路线 A）完全在主机内部用 OVS 与 bridge 连接，不需要 trunk 网卡与 fanout。

#### 9.5.3 互联网、registry 与脱机镜像

| 需要连到的资源 | 用途 | 依据 |
|---|---|---|
| Ubuntu 的 apt 来源 | 安装套件与 `setup-management-network.sh` | 【上游】VsSetup |
| `download.docker.com` | Docker CE 的 apt 来源 | 【上游】`docker.yml`、Setup |
| `sonicdev-microsoft.azurecr.io:443` | 拉取 `docker-ptf`、`docker-sonic-mgmt` | 【上游】VsSetup、Setup、`setup-container.sh` |
| `sonic-build.azurewebsites.net` | 下载预建 `sonic-vs.img.gz` | 【上游】VsSetup |
| Arista 下载网站（需帐号） | cEOS 或 vEOS 映像 | 【上游】VsSetup |
| GitHub | `git clone sonic-mgmt` | 【上游】VsSetup |
| 外部互联网（`deploy-mg` 期间） | VsSetup 注明 `deploy-mg` 需要网络，需要连到哪些站点上游没有列出 | 【上游】VsSetup |
| Docker Hub（`debian:jessie`） | cEOS 实体流程的前置映像 | 【上游】Setup |

脱机或受限网络的做法：

- 自建 Docker registry（`registry:latest`），并修改 `ansible/vars/docker_registry.yml`（【上游】Setup）。
- cEOS 映像放在内部 HTTP 服务器，配置 `ceos_image_url` 并把 `skip_ceos_image_downloading` 设为 `false`，或预先放进 `images/` 并设为 `true`（【上游】VsSetup、Setup）。
- 需要 proxy 时配置 `ansible/group_vars/all/env.yml`（【上游】VsSetup）。
- apt、pip 的内部镜像：上游没有提供，这是常见的企业作法（【估计】的建议）。

磁盘上要留空间放：cEOS tar、vEOS 的 `.iso` 与 `.vmdk`、`sonic-vs.img`、`docker-ptf`、`docker-sonic-mgmt` 与多版本映像。`root_path`（缺省 `~/veos-vm`）与 Docker 的数据目录（缺省 `/var/lib/docker`）最好放在容量最大的磁盘，必要时改 `root_path` 或搬动 Docker data-root（【估计】的建议）。

### 9.6 采购检查清单

采购前请对供应商确认下列各项；勾选代表已确认。

- [ ] CPU 支持虚拟化：Intel VT-x 或 AMD-V，且 BIOS 缺省已激活（不是出货后才发现没开）。
- [ ] 建议买实体机（bare metal）。若是云端或虚拟化平台的主机，确认平台支持嵌套虚拟化并提供 `/dev/kvm`（【上游】VsSetup 要求）。
- [ ] 内核数与内存符合 9.3 的等级（路线 A 建议 16 核／64 GB 起；多组 `t1` 以上看 192 GB 等级）。
- [ ] 磁盘为 SSD 或 NVMe，容量依 9.3 的估计；若可选，预留扩充槽。
- [ ] 网卡数量：至少一张管理网卡；实体 testbed 另加 trunk 网卡（参考 Mellanox MT27700）与连实验室网络的第二张网卡。
- [ ] 交付的操作系统为 Ubuntu 22.04 或 24.04 amd64（或允许你自行重装）。
- [ ] 取得 root 或 sudo 权限、可 SSH 登录。
- [ ] 网络：管理网段与 IP 范围已规划，不与 `10.250.0.0/24`、`172.17.0.0/16`、`172.18.0.0/16` 冲突（或你已决定要改）。
- [ ] 能连到第 9.5.3 节列出的 registry 与下载站，或已备妥脱机镜像与 proxy。
- [ ] 取得 Arista 帐号可下载 cEOS／vEOS，且已确认要用的版本与 `ceos.yml` 一致。
- [ ] 实体 testbed：fanout 交换机的端口数多于 DUT、支持 LACP／LLDP 穿透与 QinQ（上游举例 Arista 7260）；有管理交换机。
- [ ] 机房条件：机架空间、电力、散热、网络孔位已确认。
- [ ] 保固与到货验收时程已确认，便于发现规格不符时退换。

### 9.7 到货后的验收指令（皆为唯读）

以下指令只读取信息，不会修改系统。请在**新采购的服务器**上运行（不是 `${HOME}` 这台）。`kvm-ok` 需要 `cpu-checker` 套件，若没有安装可略过，用其他指令取代。网络连接检查会对外送出 HTTP 请求，但不含任何项目数据。

```bash
# 1. 操作系统与内核
cat /etc/os-release
uname -r
systemd-detect-virt            # none 代表实体机；kvm、vmware 等代表本身是 VM

# 2. CPU 与虚拟化旗标
lscpu | egrep 'Model name|^CPU\(s\)|Thread|Core|Socket|Virtualization'
egrep -c '(vmx|svm)' /proc/cpuinfo     # 大于 0 才行
ls -l /dev/kvm                          # 必须存在
kvm-ok                                  # 需 cpu-checker，输出 KVM acceleration can be used
cat /sys/module/kvm_intel/parameters/nested 2>/dev/null   # Intel 且本机是 VM 时看嵌套虚拟化是否为 Y 或 1
cat /sys/module/kvm_amd/parameters/nested 2>/dev/null     # AMD 同上

# 3. 内存与磁盘
free -h
swapon --show
df -h
lsblk -d -o NAME,SIZE,ROTA,MODEL        # ROTA 为 0 表示 SSD 或 NVMe

# 4. 网卡与网络
ip -br link
ip -br addr
lspci | grep -i ethernet
ethtool <nic> | egrep 'Speed|Link detected'     # <nic> 换成实际网卡名称
ip route

# 5. 软件版本（尚未安装的项目会回报找不到，属正常）
python3 --version
docker --version
docker info | egrep 'Server Version|Storage Driver|Docker Root Dir'
ovs-vsctl --version
virsh --version
qemu-system-x86_64 --version
ip -4 addr show docker0                 # 看 docker0 子网是否为 172.17.x.x

# 6. 对外连接（用于确认 registry 与下载站可达；脱机环境请略过）
curl -sI --max-time 10 https://download.docker.com | head -1
curl -sI --max-time 10 https://sonicdev-microsoft.azurecr.io | head -1
curl -sI --max-time 10 https://sonic-build.azurewebsites.net | head -1
```

验收标准：

| 检查 | 通过条件 | 对应章节 |
|---|---|---|
| 操作系统 | `VERSION_ID` 为 `22.04` 或 `24.04`，架构 amd64 | 9.4 |
| 虚拟化旗标 | `egrep -c '(vmx\|svm)'` 大于 0；`/dev/kvm` 存在；`kvm-ok` 通过 | 9.3.1 |
| 若主机是 VM | `nested` 为 `Y` 或 `1`（Intel 或 AMD 对应的文档） | 9.3.1 |
| 内存 | `free -h` 的 total 符合你购买的等级 | 9.3 |
| 磁盘 | `df -h` 挂载 `/`、`/var/lib/docker`、`root_path` 所在的磁盘都有足够空间；`ROTA` 为 0 | 9.3 |
| 网卡 | 管理网卡 `Link detected: yes`；实体 testbed 的 trunk 网卡速率符合采购规格 | 9.5 |
| docker0 | 若有输出，确认不是 `172.18.x.x`，否则依 8.2.7 调整 | 9.5.1 |
| 对外连接 | 第 6 组指令回传 HTTP 状态栏（任何 HTTP 回应即代表可达）；失败则改用脱机镜像 | 9.5.3 |

通过后就可以依第 8 章搭建。若任一项不通过，请在退换货期限内回报供应商。

### 9.8 本章的限制

- 【估计】字段没有上游依据，请在小规模实测后修正。尤其是磁盘容量、实体内核数与 trunk 网卡速率。
- 我没有在上游找到：每个 cEOS 容器的内存用量、T1／T2 专属的服务器规格、内核版本需求、hugepages 要求、每个映像的精确大小。
- 上游文档会演进（例如 Ubuntu 版本、cEOS 版本），采购前请再读一次[VsSetup](https://github.com/sonic-net/sonic-mgmt/blob/master/docs/testbed/README.testbed.VsSetup.md)与[Setup](https://github.com/sonic-net/sonic-mgmt/blob/master/docs/testbed/README.testbed.Setup.md)。
- 本章是依上游文档整理改写（Content was rephrased for compliance with licensing restrictions），所有指令都没有在本机运行，因为本章的目标是另一台尚未采购的服务器。

## 10. 参考数据

本机路径：见第 11 章「本机引用路径」清单。

外部链接：

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

文中内容为依上游文档整理改写（Content was rephrased for compliance with licensing restrictions）。

## 11. 本机引用路径

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

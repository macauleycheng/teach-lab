# SONiC 支持的 YANG Models 总览

本文档记录 sonic-buildimage 仓库中支持的所有 YANG model，按功能分类整理。

---

## 一、SONiC 原生 YANG Models

位于 `src/sonic-yang-models/yang-models/`，用于 Config Validation (CVL) 和 ConfigDB 配置校验。

### 1.1 路由相关

| Model | 说明 |
|-------|------|
| sonic-bgp-global | BGP 全局配置 |
| sonic-bgp-neighbor | BGP 邻居配置 |
| sonic-bgp-peergroup | BGP Peer Group |
| sonic-bgp-peerrange | BGP Peer Range |
| sonic-bgp-aggregate-address | BGP 聚合路由 |
| sonic-bgp-allowed-prefix | BGP 允许前缀 |
| sonic-bgp-bbr | BGP Best Border Router |
| sonic-bgp-common | BGP 公共定义 |
| sonic-bgp-device-global | BGP 设备全局配置 |
| sonic-bgp-internal-neighbor | BGP 内部邻居 |
| sonic-bgp-monitor | BGP Monitor Protocol |
| sonic-bgp-prefix-list | BGP 前缀列表 |
| sonic-bgp-sentinel | BGP Sentinel |
| sonic-bgp-voq-chassis-neighbor | BGP VOQ Chassis 邻居 |
| sonic-static-route | 静态路由 |
| sonic-route-common | 路由公共定义 |
| sonic-route-map | Route Map |
| sonic-routing-policy-sets | 路由策略集 |
| sonic-evpn | EVPN 配置 |
| sonic-srv6 | Segment Routing v6 |
| sonic-vrf | VRF 配置 |

### 1.2 接口 / 端口相关

| Model | 说明 |
|-------|------|
| sonic-port | 物理端口 |
| sonic-portchannel | LAG / PortChannel |
| sonic-interface | 三层接口 |
| sonic-loopback-interface | Loopback 接口 |
| sonic-vlan | VLAN |
| sonic-vlan-sub-interface | VLAN 子接口 |
| sonic-mgmt_interface | 管理接口 |
| sonic-mgmt_port | 管理端口 |
| sonic-mgmt_vrf | 管理 VRF |
| sonic-voq-inband-interface | VOQ Inband 接口 |
| sonic-system-port | 系统端口 (VOQ) |

### 1.3 QoS / Buffer 相关

| Model | 说明 |
|-------|------|
| sonic-buffer-pg | Buffer Priority Group |
| sonic-buffer-pool | Buffer Pool |
| sonic-buffer-profile | Buffer Profile |
| sonic-buffer-queue | Buffer Queue |
| sonic-buffer-port-egress-profile-list | 出口 Buffer Profile 列表 |
| sonic-buffer-port-ingress-profile-list | 入口 Buffer Profile 列表 |
| sonic-scheduler | 调度器 |
| sonic-queue | 队列 |
| sonic-wred-profile | WRED Profile |
| sonic-dot1p-tc-map | 802.1p → TC 映射 |
| sonic-dscp-tc-map | DSCP → TC 映射 |
| sonic-dscp-fc-map | DSCP → FC 映射 |
| sonic-exp-fc-map | EXP → FC 映射 |
| sonic-tc-dscp-map | TC → DSCP 映射 |
| sonic-tc-priority-group-map | TC → Priority Group 映射 |
| sonic-tc-queue-map | TC → Queue 映射 |
| sonic-mpls-tc-map | MPLS TC 映射 |
| sonic-port-qos-map | 端口 QoS Map 绑定 |
| sonic-pfc-priority-priority-group-map | PFC Priority → PG 映射 |
| sonic-pfc-priority-queue-map | PFC Priority → Queue 映射 |
| sonic-pfcwd | PFC Watchdog |
| sonic-policer | Policer |
| sonic-copp | CoPP (Control Plane Policing) |
| sonic-trimming | Trimming |
| sonic-default-lossless-buffer-parameter | 默认无损 Buffer 参数 |
| sonic-lossless-traffic-pattern | 无损流量模式 |

### 1.4 ACL / 安全相关

| Model | 说明 |
|-------|------|
| sonic-acl | ACL 规则 |
| sonic-mirror-session | 镜像会话 |
| sonic-macsec | MACsec 加密 |
| sonic-system-aaa | AAA 认证 |
| sonic-system-tacacs | TACACS+ |
| sonic-system-radius | RADIUS |
| sonic-system-ldap | LDAP |
| sonic-ssh-server | SSH 服务器 |
| sonic-fips | FIPS 安全合规 |
| sonic-passwh | 密码策略 |

### 1.5 Overlay / Tunnel

| Model | 说明 |
|-------|------|
| sonic-vxlan | VxLAN |
| sonic-tunnel | IP Tunnel |
| sonic-nvgre-tunnel | NVGRE Tunnel |
| sonic-vnet | Virtual Network |
| sonic-subnet-decap | Subnet Decap |

### 1.6 NAT / DHCP

| Model | 说明 |
|-------|------|
| sonic-nat | NAT |
| sonic-dhcpv4-relay | DHCPv4 Relay |
| sonic-dhcpv6-relay | DHCPv6 Relay |
| sonic-dhcp-server | DHCP Server |
| sonic-dhcp-server-ipv4 | DHCP Server IPv4 |

### 1.7 系统 / 平台

| Model | 说明 |
|-------|------|
| sonic-device_metadata | 设备元数据 |
| sonic-device_neighbor | 设备邻居 |
| sonic-device_neighbor_metadata | 设备邻居元数据 |
| sonic-feature | Feature 管理 |
| sonic-warm-restart | Warm Restart |
| sonic-kdump | Kdump 内核转储 |
| sonic-ntp | NTP 时间同步 |
| sonic-dns | DNS |
| sonic-syslog | Syslog |
| sonic-snmp | SNMP |
| sonic-lldp | LLDP |
| sonic-sflow | sFlow 采样 |
| sonic-crm | Critical Resource Monitoring |
| sonic-flex_counter | Flex Counter |
| sonic-debug-counter | Debug Counter |
| sonic-auto_techsupport | 自动 Tech Support |
| sonic-alarm | 告警 |
| sonic-event | 事件 |
| sonic-events-bgp | BGP 事件 |
| sonic-events-common | 公共事件 |
| sonic-events-dhcp-relay | DHCP Relay 事件 |
| sonic-events-host | 主机事件 |
| sonic-events-swss | SWSS 事件 |
| sonic-events-syncd | Syncd 事件 |
| sonic-breakout_cfg | 端口 Breakout 配置 |
| sonic-cable-length | 线缆长度 |
| sonic-chassis-module | Chassis 模块 |
| sonic-fabric-monitor | Fabric 监控 |
| sonic-fabric-port | Fabric 端口 |
| sonic-smart-switch | Smart Switch |
| sonic-console | Console |
| sonic-serial-console | 串口 Console |
| sonic-logger | Logger |
| sonic-hash | ECMP/LAG Hash |
| sonic-pbh | Policy Based Hashing |
| sonic-fine-grained-ecmp | 细粒度 ECMP |
| sonic-gnmi | gNMI 配置 |
| sonic-telemetry | Telemetry |
| sonic-telemetry_client | Telemetry Client |
| sonic-grpcclient | gRPC Client |
| sonic-high-frequency-telemetry | 高频 Telemetry |
| sonic-bmp | BMP (BGP Monitoring Protocol) |
| sonic-mclag | MC-LAG |
| sonic-mux-cable | Mux Cable (双活) |
| sonic-mux-linkmgr | Mux Link Manager |
| sonic-peer-switch | Peer Switch |
| sonic-spanning-tree | STP 生成树 |
| sonic-storm-control | 风暴控制 |
| sonic-mac-move-guard | MAC Move Guard |
| sonic-static-anycast-gateway | Static Anycast Gateway |
| sonic-neigh | 邻居表 |
| sonic-kubernetes_master | Kubernetes Master |
| sonic-restapi | REST API |
| sonic-extension | 扩展包 |
| sonic-ztp | Zero Touch Provisioning |
| sonic-versions | 版本信息 |
| sonic-banner | 登录 Banner |
| sonic-memory-statistics | 内存统计 |
| sonic-stormond-config | Storage Monitor 配置 |
| sonic-xcvrd-log | 收发器日志 |
| sonic-heartbeat | 心跳 |
| sonic-leak-control | 泄漏控制 |
| sonic-fast-linkup | 快速 Link Up |
| sonic-asic-sensors | ASIC 传感器 |
| sonic-suppress-asic-sdk-health-event | 抑制 ASIC SDK 健康事件 |
| sonic-system-defaults | 系统默认值 |
| sonic-dash | DASH (Disaggregated API for SONiC Hosts) |

### 1.8 公共类型

| Model | 说明 |
|-------|------|
| sonic-types | 基础数据类型定义 |
| sonic-route-common | 路由公共类型 |

---

## 二、OpenConfig YANG Models

位于 `src/sonic-mgmt-common/models/yang/`，用于 gNMI / REST 北向管理接口。

### 2.1 核心模型

| Model | 说明 |
|-------|------|
| openconfig-acl | ACL |
| openconfig-interfaces | 接口 |
| openconfig-if-ethernet | 以太网接口 |
| openconfig-if-ip | IP 接口 |
| openconfig-lldp | LLDP |
| openconfig-platform | 平台/硬件 |
| openconfig-sampling-sflow | sFlow 采样 |
| openconfig-system | 系统管理 |
| openconfig-mclag | MC-LAG（扩展） |

### 2.2 gNSI 安全模型

| Model | 说明 |
|-------|------|
| gnsi-authz | gNSI 授权 |
| gnsi-certz | gNSI 证书 |
| gnsi-credentialz | gNSI 凭证 |
| gnsi-pathz | gNSI 路径授权 |

### 2.3 公共依赖模型

| Model | 说明 |
|-------|------|
| iana-if-type | IANA 接口类型 |
| ietf-inet-types | IETF 网络地址类型 |
| ietf-interfaces | IETF 接口 |
| ietf-yang-types | IETF YANG 基础类型 |
| ietf-yang-library | YANG Library |
| openconfig-extensions | OC 扩展 |
| openconfig-types | OC 基础类型 |
| openconfig-yang-types | OC YANG 类型 |
| openconfig-inet-types | OC 网络类型 |
| openconfig-if-types | 接口类型 |
| openconfig-vlan / openconfig-vlan-types | VLAN |
| openconfig-packet-match / openconfig-packet-match-types | 报文匹配 |
| openconfig-platform-* (多个子模型) | 平台子组件 |
| openconfig-aaa / openconfig-aaa-types | AAA |
| openconfig-aaa-radius / openconfig-aaa-tacacs | RADIUS / TACACS |

---

## 三、FRR YANG Models

位于 `src/sonic-frr/frr/yang/`，FRR 路由套件内部使用。

### 3.1 路由协议

| Model | 说明 |
|-------|------|
| frr-bgp | BGP |
| frr-bgp-neighbor | BGP 邻居 |
| frr-bgp-common | BGP 公共 |
| frr-bgp-rpki | BGP RPKI |
| frr-bgp-route-map | BGP Route Map |
| frr-bgp-types | BGP 类型 |
| frr-ospfd | OSPFv2 |
| frr-ospf-route-map | OSPF Route Map |
| frr-isisd | IS-IS |
| frr-staticd | 静态路由 |
| frr-ripngd | RIPng |
| frr-bfdd | BFD |
| frr-vrrpd | VRRP |
| frr-pim | PIM 组播 |
| frr-pim-candidate | PIM Candidate |
| frr-gmp | IGMP/MLD |
| frr-pathd | Segment Routing / Path |

### 3.2 基础设施

| Model | 说明 |
|-------|------|
| frr-zebra | Zebra 数据平面管理 |
| frr-routing | 路由框架 |
| frr-vrf | VRF |
| frr-filter | 过滤器 |
| frr-route-map | Route Map |
| frr-route-types | 路由类型 |
| frr-nexthop | 下一跳 |
| frr-host | 主机配置 |
| frr-backend | 后端 |

### 3.3 IETF 标准模型（FRR 内嵌）

| Model | 说明 |
|-------|------|
| ietf-interfaces | 接口 |
| ietf-routing-types | 路由类型 |
| ietf-key-chain | 密钥链 |
| ietf-bgp-types | BGP 类型 |
| ietf-srv6-types | SRv6 类型 |
| ietf-netconf / ietf-netconf-acm | NETCONF |

---

## 四、模型文件位置汇总

| 分类 | 路径 |
|------|------|
| SONiC 原生 (CVL) | `src/sonic-yang-models/yang-models/` |
| SONiC CVL 副本 | `src/sonic-yang-models/cvlyang-models/` |
| OpenConfig / gNMI | `src/sonic-mgmt-common/models/yang/` |
| FRR 路由 | `src/sonic-frr/frr/yang/` |
| SWSS Common 测试 | `src/sonic-swss-common/tests/yang/` |

---

## 五、备注

- `yang-models/` 和 `cvlyang-models/` 内容基本一致，cvlyang 是用于 CVL（Config Validation Library）编译的副本。
- OpenConfig 模型通过 Transformer 映射到 SONiC ConfigDB / StateDB。
- FRR YANG 主要用于 FRR 内部管理（mgmtd），不直接暴露给 SONiC 北向接口。
- `sonic-types.yang` 定义了所有模型共用的基础类型（MAC 地址、IP 地址、端口名等）。

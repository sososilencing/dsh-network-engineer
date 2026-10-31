---
vendor: 飞塔（Fortinet）
os: FortiOS
topic: VLAN 与二层交换
devices: FortiGate 40F/60F/80F/100F/200F（VLAN Switch）、300E/400E/600E/1800F（Hardware Switch）等带硬件交换模块的型号
keywords: [VLAN, set vlanid, VLAN 子接口, 802.1Q, trunk, config system virtual-switch, hardware switch, 交换接口, MAC 地址表, diagnose netlink brctl, STP]
sources:
  - https://docs.fortinet.com/document/fortigate/7.6.8/cli-reference/317104469/config-system-interface
  - https://docs.fortinet.com/document/fortigate/7.2.14/cli-reference/101854390/config-system-virtual-switch
  - https://support.fortinet.com.cn/network_mgmt/switch_interface/vlan_hardware_switch
  - https://support.fortinet.com.cn/troubleshooting/diagnostics_cli
verified: high
updated: 2026-10-30
---

# 飞塔（Fortinet）· VLAN 与二层交换

> FortiGate 没有 `switchport mode trunk/access` 这套命令：三层 VLAN 用**子接口**（`set interface` + `set vlanid`），二层交换用 **VLAN/Hardware Switch**（`config system virtual-switch`），带 Tag 上行口用 `set trunk enable`。二层能力明显弱于交换机，复杂生成树场景建议上受管 FortiSwitch。

## 命令速查

| 功能 | 命令 | 视图 | 说明 |
|---|---|---|---|
| 建 VLAN 子接口 | `config system interface` → `edit "vlan10"` | `(interface) #` | 命名建议与 VLAN ID 一致，便于排障 |
| 指定父接口 | `set interface "port1"` | `(vlan10) #` | 承载该 VLAN 的物理口或聚合口；建好后父接口不可改 |
| 打 802.1Q Tag | `set vlanid 10` | `(vlan10) #` | 取值 1～4094，须与对端交换机 VLAN ID 一致；建好后可改 |
| 配子接口地址 | `set ip 192.168.10.1 255.255.255.0` | `(vlan10) #` | 点分掩码，不是 `/24` |
| 开管理访问 | `set allowaccess ping https ssh` | `(vlan10) #` | 不显式开启则管理面不通 |
| 上行口透传 Tag | `set trunk enable` | `(wan1) #` | 部分型号/版本需要，让物理口承载多个 VLAN Tag |
| 建交换接口（成员集合） | `config system virtual-switch` → `edit "internal"` | `(virtual-switch) #` | 两层：先建虚拟交换机，再加成员端口 |
| 绑定硬件交换芯片 | `set physical-switch "sw0"` | `(internal) #` | 型号不同芯片名可能为 `sw0`/`sw1` |
| 加/删成员口 | `config port` → `edit "port3"` / `delete port3` | `(port) #` | 成员口加入后从接口列表消失，不能再单独配 IP |
| 交换接口做 VLAN 接口 | `set vlan 10` | `(VLAN10) #` | 仅 **VLAN Switch** 支持（60F/80F/100F/200F 等）；Hardware Switch 不支持 |
| 开启 VLAN 交换模式 | `set virtual-switch-vlan enable` | `(global) #` | 全局开关，不开则 `set vlan` 不生效 |
| 交换接口设类型 | `set type hard-switch` | `(switch_1) #` | 软件交换为 `switch`，硬件/VLAN 交换为 `hard-switch` |
| 接口做 802.1X | `set security-mode 802.1X` + `set security-groups "grp"` | `(switch_1) #` | 交换接口上的端口级认证，需先配 RADIUS |
| 查交换接口列表 | `diagnose netlink brctl list` | 操作模式 | 列出设备上所有 switch 名称 |
| 查二层 MAC 表 | `diagnose netlink brctl name host <switch名>` | 操作模式 | 查该交换接口的二层转发表 |
| 查 ARP / 接口 IP | `get system arp` / `diagnose ip address list` | 操作模式 | 二层通但三层不通时先查这两条 |

## 典型示例

```text
// 场景：port1 作为上联，承载 VLAN10/20；port3、port4 接终端分属两个 VLAN
FortiGate # config system interface
FortiGate (interface) # edit "vlan10"
FortiGate (vlan10) # set vdom "root"
FortiGate (vlan10) # set interface "port1"               // 父接口（物理口/聚合口）
FortiGate (vlan10) # set vlanid 10                        // 802.1Q Tag
FortiGate (vlan10) # set ip 192.168.10.1 255.255.255.0
FortiGate (vlan10) # set allowaccess ping https ssh       // 管理面必须显式放行
FortiGate (vlan10) # set role lan
FortiGate (vlan10) # next
FortiGate (interface) # edit "vlan20"
FortiGate (vlan20) # set interface "port1"
FortiGate (vlan20) # set vlanid 20
FortiGate (vlan20) # set ip 192.168.20.1 255.255.255.0
FortiGate (vlan20) # set allowaccess ping
FortiGate (vlan20) # next
FortiGate (interface) # end

FortiGate # config system virtual-switch                    // 二层交换：建一个交换接口并加成员
FortiGate (virtual-switch) # edit "switch_1"
FortiGate (switch_1) # set physical-switch "sw0"
FortiGate (switch_1) # config port
FortiGate (port) # edit "port3"
FortiGate (port) # next
FortiGate (port) # edit "port4"
FortiGate (port) # next
FortiGate (port) # end
FortiGate (switch_1) # set type hard-switch
FortiGate (switch_1) # set ip 192.168.200.1 255.255.255.0
FortiGate (switch_1) # next
FortiGate (virtual-switch) # end

FortiGate # show full-configuration system interface vlan10   // 核对 vlanid/interface/allowaccess
FortiGate # diagnose netlink brctl list                       // 有哪些交换接口
FortiGate # diagnose netlink brctl name host switch_1         // 二层 MAC 表
FortiGate # diagnose sys 802-1x status                        // 交换接口上的 802.1X 端口状态
```

## 陷阱与版本差异

- FortiGate 是防火墙/路由设备，没有 Trunk 端口"允许 VLAN 列表"的概念：每个 VLAN 都要单独建一个子接口，VLAN 多时接口数量会膨胀。
- 物理口要加入交换接口，必须先清空 IP（0.0.0.0/0）且不能被策略/路由引用；否则 CLI 直接拒绝或行为不符合预期。
- 型号差异很关键：60F/80F/100F/200F 等是 **VLAN Switch**，可直接在交换接口上 `set vlanid` 当 VLAN 接口用；40F/300E/400E/600E/1800F 等是 **Hardware Switch**，不能设 VLAN ID，必须在它上面再建 VLAN 子接口。
- 只有 VLAN/Hardware Switch 支持 STP，Software Switch（`set type switch`）不支持；FortiGate 自身生成树能力有限，多交换机环路、多 VLAN Trunk 场景建议用受管 FortiSwitch（`config switch-controller`）。
- 同名 VLAN 子接口的 `set vlanid` 与对端不一致时，现象是"链路 up、ARP 学不到"，用 `diagnose sniffer packet port1 'vlan 10'` 抓包确认 Tag。
- 二层通了但三层不通，先看该 VLAN 接口有没有 IP、策略有没有放行（VLAN 接口在策略里就是一个 `srcintf`/`dstintf`）。

## 高危操作 ⚠️

- ⚠️ 修改正在承载业务的接口 `vlanid` 或 `set interface` 父接口会立即中断该 VLAN 全部用户；⚠️ 变更前用 `show full-configuration system interface` 备份并准备 Console。
- ⚠️ 把物理口从 `internal`/`lan` 交换接口中 `delete` 出去，会一次性断开接在该组端口上的所有终端，且恢复需要重新加成员并配 IP。
- ⚠️ `delete <接口名>` 会连带清掉引用它的防火墙策略、静态路由、DHCP 服务，可能造成整段断网；⚠️ 删除前用 GUI"查找对象引用"或官方 `diagnose` 命令确认引用关系（以官方手册为准）。
- ⚠️ 关闭 `/取消` `set virtual-switch-vlan enable` 会让已配置 `set vlan` 的交换接口停止按 VLAN 转发，影响面覆盖整台设备。

## 参考来源

1. [config system interface（docs.fortinet.com，FortiOS 7.6.8 CLI Reference）](https://docs.fortinet.com/document/fortigate/7.6.8/cli-reference/317104469/config-system-interface)
2. [config system virtual-switch（docs.fortinet.com，FortiOS 7.2.14 CLI Reference）](https://docs.fortinet.com/document/fortigate/7.2.14/cli-reference/101854390/config-system-virtual-switch)
3. [VLAN/Hardware Switch：VLAN Switch 与 Hardware Switch 差异、trunk、802.1X（FortiGate 中文一本通）](https://support.fortinet.com.cn/network_mgmt/switch_interface/vlan_hardware_switch)
4. [常用排错命令：brctl、MAC 表、ARP、接口查询（FortiGate 中文一本通）](https://support.fortinet.com.cn/troubleshooting/diagnostics_cli)

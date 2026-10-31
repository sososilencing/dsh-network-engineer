---
vendor: 思科（Cisco）
os: IOS / IOS-XE
topic: 接口配置（IP、速率双工、SVI）
devices: Catalyst 9200/9300、ISR 1000/4000、ASR 1000 等
keywords: [interface, no switchport, ip address, SVI, interface vlan, 速率双工, speed, duplex, no ip routing, show ip interface brief]
sources:
  - https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9300/software/release/17-6/configuration_guide/vlan/b_176_vlan_9300_cg/configuring_vlan_trunks.html
  - https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9300/software/release/17-6/configuration_guide/vlan/b_176_vlan_9300_cg/configuring_vlans.html
  - https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/iproute_pi/configuration/15-mt/iri-15-mt-book/iri-ip-prot-indep.html
verified: medium
updated: 2026-10-30
---

# 思科 · 接口配置（IP、速率双工、SVI）

> 适用于 IOS/IOS-XE 交换机与路由器的三层落地：交换机端口默认是二层口，要配 IP 必须先 `no switchport` 变路由口，或改用 SVI；路由器的物理口默认就是三层口，可直接 `ip address`。

## 命令速查

| 功能 | 命令 | 视图 | 说明 |
|---|---|---|---|
| 进入物理接口 | `interface gigabitethernet 1/0/2` | `(config)#` | 交换机编号为 槽/子卡/端口；路由器多为 `gigabitethernet 0/0/1` |
| 接口描述 | `description To-CORE-Uplink` | `(config-if)#` | 便于审计与排障，随配置保存 |
| 转为三层口 | `no switchport` | `(config-if)#` | 交换机端口变路由口，之后才能配 IP |
| 配置 IP | `ip address 10.1.12.1 255.255.255.0` | `(config-if)#` | 用点分十进制掩码（非 /24 写法） |
| 启用接口 | `no shutdown` | `(config-if)#` | 新接口默认多为 shutdown，需手工打开 |
| 强制速率 | `speed 1000` | `(config-if)#` | 两端必须一致；可选值随平台与光模块（以官方手册为准） |
| 强制双工 | `duplex full` | `(config-if)#` | 千兆及以上一般固定全双工；两端须一致（以官方手册为准） |
| 创建 SVI | `interface vlan 20` | `(config)#` | 三层 VLAN 接口，VLAN 必须已存在 |
| SVI 配 IP | `ip address 10.1.20.1 255.255.255.0` | `(config-if)#` | 该 IP 即 VLAN 20 网关 |
| 三层聚合口 | `interface port-channel 1` | `(config)#` | L3 EtherChannel 见 04 篇 |
| 关闭 IP 路由 | `no ip routing` | `(config)#` | ⚠️ 变纯二层设备，见高危节 |
| 无路由时的网关 | `ip default-gateway 10.1.20.254` | `(config)#` | 仅在 `no ip routing` 时生效 |
| 查看接口摘要 | `show ip interface brief` | 特权 EXEC `#` | 一屏看 IP/状态/协议是否 up |
| 查看二层属性 | `show interfaces gigabitethernet 1/0/2 switchport` | 特权 EXEC `#` | 确认是 access 还是 trunk、native VLAN |
| 查看计数与错误 | `show interfaces gigabitethernet 1/0/2` | 特权 EXEC `#` | input/output errors、CRC、drops |
| 查看接口配置 | `show running-config interface gigabitethernet 1/0/2` | 特权 EXEC `#` | 只看该口的已下发配置 |

## 典型示例

```text
SW1# configure terminal
SW1(config)# vlan 20
SW1(config-vlan)# name OFFICE
SW1(config-vlan)# exit
SW1(config)# interface vlan 20                        // 三层 VLAN 接口（SVI）充当网关
SW1(config-if)# description VLAN20-GW
SW1(config-if)# ip address 10.1.20.1 255.255.255.0
SW1(config-if)# no shutdown
SW1(config-if)# interface gigabitethernet 1/0/1
SW1(config-if)# description To-R1
SW1(config-if)# no switchport                        // 交换机口转路由口
SW1(config-if)# ip address 10.1.12.1 255.255.255.0
SW1(config-if)# no shutdown
SW1(config-if)# end
SW1# show ip interface brief
SW1# show interfaces gigabitethernet 1/0/1 switchport
SW1# copy running-config startup-config
```

路由器侧同一网段对接（无需 `no switchport`）：

```text
R1# configure terminal
R1(config)# interface gigabitethernet 0/0/1
R1(config-if)# ip address 10.1.12.2 255.255.255.0
R1(config-if)# no shutdown
R1(config-if)# end
R1# show ip interface brief
```

## 陷阱与版本差异

- `speed`/`duplex` 的可选值、以及是否允许强制，随平台与光模块差异很大（光口通常固定），具体取值以官方手册为准；两端不一致最常见的结果是链路 down 或半双工导致大量 CRC 错包。
- Catalyst 9000 系列**不支持三层子接口**：官方 VLAN Trunk 章节明确"不支持 L3 trunk、三层接口上不能用 encapsulation 关键字"，跨 VLAN 三层请用 SVI（`interface vlan`）或 `no switchport` 路由口。
- 二层口直接 `ip address` 会被拒绝；先 `no switchport` 或改 SVI。反过来，`no switchport` 会清掉该口原有 `switchport access vlan`/trunk 配置，变更前先 `show running-config interface` 留档。
- `no ip routing` 后设备变纯二层，所有 SVI/路由口不再转发，此时只能靠 `ip default-gateway` 出网；路由器上误配会直接断三层。
- 接口 `shutdown` 是默认状态之一，新建 SVI/路由口后不做 `no shutdown` 会出现"配了 IP 但不通"的典型误判。

## 高危操作 ⚠️

- ⚠️ `no ip routing`：整机停止三层转发，SVI、路由口与所有路由全部失效，远程管理可能随之中断；执行前确认有 Console 通道，并准备好 `ip routing` 回退命令。
- ⚠️ 对上行口/聚合成员口执行 `shutdown`：链路立即中断；操作前确认冗余路径可用，并避开业务窗口。
- ⚠️ 对生产接口 `no switchport` 或改 `switchport mode`：会清空该口 VLAN/trunk 参数并重建转发路径，可能瞬间断流；先备份配置并记录原参数。
- ⚠️ 强制 `speed`/`duplex` 写错（例如对端自协商）：链路可能直接 down 且不易察觉，远端设备同时变更，务必双人确认、分步执行。

## 参考来源

1. [Configuring VLAN Trunks（二层模式、L3 trunk 限制）（cisco.com）](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9300/software/release/17-6/configuration_guide/vlan/b_176_vlan_9300_cg/configuring_vlan_trunks.html)
2. [Configuring VLANs（SVI 与 Vlan 接口概念）（cisco.com）](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9300/software/release/17-6/configuration_guide/vlan/b_176_vlan_9300_cg/configuring_vlans.html)
3. [Configuring IP Routing Protocol-Independent Features（ip address / ip default-gateway / IP routing 开关）（cisco.com）](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/iproute_pi/configuration/15-mt/iri-15-mt-book/iri-ip-prot-indep.html)

---
vendor: 思科（Cisco）
os: IOS / IOS-XE
topic: VLAN 与二层
devices: Catalyst 9200/9300、ISR 1000/4000、ASR 1000 等
keywords: [VLAN, switchport mode access, switchport mode trunk, native VLAN, allowed vlan, spanning-tree mode rapid-pvst, portfast, BPDU Guard, show vlan brief, MAC 地址表]
sources:
  - https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9300/software/release/17-6/configuration_guide/vlan/b_176_vlan_9300_cg/configuring_vlans.html
  - https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9300/software/release/17-6/configuration_guide/vlan/b_176_vlan_9300_cg/configuring_vlan_trunks.html
  - https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9300/software/release/17-6/configuration_guide/lyr2/b_176_lyr2_9300_cg/configuring_spanning_tree_protocol.html
verified: high
updated: 2026-10-30
---

# 思科 · VLAN 与二层

> 适用于 Catalyst 交换机的 Access/Trunk、VLAN 库与生成树：默认封装 IEEE 802.1Q，二层口默认模式是 `dynamic auto`（会发 DTP 协商），生产环境建议显式写死 `access`/`trunk` 并关闭 DTP 协商。

## 命令速查

| 功能 | 命令 | 视图 | 说明 |
|---|---|---|---|
| 创建 VLAN | `vlan 20` | `(config)#` | 进入 `(config-vlan)#`；可用范围 1–4094 |
| 命名 VLAN | `name OFFICE` | `(config-vlan)#` | 不命名则默认 VLAN0020 |
| 配置 Access 口 | `switchport mode access` | `(config-if)#` | 强制非中继，不参与 DTP 协商 |
| 划入 Access VLAN | `switchport access vlan 20` | `(config-if)#` | VLAN 必须已存在，否则报错 |
| 配置 Trunk 口 | `switchport mode trunk` | `(config-if)#` | 永久中继模式 |
| 关闭 DTP 协商 | `switchport nonegotiate` | `(config-if)#` | 仅 access/trunk 模式可用，对端需手工配 trunk |
| 设置 native VLAN | `switchport trunk native vlan 99` | `(config-if)#` | 两端必须一致，否则可能起 STP 环路 |
| 限制允许 VLAN | `switchport trunk allowed vlan 10,20,99` | `(config-if)#` | 默认允许 1–4094；注意 `add` 与覆盖的区别 |
| 生成树模式 | `spanning-tree mode rapid-pvst` | `(config)#` | 每 VLAN 一棵树，收敛快；默认 PVST+ |
| 指定根桥 | `spanning-tree vlan 10,20 root primary` | `(config)#` | 根桥应放在核心/汇聚，不要配在接入层 |
| 边缘口加速 | `spanning-tree portfast` | `(config-if)#` | 仅接终端；接交换机/集线器会失去防环能力 |
| 边缘口防环 | `spanning-tree bpduguard enable` | `(config-if)#` | 收到 BPDU 即 error-disable 端口 |
| 查看 VLAN | `show vlan brief` | 特权 EXEC `#` | 含 vlan.dat 中的 VLAN 与端口归属 |
| 查看 Trunk | `show interfaces trunk` | 特权 EXEC `#` | 看 native VLAN、允许列表、转发状态 |
| 查看端口二层参数 | `show interfaces gigabitethernet 1/0/2 switchport` | 特权 EXEC `#` | 确认 Administrative/Native VLAN 是否如预期 |
| 查看生成树 | `show spanning-tree vlan 20` | 特权 EXEC `#` | 根桥、端口角色与状态 |
| 查看 MAC 表 | `show mac address-table` | 特权 EXEC `#` | 定位 MAC 漂移与环路 |

## 典型示例

```text
Switch# configure terminal
Switch(config)# vlan 20
Switch(config-vlan)# name OFFICE
Switch(config-vlan)# vlan 99
Switch(config-vlan)# name NATIVE
Switch(config-vlan)# exit
Switch(config)# spanning-tree mode rapid-pvst        // 全网络统一为 Rapid PVST+
Switch(config)# spanning-tree vlan 20,99 root primary
Switch(config)# interface gigabitethernet 1/0/2
Switch(config-if)# switchport mode access            // 接终端：写死 access
Switch(config-if)# switchport access vlan 20
Switch(config-if)# spanning-tree portfast            // 边缘口免等待 30 秒
Switch(config-if)# spanning-tree bpduguard enable    // 防私接交换机成环
Switch(config-if)# interface gigabitethernet 1/0/1
Switch(config-if)# switchport mode trunk             // 上行：写死 trunk
Switch(config-if)# switchport nonegotiate            // 与对端设备不跑 DTP
Switch(config-if)# switchport trunk native vlan 99   // 两端必须一致
Switch(config-if)# switchport trunk allowed vlan 10,20,99
Switch(config-if)# end
Switch# show vlan brief
Switch# show interfaces trunk
Switch# show spanning-tree vlan 20
```

## 陷阱与版本差异

- Catalyst 二层口**默认模式是 `dynamic auto`**（官方 Layer 2 Interface Modes 表），不显式配置 `access`/`trunk` 时会跑 DTP；连非思科设备或禁止 DTP 的场景必须配合 `switchport nonegotiate`，否则中继起不来。
- Trunk 两端 native VLAN 不一致会触发 STP 环路风险（官方前置条件明确）；`switchport trunk allowed vlan 10,20` 是**覆盖**允许列表，追加必须写 `add 10,20`，改错会瞬间断开未列出的 VLAN。
- VLAN 1–1005 存在 `vlan.dat`，1002–1005 为保留 VLAN 不可用；扩展 VLAN（1006–4094）在 VTP v1/v2 只存于 running-config，`copy run start` 前重启会丢。
- 一个 STP 实例数与活跃 VLAN 数挂钩，VLAN 多于支持的实例数时超出的 VLAN 不跑 STP（官方 Normal-Range 指南明确），此时应改用 MSTP 并用 allowed 列表收口。
- 成员口加入 EtherChannel 后，任何二层参数变更必须做在 `port-channel` 上，否则成员口会被 suspend（官方 VLAN 章节 restrictions）。

## 高危操作 ⚠️

- ⚠️ 在承载业务的 Trunk 上改 `switchport trunk allowed vlan`（不带 `add`）：未列出的 VLAN 立刻中断；变更前先 `show interfaces trunk` 记录现网列表并备份配置。
- ⚠️ `no vlan 20`：删除 VLAN 会让该 VLAN 内所有端口失效、业务全断；先确认无端口与 SVI 依赖，再在窗口内操作。
- ⚠️ `no spanning-tree vlan 10` 或接口上 `spanning-tree bpdufilter enable`：等于关闭该 VLAN 的防环，官方明确会引发 STP 环路；二层必须保持 STP 开启。
- ⚠️ 把 `spanning-tree vlan ... root primary` 配到接入交换机：全网流量被牵引到接入层，可能造成带宽打满与次优转发；根桥只配在核心/汇聚。

## 参考来源

1. [Configuring VLANs, VLAN Configuration Guide 17.6.x（cisco.com）](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9300/software/release/17-6/configuration_guide/vlan/b_176_vlan_9300_cg/configuring_vlans.html)
2. [Configuring VLAN Trunks（二层模式/native VLAN/allowed 列表）（cisco.com）](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9300/software/release/17-6/configuration_guide/vlan/b_176_vlan_9300_cg/configuring_vlan_trunks.html)
3. [Configuring Spanning Tree Protocol（cisco.com）](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9300/software/release/17-6/configuration_guide/lyr2/b_176_lyr2_9300_cg/configuring_spanning_tree_protocol.html)

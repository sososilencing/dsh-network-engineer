---
vendor: 瞻博（Juniper）
os: Junos
topic: VLAN 与二层
devices: EX2300/EX3400/EX4300/EX4600 交换机（ELS 版本）、QFX 系列
keywords: [VLAN, vlans vlan-id, ethernet-switching, interface-mode, access, trunk, native-vlan-id, IRB, MAC 表, RSTP]
sources:
  - https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/statement/vlan-id-edit-vlans-qfx-series.html
  - https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/statement/family-ethernet-switching-edit-interfaces-qfx-series-els.html
  - https://www.juniper.net/documentation/us/en/software/junos/interfaces-ethernet-switches/topics/topic-map/switches-interface-aggregated.html
verified: high
updated: 2026-10-30
---

# 瞻博（Juniper）· VLAN 与二层

> 适用于 ELS 版 Junos 的 EX 交换机：VLAN 先建在 `[edit vlans]`，再把物理口挂进 VLAN，中间必须经过 `family ethernet-switching`。这里没有 Cisco 的 `switchport mode`，也没有「VLAN 数据库 + 三层 SVI」一体结构，二层口和三层口由 `family` 决定。

## 命令速查

| 功能 | 命令 | 视图 | 说明 |
|---|---|---|---|
| 创建 VLAN | `set vlans VLAN10 vlan-id 10` | `[edit]` | VLAN 名 VLAN10 与 ID 10 是两回事，可不同 |
| 批量 VLAN 段 | `set vlans VLAN10 vlan-id-list 10-20` | `[edit]` | 部分平台/版本支持，视型号 |
| 口设为 access | `set interfaces ge-0/0/1 unit 0 family ethernet-switching interface-mode access` | `[edit]` | ELS 用 `interface-mode`，老版本用 `port-mode` |
| access 归属 VLAN | `set interfaces ge-0/0/1 unit 0 family ethernet-switching vlan members VLAN10` | `[edit]` | 选项是 `vlan members` 而不是 `vlan-id` |
| 口设为 trunk | `set interfaces ge-0/0/1 unit 0 family ethernet-switching interface-mode trunk` | `[edit]` | — |
| trunk 放通多个 VLAN | `... vlan members [ VLAN10 VLAN20 VLAN30 ]` | `[edit]` | 也支持 `vlan members all` |
| trunk 的 native VLAN | `set interfaces ge-0/0/1 native-vlan-id 10` | `[edit]` | 物理口层级设置；不配则未打标帧不确定归属 |
| 三层接口（IRB/SVI） | `set interfaces vlan unit 10 family inet address 10.0.0.1/24` | `[edit]` | 对应 Cisco 的 `interface Vlan10` |
| VLAN 绑定三层接口 | `set vlans VLAN10 l3-interface vlan.10` | `[edit]` | 不绑定则该 VLAN 只有二层转发 |
| 运行 RSTP | `set protocols rstp interface all` | `[edit]` | ELS 平台用 `protocols rstp`；老版本可能为 `protocols stp`（以官方手册为准） |
| 调整桥优先级 | `set protocols rstp bridge-priority 4096` | `[edit]` | 数值小者优先，缺省 32768 |
| 查看 VLAN | `show vlans` | 操作模式 | 列出 VLAN、成员口、L3 接口 |
| 查看口模式归属 | `show ethernet-switching interfaces` | 操作模式 | 核对 access/trunk 与 VLAN 成员 |
| 查看 MAC 表 | `show ethernet-switching table` | 操作模式 | 可加 `interface ge-0/0/1` 过滤 |
| 查看 STP 状态 | `show spanning-tree bridge` / `show spanning-tree interface` | 操作模式 | 看根桥、口角色与状态 |

## 典型示例

```text
user@EX-01> configure
[edit]
user@EX-01# set vlans VLAN10 vlan-id 10                  // 建 VLAN（二层广播域）
user@EX-01# set vlans VLAN20 vlan-id 20
user@EX-01# set interfaces ge-0/0/1 unit 0 family ethernet-switching interface-mode access vlan members VLAN10
user@EX-01# set interfaces ge-0/0/2 unit 0 family ethernet-switching interface-mode access vlan members VLAN20
user@EX-01# set interfaces ge-0/0/24 unit 0 family ethernet-switching interface-mode trunk vlan members [ VLAN10 VLAN20 ]
user@EX-01# set interfaces ge-0/0/24 native-vlan-id 10   // 上联口未打标帧归 VLAN10
user@EX-01# set interfaces vlan unit 10 family inet address 10.0.0.1/24   // 网关（IRB）
user@EX-01# set vlans VLAN10 l3-interface vlan.10         // 把 IRB 挂到 VLAN10
user@EX-01# set protocols rstp interface all              // 防环
user@EX-01# commit confirmed 5
commit complete
user@EX-01# run show vlans
user@EX-01# run show ethernet-switching table
user@EX-01# commit                                       // 确认变更，取消定时回滚
```

## 陷阱与版本差异

- **ELS 与非 ELS 是两套语法**：ELS（EX2300/EX3400/EX4300/EX4600、QFX 新版本）用 `interface-mode access|trunk`；早期非 ELS 版本用 `port-mode access|trunk`。照抄命令前先 `show version` 确认。
- `vlan members` 与 `vlan-id` 不要混：物理口下是 `vlan members`；`vlan-id` 是 `[edit vlans]` 或三层子接口（MX/SRX 的 `unit N vlan-id N`）用的。
- `unit 0` 不能配 `vlan-id`（该语句要求逻辑单元号 ≥1，用于 802.1Q 三层子接口），在 EX 二层口上给 `unit 0` 配 `vlan-id` 会直接 commit 报错。
- 二层口未指定 `family ethernet-switching` 时口是三层口，此时 `vlan members` 无处可挂；反过来，挂了 `family ethernet-switching` 的口不能再配 `family inet`。
- VLAN 数量与 ID 范围受型号限制：如 EX2300 最多 2024 个 VLAN、ID 可用 1–4093；EX3400 最多 4093 个（含默认 VLAN 1）。规划超限会在 commit 阶段失败（以官方手册为准）。
- `native-vlan-id` 只对 trunk 生效，且不同版本/平台可用性有差异；不确定时改用显式 tagged VLAN，避免未打标帧归属不明（以官方手册为准）。
- `rstp` 是 ELS 写法，部分老版本为 `protocols rstp` 之外的 `protocols stp`；STP 模式混用时先统一全厂取值再改。

## 高危操作 ⚠️

- ⚠️ 变更 trunk 的 `vlan members`：一次性移除正在承载管理流量/业务的 VLAN 会立即断链，且失联后无法远程回退；务必 `commit confirmed N`。
- ⚠️ 修改/删除 `l3-interface` 绑定或 IRB 地址：等于改网关，整段 VLAN 内的终端会瞬间失去三层出口。
- ⚠️ 直接操作 `native-vlan-id`：未打标帧（含多数 IPMI/裸机 PXE）归属会改变，表现为「一半终端掉线」，先在测试口验证。
- ⚠️ 改动 STP 参数（`bridge-priority`、口 cost）会触发重新收敛，全厂环网可能数秒到数十秒中断。

## 参考来源

1. [vlan-id（juniper.net）](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/statement/vlan-id-edit-vlans-qfx-series.html)
2. [ethernet-switching（family）语句（juniper.net）](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/statement/family-ethernet-switching-edit-interfaces-qfx-series-els.html)
3. [Aggregated Ethernet Interfaces Overview · ELS 与 port-mode/interface-mode 差异（juniper.net）](https://www.juniper.net/documentation/us/en/software/junos/interfaces-ethernet-switches/topics/topic-map/switches-interface-aggregated.html)

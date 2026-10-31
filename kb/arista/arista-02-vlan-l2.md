---
vendor: Arista
os: EOS
topic: VLAN 与二层交换
devices: vEOS、DCS-7050X/7060X/7280R/7500R、CCS-720XP 等 EOS 设备
keywords: [VLAN, switchport mode access, switchport mode trunk, trunk allowed vlan, native vlan, show vlan, show interfaces switchport, MAC 地址表, MSTP, spanning-tree mst configuration]
sources:
  - https://www.arista.com/en/um-eos/eos-virtual-lans-vlans
  - https://www.arista.com/en/um-eos/eos-spanning-tree-protocol
  - https://www.arista.com/en/um-eos/eos-layer-2-configuration
verified: high
updated: 2026-10-30
---

# Arista · VLAN 与二层交换

> 适用于 EOS 交换机的 VLAN 划分、Access/Trunk 端口与 MAC 表查看。**与 Cisco 的关键差异：EOS 默认生成树是 MSTP（`spanning-tree mode mstp`），不是 PVST+**；端口默认是 Access 且属 VLAN 1。

## 命令速查

| 功能 | 命令 | 视图 | 说明 |
|---|---|---|---|
| 创建 VLAN | `vlan 10` | `(config)#` | VLAN 号 1–4094；VLAN 1 默认存在 |
| 批量创建 | `vlan 10,20,30-40` | `(config)#` | 逗号 + 连字符范围 |
| 命名 VLAN | `name OFFICE` | `(config-vlan-10)#` | 最长 32 字符，不允许空格 |
| 启停 VLAN | `state active` / `state suspend` | `(config-vlan-10)#` | suspend 后该 VLAN 所有端口阻断，默认 active |
| 配为 Access | `switchport mode access` | 接口视图 | 默认即为 access |
| 指定 Access VLAN | `switchport access vlan 10` | 接口视图 | 若 VLAN 不存在会隐式创建并告警 |
| 配为 Trunk | `switchport mode trunk` | 接口视图 | 默认放行全部 VLAN（1–4094） |
| Trunk 放行列表 | `switchport trunk allowed vlan 10,20,30` | 接口视图 | 也可 `add`/`remove`/`except` |
| Trunk 原生 VLAN | `switchport trunk native vlan 99` | 接口视图 | 默认 VLAN 1，两端必须一致 |
| 批量配置端口 | `interface Ethernet 1-4` | `(config)#` | 一次进多个口，提示符 `(config-if-Et1-4)#` |
| 查看 VLAN | `show vlan` / `show vlan 10` | `>` 特权 EXEC | 状态 active/suspended 与成员口 |
| 查看端口二层属性 | `show interfaces switchport` | `>` 特权 EXEC | 模式、access/native VLAN、放行列表 |
| 查看 Trunk | `show interfaces trunk` | `>` 特权 EXEC | 各 Trunk 口的放行 VLAN |
| 查看某口所属 VLAN | `show interfaces Ethernet1 vlans` | `>` 特权 EXEC | 输出 Untagged/Tagged 两列 |
| 查看 MAC 表 | `show mac address-table` | `>` 特权 EXEC | 可加 `interface`/`vlan`/`dynamic` 过滤 |
| 生成树模式 | `spanning-tree mode mstp` | `(config)#` | **默认即 mstp**；另有 rstp / rapid-pvst / none |
| MST 区域配置 | `spanning-tree mst configuration` | `(config)#` | 进入组变更模式，`exit` 才提交 |
| MST 实例映射 | `instance 1 vlans 10,20` | `(config-mst)#` | 同一区域各交换机必须完全一致 |
| 区域名/修订号 | `name REGION-A` / `revision 3` | `(config-mst)#` | 名称+修订号+VLAN 映射共同标识区域 |
| 指定根桥 | `spanning-tree mst 0 root primary` | `(config)#` | 等价 priority 8192；`secondary` 为 16384 |
| 桥优先级 | `spanning-tree mst 0 priority 4096` | `(config)#` | 4096 的整数倍，默认 32768 |
| 接主机端口 | `spanning-tree portfast` | 接口视图 | 直连终端免学习态，接交换机勿开 |
| 根/环保护 | `spanning-tree guard root` / `guard loop` | 接口视图 | 防非法根桥与单向链路环路 |
| 查看生成树 | `show spanning-tree` / `show spanning-tree root` | `>` 特权 EXEC | 实例、角色、根桥与端口状态 |

## 典型示例

```text
switch# configure terminal
switch(config)# vlan 10,20,30
switch(config-vlan-10)# name OFFICE
switch(config-vlan-10)# exit
switch(config)# interface Ethernet 1-2
switch(config-if-Et1-2)# switchport mode access
switch(config-if-Et1-2)# switchport access vlan 10
switch(config-if-Et1-2)# spanning-tree portfast        // 接 PC/服务器，加快收敛
switch(config-if-Et1-2)# exit
switch(config)# interface Ethernet 49
switch(config-if-Et49)# switchport mode trunk
switch(config-if-Et49)# switchport trunk allowed vlan 10,20,30
switch(config-if-Et49)# switchport trunk native vlan 99   // 两端必须一致
switch(config-if-Et49)# exit
switch(config)# spanning-tree mode mstp                    // 确认是 MSTP（默认）
switch(config)# spanning-tree mst configuration
switch(config-mst)# name DC1
switch(config-mst)# revision 1
switch(config-mst)# instance 1 vlans 10,20
switch(config-mst)# instance 2 vlans 30
switch(config-mst)# exit                                  // exit 提交，abort 放弃
switch(config)# spanning-tree mst 1 root primary
switch(config)# end
switch# copy running-config startup-config                // ★ 不保存则重启丢失
switch# show vlan
switch# show interfaces switchport
switch# show spanning-tree root
```

## 陷阱与版本差异

- EOS 默认生成树是 **MSTP**；对接 Cisco 的 PVST+ 网络若需互通，可在 MSTP 侧启用 `spanning-tree mst pvst border`（边界口做 BPDU 转换），或把两侧统一到 rapid-pvst / rstp。
- MST 区域参数（`name` + `revision` + VLAN→实例映射）在区域内**必须逐台一致**，任何一台映射不同都会退化为 IST 计算，典型现象是"某几个 VLAN 突然不通"。
- MST 配置视图是**组变更模式**：`exit` 提交、`abort` 放弃；误用 `exit` 与 `abort` 会造成"配了没生效"或"想撤销却已生效"。
- 端口 `switchport mode` 切换时，EOS 会把该口从原有 VLAN 关系中摘除；Access↔Trunk 反复切换后务必回头 `show interfaces switchport` 复核。
- VLAN 命名不允许空格（官方限制），习惯写 `DC1_SERVERS` 而不是 `DC1 SERVERS`。
- 改变 VLAN-to-Instance 映射会引起流量中断，须全区域同步变更并安排维护窗口。

## 高危操作 ⚠️

- ⚠️ 在**上行 Trunk** 上收紧 `switchport trunk allowed vlan` 时漏掉管理 VLAN，会导致带内管理瞬间失联；改前先 `show interfaces trunk` 记录原列表，并保留 Console/带外通道。
- ⚠️ 误改 `switchport trunk native vlan` 只有一端变更，会造成该 VLAN 单通、且可能串流到错误广播域。
- ⚠️ `no vlan 10` 会连带删除该 VLAN 的所有成员关系与对应 SVI 配置，业务整段中断；先确认 `show vlan 10` 与 `show running-config` 中的引用。
- ⚠️ `spanning-tree mode none` 关闭全部 STP，环路时立即广播风暴打满整网，仅在严格无环拓扑中临时使用。

## 参考来源

1. [EOS User Manual · Virtual LANs (VLANs)（arista.com）](https://www.arista.com/en/um-eos/eos-virtual-lans-vlans)
2. [EOS User Manual · Spanning Tree Protocol（arista.com）](https://www.arista.com/en/um-eos/eos-spanning-tree-protocol)
3. [EOS User Manual · Layer 2 Configuration（arista.com）](https://www.arista.com/en/um-eos/eos-layer-2-configuration)

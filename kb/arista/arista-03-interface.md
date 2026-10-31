---
vendor: Arista
os: EOS
topic: 接口配置
devices: vEOS、DCS-7050X/7060X/7280R/7500R、CCS-720XP 等 EOS 设备
keywords: [interface Ethernet, no switchport, ip address, SVI, interface Vlan, ip routing, speed forced, mtu, 三层口, show interfaces status]
sources:
  - https://www.arista.com/en/um-eos/eos-ipv4
  - https://www.arista.com/en/um-eos/eos-virtual-lans-vlans
  - https://www.arista.com/en/um-eos/eos-layer-3-configuration
verified: high
updated: 2026-10-30
---

# Arista · 接口配置

> 适用于 EOS 的物理口/聚合口/SVI/子接口配置。**必须先全局 `ip routing`**（EOS 默认关闭）SVI 与路由口之间才能互通；接口掩码支持斜线写法 `10.0.0.1/24`。

## 命令速查

| 功能 | 命令 | 视图 | 说明 |
|---|---|---|---|
| 进入物理口 | `interface Ethernet1` | `(config)#` | 固定盒式用 `Ethernet1`；模块化用 `Ethernet1/1`、`Ethernet7/30/2` |
| 批量进入 | `interface Ethernet 1-4` | `(config)#` | 提示符变为 `(config-if-Et1-4)#` |
| 接口描述 | `description UPLINK-TO-SPINE` | 接口视图 | 强烈建议写上对端设备与用途 |
| 关闭/开启 | `shutdown` / `no shutdown` | 接口视图 | 物理口默认 `no shutdown` |
| 转为三层口 | `no switchport` | 接口视图 | 二→三层转换，交换机自动分配内部 VLAN |
| 三层口配 IP | `ip address 10.0.0.1/24` | 接口视图 | 也支持 `ip address 10.0.0.1 255.255.255.0` |
| 从属地址 | `ip address 192.168.1.1/24 secondary` | 接口视图 | 一个口可配多个从属地址 |
| 建 SVI | `interface Vlan10` | `(config)#` | SVI 需对应 VLAN 已存在且在 STP 转发态 |
| SVI 强制 up | `no autostate` | `(config-if-Vl10)#` | autostate 默认开启：VLAN 内无 up 端口则 SVI down |
| 全局开三层 | `ip routing` | `(config)#` | **EOS 默认关闭**，不开则 VLAN 间/路由口不通 |
| 速率双工 | `speed forced 1000full` | 接口视图 | EOS 用 `forced` 关键字；`speed auto` 恢复自协商 |
| 关闭自协商 | `speed forced 10000full` | 接口视图 | 光口/强制互联场景，两端必须匹配 |
| 设置 MTU | `mtu 9000` | 接口视图 | 路径上每一跳都要一致，否则大包被丢 |
| 子接口 | `interface Ethernet1.10` → `encapsulation dot1q vlan 10` | 子接口视图 | 单臂路由/对接 Trunk 的上联 |
| 环回口 | `interface Loopback0` → `ip address 10.255.0.1/32` | `(config)#` | OSPF router-id 与 BGP 源地址常用 |
| 恢复出厂口配置 | `default interface Ethernet1` | `(config)#` | 仅清该口，比整体回退安全 |
| 查看接口一览 | `show interfaces status` | `>` 特权 EXEC | 状态/VLAN/速率/双工，排障第一命令 |
| 查看单口详情 | `show interfaces Ethernet1` | `>` 特权 EXEC | 含 counters、errors、MTU |
| 查看 IP 汇总 | `show ip interface brief` | `>` 特权 EXEC | 各三层口地址与 up/down |
| 查看计数器 | `show interfaces counters` | `>` 特权 EXEC | 可加 `errors` / `rates` 定位丢包 |

## 典型示例

```text
switch# configure terminal
switch(config)# ip routing                        // ★ 不开则所有三层转发失效
switch(config)# vlan 10,20
switch(config)# interface Vlan10
switch(config-if-Vl10)# ip address 10.10.10.1/24  // SVI 做 VLAN 10 网关
switch(config-if-Vl10)# no shutdown
switch(config-if-Vl10)# exit
switch(config)# interface Ethernet 1-2
switch(config-if-Et1-2)# switchport mode access
switch(config-if-Et1-2)# switchport access vlan 10
switch(config-if-Et1-2)# exit
switch(config)# interface Ethernet49
switch(config-if-Et49)# no switchport              // 转为路由口
switch(config-if-Et49)# description TO-CORE-01
switch(config-if-Et49)# ip address 10.0.0.1/30
switch(config-if-Et49)# mtu 9000
switch(config-if-Et49)# exit
switch(config)# interface Ethernet50
switch(config-if-Et50)# speed forced 10000full     // 强制 10G 全双工
switch(config-if-Et50)# exit
switch(config)# end
switch# copy running-config startup-config         // ★ 不保存则重启丢失
switch# show interfaces status
switch# show ip interface brief
switch# show ip route
```

## 陷阱与版本差异

- **头号坑：忘了 `ip routing`**。EOS 默认关闭 IPv4 路由，SVI 有 IP、物理口 up，但 VLAN 间仍然不通；进 `router ospf` 时官方还会直接提示 `! IP routing not enabled`。
- 端口默认是**二层 switchport**，直接 `ip address` 不生效；必须先 `no switchport`。反向恢复用 `switchport`（或 `default interface`）。
- 速率命令是 `speed forced 1000full`（`forced` 是 EOS 特有），写成 Cisco 风格 `speed 1000` 会被拒；`speed auto` 才回到自协商。
- SVI 默认 `autostate`：该 VLAN 内没有任何 up 且 STP 转发的端口时，SVI 自动 down。MLAG 场景的本地 SVI 官方建议 `no autostate`（以官方手册为准）。
- 转成路由口后交换机会占用一个内部 VLAN，该 VLAN 号不能再被 `vlan` / `interface Vlan` 使用；`show vlan internal usage` 可查已占用范围。
- 接口名大小写与格式随平台不同（`Ethernet1` vs `Ethernet1/1`），脚本里写死编号前先 `show interfaces status` 核对。

## 高危操作 ⚠️

- ⚠️ `no ip routing` 会让全部三层转发**立即中断**（SVI、路由口、OSPF 邻居全断），是 EOS 上最典型的"一键断网"命令；变更前备份并准备同窗口回退。
- ⚠️ 在上联口执行 `shutdown` 或改 `speed`/`mtu` 会瞬间断链；光口强制速率两端不一致时链路起不来，需 Console 回退。
- ⚠️ `no interface Vlan10` 删除 SVI 会连带删除其上全部 IP 与路由，该网段网关立即消失。
- ⚠️ `default interface Ethernet1` 只清单个接口，但若该口是管理口同样会失联；先 `show running-config interfaces Ethernet1` 留存原配置。

## 参考来源

1. [EOS User Manual · IPv4（arista.com）](https://www.arista.com/en/um-eos/eos-ipv4)
2. [EOS User Manual · Virtual LANs (VLANs) / interface vlan、autostate（arista.com）](https://www.arista.com/en/um-eos/eos-virtual-lans-vlans)
3. [EOS User Manual · Layer 3 Configuration（arista.com）](https://www.arista.com/en/um-eos/eos-layer-3-configuration)

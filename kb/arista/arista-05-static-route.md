---
vendor: Arista
os: EOS
topic: 静态路由
devices: vEOS、DCS-7050X/7060X/7280R/7500R 等 EOS 设备
keywords: [ip route, 默认路由, 浮动静态, administrative distance, 管理距离, 黑洞路由 Null0, ECMP, show ip route, ip routing]
sources:
  - https://www.arista.com/en/um-eos/eos-ipv4
  - https://www.arista.com/en/um-eos/eos-layer-3-configuration
  - https://www.arista.com/en/um-eos/eos-open-shortest-path-first-version-2
verified: high
updated: 2026-10-30
---

# Arista · 静态路由

> 适用于 EOS 的静态/默认/浮动静态路由。前提是全局 `ip routing`（默认关闭）；静态路由默认管理距离 **1**，OSPF 默认 **110**，因此"给静态路由配更大 distance"才是浮动备份。

## 命令速查

| 功能 | 命令 | 视图 | 说明 |
|---|---|---|---|
| 开启三层转发 | `ip routing` | `(config)#` | **默认关闭**，不开启则路由表为空、SVI 不通 |
| 默认路由 | `ip route 0.0.0.0/0 192.14.0.4` | `(config)#` | 目的 `0.0.0.0/0`，下一跳为网关地址 |
| 明细静态路由 | `ip route 172.17.252.0/24 10.0.0.2` | `(config)#` | 前缀用 CIDR 写法，也可写掩码 |
| 指向出接口 | `ip route 172.17.252.0/24 Vlan500` | `(config)#` | 官方示例写作 `ip route 172.17.252.0/24 vlan 500` |
| 浮动静态（备份） | `ip route 0.0.0.0/0 10.0.0.2 200` | `(config)#` | distance 大于 OSPF 110 即成为备份 |
| 路由 tag | `ip route 10.0.0.0/8 10.1.1.1 tag 100` | `(config)#` | 供 route-map 过滤/重分发匹配 |
| 黑洞路由 | `ip route 10.99.0.0/16 Null0` | `(config)#` | 丢弃并避免明细回源，参数以官方手册为准 |
| VRF 内静态 | `ip route vrf MGMT 0.0.0.0/0 10.0.0.1` | `(config)#` | 管理 VRF 默认网关常用写法 |
| 严格下一跳解析 | `ip route vrf V1 10.100.100.0/24 10.10.10.2 resolution ribs system-connected` | `(config)#` | 仅当下一跳经直连路由可达才生效 |
| 等价多路径 | 同前缀写多条 `ip route` | `(config)#` | EOS 自动形成 ECMP，可调 `ip load-sharing` |
| 查看路由表 | `show ip route` | `>` 特权 EXEC | Codes 中 `C` 直连、`S` 静态、`O` OSPF |
| 查看某前缀 | `show ip route 10.1.0.0/24` | `>` 特权 EXEC | 加 `age` 看该路由存在时长 |
| 查看路由统计 | `show ip route summary` | `>` 特权 EXEC | 按来源分类的条数，判断是否超规模 |
| 查看在用网关 | `show ip route gateway` | `>` 特权 EXEC | 列出所有活跃下一跳 |
| 查看主机路由 | `show ip route host` | `>` 特权 EXEC | `R` 接口地址、`A` ARP 邻接、`B` 广播 |

## 典型示例

```text
switch# configure terminal
switch(config)# ip routing                        // ★ 前置条件，默认关闭
switch(config)# interface Vlan10
switch(config-if-Vl10)# ip address 10.10.10.1/24
switch(config-if-Vl10)# exit
switch(config)# ip route 0.0.0.0/0 10.10.10.254        // 主默认路由，AD=1
switch(config)# ip route 0.0.0.0/0 10.10.20.254 200    // 浮动备份，AD=200
switch(config)# ip route 172.17.252.0/24 Vlan500       // 指向出接口
switch(config)# ip route 10.99.0.0/16 Null0            // 黑洞，防明细回源
switch(config)# ip route 10.0.0.0/8 10.10.10.254 tag 100
switch(config)# end
switch# copy running-config startup-config             // ★ 不保存则重启丢失
switch# show ip route
switch# show ip route 0.0.0.0/0
switch# show ip route summary
```

## 陷阱与版本差异

- **忘了 `ip routing`** 是 EOS 静态路由排障的第一嫌疑：`ip route` 能敲进去、`show running-config` 也有，但 FIB 不装表、`show ip route` 里看不到，本质是三层转发未开启。
- 静态路由默认 **administrative distance = 1**，比 OSPF（110）、BGP（eBGP 20 / iBGP 200）都优；想做备份必须显式把 distance 调到大于被备份协议，例如 `200` 才被 OSPF 覆盖。
- `ip route` 的 `distance` / `tag` / `name` 等可选参数**位置随版本**，落地前用 `ip route ?` 现场确认顺序（以官方手册为准）。
- 指向出接口的静态路由（如指向 `Vlan500`）会触发对每个目的地址的 ARP 解析，大网段下容易打爆 ARP 表；能写下一跳 IP 时优先写 IP。
- 同一前缀写多条 `ip route` 即形成 ECMP，前提是各下一跳都可达；只对逐流哈希生效，单条大流不会分摊。
- `Null0` 黑洞路由能抑制明细回源，但配错前缀会直接黑洞掉业务流量，上线前用 `show ip route <前缀>` 复核。

## 高危操作 ⚠️

- ⚠️ `no ip routing` 会一次性删除所有三层转发能力（含 SVI、OSPF 邻居、静态路由），全网段中断；EOS 上最危险的"一行命令"。
- ⚠️ 删除或改错默认路由（`no ip route 0.0.0.0/0 ...`）会让所有出网流量黑洞，且往往连远程管理回程一起断，先确认有 Console/带外。
- ⚠️ 误配 `ip route 0.0.0.0/0 Null0` 或把业务网段指向 `Null0`，会造成完全静默丢包（无 ICMP 提示），极难定位。
- 变更顺序：`show running-config` 导出 → `show ip route` 留存基线 → 变更 → `show ip route <前缀>` + `ping` 验证 → 异常时删新建路由并恢复原条目 → 确认后 `copy running-config startup-config`。

## 参考来源

1. [EOS User Manual · IPv4 · Static and Default IPv4 Routes（arista.com）](https://www.arista.com/en/um-eos/eos-ipv4)
2. [EOS User Manual · Layer 3 Configuration（arista.com）](https://www.arista.com/en/um-eos/eos-layer-3-configuration)
3. [EOS User Manual · OSPFv2（含各协议默认 administrative distance 说明）（arista.com）](https://www.arista.com/en/um-eos/eos-open-shortest-path-first-version-2)

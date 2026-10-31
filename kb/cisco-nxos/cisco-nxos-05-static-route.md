---
vendor: 思科（Cisco）
os: NX-OS
topic: 静态路由
devices: Nexus 3000/5000/7000/9000 系列（三层特性需相应 license）
keywords: [静态路由, ip route, 默认路由, 浮动静态路由, 管理距离 preference, 黑洞路由, null 0, show ip static-route, show ip route]
sources:
  - https://www.cisco.com/c/en/us/td/docs/dcn/nx-os/nexus9000/103x/unicast-routing-configuration/cisco-nexus-9000-series-nx-os-unicast-routing-configuration-guide-release-103x/m_configuring_static_routing.html
  - https://www.cisco.com/c/en/us/td/docs/dcn/nx-os/nexus9000/103x/unicast-routing-configuration/cisco-nexus-9000-series-nx-os-unicast-routing-configuration-guide-release-103x.html
  - https://www.cisco.com/c/en/us/td/docs/dcn/nx-os/nexus9000/103x/configuration/interfaces/cisco-nexus-9000-nx-os-interfaces-configuration-guide-103x.html
verified: high
updated: 2026-10-30
---

# 思科 Cisco NX-OS · 静态路由

> 适用于 Nexus 的三层静态路由：出口默认路由、浮动备份、黑洞汇总与 SVI 直连型静态路由。
> 与 IOS 差异：**目标网段写 `前缀/长度`**，管理距离关键字是 `preference`（不是 `distance` 位置参数）。

## 命令速查

| 功能 | 命令 | 视图 | 说明 |
|---|---|---|---|
| 配静态路由 | `ip route 192.168.20.0/24 10.1.12.2` | `switch(config)#` | 目的写 `A.B.C.D/长度`，不能用空格掩码 |
| 默认路由 | `ip route 0.0.0.0/0 10.1.12.2` | `switch(config)#` | 出口网关；缺省 AD=1，优先于任何动态路由 |
| 指定出接口 | `ip route 192.168.20.0/24 Ethernet1/1 10.1.12.2` | `switch(config)#` | 广播型接口建议"出接口+下一跳"全指定 |
| 仅出接口（点到点） | `ip route 192.168.20.0/24 Ethernet1/1` | `switch(config)#` | 只适用于点到点；广播口必须带下一跳地址 |
| 浮动静态路由 | `ip route 192.168.20.0/24 10.1.13.3 100` | `switch(config)#` | 末尾数字即 preference（AD），范围 1–255，缺省 1；数值更大更不优 |
| 路由命名 | `ip route 192.168.20.0/24 10.1.12.2 name To-Branch` | `switch(config)#` | 便于审计，`show ip static-route` 可见 |
| 打标签 | `ip route 192.168.20.0/24 10.1.12.2 tag 100` | `switch(config)#` | 供重分发 route-map 匹配 |
| 黑洞路由 | `ip route 1.1.1.1/32 null 0` | `switch(config)#` | 下一跳 `null 0` 直接丢弃；用于汇总防环、封禁网段 |
| SVI 直连型 | `ip route 209.165.200.224/27 vlan 10` | `switch(config)#` | 无下一跳写法，前置 `feature interface-vlan` |
| VRF 内静态路由 | `vrf context RED` → `ip route 0.0.0.0/0 10.1.12.2` | `switch(config-vrf)#` | 静态路由支持 VRF，注意别配到 default VRF |
| 删除静态路由 | `no ip route 192.168.20.0/24 10.1.12.2` | `switch(config)#` | ⚠️ 参数须与原命令完全一致 |
| 只看静态路由 | `show ip static-route` | `switch#` | 查本机配置的静态路由与其下一跳解析结果 |
| 看路由表 | `show ip route` | `switch#` | 查 URIB；静态路由前面标 `S`，直连 `C`，OSPF `O` |
| 只筛静态条目 | `show ip route static` | `switch#` | 快速确认某条静态路由是否真的进了表 |

### 常见管理距离（AD，数值越小越优）

| 路由类型 | 直连 | 静态 | eBGP | EIGRP(内部) | OSPF | IS-IS | RIP | iBGP |
|---|---|---|---|---|---|---|---|---|
| NX-OS AD | 0 | 1 | 20 | 90 | 110 | 115 | 120 | 200 |

## 典型示例

```text
switch# configure terminal
switch(config)# interface Ethernet1/1
switch(config-if)# no switchport
switch(config-if)# ip address 10.1.12.1/30
switch(config-if)# no shutdown
switch(config-if)# exit
switch(config)# ip route 192.168.20.0/24 10.1.12.2                 // 主用，AD 缺省 1
switch(config)# ip route 192.168.20.0/24 10.1.13.3 100             // 备份，AD=100，主用失效后转正
switch(config)# ip route 0.0.0.0/0 10.1.12.2                       // 出口默认路由
switch(config)# ip route 10.1.0.0/16 null 0                        // 汇总黑洞，防环/防扫描
switch(config)# ip route 192.168.20.0/24 10.1.12.2 name To-Branch   // 带名字便于审计
switch(config)# end
switch# show ip static-route
switch# show ip route static
switch# show ip route
switch# copy running-config startup-config                          // 不落盘重启即丢
```

## 陷阱与版本差异

- **网段写法**：NX-OS 只认 `ip route 10.1.1.0/24 ...`；IOS 风格 `ip route 10.1.1.0 255.255.255.0 ...` 会被拒或解析异常。掩码与 ACL 反掩码也别混。
- **AD 关键字位置**：NX-OS 用末尾位置参数 `preference`（1–255，缺省 1），官方默认值表明确 Administrative distance = 1；不是 IOS 那种独立的 `distance` 子句，也不是华为的 `preference` 视图命令。
- **下一跳不可达则不入表**：官方前提写明，若 URIB 中不存在覆盖该下一跳的单播路由，静态路由不会被安装（下一跳为直连网段的情况除外）。这正是"配了但不生效"的头号原因。
- **下一跳只解析一次**：静态路由不会感知远端故障，只要本端接口 up 就保持生效，可能黑洞丢包。官方建议对该类静态路由配合 BFD 或对象跟踪（object tracking）验证下一跳可达性，语法随版本，以官方手册为准。
- **SVI 直连型静态路由**需要先 `feature interface-vlan` 并已建 VLAN 接口，官方的静态路由章节把这条列为独立流程。
- 浮动静态路由比较的是 AD 而不是带宽：备用值必须**大于**主用值，否则两条同时生效变成等价负载分担；NX-OS 默认不开启静态路由 ECMP，多路径行为以平台/版本手册为准。

## 高危操作 ⚠️

- ⚠️ 删除出口默认路由 `no ip route 0.0.0.0/0 <下一跳>` 会立即造成全网断外网，变更前先 `show ip route 0.0.0.0` 留档并备好回退命令。
- ⚠️ 严禁在在网设备下发 `ip route 0.0.0.0/0 null 0`：会丢弃全部未知目的流量，等同于全站断网。
- ⚠️ 黑洞汇总路由的掩码范围一旦写大（如把业务网段圈进去），流量会被静默丢弃且无日志；下发前用 `show ip route <汇总网段>` 核对覆盖范围。
- ⚠️ 下一跳写错不会报错，路由照样生成但转发不通；配置后必须 `ping`/`traceroute`（带 `vrf`）验证后再 `copy running-config startup-config`。

## 参考来源

1. [Cisco Nexus 9000 Series NX-OS Unicast Routing Configuration Guide, Release 10.3(x) · Configure static route（cisco.com）](https://www.cisco.com/c/en/us/td/docs/dcn/nx-os/nexus9000/103x/unicast-routing-configuration/cisco-nexus-9000-series-nx-os-unicast-routing-configuration-guide-release-103x/m_configuring_static_routing.html)
2. [Cisco Nexus 9000 Series NX-OS Unicast Routing Configuration Guide, Release 10.3(x)（cisco.com）](https://www.cisco.com/c/en/us/td/docs/dcn/nx-os/nexus9000/103x/unicast-routing-configuration/cisco-nexus-9000-series-nx-os-unicast-routing-configuration-guide-release-103x.html)
3. [Cisco Nexus 9000 Series NX-OS Interfaces Configuration Guide, Release 10.3(x)（cisco.com）](https://www.cisco.com/c/en/us/td/docs/dcn/nx-os/nexus9000/103x/configuration/interfaces/cisco-nexus-9000-nx-os-interfaces-configuration-guide-103x.html)

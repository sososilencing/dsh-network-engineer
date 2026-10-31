---
vendor: 飞塔（Fortinet）
os: FortiOS
topic: 静态路由
devices: FortiGate 40F/60F/100F/600E/1800F 等（NAT 模式路由转发）
keywords: [静态路由, config router static, set dst, set gateway, set device, set distance 10, 管理距离, priority, 黑洞路由 blackhole, distance 255, 默认路由, get router info routing-table]
sources:
  - https://docs.fortinet.com/document/fortigate/7.6.8/cli-reference/200835411/config-router-static
  - https://docs.fortinet.com/document/fortigate/6.2.5/cli-reference/531620/router-static
  - https://support.fortinet.com.cn/routing/static_route
  - https://support.fortinet.com.cn/routing/route_faq/blackhole_ineffective
verified: high
updated: 2026-10-30
---

# 飞塔（Fortinet）· 静态路由

> FortiOS 静态路由在 `config router static` 下，`edit` 的编号就是路由 ID。**`set device` 必填**（只写网关不生效）；管理距离 `distance` 默认 **10**，而 OSPF 是 **110**——即"静态优先于 OSPF"，与华为/锐捷的直觉相反。`distance 255` 的路由只在 database 里，不进转发表。

## 命令速查

| 功能 | 命令 | 视图 | 说明 |
|---|---|---|---|
| 进入静态路由 | `config router static` | 操作模式 | 条目用数字 ID，删除即 `delete <ID>` |
| 新建/编辑条目 | `edit 1` | `(static) #` | ID 全局唯一，与 GUI 顺序号同一套 |
| 目的网段 | `set dst 10.20.0.0 255.255.0.0` | `(1) #` | 点分掩码；**不配 dst 即默认路由 0.0.0.0/0** |
| 下一跳 | `set gateway 202.1.1.1` | `(1) #` | 对端设备接口地址 |
| 出接口 | `set device "wan1"` | `(1) #` | **必填**，写错或漏写该路由不工作 |
| 管理距离 | `set distance 10` | `(1) #` | 默认 10；须 ≤254，255 表示无效（见陷阱） |
| 优先级 | `set priority 0` | `(1) #` | distance 相同时比 priority，小者优先 |
| 黑洞路由 | `set blackhole enable` | `(1) #` | 匹配即丢包，常用于 VPN 断链兜底 |
| 停用但保留 | `set status disable` | `(1) #` | 条目留着不生效，排障时很方便 |
| 归属 VRF | `set vrf 0` | `(1) #` | 多 VRF 环境必须与实际 VRF 一致 |
| 查看静态路由 | `get router info routing-table static` | 操作模式 | 只看静态条目 |
| 查看整张路由表 | `get router info routing-table all` | 操作模式 | 回显 `[distance/priority]`，如 `[10/0]`、`[110/11]` |
| 查看路由数据库 | `get router info routing-table database` | 操作模式 | distance 255 的"无效路由"只能在这里看到 |
| 查看内核路由 | `diagnose ip route list` | 操作模式 | 与内核 FIB 对账时用 |
| 策略路由 | `config router policy` → `edit 1` → `set input-device "port3"` / `set output-device "wan1"` / `set gateway <IP>` | `(policy) #` | 按策略选路，优先于目的路由（以官方手册为准） |

## 典型示例

```text
// 场景：wan1 接 ISP（网关 202.1.1.1），去 10.20.0.0/16 走内网路由器 192.168.1.254
FortiGate # config router static
FortiGate (static) # edit 1
FortiGate (1) # set gateway 202.1.1.1          // 不写 dst → 默认路由 0.0.0.0/0
FortiGate (1) # set device "wan1"              // 必填，否则路由不生效
FortiGate (1) # next
FortiGate (static) # edit 2
FortiGate (2) # set dst 10.20.0.0 255.255.0.0   // 点分掩码
FortiGate (2) # set gateway 192.168.1.254
FortiGate (2) # set device "port3"
FortiGate (2) # set distance 10                 // 默认就是 10，显式写出便于比对
FortiGate (2) # next
FortiGate (static) # edit 3                     // 黑洞兜底：VPN 断链时丢弃业务网段
FortiGate (3) # set dst 192.168.123.0 255.255.255.0
FortiGate (3) # set distance 254                // 必须 ≤254；写 255 该条不生效！
FortiGate (3) # set blackhole enable
FortiGate (3) # next
FortiGate (static) # end

FortiGate # get router info routing-table all
// S*  0.0.0.0/0     [10/0]  via 202.1.1.1, wan1, [1/0]
// S   10.20.0.0/16  [10/0]  via 192.168.1.254, port3, [1/0]
// C   192.168.1.0/24 is directly connected, port3
FortiGate # get router info routing-table database | grep 192.168.123.0
FortiGate # diagnose ip route list                       // 内核 FIB 对账
```

## 陷阱与版本差异

- **FortiOS 的管理距离体系**：直连 0、静态 **10**、eBGP 20、OSPF **110**、RIP 120。因此静态路由**优先于** OSPF 学来的路由（`[10/0]` 胜过 `[110/11]`）；而华为是静态 60 / OSPF 10（OSPF 优先）、锐捷是静态 1 / OSPF 110（静态优先）——三家不能混用数值直觉，跨厂商迁移时这是最容易翻车的一条。
- `set distance 255` 表示"路由无效"：条目只出现在 `routing-table database`（回显带 `is a summary, Null`），不会进 FIB，于是流量照旧匹配默认路由被送出去 —— 这是"黑洞路由不生效"的官方典型原因，务必用 ≤254（如 254）。
- 只配 `set gateway` 不配 `set device`，路由不会工作；反之 `set device` 指向的接口 down 时该路由也会失效，双出口场景要配合链路探测/SD-WAN。
- `set dst` 省略就是默认路由；一旦有两条 `dst 0.0.0.0/0` 且 distance 相同，两条都会进表形成 ECMP（与"只能有一条默认路由"的交换机习惯不同），需要靠 distance 拉开。
- 改/删路由**不会**刷新已建立的会话——会话里缓存的还是旧下一跳，排障时要么等会话老化，要么 `diagnose sys session clear` 清掉相关会话。
- 掩码一律点分（`255.255.0.0`），写成 `/16` 会直接报语法错误；`edit` 的 ID 与 GUI 里看到的序号是同一套。

## 高危操作 ⚠️

- ⚠️ 删除或修改默认路由会让设备立即失去出口，全网断网；⚠️ 变更前 `execute backup config` 备份，并准备 Console 或第二条管理通道。
- ⚠️ 黑洞路由的 `distance` 设置过小（例如 1）会把正常业务流量按丢弃处理，现象是"目标网段完全不通"，⚠️ 只在明确需要兜底时启用并复核 distance。
- ⚠️ 目的网段/掩码敲错会形成路由环路或把大量流量导向错误出口，⚠️ 变更后用 `get router info routing-table all` + `execute traceroute` 双向验证。
- ⚠️ `delete <ID>` 是整条删除且立即生效，多 VRF/多 VDOM 环境下删错条目影响面很大；建议先用 `set status disable` 停用观察，再删除。

## 参考来源

1. [config router static（docs.fortinet.com，FortiOS 7.6.8 CLI Reference）](https://docs.fortinet.com/document/fortigate/7.6.8/cli-reference/200835411/config-router-static)
2. [router static（docs.fortinet.com，FortiOS 6.2.5 CLI Reference）](https://docs.fortinet.com/document/fortigate/6.2.5/cli-reference/531620/router-static)
3. [静态路由：CLI 写法、路径长度（distance）默认值、路由表验证（FortiGate 中文一本通）](https://support.fortinet.com.cn/routing/static_route)
4. [黑洞路由不生效：distance 255 的坑（FortiGate 中文一本通）](https://support.fortinet.com.cn/routing/route_faq/blackhole_ineffective)

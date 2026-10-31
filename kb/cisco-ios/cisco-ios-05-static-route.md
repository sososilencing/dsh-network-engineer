---
vendor: 思科（Cisco）
os: IOS / IOS-XE
topic: 静态路由
devices: Catalyst 9200/9300、ISR 1000/4000、ASR 1000 等
keywords: [静态路由, ip route, 默认路由, 浮动静态路由, 管理距离 AD, null 0, 黑洞路由, ip default-gateway, show ip route]
sources:
  - https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/iproute_pi/configuration/15-mt/iri-15-mt-book/iri-ip-prot-indep.html
  - https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/iproute_pi/configuration/15-mt/iri-15-mt-book.html
  - https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/iproute_pi/configuration/15-mt/iri-15-mt-book/iri-recursive-static-route.html
verified: high
updated: 2026-10-30
---

# 思科 · 静态路由

> 适用于 IOS/IOS-XE 的出口指向、浮动备份与黑洞防环。**关键差异**：思科静态路由默认管理距离（AD）为 **1**，数值越小越优，因此默认静态路由会**压过 OSPF（110）/RIP（120）/EIGRP（90）**；华为用 preference（静态 60）、锐捷用 AD（静态 1）——思科与锐捷同为 1，与华为 60 不是一套体系，不能直接比大小。

## 命令速查

| 功能 | 命令 | 视图 | 说明 |
|---|---|---|---|
| 配置静态路由 | `ip route 192.168.20.0 255.255.255.0 10.1.12.2` | `(config)#` | 目的网段 + 点分十进制掩码 + 下一跳 |
| 配置默认路由 | `ip route 0.0.0.0 0.0.0.0 10.1.12.2` | `(config)#` | 网关最后可用地址，覆盖所有未知目的 |
| 浮动静态路由 | `ip route 192.168.20.0 255.255.255.0 10.1.13.3 250` | `(config)#` | 末尾参数是 AD；主用默认 1，备用设 250 更大值 |
| 指定出接口 | `ip route 192.168.20.0 255.255.255.0 gigabitethernet 0/0/1 10.1.12.2` | `(config)#` | 广播型接口建议同时指出接口与下一跳 |
| 黑洞路由 | `ip route 10.1.0.0 255.255.0.0 null 0` | `(config)#` | 下一跳 null 0 直接丢弃，用于汇总防环/防扫描 |
| 删除静态路由 | `no ip route 192.168.20.0 255.255.255.0 10.1.12.2` | `(config)#` | 参数必须与原配置完全一致，否则报错 |
| 无路由时的网关 | `ip default-gateway 10.1.12.2` | `(config)#` | 仅在 `no ip routing`（二层模式）下生效 |
| 默认网络（动态协议用） | `ip default-network 192.168.0.0` | `(config)#` | 可被 IGRP/EIGRP/RIP 传播；OSPF 用 `default-information originate` |
| 查看路由表 | `show ip route` | 特权 EXEC `#` | 看 `[1/0]` 前段为 AD / 度量，S 表示静态 |
| 只看静态路由 | `show ip route static` | 特权 EXEC `#` | 确认是否真正进入路由表（而非仅在配置里） |
| 查静态配置 | `show running-config | include ip route` | 特权 EXEC `#` | 审计谁下发了哪些静态路由 |

### 常见管理距离（缺省值，数值越小越优）

| 路由来源 | 直连 | 静态 | eBGP | EIGRP 内部 | OSPF | IS-IS | RIP | iBGP | 未知 |
|---|---|---|---|---|---|---|---|---|---|
| 默认 AD | 0 | 1 | 20 | 90 | 110 | 115 | 120 | 200 | 255 |

## 典型示例

```text
R1# configure terminal
R1(config)# interface gigabitethernet 0/0/1
R1(config-if)# ip address 10.1.12.1 255.255.255.0
R1(config-if)# no shutdown
R1(config-if)# exit
R1(config)# ip route 192.168.20.0 255.255.255.0 10.1.12.2        // 主用，AD 默认 1
R1(config)# ip route 192.168.20.0 255.255.255.0 10.1.13.3 250    // 备份，浮动静态（AD 250）
R1(config)# ip route 0.0.0.0 0.0.0.0 10.1.12.2                   // 出口默认路由
R1(config)# ip route 10.1.0.0 255.255.0.0 null 0                 // 汇总黑洞，防环
R1(config)# end
R1# show ip route
R1# show ip route static
R1# show running-config | include ip route
```

## 陷阱与版本差异

- 思科静态路由 AD 默认 **1**（官方默认管理距离表），优于 OSPF 110 / RIP 120 / EIGRP 90；同目的网段存在动态路由时，静态会抢占并遮蔽动态条目，排查"路由不切换"先看 AD。
- AD 是 `ip route` 的**最后一个位置参数**：`ip route 192.168.20.0 255.255.255.0 250` 会被理解成"下一跳是 250"，语法可能被拒或生成错误路由；正确写法是先下一跳再 AD。
- 指向**出接口**的静态路由会被路由表视为直连（官方明确"lose their static nature"），并被动态协议通告；这类路由不做下一跳可达性检查，点到点/低端接入场景要清楚此副作用。
- 接口 down 或下一跳不可达时，对应静态路由会**自动从路由表移除**（官方描述）；因此浮动备份依赖的是下一跳可达性，不是链路带宽，需要更快的检测可叠加 BFD/对象跟踪。
- `ip default-gateway` 只在 `no ip routing` 时有效；三层设备上配它不会形成默认路由，别与 `ip route 0.0.0.0 0.0.0.0` 混用。

## 高危操作 ⚠️

- ⚠️ 删除出口默认路由 `no ip route 0.0.0.0 0.0.0.0 <下一跳>`：全网出网流量立刻中断；变更前备份配置并准备好回退命令，在维护窗口执行。
- ⚠️ 误配 `ip route 0.0.0.0 0.0.0.0 null 0`：所有未知目的流量被丢弃，等同断网；黑洞路由只用于具体汇总网段。
- ⚠️ 浮动备份的 AD 写反（备用值小于主用）：主备同时生效变成等价多路径，或备份抢占主用，链路规划失效；下发后必须 `show ip route` 核对两条路由的 active 状态。
- ⚠️ 下一跳写错不会报错：路由照常生成但转发不通，配置后必须 `ping`/`traceroute` 验证端到端连通性。

## 参考来源

1. [Configuring IP Routing Protocol-Independent Features（静态路由、默认路由、默认管理距离表）（cisco.com）](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/iproute_pi/configuration/15-mt/iri-15-mt-book/iri-ip-prot-indep.html)
2. [IP Routing: Protocol-Independent Configuration Guide, Cisco IOS 15M&T（cisco.com）](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/iproute_pi/configuration/15-mt/iri-15-mt-book.html)
3. [Configuring OSPF（对照 OSPF 默认 AD 110）（cisco.com）](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/iproute_ospf/configuration/15-mt/iro-15-mt-book/iro-cfg.html)

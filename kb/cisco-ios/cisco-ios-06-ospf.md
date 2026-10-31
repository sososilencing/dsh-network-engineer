---
vendor: 思科（Cisco）
os: IOS / IOS-XE
topic: OSPF
devices: Catalyst 9200/9300、ISR 1000/4000、ASR 1000 等
keywords: [OSPF, router ospf, network area, 反掩码 wildcard, passive-interface, router-id, default-information originate, show ip ospf neighbor, DR 选举, AD 110]
sources:
  - https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/iproute_ospf/configuration/15-mt/iro-15-mt-book/iro-cfg.html
  - https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/iproute_pi/configuration/15-mt/iri-15-mt-book/iri-ip-prot-indep.html
  - https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/iproute_ospf/configuration/15-mt/iro-15-mt-book.html
verified: high
updated: 2026-10-30
---

# 思科 · OSPF

> 适用于 IOS/IOS-XE 单区域/多区域 OSPFv2 部署。**关键差异**：`network` 用**反掩码（wildcard mask）**、进程号本地有效、OSPF 默认管理距离 110（intra/intera/external 均 110），静态路由（AD 1）会压过 OSPF。

## 命令速查

| 功能 | 命令 | 视图 | 说明 |
|---|---|---|---|
| 启用 OSPF 进程 | `router ospf 1` | `(config)#` | 进程号本地有效，两端可不同 |
| 宣告网段进区域 | `network 10.1.12.0 0.0.0.3 area 0` | `(config-router)#` | 第二个参数是**反掩码**，0.0.0.3 对应 /30 |
| 指定 Router ID | `router-id 1.1.1.1` | `(config-router)#` | 不配则取最高 loopback IP，其次最高物理口 IP |
| 抑制接口 Hello | `passive-interface gigabitethernet 1/0/2` | `(config-router)#` | 不建立邻居，但仍宣告该接口网段 |
| 全部设为被动 | `passive-interface default` | `(config-router)#` | 之后用 `no passive-interface` 逐个放开上联口 |
| 接口开销 | `ip ospf cost 10` | `(config-if)#` | 不改带宽值也能调选路，优于改 `bandwidth` |
| 调整 Hello/Dead | `ip ospf hello-interval 10` / `ip ospf dead-interval 40` | `(config-if)#` | 两端必须一致，否则邻居无法建立 |
| 网络类型 | `ip ospf network point-to-point` | `(config-if)#` | 点对点链路免 DR 选举，收敛更快 |
| DR 优先级 | `ip ospf priority 100` | `(config-if)#` | 数值大者优先；0 表示不参与 DR/BDR 选举 |
| 注入默认路由 | `default-information originate` | `(config-router)#` | 默认只在本地已有默认路由时才注入，加 `always` 强制 |
| 引入外部路由 | `redistribute static subnets` | `(config-router)#` | 必须带 `subnets`，否则只引入主类网段 |
| 区域汇总 | `area 1 range 10.1.0.0 255.255.0.0` | `(config-router)#` | 在 ABR 上把区域 1 明细汇总后通告其他区域 |
| 查看邻居 | `show ip ospf neighbor` | 特权 EXEC `#` | 关注 State 是否 FULL、是否有 DR/BDR |
| 查看接口参数 | `show ip ospf interface brief` | 特权 EXEC `#` | 一次核对 cost、状态、区域与邻居数 |
| 查看 OSPF 路由 | `show ip route ospf` | 特权 EXEC `#` | O 为区域内、O IA 区域间、O E2 外部 |
| 重置 OSPF 进程 | `clear ip ospf process` | 特权 EXEC `#` | ⚠️ 清邻居重建，见高危节 |

## 典型示例

```text
R1# configure terminal
R1(config)# interface loopback 0
R1(config-if)# ip address 1.1.1.1 255.255.255.255    // 稳定 Router ID
R1(config-if)# interface gigabitethernet 0/0/1
R1(config-if)# ip address 10.1.12.1 255.255.255.0
R1(config-if)# ip ospf cost 10
R1(config-if)# interface gigabitethernet 0/0/2
R1(config-if)# ip address 192.168.10.1 255.255.255.0
R1(config-if)# exit
R1(config)# router ospf 1
R1(config-router)# router-id 1.1.1.1
R1(config-router)# network 10.1.12.0 0.0.0.255 area 0     // 反掩码！不是 255.255.255.0
R1(config-router)# network 192.168.10.0 0.0.0.255 area 1
R1(config-router)# passive-interface gigabitethernet 0/0/2  // 接入口不发 Hello
R1(config-router)# default-information originate            // 向 OSPF 注入默认路由
R1(config-router)# end
R1# show ip ospf neighbor
R1# show ip ospf interface brief
R1# show ip route ospf
```

## 陷阱与版本差异

- `network` 必须写**反掩码**：`network 10.1.12.0 255.255.255.0 area 0` 掩码位全 0 匹配不上任何接口，路由一条也不宣告；这是华为/思科互转时最常踩的坑（正掩码 vs 反掩码）。
- OSPF 三类默认管理距离**都是 110**（官方说明：intra-area、interarea、external 均 110）；同目的网段若存在静态路由（AD 1），静态永远胜出，别误判为"OSPF 没起来"。
- `passive-interface` 只停 Hello 不发，接口网段照样宣告；把上联口误设 passive 会导致邻居起不来而路由还在，非常隐蔽。
- Router ID 取最高 loopback IP（无 loopback 时取最高物理口 IP）；改了 IP 或加新 loopback 后**不会自动切换**，需 `clear ip ospf process` 才生效（新版本 OSPF 重启后即生效，行为随版本略有差异）。
- `default-information originate` 默认只在本地路由表已有默认路由时才注入，需要无条件注入时加 `always`；`redistribute static` 不带 `subnets` 只引入主类网段。

## 高危操作 ⚠️

- ⚠️ `clear ip ospf process`：清空进程并重建所有邻居，收敛期间路由中断数十秒；全网点同时执行会造成大范围闪断，必须逐台执行并留观察窗口。
- ⚠️ 在现网修改 `network` 语句或区域号：会触发 SPF 重算甚至邻居重建，流量路径瞬时改变；变更前 `show ip ospf neighbor` 与 `show ip route ospf` 留基线。
- ⚠️ 两端 Hello/Dead 或认证参数不一致：邻居卡在 INIT/EXSTART 无法 FULL，业务不通且无明显告警；变更必须两端同步下发。
- ⚠️ 骨干区域 Area 0 被割裂：非骨干区域学到不完整路由，需先规划 virtual-link（或调整区域设计），不要在业务时段直接改区域拓扑。

## 参考来源

1. [Configuring OSPF（启用、接口参数、passive-interface、AD 110）（cisco.com）](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/iproute_ospf/configuration/15-mt/iro-15-mt-book/iro-cfg.html)
2. [IP Routing: Protocol-Independent Configuration Guide, Cisco IOS 15M&T（默认 AD 表、default-information originate 说明）（cisco.com）](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/iproute_pi/configuration/15-mt/iri-15-mt-book.html)
3. [Configuring IP Routing Protocol-Independent Features（默认管理距离表、default-information originate 说明）（cisco.com）](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/iproute_pi/configuration/15-mt/iri-15-mt-book/iri-ip-prot-indep.html)

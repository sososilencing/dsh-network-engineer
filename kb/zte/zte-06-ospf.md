---
vendor: 中兴（ZTE）
os: ZXR10 ROSNG / ZROS
topic: OSPF
devices: ZXR10 5900/5950/6900/8900 系列交换机、ZXR10 1800/2800/GER 系列路由器等
keywords: [中兴, ZXR10, OSPF, router ospf, router-id, network 反掩码, area 0, redistribute static, default-information originate, show ip ospf neighbor, show ip route ospf]
sources:
  - https://csl.zte.com.cn/manual_zh/topics/1684391333553.html
  - https://www.cnblogs.com/me80/p/7519120.html
  - https://www.cnblogs.com/me80/p/7519147.html
verified: medium
updated: 2026-10-30
---

# 中兴 · OSPF

> ZXR10 的 OSPF 是「Cisco 风格外壳 + 中兴细节」：`router ospf <进程号>` → `network <网段> <反掩码> area <区域>` 与 Cisco 同构，
> 但**进程视图名、接口参数的落点、掩码写法在不同版本上并不统一**（老版本 `(config-router)#`，5900E V3 系列为 `(config-ospfv2)#`/`(config-ospfv2-if)#`），照抄别家配置是最常见的翻车点。

## 命令速查

| 功能 | 命令 | 视图 | 说明 |
|---|---|---|---|
| 启动 OSPF 进程 | `router ospf 1` | 全局配置 `(config)#` | 进程号本地有效；进程视图提示符随版本为 `(config-router)#` 或 `(config-ospfv2)#`（以官方手册/售后确认） |
| 指定 Router ID | `router-id 1.1.1.1` | OSPF 视图 | 点分十进制；建议手工固定，勿依赖接口地址选举 |
| 宣告网段 | `network 10.1.12.0 0.0.0.255 area 0` | OSPF 视图 | **反掩码**；区域可写 `area 0` 或 `area 0.0.0.0`（等价性以官方手册确认） |
| 多网段/多区域 | 重复 `network ... area ...` | OSPF 视图 | 一个进程可挂多个区域，本机即成为 ABR |
| 引入静态路由 | `redistribute static` | OSPF 视图 | 可带 `metric <值>`、`metric-type 1` 或 `2`；缺省 metric/type 随版本（以官方手册/售后确认） |
| 引入直连/其他协议 | `redistribute connected` / `redistribute rip` | OSPF 视图 | 同上，参数以 `?` 补全为准 |
| 下发默认路由 | `default-information originate [always]` | OSPF 视图 | 不加 `always` 时本机需已有活动默认路由（以官方手册/售后确认） |
| 被动接口 | `passive-interface gei_1/1` | OSPF 视图 | 只宣告不建邻居；参数名/是否支持接口名随版本（以官方手册/售后确认） |
| 接口开销 | `ip ospf cost 10` | 接口视图 `(config-if)#` | 部分版本在 `(config-ospfv2-if)#` 用 `cost <值>`（以官方手册/售后确认） |
| Hello/Dead 定时器 | `ip ospf hello-interval 10` / `ip ospf dead-interval 40` | 接口视图 | 缺省 10/40，两端必须一致，否则邻居停在 Init/ExStart |
| DR 优先级 | `ip ospf priority 200` 或 `priority 200`（ospfv2-if 视图） | 接口视图 | 数值大者优先；写入方式随版本（以官方手册/售后确认） |
| 区域/接口认证 | 区域视图 `authentication` + 接口密钥命令 | 视版本 | 两端模式、Key-ID、密钥必须一致（以官方手册/售后确认） |
| 查看进程概况 | `show ip ospf` | 特权/配置视图 | 进程号、Router ID、区域数、SPF 计数 |
| 查看邻居 | `show ip ospf neighbor` | 特权/配置视图 | 状态 `Full` 才算邻接完成；`2-Way` 在 DRother 间属正常 |
| 查看接口参数 | `show ip ospf interface` | 特权/配置视图 | 核对区域、cost、Hello/Dead、DR/BDR |
| 查看链路状态库 | `show ip ospf database` | 特权/配置视图 | 排查区域间/外部 LSA 异常 |
| 查看虚拟链路 | `show ip ospf virtual-links` | 特权/配置视图 | 非骨干区域不连续时使用（实测回显见来源 2/3 同类机型） |
| 查看 OSPF 路由 | `show ip route ospf` | 特权/配置视图 | 确认外部/区域内路由是否入表 |
| 查看接口地址 | `show ip interface brief` | 特权/配置视图 | 先确认被宣告网段确实在本机 up 接口上 |
| 保存配置 | `write` | 特权视图 `ZXR10#` | 写入 `flash:/cfg/startrun.dat`，不 `write` 重启即丢 |

## 典型示例

```text
ZXR10>enable                                     // 进特权视图
ZXR10#configure terminal                         // 进全局配置
ZXR10(config)#interface gei_1/1
ZXR10(config-if)#ip address 10.1.12.1 255.255.255.0    // 注意：接口用点分「正掩码」
ZXR10(config-if)#no shutdown
ZXR10(config-if)#exit

ZXR10(config)#interface vlan 10
ZXR10(config-if)#ip address 192.168.10.1 255.255.255.0
ZXR10(config-if)#exit

ZXR10(config)#router ospf 1                      // 进程视图（本版本为 config-router，另一些版本为 config-ospfv2）
ZXR10(config-router)#router-id 1.1.1.1
ZXR10(config-router)#network 10.1.12.0 0.0.0.255 area 0        // 注意：network 用「反掩码」
ZXR10(config-router)#network 192.168.10.0 0.0.0.255 area 0
ZXR10(config-router)#redistribute static metric 20 metric-type 2   // 引入静态；参数以官方手册为准
ZXR10(config-router)#exit
ZXR10(config)#ip route 0.0.0.0 0.0.0.0 10.1.12.2               // 静态默认路由（正掩码）
ZXR10(config)#router ospf 1
ZXR10(config-router)#default-information originate always       // 无条件是下发的常用写法（以官方手册/售后确认）
ZXR10(config-router)#end
ZXR10#show ip ospf                                // 进程、Router ID、区域
ZXR10#show ip ospf neighbor                       // 期望 Full
ZXR10#show ip ospf interface                      // 核对 Area / cost / Hello-Dead
ZXR10#show ip route ospf                          // 核对计算出的路由
ZXR10#write
```

## 陷阱与版本差异

- 本机型/版本命令以官方手册为准：ZXR10 平台跨度极大（1800/2800/GER 老路由器、5900/6900/8900 交换机），进程视图名就有 `(config-router)#` 与 `(config-ospfv2)#` 两种，接口参数也有 `ip ospf cost` 与 ospfv2-if 视图 `cost` 两套写法；本文未给出实测回显的命令，一律先用 `?` 补全并与中兴售后确认。
- **正掩码与反掩码混用是头号坑**：`ip address 10.1.12.1 255.255.255.0`、`ip route 192.168.0.0 255.255.255.0 <下一跳>` 用**点分正掩码**，而 `network 10.1.12.0 0.0.0.255` 用**反掩码**；同一份配置里两种掩码并存（ACL 的 `rule ... permit 192.168.100.0 0.0.0.255` 也是反掩码），抄错一位就整段网段宣告失败。
- `network` 必须覆盖接口主 IP 所在网段：写成主机反掩码（`0.0.0.0`）或网段不匹配时，接口不收发 Hello，邻居永远起不来，且 `show ip ospf` 不会直接报错。
- `redistribute` 的缺省 metric 与 metric-type、`default-information originate` 是否支持 `always`，各版本不一致；不写参数时行为可能与预期不同（如缺省 Type-2 导致外部路由被优先），务必用 `show ip route ospf` 回看具体度量。
- 改 `router-id` 后通常要重启 OSPF 进程或整机才生效（部分版本需 `no router ospf <pid>` 重建）；重启命令各版本不同，以官方手册/售后确认为准，切勿在生产时段直接试。
- 邻居卡在 `Init`/`ExStart` 多为 Hello/Dead 不一致、区域号不一致或认证参数不一致；`2-Way` 出现在两台 DRother 之间属正常现象，不要误判故障。
- 宣告的网段只认「本机 up 接口上的主地址」；把 `interface vlan <id>` 的地址宣告出去前，先确认该 VLAN 有 up 的成员口，否则 `show ip interface brief` 是 down、OSPF 也不生效。
- 配置改完必须 `write`：ZXR10 不回自动保存，重启后未保存的 OSPF 配置全部丢失，表现为「重启后邻居全掉」。

## 高危操作 ⚠️

- ⚠️ `no router ospf <进程号>` 删除整个 OSPF 进程：该进程下所有 `network`、引入与接口参数一起消失，本机 OSPF 路由立刻撤销，跨网段业务中断；执行前 `show running-config` 导出并确认有静态/其他协议兜底。
- ⚠️ `no network <网段> <反掩码> area <区域>`：等于注销该网段的宣告，对端邻居可能直接 Down，路由表随之收敛，远端表现为「突然不通」。
- ⚠️ 修改 `router-id`：生效需重启进程，重启瞬间本机所有 OSPF 邻居重建、LSA 重算，全网路由短暂震荡；必须在维护窗口内、并提前通知上下游。
- ⚠️ 变更后立即 `write`：ZXR10 是覆盖式保存，错误配置一旦落盘，回退只能靠事先 `copy` 导出的 `startrun.dat`；标准动作是「先 `show running-config` + 导出，再改，验证通过再 `write`」。

## 参考来源

1. [中兴官方手册站 · CLI 配置（`write` 保存与 `startrun.dat` 导出流程）](https://csl.zte.com.cn/manual_zh/topics/1684391333553.html)（官方）
2. [中兴 ZXR10 6905 核心交换机配置案例（实测 `show running-config`：`ip address ... 255.255.255.0` 正掩码、`ip route`、`acl ... 0.0.0.255` 反掩码）](https://www.cnblogs.com/me80/p/7519120.html)
3. [中兴 ZXR10 GER4 核心路由器配置案例（实测 `show running-config`：接口正掩码与 ACL 反掩码并存、`write` 体系）](https://www.cnblogs.com/me80/p/7519147.html)

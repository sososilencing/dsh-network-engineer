---
vendor: 瞻博（Juniper）
os: Junos
topic: OSPF
devices: EX4300/EX4600 三层交换机、MX 系列路由器、SRX 系列防火墙
keywords: [OSPF, protocols ospf, area 0.0.0.0, interface, passive, router-id, export 策略, preference 10, 邻居, show ospf neighbor]
sources:
  - https://www.juniper.net/documentation/us/en/software/junos/ospf/topics/topic-map/configuring-ospf-areas.html
  - https://www.juniper.net/documentation/us/en/software/junos/cli/topics/topic-map/junos-configuration-commit.html
  - https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/show-route.html
verified: high
updated: 2026-10-30
---

# 瞻博（Juniper）· OSPF

> Junos 的 OSPF 不用 `network 网段 反掩码` 宣告，而是**直接在接口上挂区域**：`set protocols ospf area 0.0.0.0 interface ge-0/0/1.0`。引入外部路由必须借助 `policy-statement` 后 `export`，没有 `import-route` 一条命令的写法。

## 命令速查

| 功能 | 命令 | 视图 | 说明 |
|---|---|---|---|
| 指定 Router ID | `set routing-options router-id 1.1.1.1` | `[edit]` | 不配则取 lo0 或最低物理接口地址，易漂移 |
| 接口挂进区域 | `set protocols ospf area 0.0.0.0 interface ge-0/0/1.0` | `[edit]` | 带 `.0` 逻辑单元号；区域可写 `0` 或 `0.0.0.0` |
| 全网接口一次挂 | `set protocols ospf area 0.0.0.0 interface all` | `[edit]` | 会把管理口也拉进来，需配套排除 |
| 排除某接口 | `set protocols ospf area 0.0.0.0 interface fxp0.0 disable` | `[edit]` | 管理口/带外口必须排除 |
| 被动接口 | `set protocols ospf area 0.0.0.0 interface ge-0/0/5.0 passive` | `[edit]` | 只宣告网段、不发 Hello（对端为服务器/哑设备） |
| 接口开销 | `set protocols ospf area 0.0.0.0 interface ge-0/0/1.0 metric 20` | `[edit]` | 缺省按带宽换算，需两端核对 |
| DR 优先级 | `set protocols ospf area 0.0.0.0 interface ge-0/0/1.0 priority 200` | `[edit]` | 缺省 128，0 表示不参选；大者优先 |
| Hello/Dead 定时器 | `... hello-interval 5 dead-interval 20` | `[edit]` | 缺省 10/40（广播网），两端必须一致 |
| 区域类型 | `set protocols ospf area 0.0.0.1 stub default-metric 10` | `[edit]` | stub 需在 ABR 上配 default-metric 才会下默认路由 |
| 外部路由引入 | `set protocols ospf export EXPORT-STATIC` | `[edit]` | 需先用 policy-statement 定义引入范围 |
| 区域认证 | `set protocols ospf area 0.0.0.0 interface ge-0/0/1.0 authentication md5 1 key "xxx"` | `[edit]` | 两端 Key-ID 与密钥必须一致（以官方手册为准） |
| 参考带宽 | `set protocols ospf reference-bandwidth 10g` | `[edit]` | 影响 metric 换算，全网统一 |
| 查看邻居 | `show ospf neighbor` | 操作模式 | 状态 Full 才算邻接完成 |
| 查看邻居明细 | `show ospf neighbor detail` | 操作模式 | 含 DR/BDR、Dead 计时、区域 |
| 查看接口参数 | `show ospf interface detail` | 操作模式 | 核对 Area、metric、Hello/Dead、DR |
| 查看 OSPF 路由 | `show route protocol ospf` | 操作模式 | 方括号内 `[OSPF/10]` 为内部路由优先级 |
| 查看链路状态库 | `show ospf database` | 操作模式 | 排查区域间/外部 LSA 异常 |

## 典型示例

```text
user@R1> configure
[edit]
user@R1# set routing-options router-id 1.1.1.1                    // 手工固定，避免地址变化导致 LSA 抖动
user@R1# set protocols ospf reference-bandwidth 10g
user@R1# set protocols ospf area 0.0.0.0 interface ge-0/0/1.0     // 骨干区域直连接口
user@R1# set protocols ospf area 0.0.0.0 interface lo0.0 passive  // 宣告环回，不发 Hello
user@R1# set protocols ospf area 0.0.0.2 interface ge-0/0/2.0     // 第二个区域（本机成为 ABR）
user@R1# set protocols ospf area 0.0.0.2 interface ge-0/0/3.0 passive
user@R1# set protocols ospf area 0.0.0.0 interface ge-0/0/1.0 priority 200   // 争 DR
user@R1# set policy-options policy-statement EXPORT-STATIC term 1 from protocol static
user@R1# set policy-options policy-statement EXPORT-STATIC term 1 then accept
user@R1# set protocols ospf export EXPORT-STATIC                  // 引入静态/直连需显式 export
user@R1# commit confirmed 5
commit complete
user@R1# run show ospf neighbor                     // 期望 Full
user@R1# run show ospf interface detail | match "Area|metric|Hello"
user@R1# run show route protocol ospf               // 内部路由 [OSPF/10]，外部 [OSPF/150]
user@R1# commit                                       // 确认
```

## 陷阱与版本差异

- **没有 `network` 宣告**：接口一旦挂进 `area`，该接口所在网段就以 stub 网络形式通告；漏挂某个接口 = 该网段不进 OSPF，且不会有任何告警。
- `interface all` 很方便但危险：会把 `fxp0`/`me0` 等管理口拉进 OSPF，管理网段被泛洪甚至形成意外邻居；必须显式 `interface fxp0.0 disable`。
- 接口名要带逻辑单元（`ge-0/0/1.0`），只写 `ge-0/0/1` 在多数平台 commit 不生效或报错。
- Junos OSPF **内部路由 preference 10、外部（ASE）路由 preference 150**，数值小者优先；引出的静态路由在别的设备上看到的是 `[OSPF/150]`，此时它不如本地静态 `[Static/5]`。
- 引入路由必须 `policy-statement` + `export`，直接 `set protocols ospf export` 而策略未定义则什么也引不进去；且 export 策略里**末尾默认拒绝**，别忘 `then accept`。
- Router ID 不手工配置时可能随接口地址变化（取 loopback 中最低地址，否则取最低物理接口地址），生产网建议显式 `routing-options router-id`。
- 缺省 metric 与 `reference-bandwidth` 相关，未统一参考带宽时同带宽链路 metric 可能不同，导致选路反常；用 `show ospf interface detail` 两端比对。
- 区域认证语句在 EX / MX / SRX 与不同 Junos 版本上写法有差异（`authentication-type`、`md5`、`simple-password`），跨平台照抄前先 `set ... authentication ?` 看补全（以官方手册为准）。

## 高危操作 ⚠️

- ⚠️ 修改 Router ID 或 `reference-bandwidth`：全区域 LSA 重算、邻居重建，路由表瞬间震荡。
- ⚠️ 删除整个 OSPF 进程（`delete protocols ospf`）：本机所有 OSPF 路由消失，跨区业务中断，务必 `commit confirmed N`。
- ⚠️ 变更 Hello/Dead 或认证参数：单侧先改会立刻邻居 Down，必须在维护窗口两端同时改。
- ⚠️ 把管理口误挂进 `interface all`：管理网段被宣告且可能被对端优选，造成管理流量绕行甚至环路。

## 参考来源

1. [Configuring OSPF Areas · 单/多区域、DR 优先级 128、router-id 选择规则（juniper.net）](https://www.juniper.net/documentation/us/en/software/junos/ospf/topics/topic-map/configuring-ospf-areas.html)
2. [Commit the Configuration · 提交与 commit confirmed（juniper.net）](https://www.juniper.net/documentation/us/en/software/junos/cli/topics/topic-map/junos-configuration-commit.html)
3. [show route · 协议与 preference 显示（juniper.net）](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/show-route.html)

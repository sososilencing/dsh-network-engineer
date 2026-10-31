---
vendor: 飞塔（Fortinet）
os: FortiOS
topic: 链路聚合（802.3ad / LACP）
devices: FortiGate 40F/60F/100F/600E/1800F 等支持聚合的型号（端口数量与组合受型号限制）
keywords: [链路聚合, LACP, 802.3ad, set type aggregate, set member, lacp-mode active/passive/static, set algorithm L3/L4, diagnose netlink aggregate, 端口绑定]
sources:
  - https://docs.fortinet.com/document/fortigate/7.6.8/cli-reference/317104469/config-system-interface
  - https://docs.fortinet.com/document/fortigate/7.2.14/cli-reference/101854390/config-system-virtual-switch
  - https://support.fortinet.com.cn/network_mgmt/link_aggregation
  - https://support.fortinet.com.cn/troubleshooting/diagnostics_cli
verified: high
updated: 2026-10-30
---

# 飞塔（Fortinet）· 链路聚合

> FortiGate 的聚合口也是"接口对象"：`config system interface` → `edit agg1` → `set type aggregate` + `set member "port1" "port2"`，再选 LACP 模式（active/passive/static）与哈希算法（L3/L4）。成员口加入后从接口列表消失，必须提前清空 IP。

## 命令速查

| 功能 | 命令 | 视图 | 说明 |
|---|---|---|---|
| 进入聚合接口 | `config system interface` → `edit "agg1"` | `(interface) #` | 聚合口名称自定（如 agg1/agg2），建议与对端一致 |
| 声明类型 | `set type aggregate` | `(agg1) #` | 缺这条只是普通接口，成员参数无效 |
| 绑定成员口 | `set member "port1" "port2"` | `(agg1) #` | 成员口须先清空 IP 且未被策略/路由引用 |
| LACP 模式 | `set lacp-mode active` | `(agg1) #` | `active` 主动发 LACPDU、`passive` 等对端、`static` 手工聚合不跑 LACP |
| 负载均衡算法 | `set algorithm L3` | `(agg1) #` | `L3` 按 IP 哈希、`L4` 按四层端口哈希；默认值以回显为准 |
| 配 IP | `set ip 10.0.0.1 255.255.255.0` | `(agg1) #` | 点分掩码；聚合口就是一个三层接口 |
| 管理访问 | `set allowaccess ping https ssh` | `(agg1) #` | 覆盖式，不显式开启则管理不通 |
| 关闭设备探测 | `set device-identification disable` | `(agg1) #` | 官方建议关闭，探测耗资源且可能影响加速 |
| 查聚合/LACP 状态 | `diagnose netlink aggregate name agg1` | 操作模式 | 看 LACP mode、成员 link status、`LACP state: established`、actor/partner key |
| 查聚合口配置 | `show full-configuration system interface agg1` | 操作模式 | 核对 type/member/lacp-mode/algorithm |
| 查成员口硬件与统计 | `diagnose hardware deviceinfo nic port1` | 操作模式 | 判断是"成员没起"还是"链路层丢包" |
| 查二层交换接口 | `diagnose netlink brctl list` | 操作模式 | 确认聚合口是否被纳入某个交换接口 |

## 典型示例

```text
// 场景：port1+port2 与交换机做 LACP 主动模式聚合，承载 10.0.0.0/24
FortiGate # config system interface
FortiGate (interface) # edit "agg1"
FortiGate (agg1) # set vdom "root"
FortiGate (agg1) # set type aggregate                 // 关键：必须是 aggregate
FortiGate (agg1) # set member "port1" "port2"         // 成员物理口
FortiGate (agg1) # set lacp-mode active               // 主动发 LACPDU；对端可 active 或 passive
FortiGate (agg1) # set algorithm L3                   // 按源/目的 IP 哈希（L4 则含端口）
FortiGate (agg1) # set ip 10.0.0.1 255.255.255.0
FortiGate (agg1) # set allowaccess ping
FortiGate (agg1) # set device-identification disable   // 官方建议关闭，省资源
FortiGate (agg1) # next
FortiGate (interface) # end

// 与交换机手工聚合（对端 mode on / static）对接时改用：
//   FortiGate (agg1) # set lacp-mode static

FortiGate # diagnose netlink aggregate name agg1       // 验证：status up、成员 link status up
                                                       //        LACP state: established、MUX COLLECTING_DISTRIBUTING
FortiGate # show full-configuration system interface agg1
FortiGate # diagnose hardware deviceinfo nic port1     // 成员口速率/双工/错包
FortiGate # execute ping 10.0.0.254                    // 三层验证
```

## 陷阱与版本差异

- 成员物理口一旦加入聚合口，就从接口列表中"消失"，不能再单独配 IP/allowaccess；要拆出来必须先 `set member` 里去掉或删除聚合口。
- 三种模式要和交换机对齐：`static`（手工聚合，对应交换机静态/on 模式）、`active`、`passive`。**两端都配 passive 永远协商不起来**，现象是成员口 link up 但流量只走一条或完全不通。
- 官方中文文档对 `set lacp-mode` 的描述是"默认为动态"（参与 LACP 协商），不同版本默认值表述不一致，首次配置后用 `show full-configuration system interface agg1` 与 `diagnose netlink aggregate` 双向确认。
- `set algorithm` 只决定哈希选举，不提升单流带宽：同一条流（同源同目的 IP，L3 模式）固定走一条成员，压测要用多流。
- 官方明确建议关闭聚合口上的"设备探测"（Device detection），否则消耗较多设备资源，且可能影响流量加速。
- 可聚合端口数量和组合受型号硬件结构限制（通常要求同一硬件交换模块内的端口），小型号上并非任意两个口都能绑，选型时查硬件指南。
- 回显里的 `min-links`、`link-up-delay`、`LACP speed` 等参数在 `diagnose netlink aggregate` 可见，但其 CLI 配置字段随型号/版本差异较大，改前以官方手册为准。

## 高危操作 ⚠️

- ⚠️ 删除聚合接口或在 `set member` 中摘除成员，会让该链路立即断流；若聚合口是上联或内网主干，等于整段断网，务必在维护窗口操作。
- ⚠️ 两端 LACP 模式不匹配（一端 active/一端 static，或双 passive）会造成"物理 up、业务不通"的隐蔽故障，⚠️ 变更前后都要看 `diagnose netlink aggregate name agg1` 的 actor/partner key 是否一致。
- ⚠️ 修改 `set algorithm` 会重置哈希，已有会话可能被重新分配到另一条成员，表现为短时丢包/会话重建。
- ⚠️ 不要在承载 HA 心跳口或 FortiLink 的口上随意做聚合变更，⚠️ 可能触发 HA 切换或 FortiSwitch 脱管。

## 参考来源

1. [config system interface（docs.fortinet.com，FortiOS 7.6.8 CLI Reference，含 type aggregate / member / lacp-mode）](https://docs.fortinet.com/document/fortigate/7.6.8/cli-reference/317104469/config-system-interface)
2. [config system virtual-switch（docs.fortinet.com，FortiOS 7.2.14 CLI Reference）](https://docs.fortinet.com/document/fortigate/7.2.14/cli-reference/101854390/config-system-virtual-switch)
3. [端口聚合配置：LACP 模式、algorithm、diagnose netlink aggregate（FortiGate 中文一本通）](https://support.fortinet.com.cn/network_mgmt/link_aggregation)
4. [常用排错命令：接口与硬件信息查询（FortiGate 中文一本通）](https://support.fortinet.com.cn/troubleshooting/diagnostics_cli)

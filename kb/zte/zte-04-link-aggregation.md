---
vendor: 中兴（ZTE）
os: ZXR10 ROSNG / ZROS
topic: 链路聚合（smartgroup）
devices: ZXR10 5900/5950/6900/8900 系列交换机（含 5260/5960 等盒式型号）
keywords: [中兴, ZXR10, smartgroup, 链路聚合, LACP, mode active, mode passive, show lacp internal, 与 port-channel 差异]
sources:
  - https://xxc.lyu.edu.cn/_upload/article/files/f2/64/a9cd33264ad69c07614429415226/79074bd1-be96-47e3-b95c-f9e34595b03b.pdf
  - https://developer.aliyun.com/article/1171627
  - http://nova-minsk.com/ZTE/59E/SJ-20150114102049-005-ZXR10%205900E%20Series%20(V3.00.11)Configuration%20Guide%20(Link%20Layer).pdf
  - https://csl.zte.com.cn/manual_zh/topics/1684391333553.html
verified: medium
updated: 2026-10-30
---

# 中兴 · 链路聚合（smartgroup）

> **本主题与 Cisco 差异最大，务必先记住这一条**：ZXR10 的聚合逻辑接口叫 `smartgroup<N>`（如 `interface smartgroup1`），
> 而**加入动作是在物理成员口上敲 `smartgroup <N> mode active|passive`**——不是 Cisco 的 `channel-group 1 mode active`，
> 也不是华为/华三的 `Eth-Trunk`/`Bridge-Aggregation` 集中式写法。

## 命令速查

| 功能 | 命令 | 视图 | 说明 |
|---|---|---|---|
| 创建/进入聚合口 | `interface smartgroup1` | 全局配置 `(config)#` | 逻辑接口名固定 `smartgroup<编号>`；编号即聚合组号 |
| 成员口加入（LACP 主动） | `smartgroup 1 mode active` | 物理口视图 `(config-if)#` | 在成员口上配置；`1` 为聚合组号，`active` 为 LACP 主动端 |
| 成员口加入（LACP 被动） | `smartgroup 1 mode passive` | 物理口视图 | 两端不能同时 passive，至少一端 active |
| 聚合口设为 Trunk | `switchport mode trunk` | 聚合口视图 | VLAN 属性配在 `smartgroup` 上，**不配在成员口上** |
| 聚合口放行 VLAN | `switchport trunk vlan 10 20 100` | 聚合口视图 | 同单口的 trunk 语义 |
| 聚合口设 Access | `switchport mode access` + `switchport access vlan 10` | 聚合口视图 | 聚合口对下接服务器/终端时使用 |
| 聚合口描述 | `description LAG-To-Core01` | 聚合口视图 | 台账价值高于命令行价值，建议必配 |
| 打开聚合口 | `no shutdown` | 聚合口视图 | 聚合口新建后状态随成员口与配置而定 |
| 查看 LACP 状态 | `show lacp internal` | 特权/配置视图 | 查看本端 LACP 运行信息，判断成员口是否真的协商成功 |
| 查看聚合口流量 | `show interface smartgroup1` | 特权/配置视图 | 核对聚合口 up/down 与速率、利用率 |
| 查看成员口 | `show interface gei_1/1` | 特权/配置视图 | 成员口应显示归属聚合组且物理 up |
| 查看聚合相关配置 | `show running-config interface smartgroup1` | 特权/配置视图 | 确认 VLAN 属性配在聚合口而不是成员口 |
| 负载分担策略 | `smartgroup load-balance <策略>` | 全局配置 | 具体关键字与可选算法随版本（以官方手册/售后确认） |

## 典型示例

```text
ZXR10>enable
ZXR10#configure terminal

ZXR10(config)#interface smartgroup1              // 1) 先建逻辑聚合口
ZXR10(config-if)#description LAG-To-Core01
ZXR10(config-if)#switchport mode trunk           // 2) VLAN 属性配在聚合口上
ZXR10(config-if)#switchport trunk vlan 10 20 100
ZXR10(config-if)#exit

ZXR10(config)#interface gei_1/1                  // 3) 成员口加入聚合组（LACP 主动）
ZXR10(config-if)#smartgroup 1 mode active
ZXR10(config-if)#exit
ZXR10(config)#interface gei_1/2
ZXR10(config-if)#smartgroup 1 mode active
ZXR10(config-if)#exit

ZXR10(config)#end
ZXR10#show lacp internal                         // 4) 验证协商结果
ZXR10#show interface smartgroup1
ZXR10#show running-config interface smartgroup1
ZXR10#write
```

```text
// 对端为被动端时的典型搭配：本端 active，对端 passive
ZXR10(config)#interface gei_1/8
ZXR10(config-if)#smartgroup 1 mode passive       // 注意：两端都 passive 不建立
```

## 陷阱与版本差异

- 本机型/版本命令以官方手册为准：`interface smartgroup<N>` 的创建方式、`smartgroup <N> mode` 是否还支持静态（非 LACP）取值、负载分担关键字，各版本差异明显，未给回显的参数请先用 `?` 核对。
- **别照抄 Cisco**：Cisco 是「建 `port-channel` → 成员口 `channel-group N mode active`」；ZXR10 是「建 `smartgroup<N>` → 成员口 `smartgroup N mode active`」。语法看着像，聚合对象名与命令位置都不同，混用必报错。
- **别照抄华为/华三**：`Eth-Trunk` / `Bridge-Aggregation` + `port link-aggregation group` 在中兴上都不存在，跨厂商开局时两侧要分别按各自语法写。
- VLAN 属性必须配在 `smartgroup` 逻辑口上；配到成员口上会被聚合逻辑覆盖或直接不生效，这是"聚合 up 但业务不通"的第一大原因。
- 两端 `active/passive` 不能都为 passive；成员口速率、双工、VLAN、pvid 必须一致，否则成员口处于 selected/unselected 之外的状态而不转发。
- 删除顺序有讲究：先 `no smartgroup <N> mode ...` 把成员口摘出，再删 `smartgroup` 接口；反过来先删逻辑口，成员口可能残留归属配置导致二层异常。
- 单条大流量（单会话）在聚合链路上仍只会走一条物理链路，看到"聚合后带宽没涨"先确认是否多流；必要时调整负载分担算法（以官方手册确认可选算法）。

## 高危操作 ⚠️

- ⚠️ 在线摘除成员口（`no smartgroup N mode ...`）会让该链路流量瞬间切到剩余成员，若剩余带宽不足将丢包；变更前确认带宽余量与业务低峰窗口。
- ⚠️ 删除整个 `smartgroup` 接口会同时打断其承载的所有 VLAN 业务，且成员口归属配置可能残留；删前先导出配置（`show running-config` + `copy`）并确认无其他口引用。
- ⚠️ 修改聚合口 `switchport mode`/`switchport trunk vlan` 会重构标签处理，误删管理 VLAN 将导致失联；改动前保留 Console 通道。
- ⚠️ 两端聚合模式与成员数不一致会造成部分链路不转发甚至环路异常，改一端前先确认对端可同步变更。

## 参考来源

1. [中兴 ZXR10 配置手册（PDF，含 `smartgroup 2 mode active` / `smartgroup 1 mode passive`、`switchport mode hybrid` 等成员口配置回显）](https://xxc.lyu.edu.cn/_upload/article/files/f2/64/a9cd33264ad69c07614429415226/79074bd1-be96-47e3-b95c-f9e34595b03b.pdf)
2. [ZXR10 5900E 系列（V3.00.11）配置指导 · 链路层（官方文档镜像，含 `smartgroup` 聚合章节）](http://nova-minsk.com/ZTE/59E/SJ-20150114102049-005-ZXR10%205900E%20Series%20(V3.00.11)Configuration%20Guide%20(Link%20Layer).pdf)
3. [中兴设备 show 命令大全（`show lacp internal` 查看聚合状态）](https://developer.aliyun.com/article/1171627)
4. [中兴官方手册站 · CLI 配置（变更后 `write` 保存与配置导出）](https://csl.zte.com.cn/manual_zh/topics/1684391333553.html)（官方）

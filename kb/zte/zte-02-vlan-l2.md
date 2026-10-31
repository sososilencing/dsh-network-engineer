---
vendor: 中兴（ZTE）
os: ZXR10 ROSNG / ZROS
topic: VLAN 与二层
devices: ZXR10 5900/5950/6900/8900 系列交换机（含 5260/5960 等盒式型号）
keywords: [中兴, ZXR10, vlan, switchport mode access, switchport mode trunk, switchport access vlan, switchport trunk vlan, hybrid, show vlan, 生成树]
sources:
  - https://www.jinchutou.com/shtml/a85d8cee99dc0393c28ace1e9cf7125b.html
  - https://patents.google.com/patent/CN101262503B/en
  - https://csl.zte.com.cn/manual_zh/topics/1684391333553.html
  - http://szdingwei.net/html/6432195423.html
verified: medium
updated: 2026-10-30
---

# 中兴 · VLAN 与二层

> ZXR10 的二层是**最像 Cisco** 的一块：`vlan <id>` 建 VLAN、`interface <口>` 进接口、`switchport mode access|trunk|hybrid`
> 再配 `switchport access vlan` / `switchport trunk vlan`。差异是接口名 `gei_1/1`、trunk 放行 VLAN 用 `switchport trunk vlan` 而非 `allowed vlan`。

## 命令速查

| 功能 | 命令 | 视图 | 说明 |
|---|---|---|---|
| 创建 VLAN | `vlan 10` | 全局配置 `(config)#` | 进入 VLAN 视图 `(config-vlan)#`；VLAN 必须先建，接口才能引用 |
| VLAN 命名 | `name Office-2F` | VLAN 视图 `(config-vlan)#` | 便于台账核对；关键字以官方手册确认 |
| 进入物理端口 | `interface gei_1/1` | 全局配置 | 命名规则 `类型_槽位/端口`；FEI=百兆、GEI=千兆、XGEI=万兆 |
| 端口置为 Access | `switchport mode access` | 接口视图 `(config-if)#` | 对应 Cisco `switchport mode access` |
| 加入 Access VLAN | `switchport access vlan 10` | 接口视图 | 该端口只承载一个无标签 VLAN |
| 端口置为 Trunk | `switchport mode trunk` | 接口视图 | 上联/交换机互联口常用 |
| Trunk 放行 VLAN | `switchport trunk vlan 10 20 100` | 接口视图 | 空格分隔多个 VLAN；**不是** Cisco 的 `switchport trunk allowed vlan` |
| 端口置为 Hybrid | `switchport mode hybrid` | 接口视图 | 中兴支持的第三种模式，可同时带标签/不带标签放行多个 VLAN |
| 端口描述 | `description to-core-01-gei_1/24` | 接口视图 | 建议写「对端设备-端口-业务」 |
| 打开端口 | `no shutdown` | 接口视图 | 端口默认是否 shutdown 随型号/版本，以实际状态为准 |
| 广播风暴抑制 | `broadcast-limit 10` | 接口视图 | 阈值单位随型号；接入层防环兜底用 |
| 查看 VLAN 表 | `show vlan` | 特权/配置视图 | 输出含 `Vlan_Cfg` 与各端口成员关系，核对端口是否真的进了 VLAN |
| 查看接口二层属性 | `show running-config interface gei_2/4` | 特权/配置视图 | 可直接看到 `switchport access vlan` / `broadcast-limit` 等实际生效配置 |
| 查看端口状态 | `show interface gei_2/1` | 特权/配置视图 | 输出含 `vlan mode is access, pvid 2`，用 pvid 反查是否配错 VLAN |
| 查看 MAC/转发表 | `show mac`（部分版本为 `show fdb`） | 特权视图 | 命令名随版本差异较大，以官方手册确认 |

## 典型示例

```text
ZXR10>enable
ZXR10#configure terminal
ZXR10(config)#vlan 10                      // 建业务 VLAN 10
ZXR10(config-vlan)#name Office-2F
ZXR10(config-vlan)#exit
ZXR10(config)#vlan 20
ZXR10(config-vlan)#exit
ZXR10(config)#vlan 100                     // 管理 VLAN
ZXR10(config-vlan)#exit

ZXR10(config)#interface gei_1/1            // 接入口
ZXR10(config-if)#switchport mode access
ZXR10(config-if)#switchport access vlan 10
ZXR10(config-if)#description PC-Floor2
ZXR10(config-if)#broadcast-limit 10
ZXR10(config-if)#no shutdown
ZXR10(config-if)#exit

ZXR10(config)#interface gei_1/24           // 上联 trunk
ZXR10(config-if)#switchport mode trunk
ZXR10(config-if)#switchport trunk vlan 10 20 100
ZXR10(config-if)#exit

ZXR10(config)#end
ZXR10#show vlan                            // 核对端口成员关系
ZXR10#show running-config interface gei_1/24
ZXR10#write
```

## 陷阱与版本差异

- 本机型/版本命令以官方手册为准：VLAN 命名、MAC 表查看、生成树相关命令（`spanning-tree` 系列）在 5900/6900/8900 与盒式型号上差异较大，本文未给出回显的一律先用 `?` 核对。
- Trunk 放行 VLAN 的命令是 `switchport trunk vlan <列表>`，**不是** Cisco 的 `switchport trunk allowed vlan`；照抄 Cisco 会直接报语法错。
- 接口名是 `gei_1/1`（类型_槽位/端口），与 Cisco `Gi1/0/1` 不是"少写一个 0"这么简单：ZXR10 的 `1/1` 是「板卡 1 / 端口 1」，与 Cisco 的「槽位/子槽位/端口」三段式不一一对应，迁移配置时逐口核对，别做字符串替换。
- 端口模式有 `access` / `trunk` / `hybrid` 三种，`switchport mode hybrid` 是中兴侧常见写法；跨厂商互联时对端若无 hybrid，按 access/trunk 语义重新规划，不要硬配。
- `show interface` 输出里的 `vlan mode is access, pvid 2` 是判断"端口到底属于哪个 VLAN"的最快证据；`show vlan` 只反映成员关系，不反映 pvid。
- 部分文档把命令写作 `switch-port access vlan`（带连字符），实际设备以 `switchport` 为准，遇到报错先按设备 `?` 提示输入。
- 端口"配了 VLAN 但业务不通"时，先查 pvid 与对端 trunk 放行列表，再查生成树状态；ZXR10 的生成树默认状态与模式随型号差异大，未核实前不要假设"默认已开 MSTP"。

## 高危操作 ⚠️

- ⚠️ 在线修改上联 trunk 的 `switchport trunk vlan` 列表会瞬时中断被移除 VLAN 的转发，误删管理 VLAN 会直接失联；变更前确认管理路径存在第二条通道（Console 或带外）。
- ⚠️ 在承载业务的端口上切 `switchport mode`（access ↔ trunk ↔ hybrid）会改变标签处理方式，整段业务可能中断；必须在维护窗口逐口变更并立即用 `show vlan` 验证。
- ⚠️ 删除 VLAN（`no vlan <id>`）前先确认没有端口成员与 SVI 引用，否则端口悬空、三层网关消失。
- ⚠️ 批量改端口前先导出配置（`show running-config` + `copy ... ftp`），否则误操作只能靠手工回溯。

## 参考来源

1. [ZXR10 路由交换机基本操作和配置 V2.1 课件（端口类型 FEI/GEI/XGEI、`switchport access vlan`、`broadcast-limit`、`show interface` 回显）](https://www.jinchutou.com/shtml/a85d8cee99dc0393c28ace1e9cf7125b.html)
2. [中兴专利 CN101262503B（含 ZXR10 `vlan 10`、`interface fei_1/1`、`switchport access vlan 10` 实测配置例）](https://patents.google.com/patent/CN101262503B/en)（中兴官方申请文件）
3. [中兴官方手册站 · CLI 配置（配置保存与 startrun.dat）](https://csl.zte.com.cn/manual_zh/topics/1684391333553.html)（官方）
4. [ZXR10 1800 路由器配置流程（`switchport`/`porttype l3` 语境下的端口配置）](http://szdingwei.net/html/6432195423.html)

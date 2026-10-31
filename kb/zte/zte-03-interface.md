---
vendor: 中兴（ZTE）
os: ZXR10 ROSNG / ZROS
topic: 接口配置（二层/三层、VLAN 接口）
devices: ZXR10 5900/5950/6900/8900 系列交换机、ZXR10 1800/2800 系列路由器等
keywords: [中兴, ZXR10, interface gei_1/1, interface vlan, ip address, porttype l3, negotiation auto, speed duplex, show ip interface brief]
sources:
  - https://www.jinchutou.com/shtml/a85d8cee99dc0393c28ace1e9cf7125b.html
  - https://patents.google.com/patent/CN101262503B/en
  - http://szdingwei.net/html/6432195423.html
  - https://developer.aliyun.com/article/1171627
verified: medium
updated: 2026-10-30
---

# 中兴 · 接口配置

> ZXR10 的接口分两类：物理口 `gei_1/1`（`类型_槽位/端口`）与三层接口 `interface vlan10`。
> 物理口默认多为二层，要配 IP 必须先确定端口属性（交换机走 VLAN 接口，路由器平台用 `porttype l3` 把口变三层）。

## 命令速查

| 功能 | 命令 | 视图 | 说明 |
|---|---|---|---|
| 进入物理端口 | `interface gei_1/1` | 全局配置 `(config)#` | 类型名：`fei_`（百兆）、`gei_`（千兆）、`xgei_`（万兆）；`_槽位/端口` |
| 打开端口 | `no shutdown` | 接口视图 `(config-if)#` | 端口默认状态随型号/版本，用 `show interface` 确认 |
| 端口描述 | `description Uplink-To-Core` | 接口视图 | 出现在 `show interface` 的 `Description` 行 |
| 开启自协商 | `negotiation auto` | 接口视图 | 光口/电口均适用；手工固定速率双工前先确认对端一致 |
| 固定双工 | `duplex full` | 接口视图 | 取值 `half` / `full`；与 `speed` 通常成对配置 |
| 固定速率 | `speed 1000` | 接口视图 | 取值 `10` / `100` / `1000`；万兆口写法以版本为准 |
| 端口转三层 | `porttype l3` | 接口视图 | 路由器平台/部分型号把交换口变三层口后才可 `ip address` |
| 配置接口 IP | `ip address 10.1.1.1 255.255.255.0` | 接口视图 | **点分十进制掩码**，不是前缀长度、也不是反掩码 |
| 进入 VLAN 三层接口 | `interface vlan 10` | 全局配置 | VLAN 必须已创建；部分版本提示符为 `(config-if-vlan10)#` |
| VLAN 接口配 IP | `ip address 10.1.1.1 255.255.255.0` | VLAN 接口视图 | 交换机做网关的 standard 做法 |
| 端口加入 VLAN | `switchport access vlan 10` | 接口视图 | 三层口改二层或回切时使用，语义同 Cisco |
| 查看三层接口概览 | `show ip interface brief` | 特权/配置视图 | 一眼看 IP、状态、协议状态，排障第一条命令 |
| 查看单口明细 | `show interface gei_2/1` | 特权/配置视图 | 含双工、速率、`vlan mode is access, pvid 2`、速率/利用率统计 |
| 查看单口配置 | `show running-config interface fei_2/4` | 特权/配置视图 | 只列该口实际生效配置，核对"配了没生效"最快 |
| 端口限速/抑制 | `broadcast-limit 10` | 接口视图 | 广播抑制阈值，单位随型号（以官方手册确认） |

## 典型示例

```text
ZXR10>enable
ZXR10#configure terminal

ZXR10(config)#interface gei_1/24                     // 千兆上联口
ZXR10(config-if)#description Uplink-To-Core-01
ZXR10(config-if)#negotiation auto
ZXR10(config-if)#no shutdown
ZXR10(config-if)#exit

ZXR10(config)#vlan 10                                // 三层网关 VLAN（交换机方案）
ZXR10(config-vlan)#exit
ZXR10(config)#interface vlan 10
ZXR10(config-if)#ip address 10.1.1.1 255.255.255.0   // 点分掩码
ZXR10(config-if)#no shutdown
ZXR10(config-if)#exit

ZXR10(config)#interface gei_1/1                      // 二层接入口
ZXR10(config-if)#switchport mode access
ZXR10(config-if)#switchport access vlan 10
ZXR10(config-if)#exit

ZXR10(config)#end
ZXR10#show ip interface brief                        // 核对地址与 up/down
ZXR10#show interface gei_1/24
ZXR10#write
```

```text
// 路由器平台：把交换口变三层口后直接配 IP（ZXR10 1800 实测链）
ZXR10(config)#interface fei_1/2
ZXR10(config-if)#porttype l3
ZXR10(config-if)#ip address 192.168.1.1 255.255.255.0
ZXR10(config-if)#exit
```

## 陷阱与版本差异

- 本机型/版本命令以官方手册为准：`porttype l3`、万兆口速率写法、VLAN 接口提示符格式在不同型号/版本上不一致，未在本文给出回显的参数请先用 `?` 核对。
- 掩码一律用点分十进制（`255.255.255.0`），既不是 Cisco/华为的 `ip address 10.1.1.1 24`，也不是 OSPF 的反掩码；同一份配置里正掩码与反掩码并存，最易抄错。
- 交换机与路由器的三层做法不同：交换机是「建 VLAN → `interface vlan <id>` → 配 IP」；路由器/部分平台要先把物理口 `porttype l3` 再配 IP。直接对二层口 `ip address` 会报错或不下发。
- VLAN 接口要生效，前提是 VLAN 已创建且至少有一个 up 的成员口，否则 `show ip interface brief` 显示 down，容易误判成配错地址。
- 接口名 `gei_1/1` 与 Cisco `Gi1/0/1` 不是同一编号体系：ZXR10 是「板卡/端口」两段，Cisco 常是「槽位/子槽位/端口」三段，跨厂商迁移必须逐口核对，不能批量替换。
- `negotiation auto` 与手工 `speed`/`duplex` 互斥：先固定速率双工又留 `negotiation auto`，或两端一端自协商一端固定，都会出现速率不匹配/半双工，表现为丢包与 CRC 增长。
- 改接口 IP 后原地址立即失效，远程管理若走该地址会直接断连；变更前确认还有 Console 或另一管理地址可用。

## 高危操作 ⚠️

- ⚠️ `shutdown` 会关闭端口，链路上业务全部中断；对聚合成员口或上联口误操作会引发整段断网，执行前确认端口用途与冗余路径。
- ⚠️ 修改在线端口的 `speed`/`duplex` 会触发链路重协商，短时闪断甚至协商失败；必须在维护窗口成对修改两端。
- ⚠️ 修改或删除承载管理流量的 VLAN 接口 IP（`interface vlan <id>` / `no ip address`）会直接失联，改前保留带外通道。
- ⚠️ 变更前先 `show running-config` 导出留存，变更后用 `show ip interface brief` 与 `show interface` 复核，异常时按导出的配置回退。

## 参考来源

1. [ZXR10 路由交换机基本操作和配置 V2.1 课件（端口类型命名规则、`negotiation auto`、`duplex`、`speed`、`show interface` 回显）](https://www.jinchutou.com/shtml/a85d8cee99dc0393c28ace1e9cf7125b.html)
2. [中兴专利 CN101262503B（`interface vlan 10` + `ip address 10.10.1.1 255.255.255.0` 实测配置例）](https://patents.google.com/patent/CN101262503B/en)（中兴官方申请文件）
3. [ZXR10 1800 路由器配置流程（`porttype l3` + `ip address` 实测链）](http://szdingwei.net/html/6432195423.html)
4. [中兴设备 show 命令大全（`show ip interface brief` 等）（阿里云开发者社区）](https://developer.aliyun.com/article/1171627)

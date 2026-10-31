---
vendor: 中兴（ZTE）
os: ZXR10 ROSNG / ZROS
topic: 静态路由
devices: ZXR10 5900/5950/6900/8900 系列交换机、ZXR10 1800/2800 系列路由器等
keywords: [中兴, ZXR10, 静态路由, ip route, 默认路由, 0.0.0.0, 下一跳, 浮动静态, 路由表, show ip route]
sources:
  - http://szdingwei.net/html/6432195423.html
  - https://patents.google.com/patent/CN101262503B/en
  - https://developer.aliyun.com/article/1171627
  - https://xxc.lyu.edu.cn/_upload/article/files/f2/64/a9cd33264ad69c07614429415226/bf6d5ea3-51dd-4765-8c09-86098124dfdc.pdf
verified: medium
updated: 2026-10-30
---

# 中兴 · 静态路由

> ZXR10 静态路由用 `ip route <目的网段> <点分掩码> <下一跳>`，语法与 Cisco 高度接近——但**掩码是点分正掩码**，
> 且部分平台的路由表被拆成「协议路由表」与「转发表」两张，`show` 命令名与 Cisco 不同。

## 命令速查

| 功能 | 命令 | 视图 | 说明 |
|---|---|---|---|
| 配置静态路由 | `ip route 192.168.30.0 255.255.255.0 10.1.1.254` | 全局配置 `(config)#` | 目的网段 + 点分掩码 + 下一跳；掩码不能写 `/24` |
| 配置默认路由 | `ip route 0.0.0.0 0.0.0.0 10.1.1.254` | 全局配置 | `0.0.0.0 0.0.0.0` 即缺省路由，出方向靠它兜底 |
| 指向出接口 | `ip route 192.168.30.0 255.255.255.0 gei_1/24` | 全局配置 | 下一跳也可写成出接口（点对点链路常用），参数支持度以官方手册确认 |
| 浮动静态/优先级 | `ip route <目的> <掩码> <下一跳> <优先级>` | 全局配置 | 是否支持附加优先级参数、取值范围随版本（以官方手册/售后确认） |
| 删除静态路由 | `no ip route 192.168.30.0 255.255.255.0 10.1.1.254` | 全局配置 | 删除时必须与配置参数完全一致，否则提示找不到该路由 |
| 查看路由表 | `show ip route` | 特权/配置视图 | 交换机平台常用；具体输出字段随版本 |
| 查看协议路由表 | `show ip protocol routing` | 特权/配置视图 | 部分平台（如 M6000）用它看协议计算出的路由 |
| 查看转发表 | `show ip forwarding route` | 特权/配置视图 | 同上，看真正下发硬件转发的表项 |
| 按 VRF 查路由 | `show ip protocol routing vrf <名>` / `show ip forwarding route vrf <名>` | 特权/配置视图 | 多 VRF 场景必须带 `vrf`，否则看的是公网表 |
| 查三层接口与地址 | `show ip interface brief` | 特权/配置视图 | 静态路由不通时先确认下一跳所在接口 up 且有地址 |
| 连通性验证 | `ping <目的IP>` / `ping vrf <名> <目的IP>` | 特权视图 | 逐跳验证，避免只看 `show` 就下结论 |
| 路径追踪 | `trace <目的IP>` | 特权视图 | ZXR10 用 `trace`（不是 Cisco 的 `traceroute`） |

## 典型示例

```text
ZXR10>enable
ZXR10#configure terminal

ZXR10(config)#interface vlan 10
ZXR10(config-if)#ip address 10.1.1.1 255.255.255.0     // 下一跳所在网段的网关地址
ZXR10(config-if)#exit

ZXR10(config)#ip route 192.168.30.0 255.255.255.0 10.1.1.254   // 明细静态
ZXR10(config)#ip route 0.0.0.0 0.0.0.0 10.1.1.254              // 默认路由
ZXR10(config)#end

ZXR10#show ip route                                     // 确认 S 开头表项与下一跳
ZXR10#show ip interface brief                           // 确认出口接口 up
ZXR10#ping 192.168.30.1
ZXR10#trace 192.168.30.1
ZXR10#write
```

```text
// 双向可达性检查：对端必须有一条回程路由，否则现象是"单向通"
ZXR10(config)#ip route 10.20.1.0 255.255.255.0 10.10.1.2
ZXR10#show ip route 10.20.1.0
```

## 陷阱与版本差异

- 本机型/版本命令以官方手册为准：浮动静态的优先级参数、黑洞路由（指向 null 接口）写法、`show ip route` 的输出格式在各型号/版本上不完全一致，未给回显的请先 `?` 核对或找售后确认。
- 静态路由默认优先级（管理距离）**不要套用 Cisco 的 1 或华为的 60**：ZXR10 各平台数值体系不同，需要做浮动静态或与 OSPF 选路比较前，必须查该版本手册确认默认值与可配范围。
- 掩码是点分正掩码（`255.255.255.0`），与 Cisco 一致，但**与 OSPF `network` 用的反掩码相反**；同一份配置里两种掩码并存，是最常见的抄错点。
- 部分平台把路由拆成「协议路由表（`show ip protocol routing`）」与「转发表（`show ip forwarding route`）」；只查其中一张会出现"路由表里有、业务却不转发"的错觉。
- 下一跳必须本地直连可达（同网段且出接口 up），否则路由不进表；这是"配了但 `show` 看不到"的头号原因。
- 单向通 ≠ 本端配错：先在对端 `show ip route` 查回程路由，再查中间设备 ACL/NAT。
- 删除命令必须与配置逐字对应（含掩码与下一跳），用 `no ip route ...` 抄错一个参数就删不掉，容易误以为"删除无效"。

## 高危操作 ⚠️

- ⚠️ 默认路由指向错误下一跳会把全部未知目的地流量引入黑洞，业务表现为"内网能通、出网全断"；下发后必须立刻 `ping` 外网地址 + `show ip route` 双向验证。
- ⚠️ 远程改路由时若新路由覆盖了管理网段的可达路径，会立刻失联；改前确认存在不依赖本次变更的第二条管理路径（Console/带外）。
- ⚠️ 删除默认路由前确认已存在等价备份路由，否则删除瞬间全网中断。
- ⚠️ 变更前先 `show running-config` 导出并 `copy` 到 FTP（`copy root /cfg/startrun.dat ftp://...`），出问题按导出配置回退。

## 参考来源

1. [ZXR10 1800 路由器配置流程（`ip route 0.0.0.0 0.0.0.0 <网关>` 实测链）](http://szdingwei.net/html/6432195423.html)
2. [中兴专利 CN101262503B（`ip route 10.20.1.0 255.255.255.0 10.10.1.2` 实测配置例）](https://patents.google.com/patent/CN101262503B/en)（中兴官方申请文件）
3. [中兴设备 show 命令大全（`show ip protocol routing` / `show ip forwarding route` / `ping vrf`）](https://developer.aliyun.com/article/1171627)
4. [中兴 ZXR10 配置手册 · 静态路由章节（PDF，含静态路由命令表）](https://xxc.lyu.edu.cn/_upload/article/files/f2/64/a9cd33264ad69c07614429415226/bf6d5ea3-51dd-4765-8c09-86098124dfdc.pdf)

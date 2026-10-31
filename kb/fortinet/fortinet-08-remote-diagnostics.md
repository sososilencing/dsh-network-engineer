---
vendor: 飞塔（Fortinet）
os: FortiOS
topic: 远程管理与诊断
devices: FortiGate 40F/60F/100F/600E/1800F 等
keywords: [SSH, HTTPS 管理, set allowaccess, trusthost, get system status, diagnose sys top, diagnose sniffer packet, diagnose debug flow, diagnose debug reset, 会话表, execute ping/traceroute, execute tac report]
sources:
  - https://docs.fortinet.com/document/fortigate/7.6.1/cli-reference/303299696/diagnose-sniffer
  - https://docs.fortinet.com/document/fortigate/7.6.8/cli-reference/317104469/config-system-interface
  - https://support.fortinet.com.cn/troubleshooting/diagnostics_cli
  - https://support.fortinet.com.cn/troubleshooting/sniffer_tool/sniffer_intro
  - https://support.fortinet.com.cn/troubleshooting/debug_flow_tool/debug_flow_intro
verified: high
updated: 2026-10-30
---

# 飞塔（Fortinet）· 远程管理与诊断

> 远程管理靠接口的 `set allowaccess`（协议白名单）+ 管理员 `trusthost`（来源限制），两者都不显式配置就"命令对但连不上"。诊断链路是固定套路：`get`（看状态）→ `diagnose sniffer packet`（看包到没到）→ `diagnose debug flow`（看策略/路由怎么处理的）→ `diagnose sys session`（看会话）。**收工务必 `diagnose debug reset`。**

## 命令速查

| 功能 | 命令 | 视图 | 说明 |
|---|---|---|---|
| 开管理协议 | `set allowaccess ping https ssh` | `(port1) #` | 覆盖式；不开启则 GUI/SSH 不可达 |
| 限管理员来源 | `config system admin` → `edit "netops"` → `set trusthost1 10.10.10.0 255.255.255.0` | `(netops) #` | 与 Local-in 策略互补的第二道门 |
| 系统概况 | `get system status` | 操作模式 | 型号/固件版本/运行模式（NAT 或透明）/当前 VDOM |
| 性能概况 | `get system performance status` | 操作模式 | CPU、内存、会话数、运行时长 |
| 进程排序 | `diagnose sys top 2 30` | 操作模式 | 每 2 秒刷新前 30 个进程；输出界面 `Shift+C` 按 CPU、`Shift+M` 按内存排序 |
| 会话过滤 | `diagnose sys session filter dst 8.8.8.8` | 操作模式 | 先设过滤条件，再看/清会话 |
| 会话查看与清除 | `diagnose sys session list` / `diagnose sys session clear` | 操作模式 | `clear` 会断业务，慎用 |
| 抓包 | `diagnose sniffer packet any "host 172.16.1.100 and icmp" 4 0 a` | 操作模式 | 参数：接口 / 过滤 / 详细级 1-6 / 条数(0=不限) / 时间戳 `a`=UTC、`l`=本地 |
| Debug Flow 过滤 | `diagnose debug flow filter addr 114.114.114.114` + `diagnose debug flow filter proto 1` | 操作模式 | 支持 addr/saddr/daddr/port/sport/dport/proto/vd-name |
| Debug Flow 跟踪 | `diagnose debug flow trace start 10` | 操作模式 | 打印 10 条处理过程；IPv6 用 `trace start6` |
| 打开/关闭 debug | `diagnose debug enable` / `diagnose debug disable` / `diagnose debug reset` | 操作模式 | `disable` 只停控制台输出，`reset` 才真正停掉后台 debug |
| 设置 debug 时长 | `diagnose debug duration 0` | 操作模式 | 0=不限；默认 30 分钟 |
| 显示命中策略 | `diagnose debug flow show iprope enable` | 操作模式 | 输出中带上命中的 policy ID 与 drop 原因 |
| 路由表 | `get router info routing-table all` | 操作模式 | 排查"有没有路由"的第一条命令 |
| 连通性测试 | `execute ping-options source 192.168.1.99` + `execute ping 8.8.8.8` / `execute traceroute 8.8.8.8` | 操作模式 | 不指定源地址会用默认出口地址，回程可能不通 |
| 接口硬件 | `diagnose hardware deviceinfo nic port1` / `get sys interface transceiver` | 操作模式 | 光模块、速率、错包计数 |
| 二层转发表 | `diagnose netlink brctl name host <switch名>` | 操作模式 | 受管交换接口的 MAC 表 |
| HA 状态 | `get system ha status` / `diagnose sys ha status` | 操作模式 | 主备角色、同步状态 |
| 一键收集 | `execute tac report` | 操作模式 | 开 TAC 工单用；耗时且占资源 |
| 命令树 | `tree diagnose` / `tree execute` | 操作模式 | 不确定子命令时先看树 |

## 典型示例

```text
// 场景：内网 PC 192.168.1.10 上不了网，按"包 → 路由 → 策略 → 会话"顺序定位
FortiGate # get system status                       // 先确认版本/运行模式/VDOM
FortiGate # get router info routing-table all       // 有没有默认路由
FortiGate # diagnose sniffer packet any "host 114.114.114.114 and icmp" 4 0 l
// interfaces=[any] filters=[host 114.114.114.114 and icmp]
// 7.07 port5 in  192.168.1.10 -> 114.114.114.114: icmp: echo request
// 7.07 port1 out 192.168.89.35 -> 114.114.114.114: icmp: echo request   ← 有出说明 NAT/策略已过
FortiGate # diagnose debug flow filter addr 114.114.114.114
FortiGate # diagnose debug flow filter proto 1
FortiGate # diagnose debug flow show function-name enable
FortiGate # diagnose debug console timestamp enable
FortiGate # diagnose debug flow trace start 10
FortiGate # diagnose debug enable
// 关注这几行：find a route / Allowed by Policy-9 / SNAT ... / iprope_in_check() check failed
FortiGate # diagnose sys session filter dst 114.114.114.114
FortiGate # diagnose sys session list                // 看 no_ofld_reason、state、policy_id
FortiGate # diagnose debug flow trace stop
FortiGate # diagnose debug disable
FortiGate # diagnose debug reset                     // 必须执行，彻底关闭后台 debug

FortiGate # execute ping-options source 192.168.1.99
FortiGate # execute ping 114.114.114.114
FortiGate # execute traceroute 114.114.114.114
FortiGate # diagnose sys top 2 30                    // CPU 高时看是哪个进程
FortiGate # execute tac report                       // 需要开 case 时一键收集
```

## 陷阱与版本差异

- `diagnose debug disable` 只是停止在控制台打印，**后台 debug 可能仍在运行**；排查结束必须 `diagnose debug reset` 完全停止并清理。官方特别强调了这一点。
- debug 默认只跑 30 分钟，`diagnose debug duration 0` 才是无限时长；忘记关会把 CPU 和控制台刷爆。
- `diagnose sniffer packet` 的详细级别：1 只报文头、2 含 IP 以上数据、3 含以太网数据、4 报文头 + 进出接口（最常用）、5/6 在此基础上带数据。抓转发问题请至少用 4。
- sniffer 过滤语法接近 tcpdump（`host`/`net`/`port`/`tcp[13]&2!=0`），但抓 VLAN 帧要写 `vlan 10` 或用 `ether[14:2]` 偏移，普通 `host` 过滤看不到 Tag 信息。
- 已经被 NP/SPU 硬件加速卸载的会话，sniffer 与 debug flow 可能抓不到后续报文（官方专门有"Sniffer/Debug Flow/NP 加速的矛盾"一节），此时改看会话的 `no_ofld_reason` 或用 GUI 抓包。
- `get`/`diagnose`/`execute` 都是操作模式命令，前面不能加 `config`；`config` 只用于进入配置节点。
- 不同 FortiOS 版本/型号/VDOM 下可用命令有差异，拿不准时在命令后按 `?` 或 `tree diagnose` 查本机支持项（以官方手册为准）。

## 高危操作 ⚠️

- ⚠️ 长时间开高详细级别 sniffer 或无限时长 debug 会显著抬高 CPU 并可能丢失业务报文，⚠️ 必须限定条数（`<count>`）与时长，收工后 `diagnose debug reset`。
- ⚠️ `diagnose sys session clear` 立即中断匹配过滤条件的在线会话，用户侧表现为"突然掉线"，⚠️ 先 `diagnose sys session list` 核对范围。
- ⚠️ `execute reboot`、`execute shutdown` 中断全部业务；`execute tac report` 等收集命令在低端型号上会长时间占 CPU，⚠️ 生产环境避开高峰。
- ⚠️ 修改 `allowaccess`、管理端口或 `trusthost1` 可能立刻让当前远程会话失效，⚠️ 改前确认 Console 可用，并保留一条未受影响的入口。

## 参考来源

1. [diagnose sniffer（docs.fortinet.com，FortiOS 7.6.1 CLI Reference）](https://docs.fortinet.com/document/fortigate/7.6.1/cli-reference/303299696/diagnose-sniffer)
2. [config system interface（docs.fortinet.com，FortiOS 7.6.8 CLI Reference，allowaccess 参数）](https://docs.fortinet.com/document/fortigate/7.6.8/cli-reference/317104469/config-system-interface)
3. [常用排错命令大全：debug 开关、会话、抓包、路由、HA（FortiGate 中文一本通）](https://support.fortinet.com.cn/troubleshooting/diagnostics_cli)
4. [sniffer 工具介绍：参数含义与常用过滤条件（FortiGate 中文一本通）](https://support.fortinet.com.cn/troubleshooting/sniffer_tool/sniffer_intro)
5. [Debug Flow 介绍：filter/trace start/enable 与回显解读（FortiGate 中文一本通）](https://support.fortinet.com.cn/troubleshooting/debug_flow_tool/debug_flow_intro)

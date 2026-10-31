---
vendor: 飞塔（Fortinet）
os: FortiOS
topic: OSPF
devices: FortiGate 40F/60F/100F/600E/1800F 等（NAT 模式路由转发，需开启高级路由可见性）
keywords: [OSPF, config router ospf, router-id, config area 0.0.0.0, config network set prefix, default-information-originate, redistribute connected, passive-interface, graceful-restart, get router info ospf neighbor]
sources:
  - https://docs.fortinet.com/document/fortigate/7.6.8/cli-reference/103419153/config-router-ospf
  - https://docs.fortinet.com/document/fortigate/7.6.7/cli-reference/103419153
  - https://support.fortinet.com.cn/routing/dynamic_routing/ospf/ospf_exemple
  - https://support.fortinet.com.cn/troubleshooting/diagnostics_cli
verified: high
updated: 2026-10-30
---

# 飞塔（Fortinet）· OSPF

> FortiOS 的 OSPF 全部在 `config router ospf` 一个节点下：先 `set router-id`，再 `config area`（用点分格式 `edit 0.0.0.0`）和 `config network`（`set prefix` + **正掩码**），最后按需 `config redistribute` 与缺省路由注入。要像交换机一样看邻居，用 `get router info ospf ...`。

## 命令速查

| 功能 | 命令 | 视图 | 说明 |
|---|---|---|---|
| 进入 OSPF | `config router ospf` | 操作模式 | 需先在 GUI「系统 → 可见功能」开启高级路由 |
| 设 Router ID | `set router-id 1.1.1.1` | `(ospf) #` | 全网唯一；建议用 Loopback 地址，改动会重建邻居 |
| 建区域 | `config area` → `edit 0.0.0.0` | `(area) #` | 主干区写成 `0.0.0.0`（点分），不是数字 `0` |
| 宣告网段 | `config network` → `edit 1` → `set prefix 1.1.1.0 255.255.255.0` | `(network) #` | **正掩码**；与华为 `network x.x.x.x 0.0.0.255 area 0` 的反掩码写法相反 |
| 注入缺省路由 | `set default-information-originate enable` | `(ospf) #` | 对应 GUI 的 regular/always 语义，默认不注入 |
| 重发布直连 | `config redistribute "connected"` → `set status enable` | `(connected) #` | 把直连网段作为外部路由通告出去 |
| 重发布静态 | `config redistribute "static"` → `set status enable` | `(static) #` | 通配入口，`"rip"`/`"bgp"` 等同理 |
| 被动接口 | `config passive-interface` → `edit "port5"` | `(passive-interface) #` | 只宣告不建邻居；老版本写法为 `set passive-interface "port5"`，以 `tree router ospf` 回显为准 |
| 接口参数 | `config ospf-interface` → `edit "port3-ospf"` → `set interface "port3"` / `set cost 10` / `set hello-interval 10` / `set dead-interval 40` | `(ospf-interface) #` | 认证、cost、计时器都在这里；两端 hello/dead 必须一致 |
| 平滑重启 | `set restart-mode graceful-restart` + `set restart-period 120` | `(ospf) #` | HA 主备 + 动态路由场景几乎必配 |
| 查看邻居 | `get router info ospf neighbor all` | 操作模式 | 看邻居状态 Full / 2-Way / ExStart |
| 查看协议状态 | `get router info ospf status` | 操作模式 | Router ID、区域数、SPF 计数 |
| 查看接口 | `get router info ospf interface` | 操作模式 | 每个接口的 area、cost、邻居数、hello/dead |
| 查看 LSDB | `get router info ospf database brief` | 操作模式 | 数据库摘要 |
| 路由表验证 | `get router info routing-table all` | 操作模式 | OSPF 路由回显 `[110/xx]`（AD=110） |
| HA 路由保持 | `config system ha` → `set route-ttl 120` | `(ha) #` | 覆盖邻居重建时间，避免切换后立刻丢路由 |

## 典型示例

```text
// 场景：port3 = 1.1.1.1/24 对接三层交换机，内网 192.168.1.0/24，注入默认路由并重发布直连
FortiGate # config router ospf
FortiGate (ospf) # set router-id 1.1.1.1
FortiGate (ospf) # set default-information-originate enable   // 向邻居注入 0.0.0.0/0
FortiGate (ospf) # config area
FortiGate (area) # edit 0.0.0.0                                // 主干区域，点分格式
FortiGate (0.0.0.0) # next
FortiGate (area) # end
FortiGate (ospf) # config network
FortiGate (network) # edit 1
FortiGate (1) # set prefix 1.1.1.0 255.255.255.0               // 正掩码
FortiGate (1) # next
FortiGate (network) # edit 2
FortiGate (2) # set prefix 192.168.1.0 255.255.255.0
FortiGate (2) # next
FortiGate (network) # end
FortiGate (ospf) # config redistribute "connected"
FortiGate (connected) # set status enable                       // 重发布直连
FortiGate (connected) # end
FortiGate (ospf) # config passive-interface                     // 面向终端的口只宣告不建邻居
FortiGate (passive-interface) # edit "port5"
FortiGate (port5) # next
FortiGate (passive-interface) # end
FortiGate (ospf) # set restart-mode graceful-restart
FortiGate (ospf) # set restart-period 120
FortiGate (ospf) # end

FortiGate # get router info ospf neighbor all            // 邻居是否 Full
FortiGate # get router info ospf interface               // 接口是否在正确 area、hello/dead 是否一致
FortiGate # get router info routing-table all            // 应出现 O 192.168.2.0/24 [110/11] ...
FortiGate # diagnose ip router ospf all enable           // 排障：开 OSPF 实时 debug
FortiGate # diagnose debug enable
FortiGate # diagnose debug disable
FortiGate # diagnose debug reset                         // 彻底停 debug
```

## 陷阱与版本差异

- **掩码方向**：FortiOS 的 `set prefix` 用正掩码（`255.255.255.0`），华为 `network` 用反掩码，锐捷/Cisco 用 `0.0.0.255` 通配符；照抄别家命令是最高频的配置失败原因。
- 区域写成 `edit 0.0.0.0` 而非 `edit 0`；非主干区同理用点分（如 `edit 0.0.0.1`）。
- **OSPF AD 110 大于静态 10**：只要存在一条覆盖同网段的静态路由，OSPF 学到的路由就进不了转发表（`get router info routing-table database` 能看到但 `all` 里看不到），这与华为/锐捷的优先级直觉相反。
- 接口必须 up、IP 落在 `config network` 覆盖范围内才会建邻居；`passive-interface` 的口照旧宣告网段但不发 hello。
- 两端 `hello-interval`/`dead-interval`（默认 10/40 秒）与认证参数必须一致，否则邻居卡在 Init/ExStart。
- HA 主备 + 动态路由：默认切换后只保留原主机同步来的 FIB 10 秒，必须调 `set route-ttl` 并开启 `graceful-restart`，否则切换瞬间业务中断。
- `config passive-interface` 的子表写法在 6.x/7.x 与老版本（`set passive-interface <接口名>`）不同，落笔前用 `tree router ospf` 确认本机语法（以官方手册为准）。

## 高危操作 ⚠️

- ⚠️ 同时开 `default-information-originate` 与双向重发布（尤其把 OSPF 再重发布进静态/BGP 侧）极易形成路由环路或把内网明细路由泄露到外网，⚠️ 变更前画出重发布方向图并做好路由过滤。
- ⚠️ 修改 `set router-id` 或区域归属会触发邻居重建，全网短暂丢路由；⚠️ 生产环境应在维护窗口并配合 GR。
- ⚠️ `execute router restart`（重启路由进程）会清空所有动态邻居与动态路由，⚠️ 等于瞬断全网路由，非排障必需不要执行。
- ⚠️ HA 环境未配 `route-ttl`/GR 就升降级或重启，主备切换时会断流数十秒，⚠️ 切换演练前先确认这两个参数。

## 参考来源

1. [config router ospf（docs.fortinet.com，FortiOS 7.6.8 CLI Reference）](https://docs.fortinet.com/document/fortigate/7.6.8/cli-reference/103419153/config-router-ospf)
2. [config router ospf（docs.fortinet.com，FortiOS 7.6.7 CLI Reference）](https://docs.fortinet.com/document/fortigate/7.6.7/cli-reference/103419153)
3. [OSPF 基础配置：area/network/redistribute/route-ttl/GR（FortiGate 中文一本通）](https://support.fortinet.com.cn/routing/dynamic_routing/ospf/ospf_exemple)
4. [常用排错命令：OSPF 邻居、接口、数据库与 debug（FortiGate 中文一本通）](https://support.fortinet.com.cn/troubleshooting/diagnostics_cli)

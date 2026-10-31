---
vendor: 飞塔（Fortinet）
os: FortiOS
topic: 接口配置（IP、速率、管理访问）
devices: FortiGate 40F/60F/70F/100F/600E/1800F 等物理口与 VLAN/聚合/冗余虚拟口
keywords: [config system interface, set ip, set mode static/dhcp/pppoe, set allowaccess, append allowaccess, secondary IP, set speed, mediatype, forward-error-correction, set mtu, 接口管理访问]
sources:
  - https://docs.fortinet.com/document/fortigate/7.6.8/cli-reference/317104469/config-system-interface
  - https://docs.fortinet.com/document/fortigate/7.2.14/cli-reference/101854390/config-system-virtual-switch
  - https://support.fortinet.com.cn/network_mgmt/interfaces/interface_settings/interface_settings
  - https://support.fortinet.com.cn/troubleshooting/diagnostics_cli
verified: high
updated: 2026-10-30
---

# 飞塔（Fortinet）· 接口配置

> 接口全部在 `config system interface` 下，用 `edit port1` 进对象；**IP 用点分掩码，且管理访问（`set allowaccess`）必须显式开启，否则 ping/HTTPS/SSH 一律不通**。物理口名字不能改，只能设别名；速率/介质/FEC 是联动参数。

## 命令速查

| 功能 | 命令 | 视图 | 说明 |
|---|---|---|---|
| 进入接口对象 | `config system interface` → `edit port1` | `(interface) #` | 物理口、VLAN 子接口、聚合口、冗余口都在同一节点下 |
| 设地址模式 | `set mode static` | `(port1) #` | `static`/`dhcp`/`pppoe`；只有 static 模式下 `set ip` 才有效 |
| 配置 IP | `set ip 10.1.1.1 255.255.255.0` | `(port1) #` | 点分掩码；同一设备不同接口不能配同网段地址 |
| 管理访问 | `set allowaccess ping https ssh` | `(port1) #` | **覆盖式**赋值；不显式开启则管理面不通（各版本默认值不同，见陷阱） |
| 追加管理协议 | `append allowaccess ssh` | `(port1) #` | 保留原有协议，只新增 SSH |
| 移除单个协议 | `unselect allowaccess http` | `(port1) #` | 只删 HTTP，其余保留 |
| 辅助 IP | `set secondary-IP enable` + `config secondaryip` → `edit 1` → `set ip 192.0.2.2 255.255.255.0` | `(port1) #` / `(secondaryip) #` | 每个附加地址可单独设 allowaccess |
| 别名与描述 | `set alias WAN-ISP1` / `set description "电信100M"` | `(port1) #` | 别名 ≤25 字符且不进日志；description 更长 |
| 速率双工 | `set speed 1000full` | `(port1) #` | 可选 `10000full`/`100Gauto`/`25000full` 等，随型号支持度不同 |
| 光模块介质 | `set mediatype sr` / `lr` / `cr`（多通道为 `sr4`/`lr4`/`cr4`） | `(port1) #` | 与模块类型不匹配会导致链路不 up |
| FEC | `set forward-error-correction {disable \| cl91-rs-fec \| cl74-fc-fec}` | `(port1) #` | 1G/10G/40G 不支持；25G/100G 默认 `cl91-rs-fec` |
| MTU | `set mtu 1500` | `(port1) #` | 改小可绕开部分分片问题；TCP-MSS 另有参数，字段随版本 |
| 启停接口 | `set status down` | `(port1) #` | 等于物理关停，接口立刻不再收发 |
| 查看接口全量配置 | `show full-configuration system interface port1` | 操作模式 | 核对 allowaccess/ip/mode 的实际值 |
| 查看接口摘要 | `get system interface` / `get sys interface physical` | 操作模式 | 含 IP、链路状态 |
| 查光模块与硬件 | `get sys interface transceiver` / `get hardware nic port1` | 操作模式 | 模块型号、收发光、速率能力 |
| 查全部 IP | `diagnose ip address list` | 操作模式 | 地址冲突/漏配时先用这条 |

## 典型示例

```text
// 场景：port1 接运营商静态地址，内网管理网段为 10.10.10.0/24
FortiGate # config system interface
FortiGate (interface) # edit "port1"
FortiGate (port1) # set vdom "root"
FortiGate (port1) # set mode static                       // 静态地址
FortiGate (port1) # set ip 202.103.100.5 255.255.255.248 // 点分掩码
FortiGate (port1) # set allowaccess ping                 // WAN 口只留 ping，最小化暴露
FortiGate (port1) # set role wan
FortiGate (port1) # set alias WAN-ISP1                    // 别名便于识别
FortiGate (port1) # set status up
FortiGate (port1) # next
FortiGate (interface) # edit "port3"
FortiGate (port3) # set ip 192.168.1.99 255.255.255.0
FortiGate (port3) # set allowaccess ping https ssh        // 内网口三个协议一次配全
FortiGate (port3) # set role lan
FortiGate (port3) # set secondary-IP enable               // 辅助地址（可选）
FortiGate (port3) # config secondaryip
FortiGate (secondaryip) # edit 1
FortiGate (1) # set ip 192.168.2.99 255.255.255.0
FortiGate (1) # set allowaccess ping
FortiGate (1) # next
FortiGate (secondaryip) # end
FortiGate (port3) # next
FortiGate (interface) # end

FortiGate # show full-configuration system interface port1  // 核对 allowaccess 是否真的写进去
FortiGate # get system interface                            // 链路 up/down、IP 摘要
FortiGate # get sys interface transceiver                   // 光口先看模块与收发光
FortiGate # execute ping-options source 192.168.1.99
FortiGate # execute ping 202.103.100.1                      // 指定源地址验证连通性
```

## 陷阱与版本差异

- `set allowaccess` 是**覆盖式**的：只写 `set allowaccess ping` 会把原有的 https/ssh 全部删掉——远程管理当场失联。追加用 `append allowaccess ssh`，删除用 `unselect allowaccess http`。
- 接口默认 `allowaccess` 因版本、型号、角色（lan/wan/dmz）而不同（WAN 常见仅 `ping`），**不要凭记忆假设**，一律用 `show full-configuration system interface <名>` 核对。
- 同一 FortiGate 上多个接口不能配置同网段 IP（与交换机 SVI 的直觉不同），否则 CLI 报冲突；需要同网段多地址时用 `secondary-IP`。
- `set mode dhcp`/`pppoe` 下 `set ip` 无效，配置看起来存在但不会生效；切换模式后要重新确认地址来源。
- 改 `set speed` 时可能联动修改 mediatype 与 FEC，CLI 会提示 `The speed/mediatype/FEC of port25 will be changed ... Do you want to continue? (y/n)`，需确认后再回车。
- VLAN/聚合/冗余等虚拟接口没有 `speed`/`mediatype`/`forward-error-correction`（速率跟随成员口），照抄物理口配置会报错。
- 光模块必须与 `mediatype` 匹配（SR/LR/CR，四通道为 sr4/lr4/cr4），否则链路起不来——现象是"接口 up 但学不到对端"或彻底 down。

## 高危操作 ⚠️

- ⚠️ 修改当前管理入口接口的 IP、`allowaccess`、`status` 会立刻断开本次 GUI/SSH 会话；⚠️ 先在 Console 或另一接口准备好回退通道，并记下原值。
- ⚠️ 改 `set speed` / `set mediatype` / `set forward-error-correction` 会闪断链路，跑在线业务的 40G/100G 口尤其危险，需维护窗口。
- ⚠️ `set status down` 等同于物理关停接口，若该口是上联或心跳口，会直接断网/触发 HA 切换。
- ⚠️ 接口改名不可行（物理口名字固定），误删虚拟接口（`delete vlan10`）会连带影响引用它的策略与路由，⚠️ 删除前确认引用关系。

## 参考来源

1. [config system interface（docs.fortinet.com，FortiOS 7.6.8 CLI Reference）](https://docs.fortinet.com/document/fortigate/7.6.8/cli-reference/317104469/config-system-interface)
2. [config system virtual-switch（docs.fortinet.com，FortiOS 7.2.14 CLI Reference）](https://docs.fortinet.com/document/fortigate/7.2.14/cli-reference/101854390/config-system-virtual-switch)
3. [接口选项：allowaccess、辅助 IP、速率/介质/FEC（FortiGate 中文一本通）](https://support.fortinet.com.cn/network_mgmt/interfaces/interface_settings/interface_settings)
4. [常用排错命令：接口、光模块、ARP、IP 地址查询（FortiGate 中文一本通）](https://support.fortinet.com.cn/troubleshooting/diagnostics_cli)

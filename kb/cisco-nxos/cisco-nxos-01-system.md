---
vendor: 思科（Cisco）
os: NX-OS
topic: 系统管理与视图
devices: Nexus 3000/5000/6000/7000/9000 系列（含 9300-E/FX、9500），Nexus 9000v
keywords: [NX-OS, feature 开启特性, configure terminal, copy running-config startup-config, checkpoint, rollback running-config, ntp server, write erase, reload, show running-config]
sources:
  - https://www.cisco.com/c/en/us/td/docs/dcn/nx-os/nexus9000/103x/configuration/fundamentals/cisco-nexus-9000-nx-os-fundamentals-configuration-guide-103x.html
  - https://www.cisco.com/c/en/us/td/docs/dcn/nx-os/nexus9000/103x/configuration/system-management/cisco-nexus-9000-series-nx-os-system-management-configuration-guide-103x/m-configuring-ntp-10x.html
  - https://www.cisco.com/c/en/us/td/docs/dcn/nx-os/nexus9000/103x/configuration/security/cisco-nexus-9000-nx-os-security-configuration-guide-103x.html
verified: high
updated: 2026-10-30
---

# 思科 Cisco NX-OS · 系统管理与视图

> 适用于 Nexus 设备的视图切换、特性开关（`feature`）、配置落盘、检查点回滚与 NTP。
> 与 IOS 两个硬差异：**一切特性必须先 `feature xxx` 开启**；**改完不 `copy running-config startup-config` 就不落盘**。

## 命令速查

| 功能 | 命令 | 视图 | 说明 |
|---|---|---|---|
| 进全局配置 | `configure terminal` | `switch#` | 简写 `conf t`；提示符由 `switch#` 变 `switch(config)#` |
| 开特性（总开关） | `feature ospf` | `switch(config)#` | **前置必做**；同类还有 `feature lacp`/`interface-vlan`/`ssh`/`lldp`/`vpc`/`ntp` |
| 关特性 ⚠️ | `no feature ospf` | `switch(config)#` | **连带删除该特性下全部配置**，不是"暂停" |
| 改主机名 | `hostname N9K-SPINE-01` | `switch(config)#` | 立即生效，提示符同步变化 |
| 查看特性状态 | `show feature` | `switch#` | 看各特性 enabled/disabled，排障第一条命令 |
| 保存配置 | `copy running-config startup-config` | `switch#` 或 `switch(config)#` | 等价的 `copy run start`；**不做就等于没改** |
| 查看当前配置 | `show running-config` | `switch#` | 变更前先导出一份留存 |
| 查看启动配置 | `show startup-config` | `switch#` | 与 running 对比可发现"改了没存" |
| 建检查点 | `checkpoint before-change` | `switch#` | 给当前配置打快照，回滚的基准点 |
| 回滚到检查点 | `rollback running-config checkpoint before-change` | `switch#` | 把运行配置整体拉回快照状态 |
| 查看检查点 | `show checkpoint summary` | `switch#` | 列出已建快照与时间戳 |
| 开 NTP | `feature ntp` | `switch(config)#` | NTP 默认已启用；`no feature ntp` 关闭 |
| 指定 NTP 服务器 | `ntp server 10.10.1.20 use-vrf management` | `switch(config)#` | 带外管理网段必须带 `use-vrf management`，否则不通 |
| NTP 源接口 | `ntp source-interface mgmt0` | `switch(config)#` | 让 NTP 报文用固定源地址，便于对端 ACL |
| 校验 NTP | `show ntp status` / `show ntp peers` | `switch#` | 看 stratum 与是否 synchronized |
| 看版本/序列号 | `show version` / `show inventory` | `switch#` | 变更前后各抓一次，做基线 |
| 指定启动镜像 ⚠️ | `boot nxos bootflash:/nxos.10.3.8.bin` | `switch(config)#` | 钉住下次启动的镜像文件；路径写错/文件损坏会导致**设备起不来**，必须紧接落盘 |
| 清空配置 ⚠️ | `write erase` | `switch#` | 清 startup-config，重启后回到出厂状态 |
| 重启 ⚠️ | `reload` | `switch#` | 交互确认；`reload` 前必须先保存 |

## 典型示例

```text
switch# configure terminal
switch(config)# hostname N9K-LEAF-01
N9K-LEAF-01(config)# feature interface-vlan        // 起 SVI 的前置开关
N9K-LEAF-01(config)# feature lacp                  // 用 LACP 聚合的前置开关
N9K-LEAF-01(config)# feature ospf                  // 跑 OSPF 的前置开关
N9K-LEAF-01(config)# feature lldp                  // LLDP 全局默认关闭，必须显式开
N9K-LEAF-01(config)# feature ntp                   // NTP 默认已开，此处为确认性下发
N9K-LEAF-01(config)# ntp server 10.10.1.20 use-vrf management   // 走带外管理 VRF
N9K-LEAF-01(config)# ntp source-interface mgmt0
N9K-LEAF-01(config)# exit
N9K-LEAF-01# checkpoint before-change              // 变更前打快照
N9K-LEAF-01# show feature
N9K-LEAF-01# show ntp status
N9K-LEAF-01# copy running-config startup-config    // 关键一步：落盘
N9K-LEAF-01# show startup-config | include hostname
```

## 陷阱与版本差异

- `feature xxx` 是 NX-OS 独有的"特性包"机制：不开 `feature ospf` 时 `router ospf 1` 直接报错；同理无 `feature interface-vlan` 就建不了 SVI。IOS 用户最容易漏这一步。
- `no feature ospf` 会**删除该特性下的全部配置**（进程、接口 area、重分发），恢复时 `feature ospf` 只是重新开空进程，配置不会回来。仅想停止协议请用 `router ospf` 下的 `shutdown`。
- 配置修改只写入 running-config，掉电/重启即丢。`copy running-config startup-config` 才写 startup-config（官方文中一律按"Optional"标注，实际是必做动作）。
- LLDP 全局默认 **disabled**（接口级默认 enabled），所以 `show lldp neighbors` 没输出先查 `show feature | include lldp`。
- NTP 默认 enabled，官方默认值表：NTP Enabled / NTP 认证 Disabled / NTP logging Disabled；10.1(1) 起不再与 stratum 14、15 同步。
- 提示符形态与 IOS 不同：NX-OS 是 `switch#` → `switch(config)#` → `switch(config-if)#` → `switch(config-router)#`，没有 `(config-line)` 之外的分层子模式差异，但 `feature` 缺失时视图根本进不去。
- `boot nxos bootflash:<文件>` 只是"指定下次启动镜像"，不是升级动作：镜像必须先 `copy` 到 bootflash 并校验完整，写完立刻 `copy running-config startup-config`；只写 running-config 时重启会退回旧镜像。

## 高危操作 ⚠️

- ⚠️ `write erase` 清空 startup-config：设备重启后完全回到出厂状态（含管理 IP、feature、SSH 主机密钥），远程必失联，必须现场 Console 在旁；执行前先 `copy running-config bootflash:backup-YYYYMMDD.cfg` 备份。
- ⚠️ `reload` 前若未 `copy running-config startup-config`，本次所有变更全部丢失；带 `reload` 的脚本/批量任务要求二次确认。
- ⚠️ `boot nxos bootflash:` 指向不存在/不完整的 `.bin` 会让设备下次启动失败、停在 `loader>`，只能 Console + TFTP/USB 救援；变更前先 `show boot` 确认现有启动项，并保留上一版镜像不删。
- ⚠️ `no feature ssh` 后 22 端口并不关闭，系统会下发 deny 规则拒绝所有外部连接，等于远程管理被掐断——远程操作前先确认 Console 可用。
- ⚠️ `rollback running-config checkpoint <name>` 是整体回退：快照之后新增的配置（哪怕是别的同事加的）一起消失，回滚窗口需冻结变更。

## 参考来源

1. [Cisco Nexus 9000 Series NX-OS Fundamentals Configuration Guide, Release 10.3(x)（cisco.com）](https://www.cisco.com/c/en/us/td/docs/dcn/nx-os/nexus9000/103x/configuration/fundamentals/cisco-nexus-9000-nx-os-fundamentals-configuration-guide-103x.html)
2. [Cisco Nexus 9000 Series NX-OS System Management Configuration Guide, Release 10.3(x) · Configuring NTP（cisco.com）](https://www.cisco.com/c/en/us/td/docs/dcn/nx-os/nexus9000/103x/configuration/system-management/cisco-nexus-9000-series-nx-os-system-management-configuration-guide-103x/m-configuring-ntp-10x.html)
3. [Cisco Nexus 9000 Series NX-OS Security Configuration Guide, Release 10.3(x)（cisco.com）](https://www.cisco.com/c/en/us/td/docs/dcn/nx-os/nexus9000/103x/configuration/security/cisco-nexus-9000-nx-os-security-configuration-guide-103x.html)

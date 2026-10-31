---
vendor: Arista
os: EOS
topic: 系统管理与视图
devices: vEOS、DCS-7050X/7060X/7280R/7500R、CCS-720XP 等 EOS 设备
keywords: [EOS, configure terminal, copy running-config startup-config, 配置不自动保存, hostname, username privilege 15, role network-admin, management ssh, ntp server, show version]
sources:
  - https://www.arista.com/en/um-eos/eos-user-security
  - https://www.arista.com/assets/data/docs/Manuals/EOS-4.18.3.1F-Manual.pdf
  - https://www.arista.com/en/um-eos/eos-acls-and-route-maps
verified: high
updated: 2026-10-30
---

# Arista · 系统管理与视图

> 适用于 EOS 4.x（vEOS、7050X/7060X/7280R/7500R、CCS-720XP）。CLI 与 Cisco IOS 高度相似，但**配置绝不自动保存**：每条变更后必须执行 `copy running-config startup-config`，否则重启全部丢失。

## 命令速查

| 功能 | 命令 | 视图 | 说明 |
|---|---|---|---|
| 进入全局配置 | `configure terminal` | `>` 特权 EXEC | 可简写 `config`，进入 `(config)#` |
| 返回特权 EXEC | `end` | `(config)#` | 直接回到 `>`；`exit` 只退一级 |
| 保存配置 | `copy running-config startup-config` | `>` 特权 EXEC | **必做**，EOS 无自动保存 |
| 保存配置（简写） | `write` | `>` 特权 EXEC | 官方文档等价写法，不带参数即保存 |
| 查看运行配置 | `show running-config` | `>` 特权 EXEC | 变更前导出留存的首选命令 |
| 查看启动配置 | `show startup-config` | `>` 特权 EXEC | 与 run 对比可发现"忘了保存" |
| 设置主机名 | `hostname SW-LEAF-01` | `(config)#` | 立即生效并进入提示符 |
| 创建管理员账号 | `username admin privilege 15 role network-admin secret 0 <密码>` | `(config)#` | `secret 0` 明文、`secret 5`/`sha512` 密文 |
| 删除账号 | `no username admin` | `(config)#` | 删除前确认已存在其他管理员 |
| 设置 enable 密码 | `enable password 0 <密码>` | `(config)#` | 进入特权 EXEC 时校验 |
| 设置 root 密码 | `aaa root secret 0 <密码>` | `(config)#` | 底层 Linux root，务必设置 |
| 启用 SSH 服务 | `management ssh` → `no shutdown` | `(config)#` → `management ssh` 视图 | 后续可配 `ip access-group` 做服务 ACL |
| 启用 eAPI | `management api http-commands` → `no shutdown` | `(config)#` → eAPI 视图 | 参数随版本，以官方手册为准 |
| 配置 NTP | `ntp server 10.0.0.1` | `(config)#` | 可加 `vrf MGMT`、`prefer`、`local-interface` |
| 查看版本/运行时长 | `show version` | `>` 特权 EXEC | 确认 EOS 版本与硬件型号 |
| 查看在线用户 | `show users` | `>` 特权 EXEC | 变更前确认会话数 |

## 典型示例

```text
switch> enable                              // 进入特权 EXEC（无 enable 密码时不提示）
switch# configure terminal                  // 进入全局配置视图 (config)#
switch(config)# hostname SW-LEAF-01
SW-LEAF-01(config)# username admin privilege 15 role network-admin secret 0 Arista@2026
SW-LEAF-01(config)# enable password 0 En@ble2026
SW-LEAF-01(config)# aaa root secret 0 R00t@2026
SW-LEAF-01(config)# management ssh
SW-LEAF-01(config-mgmt-ssh)# no shutdown     // 确保 SSH 服务处于启用状态
SW-LEAF-01(config-mgmt-ssh)# exit
SW-LEAF-01(config)# ntp server 10.0.0.1 vrf MGMT
SW-LEAF-01(config)# end
SW-LEAF-01# copy running-config startup-config   // ★ 关键一步，漏了重启即丢配置
SW-LEAF-01# show startup-config | include hostname
```

验证与留存：

```text
SW-LEAF-01# show running-config > flash:backup-20261030.cfg   // 变更前备份到本地 flash
SW-LEAF-01# show version
SW-LEAF-01# show users
```

## 陷阱与版本差异

- ⚠️ 最常见的事故：改完配置直接断电/重启。EOS 的 running-config 只在内存中，**必须 `copy running-config startup-config`**；脚本变更后应 `show startup-config | diff` 或对比确认已落盘。
- EOS 提示符是反的：`>` 是特权 EXEC、`#` 是全局配置（与 Cisco 的 `>`/`#` 含义一致，但**没有非特权 `>`**，EOS 登录即处于特权 EXEC）。
- 视图提示符随版本/模式不同：接口 `(config-if-Et1)#`、VLAN `(config-vlan-10)#`、ACL `(config-acl-<name>)#`、`management ssh` 视图在部分版本显示为 `(config-mgt-ssh)#`（以官方手册为准）。
- 密码里含 `?` 时会被 CLI 当成帮助键，需先按 **Ctrl+V** 再输入问号（官方文档明确提示）。
- 不设密码的账号默认**只能从 Console 登录**；`aaa authentication policy local allow-nopassword-remote-login` 会放开远程免密登录，属严重风险项。
- `username` 的 `privilege`/`role` 参数顺序随版本略有差异，落地前建议在 vEOS 上 `username ?` 现场确认（以官方手册为准）。

## 高危操作 ⚠️

- ⚠️ `no username <最后一个管理员>` 后再保存并退出全部会话，将失去远程管理入口；变更前先建好新账号并**另开一个 SSH 会话验证**。
- ⚠️ `aaa root nopassword` / `no aaa root` 会改变底层 Linux root 登录策略，影响 TACACS+/密码恢复流程，仅在维护窗口操作。
- ⚠️ 修改 `management ssh` 或服务 ACL（`management ssh` 视图下 `ip access-group ... in`）会立即切断现有 SSH，务必保留 Console 或带外通道。
- 变更固定顺序：`show running-config` 导出 → 备份 `copy running-config flash:...` → 变更 → `show ...` 验证 → 异常时 `copy flash:backup-... running-config` 回退 → 确认后 `copy running-config startup-config`。

## 参考来源

1. [EOS User Manual · User Security（arista.com）](https://www.arista.com/en/um-eos/eos-user-security)
2. [EOS 4.18.3.1F Manual · 保存配置步骤（arista.com）](https://www.arista.com/assets/data/docs/Manuals/EOS-4.18.3.1F-Manual.pdf)
3. [EOS User Manual · ACLs and Route Maps（arista.com）](https://www.arista.com/en/um-eos/eos-acls-and-route-maps)

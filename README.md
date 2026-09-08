# ZTE G100S-17N3：使用原生数据库固化 LAN Telnet

这是一份面向 **ZTE G100S-17N3 原厂固件** 的最小操作记录：在已经拥有设备合法管理权限和一次可用 Telnet 会话的前提下，使用原生 `TelnetCfg` 数据库把 **LAN 侧** Telnet 配置保存到用户配置区，并由设备自己的 `pc` 进程管理器重启 `telnetd` 使其立即加载。

不修改 rootfs，不写 `/`，不写启动脚本、`rc.local`、init hook，也不刷机。

> 适用范围：已在 G100S-17N3 的 `TelnetCfg` 表和 `pc` 管理器上核对过字段名与服务模型。其他中兴型号、硬件版本或运营商定制固件的表结构可能不同；不要直接套用。

## 风险与前提

- Telnet 是明文协议。只允许在受信任的 LAN 内使用；不要开启 `Wan_Enable`，不要做公网映射或端口转发。
- 先从当前可用 Telnet 会话执行；本方法**不会**从零获取 Telnet 权限。
- 使用新的高强度本地口令。本文不包含任何设备、运营商或论坛账号的真实凭据。
- 每个命令均等待 shell 提示符返回后再执行下一条。重启 `telnetd` 时当前会话会断开，这是预期行为。
- 将下面的 `<USER>` 与 `<PASSWORD>` 替换为只含 `A-Za-z0-9._-` 的值，避免 shell 转义和明文泄漏问题。

## 1. 修改前读取与备份

先确认这是正确的设备和服务：

```sh
sendcmd 1 DB p TelnetCfg
sendcmd -pc show
```

输出中应有 `TelnetCfg`，以及由 `pc` 托管的 `telnetd`。把第一条命令的输出复制保存到电脑本地，作为回滚依据。

当前 G100S-17N3 上核对到的关键字段如下：

| 字段 | 作用 |
| --- | --- |
| `Lan_Enable` | LAN Telnet 监听开关 |
| `TS_UName` / `TS_UPwd` | Telnet 身份字段 |
| `TSLan_UName` / `TSLan_UPwd` | LAN Telnet 身份字段 |
| `Max_Con_Num` | 最大并发连接数 |
| `InitSecLvl` | 初始安全级别 |

## 2. 写入原生持久配置

以下只启用 LAN，不修改 WAN 开关、端口、rootfs 或启动链：

```sh
sendcmd 1 DB set TelnetCfg 0 Lan_Enable 1
sendcmd 1 DB set TelnetCfg 0 TS_UName <USER>
sendcmd 1 DB set TelnetCfg 0 TS_UPwd <PASSWORD>
sendcmd 1 DB set TelnetCfg 0 TSLan_UName <USER>
sendcmd 1 DB set TelnetCfg 0 TSLan_UPwd <PASSWORD>
sendcmd 1 DB set TelnetCfg 0 Max_Con_Num 3
sendcmd 1 DB set TelnetCfg 0 InitSecLvl 3
sendcmd 1 DB save
sync
sendcmd 1 DB p TelnetCfg
```

最后一次读取必须确认 `Lan_Enable=1`，两个用户名字段都是预期值。`sendcmd 1 DB save` 是持久化动作；只改运行时文件、只运行 `telnetd`，或只发送 `sync` 都不能替代它。

## 3. 不重启光猫，立即加载服务

重新读取托管进程并只重启 `telnetd`：

```sh
sendcmd -pc show
# 从输出中找到 telnetd 对应的 PID，例如 1234
sendcmd -pc kill 1234
```

当前 Telnet 会话会立刻断开。等待约 5–30 秒后，用一个**新**终端从 LAN 重新登录：

```sh
telnet 192.168.1.1 23
```

使用刚才设置的 `<USER>` / `<PASSWORD>`。成功进入 shell 后，执行：

```sh
sendcmd 1 DB p TelnetCfg
sendcmd -pc show
```

这证明了“数据库读回 + 受管服务重启 + 新会话登录”三层都成立，但还不等于重启持久性已验证。

## 4. 验证重启持久性

确认业务空闲、PON 正常后，使用设备正常的重启入口重启一次。设备恢复后验证：

```sh
telnet 192.168.1.1 23
# 登录成功后
sendcmd 1 DB p TelnetCfg
sendcmd -pc show
```

验收标准：LAN Telnet 能以新凭据登录；`Lan_Enable=1`；`telnetd` 仍被 `pc` 托管。若任一项不成立，不要继续反复写入，先导出当前 `TelnetCfg` 并与修改前备份对比。

## 回滚

使用仍可登录的 LAN Telnet 会话，把第 1 步备份中的字段值逐项写回，然后执行：

```sh
sendcmd 1 DB save
sync
sendcmd -pc show
# 找到 telnetd 的 PID 后，只重启该服务
sendcmd -pc kill <TELNETD_PID>
```

如果只是临时关闭 LAN Telnet，可将 `Lan_Enable` 设为 `0` 后同样 `DB save`、`sync` 并重启 `telnetd`。不要删除表、修改 `ProcCap`、修改启动脚本或改写闪存分区。

## 故障边界

- `sendcmd -pc show` 没有 `telnetd`：停止；该固件的服务模型不匹配。
- `DB p TelnetCfg` 缺字段或表不存在：停止；不要把本机字段名套到其他型号。
- 新会话无法登录：保留原会话的完整输出，恢复备份字段后再诊断。
- 重启后失效：先比对 `TelnetCfg` 读回与备份，再检查本机是否实际执行了 `DB save`；不要通过写 rootfs、`rc.local` 或 watchdog hook 强行修复。

## 证据层级

1. `DB p TelnetCfg`：配置写回；
2. 新会话登录：运行时服务已加载；
3. 正常重启后的新会话登录与读回：持久化闭环。

只有第 3 项完成，才能称为“重启后仍生效”。

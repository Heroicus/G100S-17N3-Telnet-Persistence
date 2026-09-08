# ZTE G100S-17N3：先临时开启，再可靠固化 LAN Telnet

> **更正说明（2026-09-08）**：这个型号上，`sendcmd 1 DB set TelnetCfg ...` 的“命令成功”或即时读回，**不能证明**重启后会保留。因此，不能把它作为可靠固化方案。经真实重启和 BusyBox 新会话验证的路径是：**认证的临时 Telnet → 同时修改 current/backup 两个用户配置容器 → 回包校验 → 原子替换 → 正常重启读回**。

适用对象：本记录只针对已实际核对过的 **ZTE G100S-17N3 原厂固件**。不同中兴型号、硬件版本和运营商定制固件的 Web 端点、配置容器格式、字段和进程模型都可能不同；先备份、只读核对，不能照抄到其他型号。

Telnet 为明文协议。只允许在受信任 LAN 内使用；不要开启 WAN Telnet、不要做公网映射或端口转发。

## 全流程总览

```text
已知 Web 管理凭据
  -> 认证的 factory-mode 请求启动临时 Telnet
  -> 新会话登录验证（临时态）
  -> 导出 user + backup 配置容器并做 hash
  -> 只修改 TelnetCfg 的必要字段
  -> 使用该固件匹配的 codec 回包并重新解包校验
  -> 原子替换两份容器 + sync
  -> 正常重启
  -> 新会话登录 + DB/服务读回（持久态）
```

## 0. 先做备份与网络约束

- 管理电脑应**直连光猫 LAN**，只在本地网段操作；关闭可能抢路由的 VPN/TUN。
- 先保存当前配置容器的原始副本与 hash；至少同时备份：

```text
/userconfig/cfg/db_user_cfg.xml
/userconfig/cfg/db_backup_cfg.xml
```

- 两个文件都不是普通明文 XML；不要在设备上直接 `sed`、不要只替换其中一个、不要把其它型号的容器或密钥拿来写入。
- 备份应放到管理电脑，核对文件大小与 SHA-256。没有可回退的双容器备份，不进入写入步骤。

## 1. 临时开启 Telnet：认证 factory-mode 路径

这一段解决“当前 23 端口没有监听，如何进入 shell”的前提。它依赖已有的**本机 Web 管理员账号和口令**，不是匿名接口，也不使用运营商、设备或论坛的真实凭据。

本机已验证的 `zte` 客户端会向设备的认证 factory-mode 端点发请求，返回一次临时 Telnet 登录信息并让服务在当前运行态可用。命令格式：

```sh
zte \
  -addr http://192.168.1.1 \
  -username '<WEB_ADMIN>' \
  -password '<WEB_PASSWORD>'
```

预期结果是工具输出临时 Telnet 用户名/口令。它们只在当前设备、当前运行态使用，**不要贴进文章、截图或日志**。随后在新的终端验证：

```sh
nc -vz 192.168.1.1 23
# 成功后，用工具输出的临时凭据登录
telnet 192.168.1.1 23
```

如果工具没有返回临时凭据、端口仍未监听、或新会话无法登录，就到此停止；不要猜测 factory-mode 参数、不要在 WAN 侧尝试、不要用临时启动脚本绕过。

### 为什么这是“临时”而不是“固化”

factory-mode 的效果是当前 RAM/服务态的访问入口。设备正常重启后，服务会重新按用户配置区的两个容器加载；没有修改容器时，临时 Telnet 不构成持久化。

## 2. 临时会话内只读确认

成功登录临时 Telnet 后，先确认服务模型与当前值：

```sh
sendcmd 1 DB p TelnetCfg
sendcmd -pc show
```

本机已读到的关键字段是：

| 字段 | 含义 |
| --- | --- |
| `Lan_Enable` | LAN Telnet 监听开关 |
| `TS_UName` / `TS_UPwd` | Telnet 身份字段 |
| `TSLan_UName` / `TSLan_UPwd` | LAN Telnet 身份字段 |
| `Max_Con_Num` | 最大并发连接数 |
| `InitSecLvl` | 初始安全级别 |

同时确认 `telnetd` 由 `pc` 管理。若表不存在、字段不匹配，或 `pc` 看不到 `telnetd`，停止，不套用本记录。

## 3. 可靠固化：双配置容器，而非单条 DB 命令

### 3.1 需要修改的最小字段集

在**两份**容器解包得到的 XML 内，仅修改 `TelnetCfg` 的 `Row No="0"`。不改变 WAN、端口、`ProcCap`、启动脚本或闪存分区。

```text
Lan_Enable      = 1
TS_UName        = <NEW_LAN_USER>
TS_UPwd         = <NEW_LAN_PASSWORD>
TSLan_UName     = <NEW_LAN_USER>
TSLan_UPwd      = <NEW_LAN_PASSWORD>
Max_Con_Num     = 3
InitSecLvl      = 3
```

`<NEW_LAN_USER>` 与 `<NEW_LAN_PASSWORD>` 应自行设置；不要使用设备出厂信息或临时 factory-mode 凭据。为避免 shell/工具转义问题，建议仅使用 `A-Za-z0-9._-`。

### 3.2 容器处理的不可省步骤

1. 从临时 Telnet 会话把 `db_user_cfg.xml` 和 `db_backup_cfg.xml` 拉到电脑；记录两个原始 SHA-256。
2. 使用**与该 G100S-17N3 固件版本匹配**的配置容器 codec 分别解包；确认所得 XML 都有 `TelnetCfg` 和全部上述字段。
3. 每份 XML 只改上面的 7 个字段；对修改前后做结构化 diff，禁止出现其它表或字段变化。
4. 分别回包为 user/backup 对应的容器；再将回包重新解包，逐字段比对 7 个目标值，且确认未发生其它 XML 变动。
5. 上传到 `/tmp`，在设备上比对 hash；先写入同目录隐藏临时名，再用 `mv` 分别原子替换两个正式文件，最后执行 `sync`。
6. 只进行一次设备正常重启。不要以“当前临时会话没掉线”代替重启验证。

> 本机观察到容器是加密/压缩的配置封装，而不是普通 XML。没有可复现的 `unpack → minimal diff → pack → re-unpack` 闭环，就不要写回任何容器。

### 3.3 不使用这条“看似简单”的路径

以下命令可以用于观察字段，但在本机这版固件上不应被当成可靠固化依据：

```sh
sendcmd 1 DB set TelnetCfg 0 Lan_Enable 1
sendcmd 1 DB save
```

原因是配置服务可让它返回成功或出现即时读回，却没有把相同状态可靠写入两个启动时会加载的容器。要判断“固化”只能看第 4 节的正常重启后结果。

## 4. 验收：分清三层证据

设备正常重启完成后，使用**新**终端检查：

```sh
nc -vz 192.168.1.1 23
telnet 192.168.1.1 23
# 使用第 3 节设置的新 LAN 凭据登录后：
sendcmd 1 DB p TelnetCfg
sendcmd -pc show
```

验收顺序：

1. `TelnetCfg` 读回：配置层；
2. 新会话可登录、`telnetd` 被 `pc` 托管：运行时层；
3. **正常重启后**仍能以新凭据登录，且上述两项依然成立：持久化层。

只有第 3 项通过，才可以称“固化成功”。

## 回滚

仍能进入 Telnet 时，使用备份的 user/backup 原始容器按相同“上传 → hash → 临时名 → 原子替换 → `sync` → 正常重启”流程恢复。不要只恢复一份，也不要为了抢救而改 `rc.local`、`S99modules`、`ProcCap`、rootfs 或 NAND 分区。

## 已验证边界

- 已验证：本机 G100S-17N3 上，临时入口后以双容器最小修改、正常重启、真实 BusyBox 新会话完成持久性闭环。
- 未主张：其它中兴型号/版本、WAN Telnet、公网访问、单条 `DB set` 固化、仅端口开放或仅当前会话可用。

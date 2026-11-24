# 消息

> 中文 | [English](../../en/communication/messages.md) · [索引](../index.md) · [首页](../../../README.md)

## 消息封装

每条消息的正文是一个 TOML 表，根表必须包含字符串键 `type`：

```toml
type = "setInstallLocation"
value = 'D:\Apps\Example'
```

通用规则：

- `type` 必填、字符串、camelCase、大小写敏感。
- 消息字段直接放在根表，不使用额外的 `payload` 包装。
- 未知键必须被忽略，以支持向前兼容。
- 未知 `type` 必须被接收方忽略；stub 只特殊处理 `readyToExit`，其余照常转发。
- 消息中不能出现 `null`；TOML 1.0 没有 null。空值用省略或空字符串表示。
- 不定义序号、时间戳或会话 ID；管道顺序就是消息顺序。
- 消息字段类型错误、必填字段缺失属于消息级错误，见 [错误处理](errors.md)。

## 消息一览

| 类型 | 方向 | 作用 |
| --- | --- | --- |
| `setInstallLocation` | client → backend | 设置安装位置 |
| `setCreateStartMenuFolder` | client → backend | 是否创建开始菜单文件夹 |
| `setCreateDesktopShortcut` | client → backend | 是否创建桌面快捷方式 |
| `startInstall` | client → backend | 开始安装 |
| `installFinished` | backend → client | 安装完成（成功或失败） |
| `readyToExit` | client → stub | client 已处理完毕，可以结束会话 |

除 `readyToExit` 外，所有消息都经过 stub 原样转发。

## setInstallLocation

设置待安装位置。

```toml
type = "setInstallLocation"
value = 'D:\Apps\Example'
```

| 字段 | 类型 | 必需 | 说明 |
| --- | --- | --- | --- |
| `value` | string | 是 | 安装目录 |

规则：

- 必须是绝对路径，且只允许本地盘符路径（例如 `D:\Apps\Example`）。
- 不接受 UNC、网络路径、设备路径、相对路径、包含 `..` 的路径或 NTFS ADS。
- 可以包含 `%VAR%` 环境变量引用；backend 展开后重新校验。无法展开的变量视为无效。
- 长度不能超过 32,767 个 UTF-16 代码单元。
- 如果路径已存在但不是目录，安装失败。
- backend 在 `startInstall` 前收到时覆盖当前值；重复发送时后发的有效。
- 如果 client 从未发送，backend 使用 `defaults.installLocation`。
- 收到 `startInstall` 后，该消息不再改变本次安装状态。
- 安装位置的额外安全校验见 [强制安全要求](../security/requirements.md)。

## setCreateStartMenuFolder

```toml
type = "setCreateStartMenuFolder"
value = true
```

| 字段 | 类型 | 必需 | 说明 |
| --- | --- | --- | --- |
| `value` | boolean | 是 | `true` 创建，`false` 不创建 |

- 未收到时使用 `defaults.createStartMenuFolder`。
- 重复发送时后发的有效。
- `false` 只表示本次不创建，不删除已有文件夹或快捷方式。
- 创建位置：当前用户的 `FOLDERID_Programs\<name>`。
- 具体快捷方式内容由 backend 根据 payload 决定；只有 payload 成功写入后才创建。

## setCreateDesktopShortcut

```toml
type = "setCreateDesktopShortcut"
value = false
```

| 字段 | 类型 | 必需 | 说明 |
| --- | --- | --- | --- |
| `value` | boolean | 是 | `true` 创建，`false` 不创建 |

- 未收到时使用 `defaults.createDesktopShortcut`。
- 重复发送时后发的有效。
- `false` 只表示本次不创建，不删除已有快捷方式。
- 创建位置：当前用户的 `FOLDERID_Desktop\<name>.lnk`。
- 快捷方式目标、参数和工作目录由 backend 根据 payload 决定；目标必须位于安装目录内。
- 只有 payload 成功写入后才创建。

## startInstall

```toml
type = "startInstall"
```

- 没有额外字段。
- client 必须在发送前把当前三个设置值发给 backend（推荐全部发送）。
- backend 使用当前状态安装；未收到过的设置使用 `config.toml` 默认值。
- backend 必须在开始前校验所有设置和安全条件；任何失败都通过 `installFinished` 报告。
- 第一个 `startInstall` 生效；之后的 `startInstall` 必须被忽略。
- 安装可以同步或异步，但 backend 必须最终发送且只发送一次 `installFinished`。
- 安装期间收到的设置消息必须被忽略。

## installFinished

backend 在安装尝试结束后发送。

成功：

```toml
type = "installFinished"
success = true
```

失败：

```toml
type = "installFinished"
success = false
error = "无法写入安装目录：拒绝访问"
```

| 字段 | 类型 | 必需 | 说明 |
| --- | --- | --- | --- |
| `success` | boolean | 是 | 安装是否成功 |
| `error` | string | 失败时必需，成功时可选 | 面向用户或日志的错误说明 |

- 每个会话最多发送一次；处理了 `startInstall` 就必须发送一次。
- `success = false` 时 `error` 必须提供，且不得包含 NUL。
- `success = true` 时 client 忽略 `error`。
- client 收到后显示结果；失败时展示 `error`，缺失时显示通用错误。
- backend 可以在发送后继续连接，也可以自行退出；stub 不得因此提前结束 client。

## readyToExit

```toml
type = "readyToExit"
```

- 方向固定为 client → stub；stub 必须消费，不转发给 backend。
- stub 只从 client 管道识别该消息；backend 管道上的同名消息必须忽略。
- client 必须在收到 `installFinished` 并完成结果展示后再发送。
- 只应发送一次；stub 收到第一个后进入关闭流程。
- 发送后 client 可以自行退出；stub 也会结束它。
- client 未发送 `readyToExit` 就断开时，stub 视为取消并清理。

## 状态与“后发有效”

backend 在内存中维护：

| 状态字段 | 初始值 | 更新消息 | 冻结时机 |
| --- | --- | --- | --- |
| 安装位置 | `defaults.installLocation` | `setInstallLocation` | 处理 `startInstall` |
| 创建开始菜单文件夹 | `defaults.createStartMenuFolder` | `setCreateStartMenuFolder` | 处理 `startInstall` |
| 创建桌面快捷方式 | `defaults.createDesktopShortcut` | `setCreateDesktopShortcut` | 处理 `startInstall` |
| 已开始 | `false` | `startInstall` | 第一次生效后 |
| 已完成 | `false` | 发送 `installFinished` 后 | 本会话内 |

“后发有效”的准确含义：

1. 对三个 `set*` 消息，backend 每收到一个合法消息就覆盖对应字段；不按消息去重，不比较新旧值。
2. 最终生效的是 `startInstall` 处理前最后收到的合法值。
3. 管道保证顺序；同一设置连续收到两次，第二次覆盖第一次。
4. 在 `startInstall` 之前，无效的 `set*` 消息也参与顺序：它把对应字段标记为无效；之后同一字段的合法消息可以覆盖并清除标记。
5. `startInstall` 不是“后发有效”：第一个生效，之后的忽略。
6. `readyToExit` 也是第一个生效。
7. 状态只存在于本次会话内；进程退出后不保留，下次安装重新从 `config.toml` 初始化。

## 示例

```toml
type = "setInstallLocation"
value = 'D:\Apps\Example App'
```

```toml
type = "setCreateStartMenuFolder"
value = true
```

```toml
type = "setCreateDesktopShortcut"
value = false
```

```toml
type = "startInstall"
```

```toml
type = "installFinished"
success = true
```

```toml
type = "readyToExit"
```

转发关系：

```text
set* / startInstall  client → stub → backend
installFinished      backend → stub → client
readyToExit          client → stub（消费，不转发）
```

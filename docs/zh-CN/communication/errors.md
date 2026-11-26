# 错误处理

> 中文 | [English](../../en/communication/errors.md) · [索引](../index.md) · [首页](../../../README.md)

## 原则

- **Fail closed。** 任何签名、哈希、配置、管道、路径或解压安全校验失败，都必须中止安装；不得回退到不安全行为。
- **不伪造结果。** stub 不得代替 backend 发送 `installFinished`。
- **可读错误。** 面向用户的错误必须是 UTF-8 文本，不包含 NUL，不泄露密钥或令牌。
- **清理。** 任何失败路径都必须结束子进程、关闭管道并删除会话目录。

## 错误矩阵

| 场景 | 处理 |
| --- | --- |
| `config.toml` 缺失、TOML 语法错误、协议号不支持、必填项缺失 | stub 显示错误并退出；不创建管道、不启动 client/backend |
| `foo.exe` 签名无效、不受信任或哈希清单不匹配 | stub 中止；不释放、不启动 |
| 子进程 30 秒内未连接管道 | stub 结束会话并退出 |
| 连接方 PID 与 stub 启动的子进程不一致 | stub 断开连接、结束会话；这是安全失败 |
| 帧长度非法、正文不是 UTF-8、TOML 非法、根不是表、缺少字符串 `type` | stub 视为致命协议错误，结束整个会话 |
| 单帧超过 1 MiB，或单方向缓存超过 64 帧 / 4 MiB | stub 中止会话 |
| 消息字段类型错误或必填字段缺失 | 三个 `set*`：对应字段标记为无效；后续合法消息可清除。`startInstall` 时仍有无效字段则 `installFinished success = false`。无法解析的 `installFinished`：client 按失败处理 |
| 未知消息类型或未知字段 | 接收方忽略；stub 对除 `readyToExit` 外的未知类型照常转发 |
| `installProgress` 字段非法或无法解析 | client 忽略该进度消息，继续等待 `installFinished`，不得因此中止安装 |
| backend 无法发送 `installProgress` | 进度是建议性信息；backend 可继续安装，最终仍以 `installFinished` 为准 |
| backend 在 `installFinished` 前退出或断开 | stub 关闭 client 管道；client 将 EOF 视为失败；stub 清理后退出 |
| backend 在 `installFinished` 后退出 | 正常；stub 保持 client 管道直到 `readyToExit` 或 client 断开 |
| client 未发送 `readyToExit` 就退出 | stub 视为取消：关闭 backend 管道，等待 5 秒后强制结束，清理并退出 |
| client 在 `installFinished` 前发送 `readyToExit` | stub 仍关闭会话；这是 client 违约，backend 可能被中途终止 |
| 安装失败（权限、磁盘、解压、快捷方式） | backend 发送 `installFinished success = false` 和 `error`；client 展示错误后仍发送 `readyToExit` |
| `startInstall` 重复 | 第一次生效，后续忽略 |
| `readyToExit` 重复 | 第一次生效，stub 已进入关闭流程 |

## 超时

| 项目 | 值 |
| --- | --- |
| 子进程连接管道 | 30 秒 |
| 关闭等待 | 5 秒 |
| 管道读写 | 必须有超时或可取消；不得无限阻塞 |
| 安装本身 | 不设固定超时；client 显示实时进度；没有进度消息时显示不确定进度，用户可关闭窗口取消 |

## 安装失败后的状态

- 快捷方式不得指向不存在的目标；安装失败时不得创建快捷方式。
- backend 应使用暂存目录；失败时删除暂存内容，尽量不破坏已有安装。
- 如果无法完全回滚，`error` 必须说明失败阶段和可能留下的内容。
- client 展示错误后仍然发送 `readyToExit`，让 stub 正常清理。

## 安全失败

以下情况必须立即中止，不得继续安装：

- 签名、哈希、PID、DACL、会话目录或路径校验失败；
- 安装位置指向受保护目录、重解析点或非法路径；
- ZIP 条目路径穿越、ADS、设备名、符号链接或解压限制被触发；
- 包内资源路径越界或试图读取包外文件；
- 任何组件试图执行 shell 命令或联网下载。

详细要求见 [强制安全要求](../security/requirements.md)。

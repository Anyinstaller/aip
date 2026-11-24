# 架构与组件

> 中文 | [English](../../en/overview/architecture.md) · [索引](../index.md) · [首页](../../../README.md)

AIP v1 只支持 Windows。一个安装包由三个进程组成：stub、client、backend。三者通过 Windows 命名管道交换 TOML 消息；最终用户只看到一个单独的 `foo.exe`。

## 设计目标

1. **单文件分发。** 发布物只有一个 `foo.exe`，不依赖安装包旁边的 DLL、配置文件或资源文件。
2. **界面与安装逻辑解耦。** client 和 backend 可以用任意语言、任意 UI 框架实现，只要遵守 AIP。
3. **stub 统一协调。** client 和 backend 不直接连接；stub 创建管道、转发消息、管理生命周期。
4. **消息可调试。** 管道帧内容为 UTF-8 TOML，便于日志、抓包和手写测试。
5. **默认安全。** 所有安全要求都是强制项；没有“可选加固”或“兼容模式”。
6. **单会话。** 一次 `foo.exe` 运行对应一个安装会话，不并发安装多个应用。

## 组件

| 组件 | 必需 | 职责 |
| --- | --- | --- |
| **stub** | 是 | 校验包、创建管道、启动 client/backend、双向中继、结束会话 |
| **client** | 是 | 显示 GUI、收集设置、发送开始安装、显示结果 |
| **backend** | 是 | 记住设置、解压 payload、创建快捷方式、报告完成 |
| **config.toml** | 是 | 协议号、名称、图标、背景和默认选项 |
| **payload.zip** | 是 | 要安装的应用本体 |
| **可选资源** | 否 | 背景图、图标、运行库、许可证等，全部内嵌在 `foo.exe` 中 |

## stub

stub 是 `foo.exe` 的入口和会话所有者：

- 校验 `foo.exe` 的 Authenticode 签名和内嵌组件哈希。
- 读取 `config.toml`，校验协议号和必填字段。
- 创建私有会话目录，释放 `client.exe`、`backend.exe`、`config.toml`、`payload.zip` 和资源。
- 生成随机会话 ID，创建两条命名管道。
- 启动 client 和 backend，只传递各自的 `--aip-pipe-name`。
- 双向转发 TOML 帧；识别并消费 client 的 `readyToExit`。
- 收到 `readyToExit` 或发生致命错误后，结束两个子进程、清理会话目录并退出自身。

stub 不显示常规安装界面，也不执行安装。它可以在启动失败时显示系统错误对话框。

## client

client 是安装界面进程：

- 读取运行时的 `config.toml`，用 `name`、`icon`、`background` 和 `[defaults]` 构造界面。
- 允许用户修改安装位置、开始菜单文件夹开关和桌面快捷方式开关。
- 发送 `setInstallLocation`、`setCreateStartMenuFolder`、`setCreateDesktopShortcut`，然后发送 `startInstall`。
- 接收 `installFinished`，显示成功或失败结果。
- 在界面工作全部完成后发送 `readyToExit`。

client 不得自己解压 payload、写安装目录或创建快捷方式。它也不得读取 `foo.exe` 之外的背景图或图标。

## backend

backend 是安装执行者：

- 从 `config.toml` 的 `[defaults]` 初始化本次会话状态。
- 接收 client 的设置，按“后发有效”规则更新状态。
- 收到 `startInstall` 后，严格校验安装位置和包内资源。
- 把 `payload.zip` 解压到安装位置。
- 根据开关创建当前用户的开始菜单文件夹和桌面快捷方式；具体快捷方式目标由 backend 根据 payload 决定。
- 发送一次 `installFinished`。

backend 不负责常规 GUI。所有安装路径校验、快捷方式目标校验和解压限制都必须按 [强制安全要求](../security/requirements.md) 执行。

## 信任模型

- `foo.exe` 是唯一可信入口；发布包必须签名并通过 stub 自校验。
- client 和 backend 都由 stub 从私有会话目录启动，不能读取任意本地资源路径。
- client 与 backend 之间没有直接管道，不能绕过 stub。
- 管道名、连接进程和消息大小都经过校验；不满足安全条件时立即中止（fail closed）。
- `config.toml`、`payload.zip` 和可选资源都内嵌在已签名的 `foo.exe` 中。

## 版本

当前规范为 **AIP v1**，对应 `config.toml` 中的 `protocol = 1`。不兼容修改必须提升协议号，详见 [版本与兼容性](../versioning.md)。

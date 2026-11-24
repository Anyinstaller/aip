# 管道与中继

> 中文 | [English](../../en/communication/transport.md) · [索引](../index.md) · [首页](../../../README.md)

## 拓扑

```text
┌────────┐   管道 A    ┌──────┐   管道 B    ┌─────────┐
│ client │ <=========> │ stub │ <=========> │ backend │
└────────┘             └──────┘             └─────────┘
```

- 管道 A 由 stub 提供给 client，管道 B 由 stub 提供给 backend。
- client 和 backend 不知道对方管道名，也不能直接连接。
- 所有消息都经过 stub；`readyToExit` 被 stub 消费，不转发。
- 每条管道是全双工字节流，但每个方向上的消息顺序必须保持。

## 管道名

stub 为每个会话生成不可预测的 128 位随机会话 ID，并创建：

```text
aip-<sessionId>-client
aip-<sessionId>-backend
```

完整路径：

```text
\\.\pipe\aip-<sessionId>-client
\\.\pipe\aip-<sessionId>-backend
```

`--aip-pipe-name` 传递的是不带 `\\.\pipe\` 前缀的名字，例如：

```text
aip-7f3c9a1e2b4d4e8fa6c0d5e1f2a3b4c5-client
```

## 启动参数

stub 启动两个独立进程时，只传管道名：

```text
<client.exe>  --aip-pipe-name <client-pipe-name>
<backend.exe> --aip-pipe-name <backend-pipe-name>
```

- 角色由可执行文件身份确定，没有其他角色参数。
- 进程打开管道时使用完整路径 `\\.\pipe\<pipe-name>`。
- 当前工作目录必须是会话包根目录。
- 继承环境变量；协议之外的额外参数应被忽略。
- client/backend 必须在启动后 30 秒内连接；超时视为启动失败。

## 创建与连接

1. stub 必须在启动子进程前以服务端身份创建两条管道。
2. 每条管道只创建一个实例，只接受一个连接；连接建立后立即关闭监听端。
3. stub 用 `GetNamedPipeClientProcessId` 取得连接方 PID，并必须与它启动的子进程 PID 一致；否则断开并中止会话。
4. client 打开管道时必须使用 `SECURITY_SQOS_PRESENT | SECURITY_IDENTIFICATION`，限制服务端模拟。
5. 管道句柄不可继承；stub 用 `bInheritHandles = FALSE` 启动子进程。
6. 管道使用字节模式（`PIPE_TYPE_BYTE | PIPE_READMODE_BYTE | PIPE_WAIT`）。
7. 所有管道 I/O 必须有超时或可取消；不得无限阻塞。
8. 安全属性和 DACL 要求见 [强制安全要求](../security/requirements.md)。

## 中继规则

- client → stub → backend：三个 `set*` 和 `startInstall`。
- backend → stub → client：`installFinished`。
- client → stub：`readyToExit`，由 stub 消费。
- stub 必须按接收顺序转发，不得重排、合并或重新序列化 TOML 正文。
- stub 只解析帧的 `type` 以识别 `readyToExit`；未知类型照常转发。
- stub 必须在任一方向断开或出现协议错误时结束整个会话，不能只关闭一半。

## 缓冲与限制

- 单帧最大 1 MiB。
- 每个方向最多缓存 64 帧或 4 MiB，先到者为准；超过则中止会话。
- 如果目标进程尚未连接，stub 可以有限缓冲，但不得无限堆积。
- 连接超时 30 秒；关闭等待 5 秒。
- 这些限制是强制的，不是建议。

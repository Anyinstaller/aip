# 生命周期

> 中文 | [English](../../en/overview/lifecycle.md) · [索引](../index.md) · [首页](../../../README.md)

## 启动

1. 用户运行 `foo.exe`。
2. stub 校验自身签名和内嵌组件哈希；失败则显示错误并退出。
3. stub 读取并校验 `config.toml`；协议号不支持或必填项缺失则拒绝运行。
4. stub 创建私有会话目录，释放 client、backend、config、payload 和可选资源。
5. stub 生成随机会话 ID，创建两条命名管道：

   ```text
   \\.\pipe\aip-<sessionId>-client
   \\.\pipe\aip-<sessionId>-backend
   ```

6. stub 启动 backend 和 client。启动参数只有管道名：

   ```text
   backend.exe --aip-pipe-name aip-<sessionId>-backend
   client.exe  --aip-pipe-name aip-<sessionId>-client
   ```

   角色由可执行文件身份确定，不通过命令行参数区分。

7. stub 把两个子进程加入同一个 Job Object（`JOB_OBJECT_LIMIT_KILL_ON_JOB_CLOSE`），并校验连接管道的进程 PID。
8. client/backend 必须在启动后 30 秒内连接自己的管道；超时视为启动失败。

## 正常消息流

```text
stub ── 创建管道 ──> 启动 backend ──> 启动 client
                                  │
client ── setInstallLocation ───────────────┐
client ── setCreateStartMenuFolder ────────┤
client ── setCreateDesktopShortcut ────────┤
client ── startInstall ────────────────────┤
                                  │         ▼
                                  │      stub 转发
                                  │         │
                                  │         ▼
                                  │      backend 安装
                                  │         │
backend ── installFinished ───────┴──> stub 转发 ──> client
                                  │
client ── readyToExit ──> stub ── 结束 client/backend ──> stub 退出
```

顺序要求：

1. client 可以随时发送设置消息；backend 记住最后一个合法值。
2. client 在 `startInstall` 前发送当前三个设置值（推荐全部发送，至少发送被修改过的值）。
3. backend 收到第一个 `startInstall` 后开始安装，后续 `startInstall` 忽略。
4. backend 可以发送零次或多次 `installProgress`；client 据此更新进度，但不得把进度当作完成。
5. backend 只能发送一次 `installFinished`，无论成功还是失败。
6. client 收到 `installFinished` 并完成结果展示后发送 `readyToExit`。
7. stub 收到 `readyToExit` 后结束整个会话。

## 关闭

1. stub 停止转发新消息。
2. stub 关闭两条管道。
3. stub 通知 client 和 backend 退出；最多等待 5 秒。
4. 仍未退出的进程由 stub 强制结束；Job Object 也会在 stub 退出时清理子进程。
5. stub 删除会话目录和临时文件。
6. stub 退出自身（“自杀”）。如果 `foo.exe` 从临时位置运行，可以安排退出后删除自身，但不得删除用户安装目录或用户已有的快捷方式。

## 失败与取消

- 签名、哈希、配置或管道安全校验失败：立即中止，不执行安装。
- client 在未发送 `readyToExit` 时断开：stub 视为取消，结束 backend 并清理；backend 必须使用暂存目录，避免留下半安装状态。
- backend 在 `installFinished` 前断开：stub 关闭 client 管道，client 按失败处理，stub 清理。
- backend 在 `installFinished` 后退出：正常；stub 保持 client 管道直到 `readyToExit`。
- client 在 `installFinished` 前发送 `readyToExit`：stub 仍关闭会话；这是 client 违约，backend 可能被中途终止。

完整错误矩阵见 [错误处理](../communication/errors.md)。

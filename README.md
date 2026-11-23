# Anyinstaller Protocol（AIP）

**规范版本：1.0　协议号：`1`　平台：Windows**

AIP（全称 **Anyinstaller Protocol**）是 Anyinstaller 安装包中各组件的通信与配置规范。它把安装包拆成三个职责明确的进程：

- **stub** 只负责启动、管道中继和退出；
- **client** 只负责界面与收集用户选择；
- **backend** 只负责真正的安装。

三者通过 Windows 命名管道交换 TOML 消息，安装包本体、界面资源和默认配置则来自固定的包内文件。

> 如果喜欢本项目，请 Star [导航页](https://github.com/Anyinstaller/anyinstaller)。
> If you like this project, please star the [navigation page](https://github.com/Anyinstaller/anyinstaller).

> **English summary:** AIP is a Windows-only installer protocol: a stub launches a client and a backend and relays TOML messages over Windows named pipes, while `config.toml` and `payload.zip` define the package.

本文使用 RFC 2119 风格的关键词：**必须**（MUST）、**应该**（SHOULD）、**可以**（MAY）。

---

## 目录

- [1. 概览](#1-概览)
  - [1.1 设计目标与范围](#11-设计目标与范围)
  - [1.2 六个组成部分](#12-六个组成部分)
  - [1.3 包结构](#13-包结构)
  - [1.4 启动参数](#14-启动参数)
  - [1.5 正常流程](#15-正常流程)
- [2. 通讯协议](#2-通讯协议)
  - [2.1 拓扑](#21-拓扑)
  - [2.2 命名管道](#22-命名管道)
  - [2.3 帧格式](#23-帧格式)
  - [2.4 消息格式](#24-消息格式)
  - [2.5 消息定义](#25-消息定义)
  - [2.6 状态与“后发有效”](#26-状态与后发有效)
  - [2.7 正常关闭](#27-正常关闭)
  - [2.8 异常处理](#28-异常处理)
  - [2.9 安全](#29-安全)
- [3. 配置格式](#3-配置格式)
  - [3.1 文件与语法](#31-文件与语法)
  - [3.2 结构](#32-结构)
  - [3.3 字段说明](#33-字段说明)
  - [3.4 background](#34-background)
  - [3.5 校验与回退](#35-校验与回退)
  - [3.6 完整示例](#36-完整示例)
- [4. 版本与兼容性](#4-版本与兼容性)
- [5. 完整交互示例](#5-完整交互示例)

---

## 1. 概览

### 1.1 设计目标与范围

1. **仅支持 Windows。** 通讯层使用 Windows 命名管道，不提供其他平台的替代实现。
2. **界面与逻辑解耦。** client 和 backend 可以是任意语言、任意框架实现的程序，只要遵守本协议。
3. **stub 是唯一入口。** 用户运行的是 stub；client 和 backend 都由 stub 拉起，并且只能通过 stub 通信。
4. **消息可读。** 管道帧的内容是 UTF-8 TOML，便于调试、抓包和手写测试。
5. **最小信任面。** client 与 backend 不直接建立连接；stub 控制整个会话的生命周期。
6. **单会话。** 一次 stub 运行对应一个安装会话，不跨进程保存状态，也不并发安装多个应用。

### 1.2 六个组成部分

| 组件 | 必需 | 职责 |
| --- | --- | --- |
| **stub** | 是 | 拉起 client/backend、创建管道、双向转发消息、结束会话 |
| **client** | 是 | 显示 GUI、收集用户设置、发送开始安装指令、显示结果 |
| **backend** | 是 | 记住设置、解压 payload、创建快捷方式、报告安装完成 |
| **config.toml** | 是 | 安装包配置：协议号、名称、图标、背景和默认选项 |
| **payload.zip** | 是 | 真正要安装到用户机器上的应用本体 |
| **background.webp/avif/png** | 否 | 可选的客户端背景图，随包提供 |

#### 1.2.1 stub

stub 是用户唯一直接运行的程序，通常会被重命名为 `Setup.exe` 或应用名。它**不显示常规安装界面，也不执行安装**，只做以下几件事：

1. 读取并校验包根目录下的 `config.toml`；协议号不支持或必填项缺失时，直接显示错误并退出。
2. 准备运行时包根目录：如果 client/backend 等文件被嵌入 stub，则先把它们释放到临时目录。
3. 生成随机会话 ID，创建两个命名管道（分别给 client 和 backend）。
4. 启动 backend 和 client，并通过命令行参数把各自的管道名传进去。
5. 在两条管道之间双向转发 TOML 帧，保持消息顺序和原始内容。
6. 监听 client 发来的 `readyToExit`。收到后结束 client 和 backend，清理临时文件，然后退出自身（即“自杀”）。
7. 在启动失败、管道异常等无法继续的情况下结束所有子进程，避免留下孤儿进程。

stub **必须**是安装会话的唯一所有者：client 和 backend 不得绕过 stub 直接通信。

#### 1.2.2 client

client 是安装界面进程，负责：

1. 读取 `config.toml`，使用其中的名称、图标、背景和默认选项渲染界面。
2. 允许用户修改安装位置、是否创建开始菜单文件夹、是否创建桌面快捷方式。
3. 通过管道发送 `setInstallLocation`、`setCreateStartMenuFolder`、`setCreateDesktopShortcut`，然后发送 `startInstall`。
4. 接收 `installFinished`，显示成功或失败信息，必要时执行安装后的界面动作（例如显示“完成”页）。
5. 界面工作全部完成后发送 `readyToExit`，然后可以自行退出。

client **不得**自己解压 payload、写安装目录或创建快捷方式；这些都属于 backend。

#### 1.2.3 backend

backend 是真正的安装器，负责：

1. 读取 `config.toml` 中的默认值，作为本次会话的初始状态。
2. 接收并记住 client 发来的设置。对同一个设置，后到的消息覆盖先到的消息。
3. 收到 `startInstall` 后，按当前状态执行安装：
   - 将 `payload.zip` 解压到安装位置；
   - 根据开关创建当前用户的开始菜单文件夹和桌面快捷方式；
   - 快捷方式指向哪个可执行文件由 backend 根据打包内容决定（AIP v1 只传递“是否创建”的开关，不传递目标路径）。
   - 只有在 payload 成功写入后才创建开始菜单文件夹和快捷方式；安装失败时不得留下指向不存在目标的快捷方式。
4. 安装结束后，无论成功还是失败，都向 client 发送一次 `installFinished`。
5. 保持连接，直到 stub 关闭管道或结束进程；也可以在发送完成消息后自行退出。

backend 不负责常规 GUI。它可以在必要时显示系统级错误（例如提权失败），但安装交互应由 client 完成。

安装和快捷方式默认作用于**当前用户**：

- 开始菜单：`%APPDATA%\Microsoft\Windows\Start Menu\Programs\<name>`
- 桌面：`%USERPROFILE%\Desktop\<name>.lnk`

#### 1.2.4 config.toml

用户在打包前编写（也可以由打包器生成）的静态配置文件，位于包根目录。它提供：

- 协议号；
- 应用名称；
- 图标；
- 背景；
- 三个默认值：安装位置、开始菜单文件夹、桌面快捷方式。

config.toml 在安装过程中**只读**，client、backend 和 stub 都可以读取它。

#### 1.2.5 payload.zip

要安装的应用本体。backend 会把压缩包内的条目按原目录结构解压到安装位置：

- 必须是合法的 ZIP 文件；
- 条目路径必须是相对路径；
- 禁止绝对路径、盘符路径、包含 `..` 的路径穿越条目；
- 文件名应为 UTF-8 或纯 ASCII；
- 默认行为是合并目录并覆盖同名文件；
- 推荐先解压到暂存目录，成功后再替换目标目录，避免中途失败留下半安装状态。

AIP v1 不规定 payload.zip 内部的应用清单格式；快捷方式目标等安装细节由 backend 自行决定。

#### 1.2.6 background.webp / background.avif / background.png

可选的客户端背景图，放在包根目录。它不是安装内容，只供 client 显示。三种格式的文件可以同时存在，优先级为：

1. `background.webp`
2. `background.avif`
3. `background.png`

如果 `config.toml` 中的 `background` 指定了有效值，则以配置为准；包内背景图只是配置省略时的回退。

### 1.3 包结构

一个符合 AIP v1 的安装包，逻辑结构如下：

```text
ExampleApp-Setup.exe      # stub，用户运行它
config.toml               # 必需，固定文件名
payload.zip               # 必需，固定文件名
background.webp           # 可选，也可以是 .avif / .png
client.exe                # 逻辑组件，可以被嵌入 stub
backend.exe               # 逻辑组件，可以被嵌入 stub
```

- `config.toml`、`payload.zip` 和可选的 `background.*` **必须**能以固定文件名出现在 client/backend 看到的**包根目录**中。
- client 和 backend 可以是独立可执行文件，也可以由 stub 内嵌并在运行时释放。AIP 只规定它们的行为和启动参数，不规定它们的具体文件名或存放位置。
- stub **必须**把 client 和 backend 的当前工作目录（CWD）设置为包根目录，使它们可以直接打开 `config.toml` 和 `background.*`。

### 1.4 启动参数

stub 启动两个子进程时，必须传入以下参数：

```text
<client>  --aip-role client  --aip-pipe-name <client-pipe-name>
<backend> --aip-role backend --aip-pipe-name <backend-pipe-name>
```

| 参数 | 说明 |
| --- | --- |
| `--aip-role` | 必填，值为 `client` 或 `backend`，用于区分角色 |
| `--aip-pipe-name` | 必填，本进程要连接的管道名，**不含** `\\.\pipe\` 前缀 |

client 和 backend 可以是由同一个可执行文件实现的两种角色，通过 `--aip-role` 区分；也可以是两个完全独立的程序。

补充约定：

- 进程打开管道时使用完整路径 `\\.\pipe\<pipe-name>`。
- stub **必须**在启动任何子进程之前创建好两个管道服务端。
- stub **应该**先启动 backend，再启动 client；如果同时启动，则必须在对方尚未连接时缓存消息，不能丢弃。
- client 和 backend **必须**在启动后 30 秒内连接到自己的管道；超时视为启动失败。
- 两个进程继承 stub 的环境变量；额外的、协议之外的命令行参数应被忽略。

### 1.5 正常流程

```text
stub ── 创建管道 ──> 启动 backend ──> 启动 client
                                  │
client ── 设置 / 开始安装 ──> stub ── 转发 ──> backend
                                  │
backend ── 解压、创建快捷方式 ──> 安装完成 ──> stub ── 转发 ──> client
                                  │
client ── readyToExit ──> stub ── 结束 client 与 backend ──> stub 退出
```

对应的步骤：

1. stub 校验 `config.toml`。
2. stub 生成会话 ID，创建 `...-client` 和 `...-backend` 两条命名管道。
3. stub 启动 backend，再启动 client，并分别传入管道名。
4. client 根据配置显示界面；用户修改选项时，client 把设置发给 backend。
5. 用户点击安装，client 发送当前设置（如有变化）和 `startInstall`。
6. backend 使用当前状态安装，完成后发送 `installFinished`。
7. client 显示结果，完成界面动作后发送 `readyToExit`。
8. stub 收到 `readyToExit`，关闭两条管道，结束两个子进程，清理临时目录，退出自身。

---

## 2. 通讯协议

### 2.1 拓扑

```text
┌────────┐   管道 A    ┌──────┐   管道 B    ┌─────────┐
│ client │ <=========> │ stub │ <=========> │ backend │
└────────┘             └──────┘             └─────────┘
```

- 管道 A 由 stub 提供给 client，管道 B 由 stub 提供给 backend。
- client 和 backend **不得**知道或连接对方的管道。
- stub 是双向中继：
  - client → stub → backend：设置和开始安装；
  - backend → stub → client：安装完成；
  - client → stub：`readyToExit` 被 stub 消费，不转发。
- 每条管道都是全双工字节流，但每个方向上的消息顺序必须保持不变。

### 2.2 命名管道

#### 2.2.1 管道名

stub 为每个会话生成一个不可预测的会话 ID（推荐至少 128 位随机数，例如 32 位十六进制字符串），并创建：

```text
aip-<sessionId>-client
aip-<sessionId>-backend
```

完整路径为：

```text
\\.\pipe\aip-<sessionId>-client
\\.\pipe\aip-<sessionId>-backend
```

示例：

```text
\\.\pipe\aip-7f3c9a1e2b4d4e8fa6c0d5e1f2a3b4c5-client
\\.\pipe\aip-7f3c9a1e2b4d4e8fa6c0d5e1f2a3b4c5-backend
```

`--aip-pipe-name` 传的是不带前缀的名字：

```text
aip-7f3c9a1e2b4d4e8fa6c0d5e1f2a3b4c5-client
```

#### 2.2.2 创建与连接

1. stub **必须**在启动子进程前，以服务端身份创建两条管道。
2. 每条管道只接受一个连接；第一个连接建立后，stub **应该**关闭监听端，防止第二个进程混入。
3. client/backend 以客户端身份连接自己的管道。
4. 管道使用字节模式（byte mode），不依赖 Windows 的消息模式边界；消息边界由 2.3 节的帧格式保证。
5. 管道必须允许双向读写。stub 对每条管道都有一个读循环和一个写循环，两个方向互不阻塞。
6. 任意一端关闭管道即表示该方向 EOF；未读完整的帧按 2.8 节处理。

#### 2.2.3 安全属性

- stub **应该**用仅允许当前用户访问的 DACL 创建管道。
- stub **应该**设置 `PIPE_REJECT_REMOTE_CLIENTS`，拒绝远程客户端。
- 管道名必须包含足够的随机性，不能使用固定名称。
- 除当前用户外的进程不应能连接或冒充 client/backend。

### 2.3 帧格式

两条管道都使用同一种帧格式：**4 字节小端无符号长度 + UTF-8 TOML 正文**。

```text
+----------------------+---------------------------------------+
| uint32 little-endian | TOML 正文（N 字节，UTF-8，无 BOM）    |
|        N             |                                       |
+----------------------+---------------------------------------+
```

| 字段 | 长度 | 说明 |
| --- | --- | --- |
| 长度 | 4 字节 | 无符号 32 位小端整数，仅表示正文长度，不含这 4 字节本身 |
| 正文 | N 字节 | 恰好一个 TOML 1.0 文档；根必须是表（table） |

规则：

1. `1 <= N <= 1,048,576`（1 MiB）。空帧或超过 1 MiB 的帧都是协议错误。
2. 正文必须是合法 UTF-8，不带 BOM，不带结尾 NUL。
3. 正文是一个 TOML 文档，不是“每行一个消息”。
4. 一帧只承载一条消息。TOML 内部可以有多行、注释和嵌套表，但根表只描述一条消息。
5. 发送方写入一帧时，必须先完整写入 4 字节长度，再写入正文；接收方必须处理长度头和正文被拆成多次读写的情况。
6. stub 转发时**必须**保持正文原始字节不变，不得重新序列化 TOML，也不得改变消息顺序。
7. 长度头可以重新生成，但数值必须与正文长度一致。
8. 不支持用换行、NUL 或其他分隔符代替长度头。

示例：正文为 `type = "startInstall"`（21 字节，不含换行）时，帧开头是：

```text
15 00 00 00
```

随后是 21 字节的 UTF-8 正文。

### 2.4 消息格式

每条消息的正文是一个 TOML 表，必须包含字符串键 `type`：

```toml
type = "setInstallLocation"
value = 'D:\Apps\Example'
```

通用规则：

- `type`：必填，字符串，消息类型名，使用 camelCase，大小写敏感。
- 消息的其余字段直接放在根表，不使用 `payload` 之类的额外包装。
- 未知的键**必须**被忽略，以允许向前兼容的扩展。
- 未知的 `type` **必须**被接收方忽略（stub 只特殊处理 `readyToExit`，其余照常转发）。
- 消息中不允许出现 `null`；TOML 1.0 没有 null。字段“为空”应通过省略或空字符串表达。
- 字段类型错误、必填字段缺失等都属于消息级错误，按 2.8 节处理。
- 不定义序号、时间戳或会话 ID；管道本身已经保证了会话唯一性和消息顺序。

### 2.5 消息定义

| 消息类型 | 方向 | 作用 |
| --- | --- | --- |
| `setInstallLocation` | client → backend | 设置安装位置 |
| `setCreateStartMenuFolder` | client → backend | 设置是否创建开始菜单文件夹 |
| `setCreateDesktopShortcut` | client → backend | 设置是否创建桌面快捷方式 |
| `startInstall` | client → backend | 开始安装 |
| `installFinished` | backend → client | 安装完成（成功或失败） |
| `readyToExit` | client → stub | client 已处理完毕，可以结束会话 |

除 `readyToExit` 外，所有消息都经过 stub 原样转发。

#### 2.5.1 `setInstallLocation`

设置待安装位置。backend 记住最后一次收到的值。

```toml
type = "setInstallLocation"
value = 'D:\Apps\Example'
```

| 字段 | 类型 | 必需 | 说明 |
| --- | --- | --- | --- |
| `value` | string | 是 | 安装目录 |

规则：

- `value` 必须是绝对路径：`C:\...` 形式的本地路径，或 `\\server\share\...` 形式的 UNC 路径。
- 禁止相对路径、空字符串、包含 NUL 的字符串。
- 可以包含 `%VAR%` 环境变量引用，backend 在安装时展开；无法展开的变量视为无效路径。
- 长度不能超过 32,767 个 UTF-16 代码单元（Windows 路径上限）。
- 如果该路径已存在但不是目录，安装应失败并通过 `installFinished` 报告。
- backend 在 `startInstall` 之前收到该消息时，覆盖当前的待安装位置；重复发送时**后发的有效**。
- 如果 client 从未发送该消息，backend 使用 `config.toml` 中 `defaults.installLocation` 的值。
- 收到 `startInstall` 后，该消息不再改变本次安装状态。

#### 2.5.2 `setCreateStartMenuFolder`

设置是否创建开始菜单文件夹。backend 记住最后一次收到的值。

```toml
type = "setCreateStartMenuFolder"
value = true
```

| 字段 | 类型 | 必需 | 说明 |
| --- | --- | --- | --- |
| `value` | boolean | 是 | `true` 创建，`false` 不创建 |

规则：

- 未收到该消息时，使用 `config.toml` 中 `defaults.createStartMenuFolder`。
- 重复发送时**后发的有效**。
- `false` 只表示本次不创建，不表示删除已经存在的文件夹或快捷方式。
- 创建位置为当前用户的开始菜单程序目录：
  `%APPDATA%\Microsoft\Windows\Start Menu\Programs\<name>`
- 开始菜单文件夹内具体放什么快捷方式由 backend 根据 payload 决定。

#### 2.5.3 `setCreateDesktopShortcut`

设置是否创建桌面快捷方式。backend 记住最后一次收到的值。

```toml
type = "setCreateDesktopShortcut"
value = false
```

| 字段 | 类型 | 必需 | 说明 |
| --- | --- | --- | --- |
| `value` | boolean | 是 | `true` 创建，`false` 不创建 |

规则：

- 未收到该消息时，使用 `config.toml` 中 `defaults.createDesktopShortcut`。
- 重复发送时**后发的有效**。
- `false` 只表示本次不创建，不表示删除已经存在的快捷方式。
- 创建位置为当前用户桌面：`%USERPROFILE%\Desktop\<name>.lnk`。
- 快捷方式的目标、参数和工作目录由 backend 根据 payload 决定；图标应优先使用 `config.toml` 中的 `icon`。

#### 2.5.4 `startInstall`

开始安装。它没有额外字段。

```toml
type = "startInstall"
```

规则：

- client **必须**在发送 `startInstall` 前，把当前三个设置值发送给 backend（至少发送被用户修改过的值；推荐全部发送一遍）。
- backend 收到后，使用当前状态执行安装；没有收到过的设置使用 `config.toml` 默认值。
- backend 在开始前**必须**校验所有设置。安装位置无效、payload 无法读取等错误应通过 `installFinished` 报告。
- 开始菜单文件夹和桌面快捷方式只能在 payload 成功写入后创建；安装失败时不得创建指向不存在目标的快捷方式。
- 第一个 `startInstall` 生效；安装已经开始后，后续 `startInstall` 必须被忽略，避免重复安装。
- 安装可以同步或异步执行，但 backend 必须最终发送且只发送一次 `installFinished`。
- 安装期间收到设置消息时，backend **必须**忽略它们，不得让本次安装的选项发生变化。

#### 2.5.5 `installFinished`

backend 在安装尝试结束后发送的完成消息，是 client 判断安装结果的唯一依据。

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

规则：

- 每个会话最多发送一次；只要 backend 处理了 `startInstall`，就必须发送一次，无论成功还是失败。
- `success = false` 时，`error` **必须**提供，且应是可读的 UTF-8 文本，不得包含 NUL。
- `success = true` 时，`error` 即使存在也应被 client 忽略。
- client 收到后应显示对应结果；失败时展示 `error`，缺失时显示通用错误。
- 该消息不表示 backend 进程即将退出；backend 可以继续连接，等待 stub 结束会话。
- backend 在发送该消息后可以自行退出；这属于正常行为，stub 不得因此提前结束 client。

#### 2.5.6 `readyToExit`

client 在完成所有界面工作后发送的可结束消息。

```toml
type = "readyToExit"
```

规则：

- 方向固定为 client → stub，stub **必须**消费它，不转发给 backend。
- stub 只从 client 管道识别该消息；如果 backend 管道上出现同名消息，stub **必须**忽略，不得因此关闭会话。
- client **必须**在收到 `installFinished` 并完成结果展示、必要的安装后操作后再发送。
- client 只应发送一次。stub 收到第一个 `readyToExit` 后立即进入关闭流程，后续重复消息无需处理。
- 发送后 client 可以自行退出；stub 也会结束它。
- 如果 client 在未发送 `readyToExit` 的情况下断开，stub 视为会话取消并清理（见 2.8 节）。

### 2.6 状态与“后发有效”

backend 在内存中维护本次会话的状态：

| 状态字段 | 初始值 | 更新消息 | 冻结时机 |
| --- | --- | --- | --- |
| 安装位置 | `defaults.installLocation` | `setInstallLocation` | 处理 `startInstall` 时 |
| 创建开始菜单文件夹 | `defaults.createStartMenuFolder` | `setCreateStartMenuFolder` | 处理 `startInstall` 时 |
| 创建桌面快捷方式 | `defaults.createDesktopShortcut` | `setCreateDesktopShortcut` | 处理 `startInstall` 时 |
| 已开始 | `false` | `startInstall` | 第一次生效后 |
| 已完成 | `false` | 发送 `installFinished` 后 | 本会话内保持不变 |

“后发有效”的准确含义：

1. 对三个 `set*` 消息，backend 每收到一个合法消息就覆盖对应字段，不按消息去重，也不比较新旧值；无效消息按第 6 条处理。
2. 最终生效的是 `startInstall` 处理前最后收到的值。
3. 管道保证顺序，因此不需要序号；如果同一设置连续收到两次，第二次覆盖第一次。
4. `startInstall` 不是“后发有效”：第一个生效，之后的被忽略。
5. `readyToExit` 也是第一个生效。
6. 在 `startInstall` 之前，无效的 `set*` 消息也参与顺序：它把对应字段标记为无效；之后若收到同一字段的合法消息，则以新消息为准并清除无效标记。
7. 状态只存在于本次会话内，进程退出后不保留；下次安装重新从 `config.toml` 初始化。

### 2.7 正常关闭

1. client 发送 `readyToExit`。
2. stub 停止转发新消息，关闭两条管道的读写端。
3. stub 通知 client 和 backend 退出（关闭管道句柄即可；必要时可以发送窗口关闭消息）。
4. stub 等待最多 5 秒；仍未退出的进程由 stub 强制结束。
5. stub 清理自己创建的临时目录、释放的文件和管道实例。
6. stub 退出自身。若 stub 是从临时位置运行的，可以安排退出后删除自身文件（可选）。

stub **不得**删除用户选择的安装目录或用户已有的快捷方式；它只清理本次会话的临时资源。

### 2.8 异常处理

| 场景 | 处理方式 |
| --- | --- |
| `config.toml` 缺失、语法错误、协议号不支持或必填项缺失 | stub 显示可读错误并退出；不创建管道、不启动 client/backend |
| 子进程 30 秒内未连接管道 | stub 关闭会话，结束已启动的进程并退出 |
| 帧长度非法、正文不是 UTF-8、TOML 非法、根不是表、缺少字符串 `type` | stub 视为致命协议错误，结束整个会话 |
| 单帧超过 1 MiB | 同致命协议错误 |
| 消息字段类型错误或必填字段缺失 | 若属于三个 `set*` 之一，backend 把对应字段标记为无效并记录错误；同一字段之后收到合法消息时，以新消息为准并清除无效标记。处理 `startInstall` 时若仍有无效字段，以 `installFinished success = false` 报告。client 收到无法解析的 `installFinished` 时按失败处理 |
| 未知消息类型或未知字段 | 接收方忽略；stub 对除 `readyToExit` 外的未知类型照常转发 |
| backend 在发送 `installFinished` 前退出或断开 | stub 关闭 client 管道；client 将 EOF 视为安装失败。stub 清理后退出 |
| backend 在发送 `installFinished` 后退出 | 正常；stub 保持 client 管道直到 `readyToExit` 或 client 断开 |
| client 在未发送 `readyToExit` 时退出 | stub 视为取消：关闭 backend 管道，等待 5 秒后强制结束 backend，清理并退出。backend 应使用暂存目录，尽量避免半安装 |
| client 在 `installFinished` 前发送 `readyToExit` | stub 仍按协议执行关闭；这属于 client 违约，backend 可能被中途终止 |
| 安装失败 | backend 发送 `installFinished success = false`；client 展示错误后仍应发送 `readyToExit` |
| `startInstall` 重复 | 只有第一次生效；后续忽略 |
| `readyToExit` 重复 | 只有第一次生效；stub 已进入关闭流程 |

stub 不负责伪造 `installFinished`。如果 backend 未发送完成消息就断开，client 应通过管道 EOF 或 stub 的关闭行为判断失败。

### 2.9 安全

- 所有管道仅限本机、当前用户访问。
- 管道名必须随机且不可预测，防止其他进程抢占连接。
- 单帧大小限制为 1 MiB，避免内存耗尽。
- backend 解压 ZIP 时必须拒绝路径穿越（`..`）、绝对路径、盘符路径和指向安装目录之外的符号链接。
- 安装位置必须经过规范化和校验；展开环境变量后必须是绝对路径。
- stub 不应无必要地提升权限。快捷方式默认写入当前用户目录，不需要管理员权限。
- 本协议不提供加密或认证；安全边界依赖 Windows 命名管道的访问控制和本机进程信任。

---

## 3. 配置格式

### 3.1 文件与语法

- 文件名固定为 `config.toml`，放在包根目录。
- 编码为 UTF-8，不带 BOM。
- 语法为 TOML 1.0。
- 键名大小写敏感。
- 允许注释。
- 未知的键**必须**被忽略，以便向前兼容；打包器可以额外给出警告。
- **TOML 没有 `null`。** 本文所说的“null”只表示：省略该键，或把它写成空字符串 `""`。不要写 `icon = null` 或 `background = null`，这不是合法 TOML。
- Windows 路径推荐使用 TOML 单引号字面量字符串，避免反斜杠转义：

```toml
installLocation = 'C:\Program Files\Example'
```

如果使用双引号字符串，则必须写成：

```toml
installLocation = "C:\\Program Files\\Example"
```

- 路径中的 `%VAR%` 环境变量引用由读取方展开；规则见各字段说明。

### 3.2 结构

```toml
protocol = 1
name = "示例应用"
icon = "app.ico"           # 可选
background = "acrylic"      # 可选

[defaults]
installLocation = '%LocalAppData%\Programs\示例应用'
createStartMenuFolder = true
createDesktopShortcut = true
```

### 3.3 字段说明

| 键 | 类型 | 必需 | 默认值 | 说明 |
| --- | --- | --- | --- | --- |
| `protocol` | integer | 是 | 无 | AIP 协议号，当前必须为 `1` |
| `name` | string | 是 | 无 | 应用显示名称 |
| `icon` | string | 否 | Anyinstaller 默认图标 | 图标文件路径 |
| `background` | string | 否 | 白色 / 包内背景图 | 客户端背景 |
| `defaults.installLocation` | string | 是 | 无 | 默认安装位置 |
| `defaults.createStartMenuFolder` | boolean | 是 | 无 | 默认是否创建开始菜单文件夹 |
| `defaults.createDesktopShortcut` | boolean | 是 | 无 | 默认是否创建桌面快捷方式 |

#### 3.3.1 `protocol`

```toml
protocol = 1
```

- 必须是整数，不能写成字符串 `"1"`。
- 当前规范只接受 `1`。
- stub 发现不支持的协议号时**必须**拒绝运行，并向用户显示可读的错误。
- 该字段表示 AIP 协议版本，不是应用自身的版本号。

#### 3.3.2 `name`

```toml
name = "示例应用"
```

- 必填，非空字符串。
- 建议长度 1–128 个字符。
- 用作 client 窗口标题、开始菜单文件夹名和快捷方式名。
- 不得包含控制字符。
- 如果包含 Windows 文件名非法字符（`\ / : * ? " < > |`），backend 在创建文件/文件夹时应替换为 `_`，并去掉结尾的点和空格。
- `name` 不会自动参与 `defaults.installLocation` 的计算；默认安装位置由该字段单独给出。

#### 3.3.3 `icon`

```toml
icon = 'assets\app.ico'
```

- 可选。省略或写 `icon = ""` 表示使用 Anyinstaller 默认图标。
- 推荐使用 Windows `.ico` 文件；client 可以额外支持 `.png`，但为保证快捷方式图标可用，`.ico` 兼容性最好。
- 可以是绝对路径，也可以是相对于包根目录的路径。
- 可以包含 `%VAR%` 环境变量引用。
- 文件不存在、无法展开变量、格式无法读取或路径非法时，回退到 Anyinstaller 默认图标，不中断安装。
- client 应将其用于窗口/任务栏图标；backend 在创建快捷方式时**应该**使用同一图标。

#### 3.3.4 `background`

可选，控制 client 的背景外观。支持图片路径、视觉效果和纯色，详见 [3.4](#34-background)。

#### 3.3.5 `defaults.installLocation`

```toml
[defaults]
installLocation = '%LocalAppData%\Programs\Example App'
```

- 必填，字符串。
- 必须是绝对路径：本地盘符路径或 UNC 路径。
- 可以包含 `%VAR%` 环境变量引用，例如 `%ProgramFiles%`、`%LocalAppData%`。
- 展开环境变量后必须仍是绝对路径；相对路径无效。
- 无法展开的变量视为配置错误，stub 应拒绝运行。
- 长度不能超过 32,767 个 UTF-16 代码单元。
- client 启动时用它初始化安装位置输入框；backend 用它初始化会话状态。
- 如果安装位置不存在，backend 应递归创建。
- 如果没有足够的写入权限，由 backend 通过 `installFinished` 报告失败。

#### 3.3.6 `defaults.createStartMenuFolder`

```toml
createStartMenuFolder = true
```

- 必填，布尔值。
- 作为 client 初始选项；如果用户不修改，`startInstall` 前 client 会把这个值发给 backend。
- backend 在未收到对应设置消息时也使用它。
- 具体创建位置见 2.5.2。

#### 3.3.7 `defaults.createDesktopShortcut`

```toml
createDesktopShortcut = true
```

- 必填，布尔值。
- 作为 client 初始选项；如果用户不修改，`startInstall` 前 client 会把这个值发给 backend。
- backend 在未收到对应设置消息时也使用它。
- 具体创建位置见 2.5.3。

### 3.4 background

`background` 控制 client 的背景，不参与安装。它可以取以下值：

| 值 | 含义 |
| --- | --- |
| 图片路径 | 使用用户本机或包内的图片作为背景 |
| `"acrylic"` | Windows 亚克力材质 |
| `"acrylicBlur"` | 亚克力并加强背景模糊；模糊程度由 client 决定 |
| `"mica"` | Windows 云母材质（Mica） |
| `"windowsManaged"` | 交由 Windows/DWM 管理系统背景，并跟随系统主题 |
| `"transparent"` | 完全透明背景 |
| `"#RRGGBB"` | 纯色背景，6 位十六进制，例如 `#FFFFFF`、`#1E90FF` |

特殊值匹配不区分大小写，推荐使用表中的拼写。一个值如果与特殊值匹配，就按特殊值处理；否则按图片路径处理。若要引用一个恰好叫 `mica` 的文件，请写成 `.\mica` 之类的显式相对路径。

颜色必须是 6 位十六进制 `#RRGGBB`（不区分大小写）；不接受 3 位缩写，也不接受带 alpha 的 8 位形式。

图片路径规则：

- 可以是绝对路径，也可以是相对于包根目录的路径。
- 可以包含 `%VAR%` 环境变量引用；无法展开的变量视为图片路径无效。
- 支持格式：`.png`、`.webp`、`.avif`；client 可以额外支持 `.jpg`、`.jpeg`。
- client **应该**按“覆盖窗口、保持比例、不拉伸变形”的方式绘制背景。
- 背景为图片或透明时，client **必须**保证文字可读，必要时自行叠加半透明遮罩。

选择顺序：

1. 如果 `background` 存在且非空，按上面的值解析。
2. 如果 `background` 省略、为空，或者图片路径无效，则查找包根目录的背景文件，优先级：
   `background.webp` → `background.avif` → `background.png`。
3. 如果以上都不存在或无法加载，使用白色 `#FFFFFF`。

因此：

- 不写 `background`，包内也没有 `background.*` 时，默认是白色。
- 不写 `background`，但包内有 `background.webp` 时，使用该图。
- 想让包内背景图失效、强制白色，写 `background = "#FFFFFF"`。
- `background = ""` 等同于省略，会继续尝试包内背景图。
- 图片路径存在但无法读取或无法解码时，回退到第 2 步（包内背景图），再回退白色。
- 特殊材质在当前系统不支持时，回退到白色；client 也可以用近似纯色代替，但不得崩溃。
- 纯色总是可用，不参与回退。

### 3.5 校验与回退

**致命错误**（stub 必须拒绝运行，不启动 client/backend）：

- `config.toml` 不存在、无法读取、不是合法 TOML。
- `protocol` 缺失、不是整数或不是 `1`。
- `name` 缺失、为空或类型错误。
- `[defaults]` 表缺失（或其中的三个必需字段缺失、类型错误）。
- `defaults.installLocation` 缺失、为空、包含无法展开的变量、展开后不是绝对路径。
- `defaults.createStartMenuFolder` / `defaults.createDesktopShortcut` 不是布尔值。

**可回退问题**（记录警告并继续）：

- `icon` 路径不存在、无法读取或格式不支持 → 使用 Anyinstaller 默认图标。
- `background` 值无法识别、图片不存在或无法解码 → 按 3.4 的顺序回退。
- 未知的键 → 忽略。
- `name` 中的非法文件名字符 → backend 创建文件时替换。

stub 是配置校验的守门人。client 和 backend 也可以自行读取并校验配置，但不应在 stub 已拒绝的情况下被启动。

### 3.6 完整示例

仓库内的 [`examples/config.toml`](examples/config.toml) 是一个带注释的可直接使用示例。下面给出同一结构的简版：

```toml
# AIP 1.0 示例
protocol = 1
name = "示例应用"

# 省略 icon 或写 "" 使用 Anyinstaller 默认图标
icon = 'assets\app.ico'

# 省略 background 时优先使用包内 background.webp/.avif/.png，否则白色
background = "acrylic"

[defaults]
installLocation = '%LocalAppData%\Programs\示例应用'
createStartMenuFolder = true
createDesktopShortcut = true
```

只使用默认白色背景：

```toml
protocol = 1
name = "示例应用"

[defaults]
installLocation = 'C:\Program Files\示例应用'
createStartMenuFolder = false
createDesktopShortcut = false
```

使用用户本机图片：

```toml
background = 'C:\Users\Ali\Pictures\installer-bg.avif'
```

使用纯色：

```toml
background = "#1E90FF"
```

---

## 4. 版本与兼容性

- 本文描述的是 **AIP 1.0**，对应 `config.toml` 中的 `protocol = 1`。
- `protocol` 是整数主版本号；stub、client、backend 都必须能识别它。
- 不兼容的修改（删除/重命名字段、改变已有消息语义、改变帧格式）必须提升协议号。
- 兼容的扩展（新增可选配置键、新增消息类型、新增可选消息字段）可以在同一协议号下进行；接收方必须忽略未知键和未知消息类型。
- client 和 backend 可以定义自己的扩展消息，但必须保证：
  - 不改变本规范定义消息的语义；
  - 对端忽略未知类型时不破坏标准流程；
  - 不依赖 stub 解释这些扩展消息。
- AIP v1 不定义安装进度消息。client 可以显示不确定进度条，或通过扩展消息自行实现进度显示。

一个实现要声明自己符合 AIP v1，至少应满足：

- stub：能校验配置、创建两条管道、传递启动参数、双向转发帧、处理 `readyToExit` 并清理进程。
- client：能显示配置、发送三个 `set*` 和 `startInstall`、处理 `installFinished`、发送 `readyToExit`。
- backend：能从默认值初始化状态、实现“后发有效”、执行安装、发送一次 `installFinished`。
- 配置：能读取并校验 `config.toml` 的必填字段，能处理可选字段的回退。

---

## 5. 完整交互示例

下面是一次成功安装的完整消息序列。每条代码块都是一帧的 TOML 正文（不显示 4 字节长度头）。

1. client 第一次选择安装位置：

```toml
type = "setInstallLocation"
value = 'D:\Apps\Example'
```

2. 用户又改了一次位置；backend 采用后发的值：

```toml
type = "setInstallLocation"
value = 'D:\Apps\Example App'
```

3. 创建开始菜单文件夹：

```toml
type = "setCreateStartMenuFolder"
value = true
```

4. 不创建桌面快捷方式：

```toml
type = "setCreateDesktopShortcut"
value = false
```

5. 开始安装：

```toml
type = "startInstall"
```

6. backend 完成安装：

```toml
type = "installFinished"
success = true
```

7. client 展示完成页，然后请求结束：

```toml
type = "readyToExit"
```

转发关系：

```text
1–5  client → stub → backend
6    backend → stub → client
7    client → stub（stub 消费，不转发）
```

最终 backend 的安装位置是 `D:\Apps\Example App`，创建开始菜单文件夹，不创建桌面快捷方式。

失败时，第 6 帧改为：

```toml
type = "installFinished"
success = false
error = "无法写入安装目录：拒绝访问"
```

client 展示错误后，仍然发送第 7 帧 `readyToExit`，流程与成功时相同。

---

**License：** [MIT](LICENSE)

# 单文件包结构

> 中文 | [English](../../en/overview/package.md) · [索引](../index.md) · [首页](../../../README.md)

## 发布物

AIP v1 的发布物是一个单独的 `foo.exe`。从外部看只有一个文件；逻辑上它包含：

```text
foo.exe
└─ 内嵌内容
   ├─ stub.exe        # 入口存根
   ├─ client.exe      # 安装界面
   ├─ backend.exe     # 安装执行器
   ├─ config.toml     # 配置
   ├─ payload.zip     # 应用本体
   └─ 其他可选文件    # 背景、图标、运行库、许可证等
```

这就是 `foo.exe:(stub.exe, client.exe, backend.exe, config.toml, payload.zip, ...)` 的含义。

| 内嵌组件 | 必需 | 说明 |
| --- | --- | --- |
| `stub.exe` | 是 | `foo.exe` 的入口；负责校验、释放、启动、中继和清理 |
| `client.exe` | 是 | 独立进程；角色由可执行文件本身确定 |
| `backend.exe` | 是 | 独立进程；角色由可执行文件本身确定 |
| `config.toml` | 是 | 逻辑文件名固定；打包后为运行时可读配置 |
| `payload.zip` | 是 | 逻辑文件名固定；backend 解压到安装位置 |
| 可选资源 | 否 | 例如 `assets/background.webp`、`assets/icon.ico`、运行库 |

## 容器格式

AIP 不规定 `foo.exe` 的具体容器格式。打包器可以把 stub 作为 PE 主体，把其余文件作为资源、附加数据或内嵌归档；也可以使用一个小型引导器。只要满足以下条件：

1. 最终只有一个 `foo.exe`，没有 sidecar 文件。
2. 所有运行必需的文件都在 `foo.exe` 内，包括 client、backend、config、payload 和依赖库。
3. stub 能在启动时验证签名和内嵌组件完整性。
4. stub 能把内嵌组件释放到私有会话目录，并让 client/backend 以该目录为当前工作目录。

`stub.exe`、`client.exe`、`backend.exe` 是逻辑名称。容器可以重命名它们，但必须保证：

- 入口是 stub；
- client 和 backend 是两个独立进程；
- 角色由可执行文件身份确定，**不通过命令行参数区分**；
- `config.toml` 和 `payload.zip` 在运行时以固定逻辑名出现在包根目录。

## 运行时释放

`foo.exe` 启动后，stub 按以下顺序准备运行环境：

1. 用 `WinVerifyTrust` 校验 `foo.exe` 的 Authenticode 签名。
2. 校验内嵌组件清单中的 SHA-256 哈希。
3. 在 `%LOCALAPPDATA%\Anyinstaller\Sessions\<随机ID>` 创建私有会话目录。
4. 把 `stub.exe` 以外的必需组件释放到会话目录：
   - `client.exe`
   - `backend.exe`
   - `config.toml`
   - `payload.zip`
   - 可选资源（保持包内相对路径，例如 `assets/background.webp`）
5. 把会话目录设置为只读（stub 自己保留清理权限）。
6. 创建两条命名管道，启动 client 和 backend，当前工作目录设为会话目录。

会话目录必须位于当前用户配置目录下，并使用仅当前用户和 SYSTEM 可访问的 DACL；不得使用 `%TEMP%` 根目录、公共目录或可预测路径。具体安全要求见 [强制安全要求](../security/requirements.md)。

## 资源路径重写

源 `config.toml` 中的图片路径是**打包者本机路径**：

```toml
background = 'C:\Users\Ali\Pictures\background.avif'
icon = 'C:\Users\Ali\Pictures\app.ico'
```

打包器必须：

1. 读取并验证图片；
2. 把图片嵌入 `foo.exe`，通常放在 `assets/` 下；
3. 把打包后的 `config.toml` 重写为包内相对路径：

```toml
background = "assets/background.avif"
icon = "assets/icon.ico"
```

运行时 client 只接受包内相对路径，并且只从会话目录读取，不得回读 `C:\Users\...` 等本机路径。特殊背景值（`acrylic`、`mica`、颜色等）不受影响。

如果资源缺失、无法解码或路径非法，打包器必须拒绝生成发布包；运行时发现非法包内路径时，client 必须回退白色背景或默认图标，并记录错误。详见 [背景](../configuration/background.md) 和 [打包、签名与完整性](../security/packaging.md)。

## 包根目录

运行时，client/backend 看到的包根目录至少包含：

```text
<session>\
├─ client.exe
├─ backend.exe
├─ config.toml
├─ payload.zip
└─ assets\
   ├─ background.avif   # 可选
   └─ icon.ico          # 可选
```

stub 必须把 client 和 backend 的当前工作目录设置为 `<session>`。client/backend 必须只在该目录内解析相对资源路径。

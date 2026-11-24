# 文件与语法

> 中文 | [English](../../en/configuration/file.md) · [索引](../index.md) · [首页](../../../README.md)

## 两个阶段

`config.toml` 有两个形态：

| 阶段 | 位置 | 图片路径 |
| --- | --- | --- |
| **源配置** | 打包者本机 | `icon` / `background` 可以是本机文件路径 |
| **打包后配置** | 内嵌在 `foo.exe` 中 | 只能是包内相对路径，例如 `assets/background.avif` |

打包器必须把源配置中的本机图片嵌入 `foo.exe`，并把打包后配置中的路径重写为包内相对路径。运行时 client/backend 只能读取包内资源，不得回读用户本机路径。

## 语法

- 文件名固定为 `config.toml`。
- 编码 UTF-8，不带 BOM。
- TOML 1.0。
- 键名大小写敏感。
- 允许注释。
- 最大 64 KiB。
- 已知键必须严格校验类型；未知键必须忽略，以支持向前兼容。
- **TOML 没有 `null`。** 本文中的“null”只表示省略该键，或写成空字符串 `""`。不要写 `icon = null` 或 `background = null`，这不是合法 TOML。
- Windows 路径推荐使用 TOML 单引号字面量：

```toml
installLocation = 'C:\Program Files\Example'
```

双引号字符串必须转义反斜杠：

```toml
installLocation = "C:\\Program Files\\Example"
```

## 基本结构

```toml
protocol = 1
name = "示例应用"
icon = 'C:\Users\Ali\Pictures\app.ico'          # 源配置：本机路径
background = 'C:\Users\Ali\Pictures\bg.avif'    # 源配置：本机路径

[defaults]
installLocation = '%LocalAppData%\Programs\示例应用'
createStartMenuFolder = true
createDesktopShortcut = true
```

打包后：

```toml
protocol = 1
name = "示例应用"
icon = "assets/icon.ico"
background = "assets/background.avif"

[defaults]
installLocation = '%LocalAppData%\Programs\示例应用'
createStartMenuFolder = true
createDesktopShortcut = true
```

## 路径规则

### 源配置

- `icon`、`background`：本机图片路径，可以是绝对路径，也可以是相对于源配置文件的路径。
- 打包器必须解析、验证、嵌入并重写这些路径。
- `defaults.installLocation` 可以包含 `%VAR%` 环境变量；打包器不改变它。

### 打包后配置

- `icon`、`background` 的图片路径必须是包内相对路径，不得包含盘符、UNC、`..`、设备前缀或 NTFS ADS。
- client/backend 必须以会话包根目录为基准解析，并拒绝重解析点或越界路径。
- `defaults.installLocation` 仍是安装路径模板；client/backend 展开 `%VAR%` 后校验。
- 运行时不得读取 `C:\Users\...`、网络共享或其他包外文件作为背景或图标。

## 校验

**致命错误**（stub 必须拒绝运行）：

- 文件缺失、无法读取、不是合法 TOML、超过 64 KiB。
- `protocol` 缺失、不是整数或不是 `1`。
- `name` 缺失、为空或类型错误。
- `[defaults]` 表缺失，或三个必需字段缺失/类型错误。
- `defaults.installLocation` 为空、包含无法展开的变量或展开后不是合法本地绝对路径。

**可回退问题**（记录错误并继续）：

- `icon` 路径无效或无法解码 → 使用 Anyinstaller 默认图标。
- `background` 路径无效或无法解码 → 使用白色背景。
- 未知键 → 忽略。
- `name` 中的非法文件名字符 → backend 创建文件时替换。

图片和图标路径的安全限制见 [背景](background.md) 和 [强制安全要求](../security/requirements.md)。

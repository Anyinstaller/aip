# 字段说明

> 中文 | [English](../../en/configuration/fields.md) · [索引](../index.md) · [首页](../../../README.md)

## 结构

```toml
protocol = 1
name = "示例应用"
icon = "assets/icon.ico"          # 可选
background = "assets/bg.avif"     # 可选

[defaults]
installLocation = '%LocalAppData%\Programs\示例应用'
createStartMenuFolder = true
createDesktopShortcut = true
```

| 键 | 类型 | 必需 | 默认值 | 说明 |
| --- | --- | --- | --- | --- |
| `protocol` | integer | 是 | 无 | AIP 协议号，当前必须为 `1` |
| `name` | string | 是 | 无 | 应用显示名称 |
| `icon` | string | 否 | Anyinstaller 默认图标 | 图标路径；源配置为本机路径，打包后为包内路径 |
| `background` | string | 否 | 白色 | 背景；图片路径或视觉效果/颜色 |
| `defaults.installLocation` | string | 是 | 无 | 默认安装位置 |
| `defaults.createStartMenuFolder` | boolean | 是 | 无 | 默认是否创建开始菜单文件夹 |
| `defaults.createDesktopShortcut` | boolean | 是 | 无 | 默认是否创建桌面快捷方式 |

## protocol

```toml
protocol = 1
```

- 必须是整数，不能是字符串 `"1"`。
- 当前只接受 `1`。
- 不支持时 stub 必须拒绝运行，并显示可读错误。
- 这是 AIP 协议版本，不是应用版本。

## name

```toml
name = "示例应用"
```

- 必填，非空字符串，长度 1–128 个字符。
- 用作 client 窗口标题、开始菜单文件夹名和快捷方式名。
- 不得包含控制字符。
- 包含 Windows 文件名非法字符（`\ / : * ? " < > |`）时，backend 在创建文件/文件夹时必须替换为 `_`，并去掉结尾的点和空格。
- `name` 不参与 `defaults.installLocation` 的计算；默认安装位置单独给出。

## icon

```toml
icon = 'C:\Users\Ali\Pictures\app.ico'   # 源配置
icon = "assets/icon.ico"                  # 打包后
```

- 可选。省略或 `icon = ""` 表示 Anyinstaller 默认图标。
- 推荐 `.ico`；client 可以支持 `.png`，但快捷方式图标最好由打包器转换为 `.ico`。
- 源配置中可以是本机绝对路径或相对于源配置的路径；打包器必须嵌入并重写为包内相对路径。
- 打包后只能使用包内相对路径，不得包含盘符、UNC、`..` 或设备前缀。
- 文件不存在、无法展开、格式错误或路径非法时，回退默认图标，不中断安装。
- client 用于窗口/任务栏图标；backend 创建快捷方式时应使用同一图标。
- 图标文件最大 64 MiB，解码后最大 256 百万像素；超过则按无效处理。

## background

控制 client 背景，不参与安装。支持图片路径、视觉效果和纯色，详见 [背景](background.md)。

## defaults.installLocation

```toml
[defaults]
installLocation = '%LocalAppData%\Programs\Example App'
```

- 必填，字符串。
- 必须是本地绝对路径（盘符路径），不接受 UNC、网络路径或设备路径。
- 可以包含 `%VAR%` 环境变量引用；展开后必须仍是本地绝对路径。
- 无法展开的变量视为配置错误，stub 拒绝运行。
- 长度不超过 32,767 个 UTF-16 代码单元。
- client 用它初始化安装位置输入框；backend 用它初始化会话状态。
- 安装位置的安全校验（受保护目录、重解析点、暂存目录等）见 [强制安全要求](../security/requirements.md)。

## defaults.createStartMenuFolder

```toml
createStartMenuFolder = true
```

- 必填，布尔值。
- client 的初始选项；用户不修改时，client 在 `startInstall` 前发送该值。
- backend 未收到对应消息时也使用它。
- 创建位置为当前用户的 `FOLDERID_Programs`。

## defaults.createDesktopShortcut

```toml
createDesktopShortcut = true
```

- 必填，布尔值。
- client 的初始选项；用户不修改时，client 在 `startInstall` 前发送该值。
- backend 未收到对应消息时也使用它。
- 创建位置为当前用户的 `FOLDERID_Desktop`。

## 示例

完整源配置见 [`examples/config.toml`](../../../examples/config.toml)，打包后配置见 [`examples/config.packaged.toml`](../../../examples/config.packaged.toml)。

```toml
protocol = 1
name = "示例应用"
icon = "assets/icon.ico"
background = "acrylic"

[defaults]
installLocation = '%LocalAppData%\Programs\示例应用'
createStartMenuFolder = true
createDesktopShortcut = true
```

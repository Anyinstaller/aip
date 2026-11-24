# 背景

> 中文 | [English](../../en/configuration/background.md) · [索引](../index.md) · [首页](../../../README.md)

`background` 控制 client 的背景，不参与安装。它可以是本机图片、包内图片、Windows 视觉效果或纯色。

## 取值

| 值 | 含义 |
| --- | --- |
| 图片路径 | 使用图片作为背景 |
| `"acrylic"` | Windows 亚克力材质 |
| `"acrylicBlur"` | 亚克力并加强背景模糊；模糊程度由 client 决定 |
| `"mica"` | Windows 云母材质（Mica） |
| `"windowsManaged"` | 交由 Windows/DWM 管理系统背景，并跟随系统主题 |
| `"transparent"` | 完全透明背景 |
| `"#RRGGBB"` | 纯色，6 位十六进制，例如 `#FFFFFF`、`#1E90FF` |

特殊值匹配不区分大小写，推荐使用表中拼写。一个值如果匹配特殊值，就按特殊值处理；否则按图片路径处理。要引用恰好叫 `mica` 的文件，请写成 `.\mica` 这样的显式相对路径。

颜色必须是 6 位十六进制 `#RRGGBB`；不接受 3 位缩写，也不接受带 alpha 的 8 位形式。

## 图片路径的两个阶段

### 源配置

用户/打包者在本机写：

```toml
background = 'C:\Users\Ali\Pictures\background.avif'
```

可以是绝对路径，也可以是相对于源配置文件的路径。

### 打包

打包器必须：

1. 解析并验证图片（存在、可解码、未超过限制）；
2. 把图片嵌入 `foo.exe`，推荐放在 `assets/background.<ext>`；
3. 把打包后 `config.toml` 重写为包内相对路径：

```toml
background = "assets/background.avif"
```

4. 确保打包后配置中不再出现本机绝对路径、盘符、UNC 或 `..`。

如果图片缺失、无法解码或路径非法，打包器必须拒绝生成发布包。

### 运行时

client 只能接受包内相对路径：

- 以会话包根目录为基准解析；
- 拒绝绝对路径、盘符、UNC、设备前缀、`..`、NTFS ADS 和重解析点；
- 解析后的真实路径必须仍位于包根目录内；
- 不得读取 `C:\Users\...`、网络共享或任何包外文件。

违反以上规则时，client 必须回退白色背景并记录错误，不得尝试读取包外路径。

## 默认值

- `background` 省略、为空字符串或无效时，默认是白色 `#FFFFFF`。
- AIP v1 **不自动**查找固定的 `background.webp/avif/png`。如果打包者想提供默认背景图，必须在打包后 `config.toml` 中写入对应的包内路径。
- 特殊材质在当前系统不支持时，回退白色；client 可以用近似纯色代替，但不得崩溃。
- 纯色总是可用，不参与回退。

## 图片格式与限制

- 支持 `.png`、`.webp`、`.avif`；client 可以额外支持 `.jpg`、`.jpeg`。
- 单个背景文件最大 64 MiB。
- 解码后最大 256 百万像素，单边最大 16,384 像素。
- 超过限制、解码失败或格式不支持时，按无效图片处理，回退白色。
- client 应按“覆盖窗口、保持比例、不拉伸变形”的方式绘制背景。
- 背景为图片或透明时，client 必须保证文字可读，必要时自行叠加半透明遮罩。

## 示例

源配置：

```toml
background = 'C:\Users\Ali\Pictures\installer-bg.avif'
```

打包后：

```toml
background = "assets/background.avif"
```

亚克力：

```toml
background = "acrylic"
```

纯色：

```toml
background = "#1E90FF"
```

默认白色（省略）：

```toml
# background 未设置
```

资源路径和图片解码的强制安全要求见 [强制安全要求](../security/requirements.md)。

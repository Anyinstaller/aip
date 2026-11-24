# Anyinstaller Protocol (AIP)

**AIP v1 · 协议号 / protocol number `1` · 仅 Windows / Windows only**

AIP（Anyinstaller Protocol）定义 Anyinstaller 安装包的组件、单文件包结构、Windows 命名管道通讯、TOML 消息和 `config.toml` 配置格式。最终发布物是一个单独的 `foo.exe`，内部包含 stub、client、backend、`config.toml`、`payload.zip` 和可选资源。

AIP (Anyinstaller Protocol) defines the components, single-file package layout, Windows named-pipe transport, TOML messages, and `config.toml` format for Anyinstaller packages. The release artifact is a single `foo.exe` containing the stub, client, backend, `config.toml`, `payload.zip`, and optional assets.

> 如果喜欢本项目，请 Star [导航页](https://github.com/Anyinstaller/anyinstaller)。
> If you like this project, please star the [navigation page](https://github.com/Anyinstaller/anyinstaller).

## 文档 / Documentation

- **中文**：[AIP 协议文档](docs/zh-CN/index.md)
- **English**：[AIP protocol documentation](docs/en/index.md)

## 快速信息 / Quick facts

- 仅支持 Windows 10/11，使用 Windows 命名管道。  
  Windows 10/11 only; communication uses Windows named pipes.
- client 与 backend 不直接连接，全部消息由 stub 中继。  
  The client and backend never connect directly; the stub relays every message.
- 管道内容为 TOML；帧使用 4 字节小端长度前缀。  
  Pipe frames carry TOML with a 4-byte little-endian length prefix.
- 所有安全要求都是强制项，没有“可选安全项”。  
  Every security requirement is mandatory; there are no optional security features.
- 单文件包、启动参数、资源路径重写和签名要求见文档。  
  See the documentation for the single-file package, startup arguments, asset path rewriting, and signing requirements.

## 许可证 / License

[MIT](LICENSE)

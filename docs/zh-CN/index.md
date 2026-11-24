# AIP 文档

> [English](../en/index.md) · [首页](../../README.md)

AIP（全称 **Anyinstaller Protocol**）是 Anyinstaller 安装包的通讯与配置规范。

**版本：** AIP v1　**协议号：** `1`　**平台：** Windows 10/11

AIP 把安装包拆成三个进程：

- **stub**：启动、管道中继、生命周期和清理；
- **client**：GUI 和用户选择；
- **backend**：真正的安装逻辑。

最终发布物是一个单独的 `foo.exe`。client 与 backend 不直接连接，所有消息都经过 stub。所有安全要求均为强制项，没有可选的安全功能。

## 概览

- [架构与组件](overview/architecture.md)
- [单文件包结构](overview/package.md)
- [生命周期](overview/lifecycle.md)

## 通讯协议

- [管道与中继](communication/transport.md)
- [帧格式](communication/framing.md)
- [消息](communication/messages.md)
- [错误处理](communication/errors.md)

## 配置格式

- [文件与语法](configuration/file.md)
- [字段说明](configuration/fields.md)
- [背景](configuration/background.md)

## 安全

- [强制安全要求](security/requirements.md)
- [打包、签名与完整性](security/packaging.md)

## 其他

- [版本与兼容性](versioning.md)
- 示例：[源配置](../../examples/config.toml) · [打包后配置](../../examples/config.packaged.toml)
- [AIP 首页](../../README.md)

> 安全页中的每一条都是 **必须**（MUST）。不满足任何一条，就不符合 AIP v1。

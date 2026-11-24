# AIP Documentation

> [中文](../zh-CN/index.md) · [Home](../../README.md)

AIP (**Anyinstaller Protocol**) is the communication and configuration protocol for Anyinstaller packages.

**Version:** AIP v1　**Protocol number:** `1`　**Platform:** Windows 10/11

AIP splits an installer into three processes:

- **stub**: launch, pipe relay, lifecycle, and cleanup;
- **client**: GUI and user choices;
- **backend**: the actual installation logic.

The distributed artifact is a single `foo.exe`. The client and backend never connect to each other directly; all messages go through the stub. Every security requirement is mandatory; there are no optional security features.

## Overview

- [Architecture and components](overview/architecture.md)
- [Single-file package](overview/package.md)
- [Lifecycle](overview/lifecycle.md)

## Communication

- [Pipes and relay](communication/transport.md)
- [Frame format](communication/framing.md)
- [Messages](communication/messages.md)
- [Error handling](communication/errors.md)

## Configuration

- [File and syntax](configuration/file.md)
- [Fields](configuration/fields.md)
- [Background](configuration/background.md)

## Security

- [Mandatory security requirements](security/requirements.md)
- [Packaging, signing, and integrity](security/packaging.md)

## Other

- [Versioning and compatibility](versioning.md)
- Examples: [source config](../../examples/config.toml) · [packaged config](../../examples/config.packaged.toml)
- [AIP home](../../README.md)

> Every item on the security pages is **MUST**. Failing any one item means the implementation is not AIP v1 conformant.

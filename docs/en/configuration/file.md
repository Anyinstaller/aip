# File and syntax

> [中文](../../zh-CN/configuration/file.md) | English · [Index](../index.md) · [Home](../../../README.md)

## Two phases

`config.toml` has two forms:

| Phase | Location | Image paths |
| --- | --- | --- |
| **Source config** | On the packager's machine | `icon` / `background` may be local file paths |
| **Packaged config** | Embedded in `foo.exe` | Package-internal relative paths only, e.g. `assets/background.avif` |

The packager must embed source images in `foo.exe` and rewrite the packaged config to package-internal relative paths. At runtime, client/backend may read package assets only and must not read local paths from the user's machine.

## Syntax

- File name is fixed: `config.toml`.
- Encoding: UTF-8, no BOM.
- TOML 1.0.
- Key names are case-sensitive.
- Comments are allowed.
- Maximum size: 64 KiB.
- Known keys must be strictly type-checked; unknown keys must be ignored for forward compatibility.
- **TOML has no `null`.** In this specification, "null" means omit the key or write an empty string `""`. Do not write `icon = null` or `background = null`; that is not valid TOML.
- Windows paths should use TOML literal strings:

```toml
installLocation = 'C:\Program Files\Example'
```

Basic strings must escape backslashes:

```toml
installLocation = "C:\\Program Files\\Example"
```

## Basic structure

```toml
protocol = 1
name = "Example App"
icon = 'C:\Users\Ali\Pictures\app.ico'          # source: local path
background = 'C:\Users\Ali\Pictures\bg.avif'    # source: local path

[defaults]
installLocation = '%LocalAppData%\Programs\Example App'
createStartMenuFolder = true
createDesktopShortcut = true
```

Packaged:

```toml
protocol = 1
name = "Example App"
icon = "assets/icon.ico"
background = "assets/background.avif"

[defaults]
installLocation = '%LocalAppData%\Programs\Example App'
createStartMenuFolder = true
createDesktopShortcut = true
```

## Path rules

### Source config

- `icon`, `background`: local image paths, absolute or relative to the source config file.
- The packager must resolve, validate, embed, and rewrite them.
- `defaults.installLocation` may contain `%VAR%` environment references; the packager leaves it unchanged.

### Packaged config

- `icon`, `background` image paths must be package-internal relative paths, with no drive letter, UNC, `..`, device prefix, or NTFS ADS.
- Client/backend resolve them against the session package root and reject reparse points or escaping paths.
- `defaults.installLocation` remains an install path template; client/backend expand `%VAR%` and validate it.
- At runtime, no component may read `C:\Users\...`, a network share, or any file outside the package as a background or icon.

## Validation

**Fatal errors** (the stub must refuse to run):

- File missing, unreadable, invalid TOML, or over 64 KiB.
- `protocol` missing, not an integer, or not `1`.
- `name` missing, empty, or wrong type.
- `[defaults]` missing, or any of its three required fields missing/wrong type.
- `defaults.installLocation` empty, containing an unresolvable variable, or not a valid local absolute path after expansion.

**Recoverable issues** (log and continue):

- Invalid or undecodable `icon` path → use the Anyinstaller default icon.
- Invalid or undecodable `background` path → use a white background.
- Unknown keys → ignore.
- Invalid filename characters in `name` → the backend replaces them when creating files.

See [Background](background.md) and the [mandatory security requirements](../security/requirements.md) for image path security limits.

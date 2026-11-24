# Fields

> [中文](../../zh-CN/configuration/fields.md) | English · [Index](../index.md) · [Home](../../../README.md)

## Structure

```toml
protocol = 1
name = "Example App"
icon = "assets/icon.ico"          # optional
background = "assets/bg.avif"     # optional

[defaults]
installLocation = '%LocalAppData%\Programs\Example App'
createStartMenuFolder = true
createDesktopShortcut = true
```

| Key | Type | Required | Default | Description |
| --- | --- | --- | --- | --- |
| `protocol` | integer | Yes | None | AIP protocol number; currently must be `1` |
| `name` | string | Yes | None | Application display name |
| `icon` | string | No | Anyinstaller default icon | Icon path; local in source config, internal after packaging |
| `background` | string | No | White | Background image, visual effect, or color |
| `defaults.installLocation` | string | Yes | None | Default install location |
| `defaults.createStartMenuFolder` | boolean | Yes | None | Default for creating a Start Menu folder |
| `defaults.createDesktopShortcut` | boolean | Yes | None | Default for creating a desktop shortcut |

## protocol

```toml
protocol = 1
```

- Must be an integer, not the string `"1"`.
- Only `1` is accepted.
- An unsupported value must make the stub refuse to run and show a readable error.
- This is the AIP protocol version, not the application version.

## name

```toml
name = "Example App"
```

- Required, non-empty string, 1–128 characters.
- Used as the client window title, Start Menu folder name, and shortcut name.
- Must not contain control characters.
- If it contains Windows filename-invalid characters (`\ / : * ? " < > |`), the backend must replace them with `_` when creating files/folders and strip trailing dots and spaces.
- `name` does not participate in `defaults.installLocation`; the default location is given separately.

## icon

```toml
icon = 'C:\Users\Ali\Pictures\app.ico'   # source
icon = "assets/icon.ico"                  # packaged
```

- Optional. Omitted or `icon = ""` means the Anyinstaller default icon.
- `.ico` is recommended; the client may support `.png`, but the packager should convert shortcut icons to `.ico`.
- In source config, it may be a local absolute path or a path relative to the source config; the packager must embed it and rewrite it to a package-internal relative path.
- Packaged config may use only package-internal relative paths, with no drive, UNC, `..`, or device prefix.
- If the file is missing, cannot be expanded, has an invalid format, or has an invalid path, fall back to the default icon without aborting the installation.
- The client uses it for the window/taskbar icon; the backend should use the same icon for shortcuts.
- Maximum icon file size: 64 MiB; maximum decoded size: 256 megapixels. Larger files are treated as invalid.

## background

Controls the client background and does not participate in installation. It supports image paths, visual effects, and solid colors; see [Background](background.md).

## defaults.installLocation

```toml
[defaults]
installLocation = '%LocalAppData%\Programs\Example App'
```

- Required string.
- Must be a local absolute drive path; UNC, network, and device paths are rejected.
- May contain `%VAR%` environment references; after expansion it must still be a local absolute path.
- An unresolvable variable is a config error; the stub refuses to run.
- Maximum length: 32,767 UTF-16 code units.
- The client uses it to initialize the install-location field; the backend uses it to initialize session state.
- Install-location security checks (protected directories, reparse points, staging) are in the [mandatory security requirements](../security/requirements.md).

## defaults.createStartMenuFolder

```toml
createStartMenuFolder = true
```

- Required boolean.
- Initial client choice; if the user does not change it, the client sends this value before `startInstall`.
- The backend also uses it if the corresponding message was never received.
- Created under the current user's `FOLDERID_Programs`.

## defaults.createDesktopShortcut

```toml
createDesktopShortcut = true
```

- Required boolean.
- Initial client choice; if the user does not change it, the client sends this value before `startInstall`.
- The backend also uses it if the corresponding message was never received.
- Created under the current user's `FOLDERID_Desktop`.

## Example

See [`examples/config.toml`](../../../examples/config.toml) for a complete source config and [`examples/config.packaged.toml`](../../../examples/config.packaged.toml) for the packaged form.

```toml
protocol = 1
name = "Example App"
icon = "assets/icon.ico"
background = "acrylic"

[defaults]
installLocation = '%LocalAppData%\Programs\Example App'
createStartMenuFolder = true
createDesktopShortcut = true
```

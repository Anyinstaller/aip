# Background

> [中文](../../zh-CN/configuration/background.md) | English · [Index](../index.md) · [Home](../../../README.md)

`background` controls the client background and does not participate in installation. It may be a local image, a packaged image, a Windows visual effect, or a solid color.

## Values

| Value | Meaning |
| --- | --- |
| Image path | Use an image as the background |
| `"acrylic"` | Windows Acrylic material |
| `"acrylicBlur"` | Acrylic with stronger background blur; the client decides the blur amount |
| `"mica"` | Windows Mica material |
| `"windowsManaged"` | Let Windows/DWM manage the system backdrop and follow the system theme |
| `"transparent"` | Fully transparent background |
| `"#RRGGBB"` | Solid color, 6-digit hex, e.g. `#FFFFFF`, `#1E90FF` |

Special values are case-insensitive; the spellings above are recommended. If a value matches a special value, it is treated as that special value; otherwise it is treated as an image path. To reference a file literally named `mica`, use an explicit relative path such as `.\mica`.

Colors must be 6-digit hex `#RRGGBB`; 3-digit shorthand and 8-digit alpha forms are not accepted.

## Two phases for image paths

### Source config

The user/packager writes a local path:

```toml
background = 'C:\Users\Ali\Pictures\background.avif'
```

It may be absolute or relative to the source config file.

### Packaging

The packager must:

1. Resolve and validate the image (exists, decodable, within limits);
2. Embed it in `foo.exe`, preferably at `assets/background.<ext>`;
3. Rewrite the packaged `config.toml` to a package-internal relative path:

```toml
background = "assets/background.avif"
```

4. Ensure the packaged config contains no local absolute path, drive letter, UNC path, or `..`.

If the image is missing, cannot be decoded, or has an invalid path, the packager must refuse to produce the release.

### Runtime

The client may accept only package-internal relative paths:

- resolve against the session package root;
- reject absolute paths, drive letters, UNC, device prefixes, `..`, NTFS ADS, and reparse points;
- the resolved real path must remain inside the package root;
- must not read `C:\Users\...`, a network share, or any file outside the package.

If any rule is violated, the client must fall back to a white background and log an error; it must not attempt to read an external path.

## Defaults

- If `background` is omitted, empty, or invalid, the default is white `#FFFFFF`.
- AIP v1 does **not** automatically look for a fixed `background.webp/avif/png`. If the packager wants a default image, it must write the corresponding package-internal path into the packaged `config.toml`.
- If a special material is unsupported on the current system, fall back to white; the client may use an approximate solid color but must not crash.
- Solid colors always work and do not participate in fallback.

## Image formats and limits

- Supported: `.png`, `.webp`, `.avif`; the client may also support `.jpg`, `.jpeg`.
- Maximum background file size: 64 MiB.
- Maximum decoded size: 256 megapixels, maximum 16,384 pixels per side.
- Files over the limits, undecodable files, or unsupported formats are treated as invalid and fall back to white.
- The client should draw the background with "cover, preserve aspect ratio, no distortion".
- For images or transparency, the client must keep text readable and may add a semi-transparent scrim.

## Examples

Source config:

```toml
background = 'C:\Users\Ali\Pictures\installer-bg.avif'
```

Packaged:

```toml
background = "assets/background.avif"
```

Acrylic:

```toml
background = "acrylic"
```

Solid color:

```toml
background = "#1E90FF"
```

Default white (omitted):

```toml
# background is not set
```

See the [mandatory security requirements](../security/requirements.md) for asset path and image decoding requirements.

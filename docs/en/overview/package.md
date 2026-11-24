# Single-file package

> [中文](../../zh-CN/overview/package.md) | English · [Index](../index.md) · [Home](../../../README.md)

## Release artifact

The AIP v1 release artifact is a single `foo.exe`. From the outside there is only one file; logically it contains:

```text
foo.exe
└─ embedded content
   ├─ stub.exe        # entry stub
   ├─ client.exe      # installer UI
   ├─ backend.exe     # installer engine
   ├─ config.toml     # configuration
   ├─ payload.zip     # application payload
   └─ other optional files  # background, icon, runtime libraries, licenses, etc.
```

This is what `foo.exe:(stub.exe, client.exe, backend.exe, config.toml, payload.zip, ...)` means.

| Embedded component | Required | Description |
| --- | --- | --- |
| `stub.exe` | Yes | Entry point of `foo.exe`; validation, extraction, launch, relay, cleanup |
| `client.exe` | Yes | Separate process; its role is determined by the executable itself |
| `backend.exe` | Yes | Separate process; its role is determined by the executable itself |
| `config.toml` | Yes | Fixed logical name; runtime-readable configuration |
| `payload.zip` | Yes | Fixed logical name; extracted by the backend |
| Optional assets | No | For example `assets/background.webp`, `assets/icon.ico`, runtime libraries |

## Container format

AIP does not mandate a specific container format for `foo.exe`. The packager may use the stub as the PE body and append the other files as resources, overlay data, or an embedded archive; it may also use a small bootstrapper. The following are required:

1. The final artifact is one `foo.exe` with no sidecar files.
2. Every runtime dependency is inside `foo.exe`, including client, backend, config, payload, and libraries.
3. The stub can verify the signature and embedded component integrity at startup.
4. The stub can extract embedded components into a private session directory and make that directory the current working directory for client/backend.

`stub.exe`, `client.exe`, and `backend.exe` are logical names. The container may rename them, but it must guarantee:

- the entry point is the stub;
- client and backend are two separate processes;
- the role is determined by executable identity, **not by a command-line argument**;
- `config.toml` and `payload.zip` appear under their fixed logical names at the package root at runtime.

## Runtime extraction

When `foo.exe` starts, the stub prepares the runtime environment:

1. Verify the Authenticode signature of `foo.exe` with `WinVerifyTrust`.
2. Verify the SHA-256 hashes in the embedded component manifest.
3. Create a private session directory at `%LOCALAPPDATA%\Anyinstaller\Sessions\<random ID>`.
4. Extract every required component except the running stub into the session directory:
   - `client.exe`
   - `backend.exe`
   - `config.toml`
   - `payload.zip`
   - optional assets, preserving their package-relative paths such as `assets/background.webp`
5. Make the session directory read-only (the stub keeps cleanup rights).
6. Create two named pipes, launch client and backend, and set the session directory as their current working directory.

The session directory must be under the current user's local application data and must use a DACL that allows only the current user and SYSTEM; it must not use the `%TEMP%` root, a shared directory, or a predictable path. See the [mandatory security requirements](../security/requirements.md).

## Asset path rewriting

In the source `config.toml`, image paths are **paths on the packager's machine**:

```toml
background = 'C:\Users\Ali\Pictures\background.avif'
icon = 'C:\Users\Ali\Pictures\app.ico'
```

The packager must:

1. Read and validate the images;
2. Embed them in `foo.exe`, normally under `assets/`;
3. Rewrite the packaged `config.toml` to package-internal relative paths:

```toml
background = "assets/background.avif"
icon = "assets/icon.ico"
```

At runtime, the client accepts only package-internal relative paths and reads them only from the session directory; it must not read `C:\Users\...` or any other local path. Special background values (`acrylic`, `mica`, colors, etc.) are unchanged.

If an asset is missing, cannot be decoded, or has an invalid path, the packager must refuse to produce the release. At runtime, an invalid package-internal path must make the client fall back to a white background or the default icon and log an error. See [Background](../configuration/background.md) and [Packaging, signing, and integrity](../security/packaging.md).

## Package root

At runtime, the package root seen by client/backend contains at least:

```text
<session>\
├─ client.exe
├─ backend.exe
├─ config.toml
├─ payload.zip
└─ assets\
   ├─ background.avif   # optional
   └─ icon.ico          # optional
```

The stub must set the current working directory of client and backend to `<session>`. Client and backend must resolve relative asset paths only inside that directory.

# Packaging, signing, and integrity

> [中文](../../zh-CN/security/packaging.md) | English · [Index](../index.md) · [Home](../../../README.md)

This page defines what the packager must do and what `foo.exe` must satisfy before release. Every item is mandatory (MUST).

## Packager responsibilities

The packager takes these inputs:

```text
build\
├─ stub.exe
├─ client.exe
├─ backend.exe
├─ config.toml        # source config; image paths may be local
├─ payload.zip
└─ assets\            # optional
```

The output is exactly one `foo.exe`. The packager must:

1. Validate the source `config.toml`: syntax, protocol number, required fields, `name` length and characters.
2. Validate `payload.zip`: valid ZIP, entry count, total size, compression ratio, legal paths.
3. Resolve the `icon` / `background` source image paths: they must exist, be decodable, and satisfy size and pixel limits.
4. Copy images into the package's `assets/` and rewrite the packaged config paths to package-internal relative paths.
5. Build a component manifest recording each embedded file's logical name, SHA-256, size, and package-relative path.
6. Decide whether client upgrades must preserve user data — AIP does not mandate this; the backend implements it.
7. Use the stub as the PE body and append the remaining components as resources, overlay data, or an embedded archive.
8. Produce `foo.exe` and run a self-extraction round-trip test: parse and verify every component with the same rules used at runtime.
9. Sign `foo.exe` and add an RFC 3161 timestamp.
10. After signing, verify the signature and manifest again to confirm the release is accepted by the stub.

If any step fails, the packager must exit non-zero and must not leave a partial `foo.exe`.

## Component manifest

The manifest must contain at least:

| Field | Description |
| --- | --- |
| Logical name | `stub.exe`, `client.exe`, `backend.exe`, `config.toml`, `payload.zip`, asset paths |
| Package-relative path | Destination path when extracted to the session directory |
| Size | Byte count |
| SHA-256 | Hash of the component's original bytes |
| Type | Executable, config file, archive, asset |

The manifest itself must be covered by the `foo.exe` signature. The stub must verify hashes both before and after extraction; any mismatch must abort.

## Signing

1. `foo.exe` must be Authenticode-signed.
2. It must be timestamped so the signature remains valid after the certificate expires.
3. A trusted public code-signing certificate must be used. An EV certificate reduces SmartScreen warnings but does not replace signing and integrity verification.
4. Signing must happen after packaging, and it must not bypass the manifest hash checks.
5. Unsigned, test-signed, or self-signed-root packages must not be distributed.
6. The signing private key must not be stored under a path writable by the installer.

## Integrity

1. At startup the stub must verify its own signature with `WinVerifyTrust`, including revocation checking.
2. The stub must verify the manifest signature and the SHA-256 of every component.
3. After extraction to the session directory, hashes must be verified again to detect replacement during extraction.
4. A verification failure must fail closed: do not launch client/backend, do not install, show a readable error.
5. Options such as "ignore certificate errors" or "warn only" are forbidden.
6. Additional components or updated manifests must not be downloaded from the network.

## Single-file constraint

1. The release directory may contain only `foo.exe`; DLLs, config files, asset files, `*.pdb` files, and install scripts are forbidden.
2. Everything needed at runtime must be embedded, including the VC runtime, .NET runtime dependencies, icons, and backgrounds.
3. Do not depend on any preinstalled runtime version; if a dependency exists, the packager must embed it and record it in the manifest.
4. `foo.exe` must not modify itself or write files next to itself at runtime.
5. The package must not require the user to unpack it first, and it must not create temporary sidecar files for external programs.

## Asset handling

1. Source images must be local files; the packager must not fetch assets from network URLs.
2. The packager must reject unexpected formats such as SVG, HTML, or executables disguised as images.
3. The packager must decode and re-encode images to a supported format, stripping metadata and extra data chunks.
4. Package asset paths must be forward-slash relative paths such as `assets/background.avif`.
5. Package paths must not contain `..`, drive letters, UNC, device prefixes, ADS, or control characters.
6. Every asset path in the packaged config must have a matching component in the manifest.

## payload.zip

1. It must be a valid ZIP using a supported compression method (store or deflate).
2. It must not contain encrypted entries, symlinks, hard links, or reparse points.
3. It must not contain executable install scripts as "install logic"; install logic belongs only in the backend.
4. Entry paths must be relative and contain no `..`, drive letters, UNC, ADS, device names, or trailing dots/spaces.
5. The packager must compute entry count, total uncompressed size, single-file size, and compression ratio, consistent with runtime limits.
6. Empty ZIPs (zero entries) must be rejected.

## Pre-release checklist

Confirm every item before release:

- [ ] `foo.exe` is the only file in the directory.
- [ ] `foo.exe` is signed and timestamped; `signtool verify /pa /all` passes.
- [ ] The component manifest covers every embedded file with correct SHA-256.
- [ ] The packaged `config.toml` contains no local absolute paths.
- [ ] `protocol = 1`, with complete `name` and `[defaults]`.
- [ ] Every image decodes and is within size and pixel limits.
- [ ] `payload.zip` passes traversal, ZIP-bomb, and device-name checks.
- [ ] Full install, cancellation, and failure rollback are tested on a clean user account.
- [ ] Offline testing confirms no download behavior.
- [ ] Process Monitor confirms no reads or writes outside the package or install location.

See the [mandatory security requirements](requirements.md) for runtime security requirements.

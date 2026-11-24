# Architecture and components

> [中文](../../zh-CN/overview/architecture.md) | English · [Index](../index.md) · [Home](../../../README.md)

AIP v1 is Windows-only. An installer consists of three processes: stub, client, and backend. They exchange TOML messages over Windows named pipes; the end user sees only a single `foo.exe`.

## Design goals

1. **Single-file distribution.** The release is one `foo.exe` with no sidecar DLLs, config files, or asset files.
2. **UI and installation logic are decoupled.** The client and backend can be written in any language or UI framework as long as they follow AIP.
3. **The stub coordinates everything.** The client and backend never connect directly; the stub creates pipes, relays messages, and owns the lifecycle.
4. **Debuggable messages.** Pipe frames contain UTF-8 TOML, which is easy to log, capture, and test by hand.
5. **Secure by default.** Every security requirement is mandatory; there is no optional hardening or compatibility mode.
6. **One session.** Each run of `foo.exe` is one installation session; it does not install multiple applications concurrently.

## Components

| Component | Required | Responsibility |
| --- | --- | --- |
| **stub** | Yes | Validate the package, create pipes, launch client/backend, relay both directions, end the session |
| **client** | Yes | Show the GUI, collect settings, start the install, show the result |
| **backend** | Yes | Remember settings, extract the payload, create shortcuts, report completion |
| **config.toml** | Yes | Protocol number, name, icon, background, and defaults |
| **payload.zip** | Yes | The application being installed |
| **Optional assets** | No | Background, icon, runtime libraries, licenses, etc.; all embedded in `foo.exe` |

## stub

The stub is the entry point of `foo.exe` and the owner of the session:

- Verify the Authenticode signature of `foo.exe` and the hashes of embedded components.
- Read `config.toml` and validate the protocol number and required fields.
- Create a private session directory and extract `client.exe`, `backend.exe`, `config.toml`, `payload.zip`, and assets.
- Generate a random session ID and create two named pipes.
- Launch the client and backend, passing only their own `--aip-pipe-name`.
- Relay TOML frames in both directions; recognize and consume the client's `readyToExit`.
- On `readyToExit` or a fatal error, terminate both children, delete the session directory, and exit.

The stub does not show the regular installation UI and does not perform the installation. It may show a system error dialog on startup failure.

## client

The client is the installation UI process:

- Read the runtime `config.toml` and build the UI from `name`, `icon`, `background`, and `[defaults]`.
- Let the user change the install location, Start Menu folder flag, and desktop shortcut flag.
- Send `setInstallLocation`, `setCreateStartMenuFolder`, and `setCreateDesktopShortcut`, then `startInstall`.
- Receive `installFinished` and show success or failure.
- Send `readyToExit` after all UI work is complete.

The client must not extract the payload, write the install directory, or create shortcuts. It must not read background or icon files from outside `foo.exe`.

## backend

The backend performs the installation:

- Initialize session state from `[defaults]` in `config.toml`.
- Receive client settings and update state using last-write-wins.
- On `startInstall`, strictly validate the install location and package assets.
- Extract `payload.zip` into the install location.
- Create the current user's Start Menu folder and desktop shortcut according to the flags; the shortcut target is determined by the backend from the payload.
- Send exactly one `installFinished`.

The backend does not own the regular GUI. All path validation, shortcut target validation, and extraction limits must follow the [mandatory security requirements](../security/requirements.md).

## Trust model

- `foo.exe` is the only trusted entry point; release packages must be signed and self-verified by the stub.
- The client and backend are launched by the stub from a private session directory and cannot read arbitrary local asset paths.
- There is no direct pipe between client and backend; neither can bypass the stub.
- Pipe names, connecting processes, and message sizes are validated; any security failure aborts immediately (fail closed).
- `config.toml`, `payload.zip`, and optional assets are embedded in the signed `foo.exe`.

## Version

The current specification is **AIP v1**, corresponding to `protocol = 1` in `config.toml`. Incompatible changes must increase the protocol number; see [Versioning and compatibility](../versioning.md).

# Error handling

> [中文](../../zh-CN/communication/errors.md) | English · [Index](../index.md) · [Home](../../../README.md)

## Principles

- **Fail closed.** Any failure of signature, hash, config, pipe, path, or extraction security checks must abort the installation; never fall back to an unsafe behavior.
- **Never fabricate results.** The stub must not send `installFinished` on behalf of the backend.
- **Readable errors.** User-facing errors must be UTF-8 text, contain no NUL, and reveal no keys or tokens.
- **Clean up.** Every failure path must terminate children, close pipes, and delete the session directory.

## Error matrix

| Scenario | Handling |
| --- | --- |
| `config.toml` missing, TOML syntax error, unsupported protocol, missing required field | Stub shows an error and exits; no pipes, no client/backend |
| `foo.exe` signature invalid/untrusted or hash manifest mismatch | Stub aborts; no extraction, no launch |
| Child does not connect to its pipe within 30 seconds | Stub ends the session and exits |
| Connecting PID does not match the child launched by the stub | Stub disconnects and aborts; this is a security failure |
| Invalid frame length, non-UTF-8 body, invalid TOML, non-table root, missing string `type` | Stub treats it as a fatal protocol error and ends the session |
| Frame over 1 MiB, or per-direction buffer over 64 frames / 4 MiB | Stub aborts the session |
| Wrong field type or missing required field | For the three `set*` messages, mark that field invalid; a later valid message may clear it. If a field is still invalid at `startInstall`, send `installFinished success = false`. An unparsable `installFinished` makes the client treat the install as failed |
| Unknown message type or unknown field | Receiver ignores it; the stub forwards unknown types except `readyToExit` |
| Malformed or unparsable `installProgress` | The client ignores that progress message and keeps waiting for `installFinished`; it must not abort installation because of it |
| Backend cannot send `installProgress` | Progress is advisory; the backend may continue, and `installFinished` remains authoritative |
| Backend exits or disconnects before `installFinished` | Stub closes the client pipe; client treats EOF as failure; stub cleans up and exits |
| Backend exits after `installFinished` | Normal; stub keeps the client pipe until `readyToExit` or client disconnect |
| Client exits without `readyToExit` | Stub treats it as cancellation: closes the backend pipe, waits 5 seconds, force-terminates, cleans up, exits |
| Client sends `readyToExit` before `installFinished` | Stub still closes the session; this is a client violation and the backend may be terminated mid-install |
| Installation failure (permissions, disk, extraction, shortcut) | Backend sends `installFinished success = false` with `error`; client shows the error and still sends `readyToExit` |
| Duplicate `startInstall` | First one takes effect; later ones are ignored |
| Duplicate `readyToExit` | First one takes effect; the stub is already shutting down |

## Timeouts

| Item | Value |
| --- | --- |
| Child connects to pipe | 30 seconds |
| Shutdown grace | 5 seconds |
| Pipe reads/writes | Must have a timeout or be cancelable; no indefinite blocking |
| Installation itself | No fixed timeout; the client shows live progress, or indeterminate progress when no progress messages arrive; the user can close the window to cancel |

## State after failure

- Shortcuts must not point to missing targets; do not create shortcuts when installation fails.
- The backend should use a staging directory; on failure, delete staging content and avoid damaging an existing installation.
- If a full rollback is impossible, `error` must state which phase failed and what may remain.
- After showing the error, the client still sends `readyToExit` so the stub can clean up normally.

## Security failures

The following must abort immediately, with no further installation:

- signature, hash, PID, DACL, session directory, or path validation failure;
- install location is a protected directory, reparse point, or otherwise invalid path;
- a ZIP entry attempts traversal, ADS, device names, symlinks, or exceeds extraction limits;
- a package asset path escapes the package root or attempts to read an external file;
- any component attempts to execute a shell command or download content.

See the [mandatory security requirements](../security/requirements.md) for details.

# Lifecycle

> [中文](../../zh-CN/overview/lifecycle.md) | English · [Index](../index.md) · [Home](../../../README.md)

## Startup

1. The user runs `foo.exe`.
2. The stub verifies its own signature and the hashes of embedded components; on failure it shows an error and exits.
3. The stub reads and validates `config.toml`; an unsupported protocol number or a missing required field aborts startup.
4. The stub creates a private session directory and extracts client, backend, config, payload, and optional assets.
5. The stub generates a random session ID and creates two named pipes:

   ```text
   \\.\pipe\aip-<sessionId>-client
   \\.\pipe\aip-<sessionId>-backend
   ```

6. The stub launches backend and client. The only AIP argument is the pipe name:

   ```text
   backend.exe --aip-pipe-name aip-<sessionId>-backend
   client.exe  --aip-pipe-name aip-<sessionId>-client
   ```

   The role is determined by the executable identity, not by a command-line argument.

7. The stub puts both children in a Job Object (`JOB_OBJECT_LIMIT_KILL_ON_JOB_CLOSE`) and verifies the PID of each pipe peer.
8. Client/backend must connect to their pipes within 30 seconds of launch; otherwise startup fails.

## Normal message flow

```text
stub ── create pipes ──> launch backend ──> launch client
                                  │
client ── setInstallLocation ───────────────┐
client ── setCreateStartMenuFolder ────────┤
client ── setCreateDesktopShortcut ────────┤
client ── startInstall ────────────────────┤
                                  │         ▼
                                  │      stub relays
                                  │         │
                                  │         ▼
                                  │      backend installs
                                  │         │
backend ── installFinished ───────┴──> stub relays ──> client
                                  │
client ── readyToExit ──> stub ── terminate client/backend ──> stub exits
```

Ordering requirements:

1. The client may send settings at any time; the backend remembers the last valid value.
2. Before `startInstall`, the client sends its current three settings (preferably all of them, at least the ones the user changed).
3. The backend starts the installation on the first `startInstall`; later `startInstall` messages are ignored.
4. The backend may send zero or more `installProgress` messages; the client updates progress but must never treat progress as completion.
5. The backend sends exactly one `installFinished`, whether the installation succeeds or fails.
6. After receiving `installFinished` and finishing the result UI, the client sends `readyToExit`.
7. The stub ends the session when it receives `readyToExit`.

## Shutdown

1. The stub stops relaying new messages.
2. The stub closes both pipes.
3. The stub asks client and backend to exit and waits up to 5 seconds.
4. Any remaining process is forcibly terminated; the Job Object also cleans up children when the stub exits.
5. The stub deletes the session directory and temporary files.
6. The stub exits (the "self-kill"). If `foo.exe` is running from a temporary location, it may arrange deletion of its own file, but it must not delete the user's install directory or pre-existing shortcuts.

## Failure and cancellation

- Signature, hash, config, or pipe security failure: abort immediately; do not install.
- The client disconnects without `readyToExit`: the stub treats the session as cancelled, terminates the backend, and cleans up; the backend must use a staging directory to avoid a half-installed state.
- The backend disconnects before `installFinished`: the stub closes the client pipe; the client treats this as failure; the stub cleans up.
- The backend exits after `installFinished`: normal; the stub keeps the client pipe open until `readyToExit`.
- The client sends `readyToExit` before `installFinished`: the stub still closes the session; this is a client violation and the backend may be terminated mid-installation.

See [Error handling](../communication/errors.md) for the full error matrix.

# Pipes and relay

> [中文](../../zh-CN/communication/transport.md) | English · [Index](../index.md) · [Home](../../../README.md)

## Topology

```text
┌────────┐   pipe A    ┌──────┐   pipe B    ┌─────────┐
│ client │ <=========> │ stub │ <=========> │ backend │
└────────┘             └──────┘             └─────────┘
```

- Pipe A is provided by the stub to the client; pipe B to the backend.
- The client and backend do not know each other's pipe name and cannot connect directly.
- All messages pass through the stub; `readyToExit` is consumed by the stub and not forwarded.
- Each pipe is a full-duplex byte stream, but message order within each direction must be preserved.

## Pipe names

The stub generates an unpredictable 128-bit random session ID and creates:

```text
aip-<sessionId>-client
aip-<sessionId>-backend
```

Full paths:

```text
\\.\pipe\aip-<sessionId>-client
\\.\pipe\aip-<sessionId>-backend
```

`--aip-pipe-name` receives the name without the `\\.\pipe\` prefix, for example:

```text
aip-7f3c9a1e2b4d4e8fa6c0d5e1f2a3b4c5-client
```

## Startup arguments

The stub launches two separate processes and passes only the pipe name:

```text
<client.exe>  --aip-pipe-name <client-pipe-name>
<backend.exe> --aip-pipe-name <backend-pipe-name>
```

- The role is determined by executable identity; no other role argument exists.
- Processes open the full path `\\.\pipe\<pipe-name>`.
- The current working directory must be the session package root.
- Environment variables are inherited; extra non-AIP arguments must be ignored.
- Client/backend must connect within 30 seconds of launch; timeout is a startup failure.

## Creation and connection

1. The stub must create both pipes as servers before launching children.
2. Each pipe has exactly one instance and accepts one connection; the listening end must be closed immediately after connection.
3. The stub must obtain the connecting PID with `GetNamedPipeClientProcessId`; it must equal the PID of the child it launched. Otherwise disconnect and abort.
4. The client must open its pipe with `SECURITY_SQOS_PRESENT | SECURITY_IDENTIFICATION` to limit server impersonation.
5. Pipe handles must not be inheritable; the stub launches children with `bInheritHandles = FALSE`.
6. Pipes use byte mode (`PIPE_TYPE_BYTE | PIPE_READMODE_BYTE | PIPE_WAIT`).
7. All pipe I/O must have a timeout or be cancelable; it must never block indefinitely.
8. Security attributes and DACL requirements are in the [mandatory security requirements](../security/requirements.md).

## Relay rules

- client → stub → backend: the three `set*` messages and `startInstall`.
- backend → stub → client: `installFinished`.
- client → stub: `readyToExit`, consumed by the stub.
- The stub must forward in receive order and must not reorder, coalesce, or reserialize TOML bodies.
- The stub parses only the frame `type` to recognize `readyToExit`; unknown types are forwarded unchanged.
- If either direction disconnects or a protocol error occurs, the stub must end the whole session, not just one side.

## Buffering and limits

- Maximum frame size: 1 MiB.
- Maximum buffered data per direction: 64 frames or 4 MiB, whichever is reached first; exceeding it aborts the session.
- If the destination has not connected yet, the stub may buffer within those limits; it must not grow without bound.
- Connection timeout: 30 seconds; shutdown grace: 5 seconds.
- These limits are mandatory, not recommendations.

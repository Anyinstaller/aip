# Messages

> [中文](../../zh-CN/communication/messages.md) | English · [Index](../index.md) · [Home](../../../README.md)

## Message envelope

Every message body is a TOML table whose root must contain the string key `type`:

```toml
type = "setInstallLocation"
value = 'D:\Apps\Example'
```

General rules:

- `type` is required, a string, camelCase, and case-sensitive.
- Message fields live directly in the root table; there is no extra `payload` wrapper.
- Unknown keys must be ignored for forward compatibility.
- Unknown `type` values must be ignored by receivers; the stub special-cases only `readyToExit` and forwards everything else.
- Messages must not contain `null`; TOML 1.0 has no null. Use omission or an empty string.
- No sequence numbers, timestamps, or session IDs are defined; pipe order is message order.
- Wrong field types or missing required fields are message-level errors; see [Error handling](errors.md).

## Message summary

| Type | Direction | Purpose |
| --- | --- | --- |
| `setInstallLocation` | client → backend | Set the install location |
| `setCreateStartMenuFolder` | client → backend | Create a Start Menu folder or not |
| `setCreateDesktopShortcut` | client → backend | Create a desktop shortcut or not |
| `startInstall` | client → backend | Start the installation |
| `installProgress` | backend → client | Advisory installation progress (optional) |
| `installFinished` | backend → client | Installation finished (success or failure) |
| `readyToExit` | client → stub | Client is done; the session may end |

Except for `readyToExit`, every message is forwarded unchanged by the stub.

## setInstallLocation

```toml
type = "setInstallLocation"
value = 'D:\Apps\Example'
```

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `value` | string | Yes | Install directory |

- Must be an absolute local drive path such as `D:\Apps\Example`.
- UNC paths, network paths, device paths, relative paths, paths containing `..`, and NTFS ADS are rejected.
- `%VAR%` environment references are allowed; the backend expands them and validates the result. Unresolvable variables make the value invalid.
- Maximum length: 32,767 UTF-16 code units.
- If the path exists but is not a directory, installation fails.
- The backend overwrites its current value when the message arrives before `startInstall`; if repeated, the later message wins.
- If the client never sends it, the backend uses `defaults.installLocation`.
- After `startInstall`, this message no longer changes the session state.
- Additional path security checks are in the [mandatory security requirements](../security/requirements.md).

## setCreateStartMenuFolder

```toml
type = "setCreateStartMenuFolder"
value = true
```

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `value` | boolean | Yes | `true` creates; `false` does not |

- If never received, `defaults.createStartMenuFolder` is used.
- If repeated, the later message wins.
- `false` only means "do not create"; it does not delete an existing folder or shortcut.
- Creation location: the current user's `FOLDERID_Programs\<name>`.
- The backend decides the shortcut contents from the payload; shortcuts are created only after the payload is successfully written.

## setCreateDesktopShortcut

```toml
type = "setCreateDesktopShortcut"
value = false
```

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `value` | boolean | Yes | `true` creates; `false` does not |

- If never received, `defaults.createDesktopShortcut` is used.
- If repeated, the later message wins.
- `false` only means "do not create"; it does not delete an existing shortcut.
- Creation location: the current user's `FOLDERID_Desktop\<name>.lnk`.
- The shortcut target, arguments, and working directory are determined by the backend from the payload; the target must be inside the install directory.
- Shortcuts are created only after the payload is successfully written.

## startInstall

```toml
type = "startInstall"
```

- No additional fields.
- Before sending it, the client must send its current three settings (all of them is recommended).
- The backend installs using current state; settings never received use `config.toml` defaults.
- The backend must validate all settings and security conditions before starting; any failure is reported through `installFinished`.
- The first `startInstall` takes effect; later ones must be ignored.
- Installation may be synchronous or asynchronous, but the backend must eventually send exactly one `installFinished`.
- Settings received during installation must be ignored.

## installProgress

The backend reports advisory installation progress to the client for progress bars and stage labels.

```toml
type = "installProgress"
stage = "extracting"
percent = 42
current = 5242880
total = 12582912
```

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `stage` | string | Yes | Stable stage identifier used for localized labels |
| `percent` | integer | Yes | Overall progress in `0..=100` |
| `current` | integer | No | Amount completed in the current stage; non-negative |
| `total` | integer | No | Amount expected in the current stage; non-negative; `current <= total` when both are present |

Recommended stages:

| `stage` | Suggested percent | Meaning |
| --- | --- | --- |
| `validating` | 0 | Validate configuration and install arguments |
| `preparing` | 5 | Create staging storage and prepare |
| `extracting` | 10–78 | Extract payload; may include byte `current` / `total` |
| `verifying` | 80 | Re-check extracted contents |
| `committing` | 85 | Commit the install directory |
| `creatingShortcuts` | 92–96 | Create shortcuts |
| `finishing` | 99–100 | Cleanup and finalization |

Rules:

- `installProgress` is advisory, not a completion signal; only `installFinished` completes installation.
- The backend may omit progress entirely; older clients must remain on indeterminate progress.
- `percent` should be non-decreasing within a session.
- The backend may send progress frequently while extracting, but should suppress unchanged values.
- The client must ignore malformed progress messages, including out-of-range `percent` or wrong field types; it must not abort installation because of them.
- The client should ignore unknown `stage` details while still using a valid `percent`.
- Unknown fields must be ignored for forward compatibility.
- The backend must not send `installProgress` after `installFinished`.
- Adding stage values or this message does not require a protocol-number bump.

## installFinished

The backend sends this after an installation attempt.

Success:

```toml
type = "installFinished"
success = true
```

Failure:

```toml
type = "installFinished"
success = false
error = "Access denied while writing the install directory"
```

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `success` | boolean | Yes | Whether installation succeeded |
| `error` | string | Required on failure; optional on success | Human-readable error |

- At most once per session; if `startInstall` was processed, it must be sent once.
- On `success = false`, `error` must be present and must not contain NUL.
- On `success = true`, the client ignores `error`.
- The client displays the result; on failure it shows `error` or a generic message if missing.
- The backend may stay connected or exit after sending; the stub must not end the client early because of that.

## readyToExit

```toml
type = "readyToExit"
```

- Direction is fixed: client → stub; the stub must consume it and not forward it to the backend.
- The stub recognizes it only on the client pipe; a message with the same type on the backend pipe must be ignored.
- The client must send it after receiving `installFinished` and finishing the result UI.
- It should be sent once; the stub enters shutdown on the first one.
- The client may exit after sending; the stub also terminates it.
- If the client disconnects without `readyToExit`, the stub treats the session as cancelled and cleans up.

## State and last-write-wins

The backend keeps in-memory state:

| State field | Initial value | Updating message | Frozen when |
| --- | --- | --- | --- |
| Install location | `defaults.installLocation` | `setInstallLocation` | `startInstall` is processed |
| Create Start Menu folder | `defaults.createStartMenuFolder` | `setCreateStartMenuFolder` | `startInstall` is processed |
| Create desktop shortcut | `defaults.createDesktopShortcut` | `setCreateDesktopShortcut` | `startInstall` is processed |
| Started | `false` | `startInstall` | First one takes effect |
| Finished | `false` | After `installFinished` is sent | For the rest of the session |

What "last-write-wins" means:

1. For the three `set*` messages, each valid message overwrites its field; no deduplication and no value comparison.
2. The value in effect is the last valid value received before `startInstall` is processed.
3. Pipe order is guaranteed; if the same setting arrives twice, the second overwrites the first.
4. Before `startInstall`, an invalid `set*` message also participates in ordering: it marks that field invalid; a later valid message for the same field overwrites it and clears the mark.
5. `startInstall` is not last-write-wins: the first one takes effect, later ones are ignored.
6. `readyToExit` is also first-one-wins.
7. State exists only for this session; it is not persisted and is reinitialized from `config.toml` next time.

## Example

```toml
type = "setInstallLocation"
value = 'D:\Apps\Example App'
```

```toml
type = "setCreateStartMenuFolder"
value = true
```

```toml
type = "setCreateDesktopShortcut"
value = false
```

```toml
type = "startInstall"
```

```toml
type = "installProgress"
stage = "finishing"
percent = 100
```

```toml
type = "installFinished"
success = true
```

```toml
type = "readyToExit"
```

Relay paths:

```text
set* / startInstall  client → stub → backend
installProgress      backend → stub → client
installFinished      backend → stub → client
readyToExit          client → stub (consumed, not forwarded)
```

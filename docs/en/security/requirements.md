# Mandatory security requirements

> [中文](../../zh-CN/security/requirements.md) | English · [Index](../index.md) · [Home](../../../README.md)

AIP v1 has **no optional security items**. Every item on this page is **MUST**. Failing any one item means the implementation is not AIP v1 conformant and must not be distributed as a release package.

## Threat model

The following threats must be assumed:

- other local processes attempt to squat on or impersonate the named pipes;
- an attacker modifies `foo.exe` or any embedded component;
- a malicious `payload.zip` uses path traversal, ZIP bombs, ADS, symlinks, or device names;
- a malicious config makes the install write into system directories, read files outside the package, or create malicious shortcuts;
- the temporary directory is readable, writable, or replaceable by another process;
- the installer is tricked into performing unintended operations with elevated rights;
- DLL search-order hijacking, uninitialized input, or unbounded memory allocation.

## 1. Processes and privileges

1. The stub, client, and backend must run as the same user, in the same logon session, at the same integrity level.
2. The stub must not start at medium integrity and elevate only the backend; mixed-integrity pipes are forbidden. If administrator rights are required, the `foo.exe` manifest must declare `requireAdministrator` for the whole package; otherwise it must declare `asInvoker`.
3. The stub must create a Job Object with `JOB_OBJECT_LIMIT_KILL_ON_JOB_CLOSE` and assign both client and backend to it. If the stub dies, the system must kill the children.
4. The stub must launch children with `bInheritHandles = FALSE`; pipe handles, process handles, and other kernel handles must not be inherited.
5. Client and backend must be launched from the private session directory. The stub must verify that the actual image path is inside that directory and is not a reparse point.
6. The stub must not call `ImpersonateNamedPipeClient` on client/backend, must not enable `SeDebugPrivilege`, and must not request privileges unrelated to the task.
7. Every security check failure must fail closed: terminate children, delete the session directory, do not install, do not create shortcuts.
8. No component may execute a shell command that comes from a pipe, config, or payload.

## 2. Named pipes

1. Pipes must use a DACL that allows only the current user SID; Everyone, Users, anonymous, and remote principals must be denied.
2. Pipes must set `PIPE_REJECT_REMOTE_CLIENTS`.
3. Pipe names must contain at least a 128-bit random session ID; fixed or predictable names are forbidden.
4. Each pipe has exactly one instance and accepts one connection; the listening end must be closed immediately after the connection is established.
5. The stub must obtain the connecting PID with `GetNamedPipeClientProcessId` and compare it with the child PID it launched; it must validate the logon session with `GetNamedPipeClientSessionId`; it must verify that the connecting process token's user SID matches the current user. Any mismatch must disconnect and abort.
6. The client must open its pipe with `SECURITY_SQOS_PRESENT | SECURITY_IDENTIFICATION` to limit server impersonation.
7. Pipes must operate in byte mode (`PIPE_TYPE_BYTE | PIPE_READMODE_BYTE | PIPE_WAIT`).
8. All pipe I/O must use overlapped I/O or an equivalent mechanism with timeouts/cancellation; indefinite blocking is forbidden.
9. Maximum frame size is 1 MiB; each direction may buffer at most 64 frames or 4 MiB. Exceeding either limit must abort the session.
10. The stub parses only the frame `type` to recognize `readyToExit`; it must not execute message content or start new processes based on it.

## 3. Session directory and extraction

1. The session directory must be created at `%LOCALAPPDATA%\Anyinstaller\Sessions\<random ID>`; the `%TEMP%` root, shared directories, and predictable paths are forbidden.
2. The directory must use a DACL that allows only the current user and SYSTEM, and must not inherit permissive ACEs from its parent.
3. Directory creation must use `CREATE_NEW` semantics; existing paths, reparse points, and symlinks must be rejected.
4. After extraction, every component's SHA-256 must be verified against the manifest; any mismatch must abort.
5. After extraction, the session directory must be made read-only; client/backend may only read, never write.
6. The stub must delete the session directory on normal exit, abnormal exit, and forced termination; files that cannot be deleted must be cleaned up after reboot.
7. `config.toml`, `payload.zip`, and assets must never be extracted to the install directory, the desktop, or any other persistent location.

## 4. Install location

1. `installLocation` must be an absolute local drive path such as `D:\Apps\Example` after `%VAR%` expansion.
2. The following must be rejected: UNC/network paths, device paths (`\\.\`, `\\?\`), relative paths, drive-relative paths (`C:foo`), paths containing `..`, NTFS ADS, empty strings, and strings containing NUL.
3. The path must be normalized with `GetFullPathNameW` or an equivalent API and re-validated afterwards.
4. The following must be rejected: drive roots, `%SystemRoot%` and its subdirectories, `%ProgramFiles%` / `%ProgramFiles(x86)%` roots, the `%ProgramData%` root, the user profile root, the desktop root, the Start Menu root, and the session directory.
5. Every path component from the volume root to the target must be checked; reparse points, symlinks, and junctions must be rejected to prevent redirection.
6. Installation must first create a randomly named private staging directory under the target's parent directory, with a DACL that allows only the current user and SYSTEM; extraction and verification complete before commit.
7. If the commit fails, the staging directory must be deleted; files must never be written outside the install location, the Start Menu, or the desktop.
8. No program, script, or DLL from the payload may be executed during installation.

## 5. Shortcuts

1. The current user's `FOLDERID_Programs` and `FOLDERID_Desktop` must be obtained with `SHGetKnownFolderPath`; environment-variable concatenation alone is insufficient.
2. `.lnk` files must be created with `IShellLinkW` + `IPersistFile`; hand-written binary shortcut files are forbidden.
3. The shortcut target must be an existing, non-reparse file inside the install directory and must still resolve inside it after normalization.
4. The shortcut name must derive from the sanitized `name`, be length-limited, and contain no path separators.
5. Shortcut arguments and working directory must be fixed by the backend; arbitrary command lines from config or pipes are forbidden.
6. `createStartMenuFolder = false` / `createDesktopShortcut = false` means "do not create" only; existing user files must not be deleted.
7. Shortcuts may be created only after the payload is successfully written and verified; a failed install must not leave shortcuts pointing at missing targets.

## 6. ZIP extraction

1. Only valid ZIP files are accepted; every entry's CRC and actual uncompressed size must be verified.
2. The following must be rejected: absolute paths, drive paths, UNC, `..`, NTFS ADS, device names (`CON`, `NUL`, `COM1`, etc.), trailing dots/spaces, empty names, and overlong paths.
3. Symlinks, reparse points, encrypted entries, and unsupported compression methods must be rejected.
4. Limits: at most 100,000 entries; total uncompressed size at most 8 GiB; single file at most 4 GiB; compression ratio at most 100:1; single path at most 32,767 UTF-16 code units.
5. Every entry must be normalized before writing and the final real path must be confirmed inside the staging directory; reparse points must be rejected.
6. Extraction must not overwrite anything outside the staging directory and must not inherit dangerous file attributes from the ZIP.
7. After extraction, entry count, total size, and paths must be checked again; any anomaly must abort and clean up.

## 7. Configuration and assets

1. `config.toml` must be at most 64 KiB, UTF-8 without BOM, and every known field must be strictly type- and length-checked.
2. In the packaged config, `icon` / `background` image paths must be package-internal relative paths; absolute paths, drive letters, UNC, device prefixes, `..`, ADS, and reparse points must be rejected.
3. Assets must resolve against the session package root, and the resolved real path must remain inside that root; reading files outside the package is forbidden.
4. Each icon/background file must be at most 64 MiB; at most 256 megapixels decoded and 16,384 pixels per side.
5. The packager must embed local image paths from the source config into `foo.exe` and rewrite them to package-internal paths; the packaged config must contain no local absolute paths.
6. At runtime, an invalid asset path must fall back to the default icon or white background and log an error; reading an external path is forbidden.
7. `%VAR%` expansion in the config must use the current process environment; unresolvable variables are errors and must not be left as literals to continue the install.

## 8. Network and execution

1. AIP communication is limited to local named pipes; network listeners, network connections, and remote pipes are forbidden.
2. During installation, stub/client/backend must not download, execute, or update remote content; OS signature-revocation checks are the only exception.
3. String commands must not be executed through `cmd.exe`, PowerShell, `ShellExecute`, or unconstrained `CreateProcess`.
4. Client/backend must be launched with `CreateProcessW` using an explicit image path, current directory, and arguments; current-directory search is forbidden.
5. DLLs must not be loaded from user-writable directories; a safe DLL search strategy must be used (for example `SetDefaultDllDirectories(LOAD_LIBRARY_SEARCH_SYSTEM32)`).

## 9. Binary hardening

1. All native components must enable ASLR (`/DYNAMICBASE`), DEP (`/NXCOMPAT`), CFG (`/guard:cf`), stack protection (`/GS`), and high-entropy VA (`/HIGHENTROPYVA`).
2. Safe string and memory APIs must be used, and every length from pipes, config, ZIP, and the file system must be validated.
3. Current-directory DLL search, `LoadLibrary` from user paths, unquoted paths, and predictable temporary file names must be avoided.
4. Release `foo.exe` must be Authenticode-signed and timestamped; unsigned packages must not be distributed.
5. Dependency libraries must come from trusted sources, receive security updates, and must not bundle versions with known exploitable vulnerabilities.

## 10. Failure, logging, and compliance

1. Any security check failure must abort immediately: no continued installation, no shortcuts, no fallback to an unsafe path.
2. Error messages must be UTF-8 text without NUL; keys, tokens, and full credentials must never be printed.
3. Logs must not be written into the install directory or package assets; if logging is enabled, it must go to the session directory or a user log directory and be deleted during cleanup.
4. Every item on this page is a necessary condition for AIP v1 conformance. Implementations must not disable these requirements via config, command-line switches, or environment variables.

See [Packaging, signing, and integrity](packaging.md) for the mandatory packaging requirements.

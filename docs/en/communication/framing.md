# Frame format

> [中文](../../zh-CN/communication/framing.md) | English · [Index](../index.md) · [Home](../../../README.md)

Both pipes use the same frame format: **4-byte little-endian unsigned length + UTF-8 TOML body**.

```text
+----------------------+---------------------------------------+
| uint32 little-endian | TOML body (N bytes, UTF-8, no BOM)    |
|        N             |                                       |
+----------------------+---------------------------------------+
```

| Field | Length | Description |
| --- | --- | --- |
| Length | 4 bytes | Unsigned 32-bit little-endian integer; body length only, excluding these 4 bytes |
| Body | N bytes | Exactly one TOML 1.0 document; the root must be a table |

## Rules

1. `1 <= N <= 1,048,576` (1 MiB). Empty frames or frames larger than 1 MiB are protocol errors.
2. The body must be valid UTF-8, with no BOM and no trailing NUL.
3. The body is one TOML document, not "one message per line".
4. One frame carries one message; the TOML may contain multiple lines, comments, or nested tables.
5. The sender must write the 4-byte length first, then the body; the receiver must handle the length or body arriving in multiple reads/writes.
6. One write may contain multiple frames; the receiver must loop by length.
7. The stub must forward the body byte-for-byte and must not reserialize TOML.
8. The length header may be regenerated, but its value must match the body length.
9. Newlines, NUL, or other delimiters must not replace the length header.
10. Reads must be bounded and time-limited; buffering per direction must not exceed 64 frames or 4 MiB.

## Example

For the body `type = "startInstall"` (21 bytes, no newline), the frame starts with:

```text
15 00 00 00
```

followed by 21 bytes of UTF-8 body.

## Invalid frames

Any of the following is a fatal protocol error; the stub must end the session:

- length is less than 1 or greater than 1 MiB;
- body is not valid UTF-8;
- body is not valid TOML 1.0;
- TOML root is not a table;
- `type` is missing or not a string;
- frame is truncated or the pipe closes in the middle of a frame.

Message field errors (for example, a wrong `value` type) are not frame errors; see [Messages](messages.md) and [Error handling](errors.md).

# Versioning and compatibility

> [中文](../zh-CN/versioning.md) | English · [Index](index.md) · [Home](../../README.md)

## Version identifiers

| Item | Current value |
| --- | --- |
| Specification name | Anyinstaller Protocol (AIP) |
| Specification version | AIP v1 |
| Protocol number | `1` |
| Pipe frame format | 4-byte little-endian length + UTF-8 TOML |
| Platform | Windows 10/11 |

`protocol = 1` in `config.toml` is the only machine-readable version identifier. The stub must reject unrecognized protocol numbers; it must not guess or downgrade.

## Compatibility rules

The following changes do **not** require a new protocol number; they only need to remain backward compatible:

- adding optional config fields;
- adding new message types such as `installProgress`;
- adding new special background values (provided old implementations treat them as invalid image paths and fall back to white);
- adding new optional asset files;
- fixing documentation errors without changing wire behavior.

The following changes **require** a new protocol number:

- changing the frame format, length field, or encoding;
- changing required fields, types, or semantics of existing messages;
- changing last-write-wins or send-exactly-once rules;
- changing startup arguments;
- changing package structure or asset path resolution rules;
- changing security requirements such that old implementations become non-conformant;
- removing or renaming existing messages, config fields, or special background values.

## Forward compatibility

1. Receivers must ignore unknown fields and unknown message types.
2. The stub must forward unknown messages other than `readyToExit`.
3. Client/backend must not crash or refuse to run because of an unknown optional field.
4. An unknown special `background` value is treated as an image path; if it cannot be resolved, fall back to white.

## Protocol mismatch

- `protocol` missing, not an integer, or not `1`: the stub shows an error and exits; it does not extract components or launch children.
- A future protocol number (for example `2`): an AIP v1 stub must reject it; it must not attempt compatibility.
- The packager must write the protocol number it supports; it must not write non-numeric values such as "latest".

## Specification change process

1. Modify the documentation in this repository.
2. Decide whether a new protocol number is required.
3. If so, update all affected examples and `examples/` at the same time.
4. State the compatibility impact in the commit message.
5. Documentation for older protocol numbers must remain in version history; it must not be overwritten in place.

## Application version

AIP does not define an application version. `name` is a display name, not a version. The backend may decide how to record an application version in the install directory, shortcuts, or registry, but that is outside AIP.

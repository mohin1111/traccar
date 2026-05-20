# StringProtocolEncoder.java

**Role:** Abstract subclass of `BaseProtocolEncoder` adding a `formatCommand(Command, String format, String... keys)` helper for building AT-command-style strings from command attributes.
**Fits in:** Extended by text-based protocol encoders (e.g., `ItsProtocolEncoder`). The `setTextCommandEncoder(StringProtocolEncoder)` call in `BaseProtocol` also wires this for SMS delivery.
**Read next:** [[BaseProtocolEncoder.java]] (parent), [[ItsProtocolEncoder.java]] (concrete example)

## Public API

- `formatCommand(Command command, String format, ValueFormatter valueFormatter, String... keys)` (line 30) — substitutes named keys from the command's attribute map into a `String.format` template. Special key `Command.KEY_UNIQUE_ID` is replaced with the device's IMEI.
- `formatCommand(Command command, String format, String... keys)` (line 55) — no-formatter variant; uses `Object.toString()` for each attribute.
- `ValueFormatter` inner interface (line 26) — `formatValue(String key, Object value)` for custom formatting.

## Key flows

### `formatCommand` logic (lines 31-53)
For each key:
1. `KEY_UNIQUE_ID` → `getUniqueId(deviceId)`.
2. Otherwise: look up in `command.getAttributes()`, apply formatter if provided, fall back to `toString()`, or empty string if null.
3. Return `String.format(format, values[])`.

## Gotchas / non-obvious

- `format` uses positional `%s` placeholders matching the order of `keys`. Key order in the `keys` vararg must match placeholder order in `format`.
- `null` attribute values become empty string, not `"null"`.

## Line index

- 26-28 — `ValueFormatter` inner interface
- 30-53 — `formatCommand` with formatter
- 55-57 — `formatCommand` without formatter

# `helper/` — Parsing, units, logging and general utilities

31 utility classes with no mutual dependencies. The most frequently imported package in the codebase. Protocol decoders use `Parser` / `PatternBuilder` / `BitUtil` / `BcdUtil`; the API layer uses `DateUtil` / `UnitsConverter`; handlers use `Log` / `PositionLogger`.

## File index

| File | One-liner |
|---|---|
| `Parser.java` | Regex-based field extractor: `nextInt()`, `nextDouble()`, `nextHex()`, `nextString()` etc. with null-safety |
| `PatternBuilder.java` | Fluent regex builder with named groups + type tokens for protocol decoder patterns |
| `BitUtil.java` | Static bit-field helpers: `check(value, bit)`, `range(value, from, to)`, `between(value, from, to)` |
| `BcdUtil.java` | BCD (Binary Coded Decimal) encode/decode for protocol fields |
| `BitBuffer.java` | Bit-level reader over a `ByteBuf` — reads arbitrary bit widths |
| `BufferUtil.java` | Netty `ByteBuf` search utilities (indexOf, readHex, etc.) |
| `Checksum.java` | NMEA, CRC16, CRC32, XOR, sum checksums used across protocol decoders |
| `DataConverter.java` | Hex-string ↔ byte[] conversions |
| `DateBuilder.java` | Fluent date assembler: `setDate(y,m,d).setTime(h,m,s).getDate()` with timezone |
| `DateUtil.java` | ISO-8601 / RFC-3339 formatting; `parseDate` for multiple formats; timezone utils |
| `DistanceCalculator.java` | Haversine distance between two lat/lon points (meters) |
| `Hashing.java` | Password hashing (salted SHA-512) + verification |
| `Log.java` | SLF4J logger setup: reads `logger.*` config keys, initializes logback |
| `LogAction.java` | Audit log helper: `logCreate/Update/Delete/Login` writes to `tc_logs` via `storage` |
| `NetworkUtil.java` | Extracts IP address from Netty `Channel` |
| `ObdDecoder.java` | Decodes OBD-II PIDs (fault codes, RPM, coolant temp, etc.) from raw byte values |
| `ObjectMapperContextResolver.java` | JAX-RS `ContextResolver<ObjectMapper>` for Jersey + Jackson integration |
| `PatternUtil.java` | Pattern compile helper with error logging |
| `PositionLogger.java` | Formats position as structured log line (device, protocol, coords, speed, etc.) |
| `ReflectionCache.java` | Cache of `Method` objects by class — used by `QueryBuilder` for reflection ORM |
| `SessionHelper.java` | Sets/reads user from HTTP session; calls `logAction` on login |
| `StringUtil.java` | `containsHex`, `sanitizePhoneNumber`, capitalizer, split utilities |
| `UnitsConverter.java` | Speed/distance/fuel conversions: knots↔kph↔mph, meters↔feet↔miles, liters↔gallons |
| `WebHelper.java` | Reconstructs server base URL from config |
| `ClassScanner.java` | Scans classpath for subclasses of a given class (used for protocol auto-discovery) |
| `ConcurrentWeakValueMap.java` | Thread-safe map with weak values (GC-able entries) |
| `model/` | Sub-package: `UserUtil`, `DeviceUtil`, `PositionUtil`, `AttributeUtil` — domain-level helper logic |

## Most important files for protocol development

1. **`Parser.java`** — every string-based protocol decoder uses this. Wraps a `Matcher` and provides typed `next*()` extractors. Returns `null` (not exception) on missing optional groups.
2. **`PatternBuilder.java`** — builds Java regex patterns from human-readable token sequences. Tokens like `NUMBER()`, `COORDINATE()`, `HEX()`, `COMMA`, `EOL` etc. See existing protocol decoders for usage.
3. **`BitUtil.java`** — essential for binary protocol decoders. `BitUtil.check(status, 0)` extracts bit 0; `BitUtil.range(value, 4, 8)` extracts bits 4-7.
4. **`DateBuilder.java`** — builds `Date` from parsed components. Handles ambiguous 2-digit years and 12/24-hour clock differences.

## Most important files for API/business logic

1. **`UnitsConverter.java`** — all speed values stored internally as knots. `knotsFromKph(speed)` for GPS devices reporting km/h. Report outputs convert back.
2. **`Hashing.java`** — `createHash(password)` → salted SHA-512 stored in `tc_users`. `validatePassword(input, hash, salt)`.
3. **`LogAction.java`** — all REST mutations call `logAction.*` for the audit trail written to `tc_logs`.
4. **`DateUtil.java`** — `formatDate(date)` → ISO-8601 string used in all API responses.

## `model/` sub-package

- `AttributeUtil` — walks device → group → server → config hierarchy to resolve a `ConfigKey` value for a given device. The implementation of [[LookupContext.java]].
- `UserUtil` — language, speed unit, distance unit from user/server attributes.
- `DeviceUtil` — iterates devices+groups for a user.
- `PositionUtil` — trip/stop detection logic shared by reports.

## How to add a utility

Prefer a static method on an existing file (`StringUtil`, `BitUtil`, etc.). Only create a new file if the utility is unrelated to existing categories. No Spring/Guice injection — all helpers are pure static or Guice-free instantiation.

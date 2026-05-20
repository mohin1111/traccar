# DateParameterConverterProvider.java

**Role:** JAX-RS `ParamConverterProvider` that maps ISO 8601 date strings in query parameters (e.g., `?from=2024-01-01T00:00:00Z`) to `java.util.Date` objects. Without this, Jersey would not know how to parse `Date` query params.
**Fits in:** Registered as a Jersey provider. Applies to all `@QueryParam` / `@FormParam` of type `Date` across all resources (ReportResource, PositionResource, etc.).
**Read next:** `DateUtil.java` (the actual parse/format logic)

## Public API

- `getConverter(rawType, genericType, annotations)` (line 43) — returns `DateParameterConverter` if the target type is `Date`, else null.
- Inner `DateParameterConverter.fromString(value)` (line 31) — delegates to `DateUtil.parseDate(value)`.
- Inner `DateParameterConverter.toString(value)` (line 36) — delegates to `DateUtil.formatDate(value)`.

## Gotchas / non-obvious

- Returns null for non-Date types — Jersey continues its own conversion chain. Safe.
- `DateUtil.parseDate` handles ISO 8601 with timezone. Passing a bare date without time may produce midnight UTC.

## Line index

- 28-40 — inner `DateParameterConverter`
- 43-49 — `getConverter` provider method

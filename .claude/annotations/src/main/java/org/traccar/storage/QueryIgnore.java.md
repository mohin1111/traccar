# QueryIgnore.java

**Role:** Method-level annotation that tells `ReflectionCache` to skip a getter/setter when building the column list. Prevents non-DB-backed computed properties from appearing in SQL.
**Fits in:** Checked inside `Columns.getAllColumns()` (via `ReflectionCache` property `queryIgnore()` flag). Companion to [[StorageName.java]].
**Read next:** [[storage/query/Columns.java]] (consumer), [[DatabaseStorage.java]] (drives everything)

## Public API

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface QueryIgnore {}
```

No elements — presence/absence is the signal.

## Key flows

### Usage in model classes
```java
@QueryIgnore
public String getSomeComputedThing() { ... }
```

`Columns.getAllColumns()` filters: `!entry.getValue().queryIgnore()`, so this getter is excluded from INSERT and UPDATE column lists.

## Gotchas / non-obvious

- Place on the **getter** (`get…` prefix), not a setter — `ReflectionCache` builds the column list from getters.
- Without `@QueryIgnore`, any public `getX()` method whose name matches a JavaBean convention will be assumed to map to a DB column `x`. Spurious methods will cause SQL errors (`Unknown column`).
- `RUNTIME` retention required — same reason as `@StorageName`.

## Line index

- 23 — `@Target(ElementType.METHOD)`
- 24 — `@Retention(RetentionPolicy.RUNTIME)`

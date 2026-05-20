# StorageName.java

**Role:** The sole annotation that maps a Java model class to a database table name. Without it on a class, `DatabaseStorage` throws `StorageException("StorageName annotation is missing")`.
**Fits in:** Applied to every entity in `model/` that needs DB persistence. Read at runtime by `DatabaseStorage.getStorageName()` (line 247) via reflection.
**Read next:** [[DatabaseStorage.java]] (consumer), [[QueryIgnore.java]] (companion annotation)

## Public API

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface StorageName {
    String value();
}
```

Single `value()` element — the exact SQL table name string (e.g., `"tc_devices"`, `"tc_positions"`).

## Key flows

### Usage in model classes
```java
@StorageName("tc_devices")
public class Device extends GroupedModel { ... }
```

### Runtime lookup in DatabaseStorage (line 247-252)
```java
StorageName storageName = clazz.getAnnotation(StorageName.class);
if (storageName == null) throw new StorageException("StorageName annotation is missing");
return storageName.value();
```

## Gotchas / non-obvious

- **`RUNTIME` retention is required** — without it, the annotation is stripped at compile time and reflection cannot find it. Do not change retention.
- **`TYPE` target only** — cannot be placed on methods or fields; only on class declarations.
- All Traccar table names follow the `tc_` prefix convention (e.g., `tc_users`, `tc_positions`, `tc_geofences`). Follow this when adding new model classes.
- `Permission` uses a static method `Permission.getStorageName(ownerClass, propertyClass)` to derive junction-table names (e.g., `tc_user_device`) — those junction tables do NOT carry `@StorageName`.

## Line index

- 23 — `@Target(ElementType.TYPE)`
- 24 — `@Retention(RetentionPolicy.RUNTIME)`
- 26-27 — annotation body with `String value()`

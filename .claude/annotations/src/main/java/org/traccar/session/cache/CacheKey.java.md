# CacheKey.java

**Role:** Immutable record combining `(Class<? extends BaseModel>, long id)` into a hash-map key for the cache graph. Provides both a direct constructor and a convenience constructor from a `BaseModel` instance.
**Fits in:** Used exclusively inside [[CacheGraph.java]] as keys for `roots` and `nodes` maps.
**Read next:** [[CacheGraph.java]], [[CacheNode.java]]

## Public API

```java
record CacheKey(Class<? extends BaseModel> clazz, long id) {
    CacheKey(BaseModel object) {
        this(object.getClass(), object.getId());
    }
}
```

Record equality uses both `clazz` and `id` — `(Device.class, 42)` ≠ `(User.class, 42)`.

## Gotchas / non-obvious

- **Package-private** — `CacheKey` has no access modifier; only accessible within `session.cache`. Not meant for use outside the cache subsystem.
- Type-safe: using the `Class<?>` as part of the key prevents accidental collision between a `Device` with id 5 and a `User` with id 5.

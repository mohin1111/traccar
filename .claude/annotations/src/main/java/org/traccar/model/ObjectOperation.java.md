# ObjectOperation.java

**Role:** Enum for CRUD operation types used in broadcast/cache invalidation messages. `BroadcastService` publishes `ObjectOperation` along with the affected entity class and id so all nodes in a cluster can update their caches.
**Fits in:** Used by `BroadcastService.updateObject()` and consumed by `CacheManager` to apply incremental cache updates.
**Read next:** [[Action.java]] (audit log uses string action types)

## Public API

- `ADD` — object was created.
- `UPDATE` — object was modified.
- `DELETE` — object was deleted.

## Line index

- 3 — `public enum ObjectOperation { ADD, UPDATE, DELETE, }`

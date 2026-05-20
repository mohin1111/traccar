# OrderResource.java

**Role:** REST resource at `/api/orders` for order/task records. Minimal — all logic in SimpleObjectResource. Sorted by description.
**Fits in:** Entity: `Order` (tc_orders). Extends SimpleObjectResource. Orders represent dispatch tasks in fleet workflows.
**Read next:** [[SimpleObjectResource.java]], `Order.java` model

## Public API

Full CRUD + user-filtered list. Constructor: `super(Order.class, "description", List.of("description"))`.

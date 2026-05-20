# Order.java

**Role:** Basic dispatch order / trip assignment entity. `@StorageName("tc_orders")` → `tc_orders`. Minimal: a unique order ID, description, and from/to addresses. Extends `ExtendedModel` for a JSON attributes bag.
**Fits in:** Linked to devices via `tc_device_order`. Early/stub feature; not heavily used in v6.13.3 processing pipeline.
**Read next:** [[ExtendedModel.java]] (parent)

## Public API

### DB-mapped fields
- `uniqueId String` (line 22) — external order reference number.
- `description String` (line 32) — order description.
- `fromAddress String` (line 42) — pickup address.
- `toAddress String` (line 52) — delivery address.
- Inherited `attributes AttributeMap` — extra order metadata.

## Line index

- 20 — `@StorageName("tc_orders")`
- 21 — `class Order extends ExtendedModel`
- 22-30 — uniqueId
- 32-40 — description
- 42-50 — fromAddress
- 52-61 — toAddress

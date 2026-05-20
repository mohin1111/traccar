# `geofence/` — Geofence geometry types

Four files providing geometric containment tests for geofences stored as WKT strings in `tc_geofences.area`.

## File index

| File | One-liner |
|---|---|
| `GeofenceGeometry.java` | Abstract base: WKT parse helpers, `containsPoint(lat, lon)`, `calculateArea()`, `toWkt()` |
| `GeofenceCircle.java` | `CIRCLE (lat lon, radius)` — custom WKT format; Haversine containment |
| `GeofencePolygon.java` | `POLYGON ((lat1 lon1, ...))` — point-in-polygon via ray casting; handles dateline crossing |
| `GeofencePolyline.java` | `LINESTRING (lat1 lon1, ...)` — contains if within a buffer distance of any segment |

## Dependency graph

```
GeofenceHandler (handler/) → reads Geofence.getArea() → GeofenceGeometry subclass
GeofenceGeometry instantiated by: Geofence.getGeometry() (model/Geofence.java)

GeofencePolygon → Spatial4j / JTS (area calculation)
GeofenceCircle → UnitsConverter (radius units)
```

## WKT format conventions

Traccar geofences are stored using a lat-first WKT convention (opposite of standard lon-first WKT):
- `CIRCLE (18.9752 72.8258, 500)` — center lat lon, radius in meters
- `POLYGON ((18.9752 72.8258, 18.9760 72.8265, 18.9745 72.8270, 18.9752 72.8258))`
- `LINESTRING (18.9752 72.8258, 18.9760 72.8265)`

**Do not pass these to standard WKT parsers** — the lat/lon order is reversed relative to GeoJSON and standard WKT.

## Containment algorithms

- **Circle**: Haversine distance from center ≤ radius.
- **Polygon**: Jordan curve theorem (ray casting) with antimeridian (dateline) normalization. Pre-computes `constant[]` and `multiple[]` arrays in constructor for O(n) per test.
- **Polyline**: minimum perpendicular distance from point to each segment ≤ buffer distance.

## Performance note

`GeofencePolygon` precomputes edge coefficients in the constructor. For high-frequency containment tests (many devices × many geofences), cache the parsed `GeofenceGeometry` objects. `CacheManager` in `session/` does this automatically.

## How to add a new geometry type

1. Extend `GeofenceGeometry`.
2. Implement `containsPointInternal(lat, lon)`, `calculateArea()`, `toWkt()`, and a constructor parsing the custom WKT prefix.
3. Add a prefix match in `Geofence.getGeometry()` factory method.
4. The `GeofenceHandler` and API code will work automatically.

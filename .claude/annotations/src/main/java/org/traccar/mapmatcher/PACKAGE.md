# `mapmatcher/` — Road snapping (map matching)

Two files providing road snapping for positions: snaps a raw GPS coordinate to the nearest road segment.

## File index

| File | One-liner |
|---|---|
| `MapMatcher.java` | Interface: `getPoint(lat, lon, MapMatcherCallback)` — async; callback returns snapped lat/lon |
| `TraccarMapMatcher.java` | HTTP client hitting the Traccar map-matching service (OSRM/Valhalla wrapper) |

## Dependency graph

```
MapMatcherHandler (handler/) → MapMatcher? (@Nullable, if configured)
TraccarMapMatcher → JAX-RS async GET → external map-matching API
```

## Configuration

```xml
<entry key="mapMatcher.enable">true</entry>
<entry key="mapMatcher.url">https://map-matcher.traccar.com/</entry>
```

The Traccar cloud map-matching service is an OSRM wrapper. Self-hosted OSRM or Valhalla can be pointed to via `mapMatcher.url`.

## When map matching fires

`MapMatcherHandler` runs in the pipeline after `GeolocationHandler` and before `DistanceHandler`. It fires for every valid position with GPS coordinates. The snapped coordinate replaces the raw GPS lat/lon in the position.

## Transport OS note

Map matching is useful for trip replay (snapping GPS points to road centerlines) but adds 10-100ms latency per position from a cloud API call. For high-frequency AIS-140 positions (30s interval), enable only if road snapping accuracy is required for reporting. Disable for raw position archive.

# `speedlimit/` — Road speed limit lookups

Three files providing road speed limit data via the Overpass API (OpenStreetMap) for use in overspeed detection.

## File index

| File | One-liner |
|---|---|
| `SpeedLimitProvider.java` | Interface: `getSpeedLimit(lat, lon, SpeedLimitProviderCallback callback)` |
| `SpeedLimitException.java` | Checked exception for provider failures |
| `OverpassSpeedLimitProvider.java` | Queries Overpass API for `way[highway][maxspeed]` within a configurable accuracy radius |

## Dependency graph

```
SpeedLimitHandler (handler/) → SpeedLimitProvider? (@Nullable, if enabled)
OverpassSpeedLimitProvider → JAX-RS async GET to Overpass API
```

## Configuration

```xml
<entry key="speedLimit.enable">true</entry>
<entry key="speedLimit.url">https://overpass-api.de/api/interpreter</entry>
<entry key="speedLimit.accuracy">25</entry>  <!-- radius in meters -->
```

## How it works

`OverpassSpeedLimitProvider` sends an Overpass QL query for OSM ways tagged `[highway][maxspeed]` within `accuracy` meters of the position coordinates. Parses the first result's `maxspeed` tag — supports `"60"` (km/h), `"40 mph"` (converts), `"30 knots"` (passes through as-is).

The returned speed limit (in knots) is stored as `position.attributes["speedLimit"]`. `OverspeedEventHandler` compares `position.getSpeed()` against this value.

## Performance note

Each position triggers an async Overpass API call. At 30-second reporting intervals for 100 devices = ~3 req/s. Public Overpass API has rate limits; for production use a self-hosted Overpass instance or cache results spatially.

## Transport OS note

Speed limits on Indian national/state highways are set by MoRTH regulations (100 km/h for cars, 80 km/h for trucks on NH), not always in OSM `maxspeed` tags. For AIS-140 overspeed reporting against regulatory limits, configure a **fixed** overspeed threshold per device attribute (`speedLimit` attribute on the device) rather than relying on Overpass data.

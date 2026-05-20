# `geocoder/` — Reverse geocoding providers

24 reverse-geocoding provider implementations that convert (lat, lon) coordinates to human-readable addresses. Injected into `GeocoderHandler` in the position pipeline and into `NotificationManager` for on-request geocoding.

## File index

| File | One-liner |
|---|---|
| `Geocoder.java` | Interface: `getAddress(lat, lon, callback)` + `setStatisticsManager()` |
| `JsonGeocoder.java` | Abstract base: async JAX-RS GET, LRU cache, stats registration |
| `Address.java` | Structured address model: house, street, suburb, settlement, district, state, country, postcode |
| `AddressFormat.java` | Configurable address format string (e.g., `%h %s, %S, %r, %c`) |
| `MapmyIndiaGeocoder.java` | **MapmyIndia reverse geocoder** — key relevance for Transport OS India |
| `NominatimGeocoder.java` | OpenStreetMap Nominatim (self-hostable, free) |
| `GoogleGeocoder.java` | Google Maps Geocoding API |
| `BingMapsGeocoder.java` | Bing Maps |
| `HereGeocoder.java` | HERE Technologies |
| `TomTomGeocoder.java` | TomTom |
| `MapboxGeocoder.java` | Mapbox |
| `MapQuestGeocoder.java` | MapQuest |
| `LocationIqGeocoder.java` | LocationIQ |
| `GeoapifyGeocoder.java` | Geoapify |
| `OpenCageGeocoder.java` | OpenCage Data |
| `GeocodeFarmGeocoder.java` | Geocode.farm |
| `GeocodeXyzGeocoder.java` | Geocode.xyz |
| `GisgraphyGeocoder.java` | Gisgraphy (self-hostable) |
| `FactualGeocoder.java` | Factual (deprecated API) |
| `BanGeocoder.java` | BAN (Base Adresse Nationale — France only) |
| `PositionStackGeocoder.java` | Positionstack |
| `PlusCodesGeocoder.java` | Google Plus Codes (Open Location Code) |
| `GeocodeJsonGeocoder.java` | GeoJSON-based providers |
| `MapTilerGeocoder.java` | MapTiler |

## Dependency graph

```
GeocoderHandler (handler/) → Geocoder (bound instance)
NotificationManager (database/) → Geocoder? (@Nullable, on request)

Geocoder implementations all extend JsonGeocoder
 └── JsonGeocoder → JAX-RS Client + Caffeine LRU cache + StatisticsManager
```

## Provider selection

Set in `traccar.xml`:
```xml
<entry key="geocoder.enable">true</entry>
<entry key="geocoder.type">mapmyindia</entry>   <!-- or google, nominatim, mapbox, etc. -->
<entry key="geocoder.key">YOUR_API_KEY</entry>
<entry key="geocoder.cacheSize">512</entry>
<entry key="geocoder.onRequest">false</entry>   <!-- false=inline, true=deferred -->
```

## Cache behavior (`JsonGeocoder`)

- Caffeine `maximumSize(cacheSize)` LRU cache keyed by `"lat,lon"` string.
- `cacheSize=0` disables caching.
- Cache is in-memory only; cleared on restart.
- Async callback version (`callback != null`) returns cached result synchronously if available.

## `geocodeOnRequest` vs inline

- `false` (default): geocoding happens inline in `GeocoderHandler` during position processing — adds latency to the pipeline but ensures address is always available.
- `true`: geocoding deferred — `NotificationManager` geocodes on-demand before notification delivery only. Pipeline latency unaffected but addresses not stored in positions.

## MapmyIndia for Transport OS

`MapmyIndiaGeocoder` hits `https://apis.mapmyindia.com/<key>/rev_geocode?lat=%f&lng=%f`. Parses `results[0]` for `formatted_address`, `house_number`/`house_name`, `street`, `locality`/`sublocality`, `city`/`village`, `district`/`subDistrict`, `state`, `pincode`.

Configure: `geocoder.type=mapmyindia`, `geocoder.key=<MapmyIndia API key>`. Best accuracy for Indian addresses including pincode (essential for AIS-140 compliance reporting).

## How to add a new geocoder

1. Extend `JsonGeocoder`.
2. Pass URL template with `%f`/`%f` placeholders for lat/lon to `super()`.
3. Implement `parseAddress(JsonObject json)` → return `Address`.
4. Add `case "myprovider":` in `MainModule` geocoder binding switch.
5. Add `Keys.GEOCODER_*` config keys if the provider needs non-standard parameters.

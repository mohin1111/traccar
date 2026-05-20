# `geolocation/` — Cell tower / Wi-Fi geolocation providers

Four files that resolve a device's position from cell tower and Wi-Fi access point data when GPS is unavailable. Used by `GeolocationHandler` in the position pipeline.

## File index

| File | One-liner |
|---|---|
| `GeolocationProvider.java` | Interface: `getLocation(Network network, LocationProviderCallback callback)` |
| `GeolocationException.java` | Checked exception for provider failures |
| `GoogleGeolocationProvider.java` | Google Maps Geolocation API (cell + Wi-Fi, requires Maps Platform key) |
| `OpenCellIdGeolocationProvider.java` | OpenCelliD API (cell towers only, free tier available) |
| `UniversalGeolocationProvider.java` | Universal Geolocation API (cell + Wi-Fi) |
| `UnwiredGeolocationProvider.java` | Unwired Labs / HERE geolocation (cell + Wi-Fi) |

## Dependency graph

```
GeolocationHandler (handler/) → GeolocationProvider (bound instance)
 └── GeolocationProvider → JAX-RS async HTTP + StatisticsManager.registerGeolocationRequest()

Network (model/) → CellTower[] + WifiAccessPoint[] → passed to provider
```

## Configuration

```xml
<entry key="geolocation.enable">true</entry>
<entry key="geolocation.type">google</entry>   <!-- or opencellid, universal, unwired -->
<entry key="geolocation.key">YOUR_API_KEY</entry>
<entry key="geolocation.url">https://custom-endpoint/</entry>  <!-- optional override -->
```

## When geolocation fires

`GeolocationHandler` runs in the position pipeline (step 4 of `ProcessingHandler`). It triggers when:
- `position.getLatitude() == 0 && position.getLongitude() == 0` (no GPS fix), AND
- `position.getNetwork()` contains at least one cell tower or Wi-Fi AP.

The handler calls `provider.getLocation(network, callback)` asynchronously; the callback updates the position coordinates and accuracy, then continues pipeline processing.

## AIS-140 relevance

AIS-140 VLT devices report cell tower data (`CellTower` with MCC=404/405 for India, MNC, LAC, CellId). When GPS signal is lost (tunnel, urban canyon), enabling `geolocation` with `opencellid` or `google` provides cell-based location fallback. Essential for continuous tracking compliance in Indian highways.

## How to add a new geolocation provider

1. Implement `GeolocationProvider`.
2. POST or GET the JSON payload from `Network` (cell towers + Wi-Fi) to the provider API.
3. On success call `callback.onSuccess(lat, lon, accuracy)`.
4. Add `case "myprovider":` in `MainModule` geolocation binding switch.

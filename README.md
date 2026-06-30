# homebridge-http-climate

A Homebridge plugin that polls an HTTP/HTTPS endpoint and exposes **temperature and humidity** as HomeKit sensors — both from the same request.

Forked from [homebridge-http-temperature](https://github.com/metbosch/homebridge-http-temperature) by @metbosch.

## Installation

```bash
npm install -g homebridge-http-climate
```

Or directly from GitHub:

```bash
npm install -g git+https://github.com/Kohlibrie/homebridge-http-climate.git
```

Then add an `HttpClimate` accessory to your Homebridge `config.json`.

## Configuration

| Field | Required | Default | Description |
|---|---|---|---|
| `url` | yes | — | Endpoint URL (must start with `http://` or `https://`) |
| `name` | yes | — | Accessory name shown in Home app |
| `field_name` | no | `"temperature"` | Dot-path into JSON response for temperature. Empty string `""` means the raw body is the value. |
| `humidity_field_name` | no | — | Dot-path into JSON response for humidity (0–100). If omitted, no humidity sensor is created. |
| `units` | no | `"C"` | Temperature units returned by the endpoint: `"C"` or `"F"` (auto-converted to Celsius for HomeKit) |
| `update_interval` | no | `120000` | Polling interval in ms. Set to `0` to only update on HomeKit request. |
| `timeout` | no | `5000` | Request timeout in ms |
| `auth` | no | — | Object with `user` and `pass` for HTTP Basic Auth |
| `http_headers` | no | — | Object with custom request headers |
| `http_method` | no | `"GET"` | HTTP method |
| `http_protocol` | no | inferred from URL | Force `"http"` or `"https"` |
| `min_temp` | no | `-100` | Minimum expected temperature (out-of-bounds values are rejected) |
| `max_temp` | no | `130` | Maximum expected temperature |
| `debug` | no | `false` | Enable verbose logging |

## Examples

### Temperature + Humidity from the same endpoint

Endpoint response:
```json
{
  "humidity": "41",
  "temperature": "22.5"
}
```

Config:
```json
{
  "accessory": "HttpClimate",
  "name": "Outside",
  "url": "http://192.168.1.100/sensor",
  "field_name": "temperature",
  "humidity_field_name": "humidity"
}
```

Apple Home will show two tiles under the same accessory — a thermometer and a humidity sensor.

### Nested JSON path

Endpoint response:
```json
{
  "value": {
    "temp_int": 22.5,
    "humidity": 41
  }
}
```

Config:
```json
{
  "accessory": "HttpClimate",
  "name": "Outside",
  "url": "http://192.168.1.100/sensor",
  "field_name": "value.temp_int",
  "humidity_field_name": "value.humidity"
}
```

### Temperature only (backward compatible)

```json
{
  "accessory": "HttpClimate",
  "name": "Outside",
  "url": "http://192.168.1.100/sensor",
  "field_name": "temperature"
}
```

### Raw body as temperature value

When the endpoint returns just a plain number (e.g. `22.5`):

```json
{
  "accessory": "HttpClimate",
  "name": "Outside",
  "url": "http://192.168.1.100/temp",
  "field_name": ""
}
```

### HTTPS with Basic Auth

```json
{
  "accessory": "HttpClimate",
  "name": "Outside",
  "url": "https://192.168.1.100/sensor",
  "field_name": "temperature",
  "humidity_field_name": "humidity",
  "auth": {
    "user": "admin",
    "pass": "secret"
  }
}
```

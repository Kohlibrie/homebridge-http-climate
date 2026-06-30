# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is a Homebridge accessory plugin (`homebridge-http-climate`) that polls a single HTTP/HTTPS endpoint for climate data and exposes it as HomeKit `TemperatureSensor` and optionally `HumiditySensor` accessories. The entire plugin lives in a single file: `index.js`.

There are no build steps, no test suite, and no npm scripts. The plugin is consumed by homebridge at runtime via `require()`.

## Installation & Development

```bash
npm install

# Install globally for use with a local homebridge instance
npm install -g homebridge
npm install -g .
```

To test changes, restart homebridge with a `config.json` that includes an `HttpTemperature` accessory entry (see `sample-config.json` for the bridge/accessory structure).

## Architecture

The plugin exports a single factory function that homebridge calls with its `homebridge` object:

```js
module.exports = function (homebridge) { ... }
```

This factory captures `Service` and `Characteristic` from `homebridge.hap` and registers the `HttpClimate` accessory class.

**`HttpTemperature` constructor** — reads all config fields and builds `request_opts` for Node's `http`/`https` module. Protocol is inferred from the URL unless `http_protocol` is set explicitly (known bug: the code reads `config.protocol` instead of `config.http_protocol` when `config.http_protocol` is truthy).

**Key prototype methods:**

- `getServices()` — called once by homebridge at startup. Creates `AccessoryInformation` and `TemperatureSensor` services, wires `getState` to `CurrentTemperature`'s `get` event. If `humidity_field_name` is configured, also creates a `HumiditySensor` service wired to `getHumidityState`. Starts `setInterval` polling if `update_interval > 0`.
- `updateState()` — makes the HTTP request and stores the result as a `Promise<{temperature, humidity}>` in `this.last_value`. Uses `this.waiting_response` to skip overlapping requests. On success, pushes values and `NO_FAULT` to characteristics; on error, sets `GENERAL_FAULT`.
- `getState(callback)` — triggers `updateState()` then resolves `this.last_value` to call the HAP callback with `data.temperature`.
- `getHumidityState(callback)` — triggers `updateState()` (reuses any in-flight request) then resolves `this.last_value` to call the HAP callback with `data.humidity`.
- `getFromObject(obj, path, def)` — resolves dot-and-bracket-notation paths (e.g. `values.[0].temp2`) into nested JSON objects.

**Data flow (single HTTP request, both sensors):**

1. HTTP response body received
2. JSON parsed once if either `field_name` or `humidity_field_name` is a non-empty path
3. Temperature: extract via `field_name` path (or raw body if `""`), strip non-numeric chars, parse as `Number`, check bounds, convert F→C if needed
4. Humidity: extract via `humidity_field_name` path (or raw body if `""`), strip non-numeric chars, parse as `Number`, clamp to 0–100
5. Both values pushed to their respective characteristics

**`field_name` / `humidity_field_name` behavior:** empty string `""` means the raw response body is the value; any non-empty string is treated as a dot-path into the parsed JSON. Omitting `humidity_field_name` entirely disables the humidity sensor.

## Config Reference

Mandatory: `url`, `name`.

| Field | Default | Description |
|---|---|---|
| `field_name` | `"temperature"` | Dot-path for temperature in JSON response; `""` uses raw body |
| `humidity_field_name` | — | Dot-path for humidity; omit to disable humidity sensor |
| `units` | `"C"` | `"C"` or `"F"` — endpoint's units; `"F"` is converted to Celsius for HomeKit |
| `update_interval` | `120000` | Polling interval in ms; `0` disables polling |
| `timeout` | `5000` | Request timeout in ms |
| `auth` | — | `{user, pass}` for HTTP Basic Auth |
| `http_headers` | — | Object of custom request headers |
| `http_method` | `"GET"` | HTTP method |
| `min_temp` / `max_temp` | `-100` / `130` | Out-of-bounds temperature values are rejected with an error |
| `debug` | `false` | Enable verbose logging |

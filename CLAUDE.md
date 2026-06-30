# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is a Homebridge accessory plugin (`homebridge-http-temperature`) that polls an HTTP/HTTPS endpoint for temperature data and exposes it as a HomeKit `TemperatureSensor`. The entire plugin lives in a single file: `index.js`.

There are no build steps, no test suite, and no npm scripts. The plugin is consumed by homebridge at runtime via `require()`.

## Installation & Development

```bash
# Install dependencies (none beyond Node built-ins; package-lock exists for lockfile consistency)
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

This factory captures `Service` and `Characteristic` from `homebridge.hap` and registers the `HttpTemperature` accessory class.

**`HttpTemperature` constructor** — reads all config fields and builds `request_opts` for Node's `http`/`https` module. Protocol is inferred from the URL unless `http_protocol` is set explicitly (note: the code reads `config.protocol` when `config.http_protocol` is truthy — this is a known bug in the existing code).

**Key prototype methods:**

- `getServices()` — called once by homebridge at startup. Creates two HAP services (`AccessoryInformation`, `TemperatureSensor`), wires `getState` to the `CurrentTemperature` characteristic's `get` event, and optionally starts a polling interval via `setInterval`.
- `updateState()` — makes the HTTP request; stores the result as a `Promise` in `this.last_value`. Uses `this.waiting_response` to skip overlapping requests. On success, pushes the value and `NO_FAULT` to the characteristics; on error, sets `GENERAL_FAULT`.
- `getState(callback)` — called by HomeKit on demand. Triggers `updateState()` then resolves `this.last_value` to call the HAP callback.
- `getFromObject(obj, path, def)` — resolves dot-and-bracket-notation paths (e.g. `values.[0].temp2`) into nested JSON objects.

**Data flow:** HTTP response body → strip non-numeric characters → `Number()` → bounds check → optional Fahrenheit→Celsius conversion → pushed to `Characteristic.CurrentTemperature`.

**`field_name` behavior:** empty string means the raw body is the temperature value; any non-empty string is treated as a dot-path into the parsed JSON.

## Config Reference

Mandatory: `url`, `name`.  
Key optional fields: `field_name` (default `"temperature"`), `units` (`"C"`/`"F"`, default `"C"`), `update_interval` (ms, default `120000`; set to `0` to disable polling), `timeout` (ms, default `5000`), `auth` (`{user, pass}`), `http_headers`, `http_method`, `min_temp`, `max_temp`.

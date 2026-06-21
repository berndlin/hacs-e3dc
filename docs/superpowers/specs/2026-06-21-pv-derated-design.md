# PV Derated Binary Sensor Design

## Goal

Expose the E3DC `pvDerated` system-status flag as a regular Home Assistant
binary sensor so it can be used directly in automations.

## Current Context

- `python-e3dc` already exposes `get_system_status()`.
- This integration already queries `get_system_status()` in diagnostics.
- Runtime polling currently does not move any system-status bits into
  `coordinator.data`.
- `binary_sensor.py` already builds entities from coordinator-backed
  description entries.

## Approved Approach

### Data flow

Add a new `E3DCProxy.get_system_status()` method in
`custom_components/e3dc_rscp/e3dc_proxy.py`. The coordinator will call it
through `hass.async_add_executor_job()` during the normal update cycle and store
`pvDerated` as `coordinator.data["system-pv-derated"]`.

### Entity model

Add a new binary sensor description in
`custom_components/e3dc_rscp/binary_sensor.py` with:

- `key="system-pv-derated"`
- `translation_key="system-pv-derated"`
- `device_class=BinarySensorDeviceClass.RUNNING`
- solar-themed on/off icons matching the existing binary-sensor style

The visible entity name will be `PV derated`, following the repo's existing
English naming and capitalization patterns.

### Diagnostics

Because this adds a new coordinator data key, `custom_components/e3dc_rscp/diagnostics.py`
must expose that key in the diagnostics dump in the same change.

### Translations

Add the new translation key to both:

- `custom_components/e3dc_rscp/strings.json`
- `custom_components/e3dc_rscp/translations/en.json`

## Error Handling

- Proxy access remains wrapped by `@e3dc_call`.
- Coordinator loader should mirror the existing pattern: log a warning on
  `HomeAssistantError` and keep previous coordinator state untouched for this
  key on failed refreshes.

## Verification

Use the locally available verification surface for this repo:

- Python syntax compilation for changed Python files
- JSON validity check for changed translation files
- Git diff review to confirm naming and wiring consistency

There is no existing test suite in this repository, so this change will rely on
those checks plus Home Assistant runtime validation by the user.

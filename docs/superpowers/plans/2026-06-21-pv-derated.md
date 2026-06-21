# PV Derated Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Expose E3DC `pvDerated` as a coordinator-backed Home Assistant binary sensor.

**Architecture:** Add a proxy method for `get_system_status()`, load `pvDerated` during normal coordinator refreshes, surface it through a new binary-sensor description, and register the new coordinator key in diagnostics and English translations.

**Tech Stack:** Python 3.11, Home Assistant `DataUpdateCoordinator`, `python-e3dc`, JSON translations

---

### Task 1: Add runtime access to the system-status flag

**Files:**
- Modify: `custom_components/e3dc_rscp/e3dc_proxy.py`
- Modify: `custom_components/e3dc_rscp/coordinator.py`

- [ ] **Step 1: Add the proxy method**

```python
    @e3dc_call
    def get_system_status(self) -> dict[str, Any]:
        """Load E3DC system status flags."""
        return self.e3dc.get_system_status(keepAlive=True)
```

- [ ] **Step 2: Add the coordinator loader**

```python
    async def _load_and_process_system_status(self) -> None:
        """Load and process E3DC system status flags."""
        try:
            system_status: dict[str, Any] = await self.hass.async_add_executor_job(
                self.proxy.get_system_status
            )
        except HomeAssistantError as ex:
            _LOGGER.warning("Failed to load system status, not updating data: %s", ex)
            return

        self._mydata["system-pv-derated"] = bool(system_status["pvDerated"])
```

- [ ] **Step 3: Call the loader from the update cycle**

```python
        _LOGGER.debug("Polling general status information")
        await self._load_and_process_poll()

        _LOGGER.debug("Polling system status information")
        await self._load_and_process_system_status()
```

### Task 2: Add the entity and translation keys

**Files:**
- Modify: `custom_components/e3dc_rscp/binary_sensor.py`
- Modify: `custom_components/e3dc_rscp/strings.json`
- Modify: `custom_components/e3dc_rscp/translations/en.json`

- [ ] **Step 1: Add the binary sensor description**

```python
    E3DCBinarySensorEntityDescription(
        key="system-pv-derated",
        translation_key="system-pv-derated",
        device_class=BinarySensorDeviceClass.RUNNING,
        on_icon="mdi:solar-power-variant",
        off_icon="mdi:solar-power-variant-outline",
    ),
```

- [ ] **Step 2: Add English translation entries**

```json
"system-pv-derated": {
  "name": "PV derated"
}
```

### Task 3: Keep diagnostics in sync

**Files:**
- Modify: `custom_components/e3dc_rscp/diagnostics.py`

- [ ] **Step 1: Add the new coordinator data key to diagnostics**

```python
            "current_data": self.coordinator.data,
            "current_system_pv_derated": self.coordinator.data.get(
                "system-pv-derated"
            ),
```

### Task 4: Verify the patch

**Files:**
- Verify: `custom_components/e3dc_rscp/e3dc_proxy.py`
- Verify: `custom_components/e3dc_rscp/coordinator.py`
- Verify: `custom_components/e3dc_rscp/binary_sensor.py`
- Verify: `custom_components/e3dc_rscp/diagnostics.py`
- Verify: `custom_components/e3dc_rscp/strings.json`
- Verify: `custom_components/e3dc_rscp/translations/en.json`

- [ ] **Step 1: Run Python syntax compilation**

```powershell
& 'C:\Users\Ad\AppData\Local\Programs\Python\Python313\python.exe' -m py_compile `
  'custom_components\e3dc_rscp\e3dc_proxy.py' `
  'custom_components\e3dc_rscp\coordinator.py' `
  'custom_components\e3dc_rscp\binary_sensor.py' `
  'custom_components\e3dc_rscp\diagnostics.py'
```

- [ ] **Step 2: Run JSON validation**

```powershell
& 'C:\Users\Ad\AppData\Local\Programs\Python\Python313\python.exe' -c "import json, pathlib; [json.loads(pathlib.Path(p).read_text(encoding='utf-8')) for p in ['custom_components/e3dc_rscp/strings.json', 'custom_components/e3dc_rscp/translations/en.json']]; print('json ok')"
```

- [ ] **Step 3: Review the diff**

```powershell
git diff -- custom_components/e3dc_rscp/e3dc_proxy.py custom_components/e3dc_rscp/coordinator.py custom_components/e3dc_rscp/binary_sensor.py custom_components/e3dc_rscp/diagnostics.py custom_components/e3dc_rscp/strings.json custom_components/e3dc_rscp/translations/en.json
```

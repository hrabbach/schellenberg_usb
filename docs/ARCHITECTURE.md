<!-- generated-by: gsd-doc-writer -->
# Architecture

## System Overview

The Schellenberg USB Integration is a Home Assistant custom component that bridges
Schellenberg roller-shutter motors to the HA platform over a USB Funk-Stick. The stick
connects to the host via a serial port at a fixed 112500 bps baud rate and speaks a
proprietary binary-text protocol. The integration exposes each paired motor as a
`cover` entity with time-based position tracking, the USB stick itself as three `sensor`
entities (connection status, firmware version, operating mode), and an LED switch. All
I/O is asynchronous — there is no polling; entities update via HA's dispatcher mechanism.

The system is designed around a fundamental hardware constraint: non-bidirectional
("timed") motors send no confirmation of movement. Control and position tracking for
those motors are purely time-based, using `time.monotonic()` and pre-measured travel
times. Bidirectional motors transmit `ss`-prefix frames back to the stick, which the
integration uses for event-driven state updates.

---

## Component Map

```
┌─────────────────────────────────────────────────────────┐
│  Home Assistant UI / REST API                           │
└────────────────────┬────────────────────────────────────┘
                     │ config_entries / entity platform
┌────────────────────▼────────────────────────────────────┐
│  __init__.py                                            │
│  async_setup_entry — creates SchellenbergUsbApi,        │
│  stores in entry.runtime_data, forwards to platforms,   │
│  tracks subentry additions, live-applies hub options    │
└────┬──────────────────────────────────────┬─────────────┘
     │ entry.runtime_data (api)             │ async_forward_entry_setups
     ▼                                      ▼
┌──────────────────┐          ┌─────────────────────────────────┐
│  api.py          │          │  cover.py / sensor.py / switch.py│
│  SchellenbergUsbApi         │  SchellenbergCover              │
│  SchellenbergProtocol       │  SchellenbergConnectionSensor   │
│                  │          │  SchellenbergVersionSensor      │
│  serial link     │          │  SchellenbergModeSensor         │
│  112500 bps      │          │  SchellenbergLedSwitch          │
└────────┬─────────┘          └────────────┬────────────────────┘
         │ async_dispatcher_send           │ async_dispatcher_connect
         └──────────────┬──────────────────┘
                        │ HA dispatcher bus
              SIGNAL_DEVICE_EVENT_{device_id}
              SIGNAL_STICK_STATUS_UPDATED
              SIGNAL_CALIBRATION_COMPLETED
```

**Config flow tree:**

```
SchellenbergUsbConfigFlow (config_flow.py)
├── async_step_user          — manual serial port entry
├── async_step_usb           — USB auto-discovery (VID 16C0 / PID 05E1)
└── SchellenbergPairingSubentryFlow
    ├── async_step_menu      — pair / manual_add choice
    ├── async_step_pair      — auto-pair via stick
    ├── async_step_manual_add — enum + mode entry
    ├── async_step_manual_position — initial position for timed motors
    ├── async_step_reconfigure — routes by motor type (bidir vs timed)
    ├── CalibrationFlowHandler (options_flow_calibration.py)
    │   — event-driven calibration for bidirectional motors
    └── TimedCalibrationFlowHandler (options_flow_timed_calibration.py)
        — button-press timing for non-bidirectional (timed) motors
```

---

## Component Responsibilities

| Module | Class / Function | Responsibility |
|---|---|---|
| `api.py` | `SchellenbergUsbApi` | Serial connection lifecycle, command transmission, stick-busy retry queue, device registry (`_registered_devices`), pairing coordination, futures for async serial responses |
| `api.py` | `SchellenbergProtocol` | `asyncio.Protocol` subclass; buffers incoming bytes, splits on `\n`, dispatches complete lines to `SchellenbergUsbApi._handle_message` |
| `__init__.py` | `async_setup_entry` | Creates `SchellenbergUsbApi`, stores it in `entry.runtime_data`, bootstraps hub subentry, forwards to platforms, registers `_on_entry_updated` to detect subentry changes |
| `cover.py` | `SchellenbergCover` | HA `CoverEntity` + `RestoreEntity`; open/close/stop/set-position; position tracking loop (200 ms tick, 1 s HA push); bidirectional vs timed branching; calibration persistence |
| `cover.py` | `_get_cal_store` / `_save_calibration` | HA `Store` wrapper for `.storage/schellenberg_usb_calibration`; shared across all cover entities via `hass.data` |
| `sensor.py` | `SchellenbergConnectionSensor` / `SchellenbergVersionSensor` / `SchellenbergModeSensor` | Expose `api.is_connected`, `api.device_version`, `api.device_mode`; update on `SIGNAL_STICK_STATUS_UPDATED` |
| `switch.py` | `SchellenbergLedSwitch` | LED on/off/blink by delegating to `api.led_on()` / `api.led_off()` |
| `config_flow.py` | `SchellenbergUsbConfigFlow` | Hub config flow: manual serial port entry + USB auto-discovery |
| `config_flow.py` | `SchellenbergPairingSubentryFlow` | Subentry flow for blind devices; delegates calibration steps to handler classes |
| `options_flow.py` | `SchellenbergOptionsFlowHandler` | Hub options: change serial port, toggle `ignore_unknown`; port change triggers reload, toggle is live-applied without reload |
| `options_flow_calibration.py` | `CalibrationFlowHandler` | Event-driven calibration for bidirectional motors: waits for `SIGNAL_DEVICE_EVENT_{id}` start/stop events; emits `SIGNAL_CALIBRATION_COMPLETED` with `final_position=0` |
| `options_flow_timed_calibration.py` | `TimedCalibrationFlowHandler` | Button-press timing calibration for non-bidirectional motors: sends drive command, user presses a form button when motor reaches endstop, records `time.monotonic()` delta; emits `SIGNAL_CALIBRATION_COMPLETED` with `final_position=100` |
| `const.py` | constants | `DOMAIN`, `CMD_*`, `CONF_*`, `SIGNAL_*` strings, `SchellenbergConfigEntry` type alias, calibration guard constants |

---

## Serial Protocol Layer

### Physical link

- **Baud rate:** 112500 bps (fixed; not configurable)
- **USB device:** VID 16C0, PID 05E1, manufacturer "van ooijen" (Schellenberg USB Funk-Stick)
- **Framing:** newline-terminated ASCII lines

### Connection lifecycle (`api.py:SchellenbergUsbApi.connect`)

1. `serial_asyncio.create_serial_connection` creates a `SchellenbergProtocol` instance.
2. `verify_device()` sends `!?` (`CMD_VERIFY`) and awaits an `RFTU_V*` response via `_verify_future` (timeout: `VERIFY_TIMEOUT` = 5 s).
3. If mode is not `listening`, a lowercase command (`hello`) is sent to enter listening mode (B:2).
4. `get_device_id()` sends `sr` (`CMD_GET_DEVICE_ID`) and awaits an `sr{6-char-id}` response via `_device_id_future`.
5. On `SerialException`/`OSError`, retry is scheduled via `hass.loop.call_later(5, ...)`.

### Message parsing (`api.py:SchellenbergUsbApi._handle_message`)

| Prefix | Format | Action |
|---|---|---|
| `RFTU_` | `RFTU_V20 F:<date> B:<mode>` | Sets `_device_version`, `_device_mode`; resolves `_verify_future`; fires `SIGNAL_STICK_STATUS_UPDATED` |
| `t1` / `t0` | — | Transmit ACK; ignored |
| `tE` | — | Stick busy; schedules `_retry_command_after_delay()` (100 ms) to resend `_pending_retry_command` |
| `sr{6}` | `sr5D3E7C` | Device ID response; resolves `_device_id_future` |
| `sl{...}` | `sl00BE{6-char-id}...` | Pairing/list response; device ID extracted at `[6:12]`; resolves `_pairing_future` during pairing |
| `ss{...}` | `ss{enum:2}{device_id:6}{incr:4}{cmd:2}{pad:2}{rssi:2}` | Inbound device event; device enum at `[2:4]`, device ID at `[4:10]`, command at `[14:16]`; dispatches `SIGNAL_DEVICE_EVENT_{device_id}` |

### Outbound command format

All device control commands use the `CMD_TRANSMIT` prefix (`ss`):

```
ss{device_enum:2}{repeat:1}{command:2}{padding:4}
```

Example — open blind with enum `10`:
```
ss109010000
```

Literal command values (from `const.py`):

| Constant | Value | Meaning |
|---|---|---|
| `CMD_STOP` | `00` | Stop |
| `CMD_UP` | `01` | Open (up) |
| `CMD_DOWN` | `02` | Close (down) |
| `CMD_PAIR` | `60` | Pair with device |
| `CMD_SET_UPPER_ENDPOINT` | `61` | Set upper travel endpoint |
| `CMD_SET_LOWER_ENDPOINT` | `62` | Set lower travel endpoint |
| `CMD_ALLOW_PAIRING` | `40` | Make device accept new remote |
| `CMD_MANUAL_UP` | `41` | Hold-up (button simulation) |
| `CMD_MANUAL_DOWN` | `42` | Hold-down (button simulation) |

Stick system commands are uppercase with `!` prefix: `!?` (verify), `!B` (bootloader), `!G` (initial), `!R` (reboot). Lowercase commands control the stick itself: `so+`/`so-` (LED on/off), `so1`–`so9` (LED blink), `sr` (get device ID), `sp` (enter pairing mode).

---

## Dispatcher Signal Flow

The integration uses HA's `async_dispatcher_send` / `async_dispatcher_connect` for decoupled intra-process communication. No external message bus is used.

### Signals defined in `const.py`

| Signal | Sender | Receivers | Payload |
|---|---|---|---|
| `SIGNAL_DEVICE_EVENT_{device_id}` | `SchellenbergUsbApi._handle_message` | `SchellenbergCover._handle_event` | `command: str` (e.g., `"01"`, `"02"`, `"00"`) |
| `SIGNAL_STICK_STATUS_UPDATED` | `SchellenbergUsbApi._update_status` | `SchellenbergBaseSensor._handle_status_update`, `SchellenbergCover._handle_status_update` | (no payload) |
| `SIGNAL_CALIBRATION_COMPLETED` | `CalibrationFlowHandler._save_calibration_data`, `TimedCalibrationFlowHandler._emit_calibration_signal` | `SchellenbergCover._handle_calibration_completed` | `device_id, open_time, close_time, final_position` |

### Signal routing detail

`SIGNAL_DEVICE_EVENT_{device_id}` is a per-device signal string — the device ID is
embedded in the signal name (`f"{SIGNAL_DEVICE_EVENT}_{device_id}"`). Each
`SchellenbergCover` subscribes on `async_added_to_hass` and unsubscribes via
`async_on_remove`. Timed motor entities subscribe but immediately return without
side-effects when `_is_bidirectional` is `False` (guard in `_handle_event`).

`SIGNAL_CALIBRATION_COMPLETED` is broadcast to all cover entities; each entity
filters on the `device_id` argument in `_handle_calibration_completed`.

---

## Bidirectional vs Timed Motor Control

The `CONF_BIDIRECTIONAL` flag (stored in `ConfigSubentry.data`) governs which code
path is active for a given motor. Default is `True` so legacy auto-paired subentries
without the key are treated as bidirectional.

### Bidirectional motors

- Transmit inbound `ss`-frame events on movement start (`01`), stop (`00`), close (`02`).
- `SchellenbergCover._handle_event` reacts to these events to set `_attr_is_opening`,
  `_attr_is_closing`, start the position-tracking loop, and snap position on stop.
- Calibration uses `CalibrationFlowHandler`, which subscribes to
  `SIGNAL_DEVICE_EVENT_{device_id}` to detect movement start and stop events, then
  measures elapsed `time.time()` between them.
- `set_cover_position` is always available.

### Timed (non-bidirectional) motors

- Produce no inbound frames. The `_handle_event` guard returns immediately without
  mutating state.
- Movement is initiated by `async_open_cover` / `async_close_cover` calling
  `api.control_blind()`. Position is computed entirely from `time.monotonic()` delta
  and the stored travel times.
- `set_cover_position` requires `_is_calibrated` to be `True`; uncalibrated timed
  motors ignore the command.
- Restart behaviour: if the last persisted state was `opening`, position snaps to
  100%; if `closing`, position snaps to 0%; idle states restore from `RestoreEntity`.
  If no prior state exists, position defaults to 100% (assume open).
- Calibration uses `TimedCalibrationFlowHandler` — event-free, pure form-button
  timing (see Calibration section below).

### Position tracking loop (`cover.py:SchellenbergCover._async_position_update_loop`)

Both motor types share the same loop once movement starts:

1. Wakes every 200 ms (`asyncio.sleep(0.2)`).
2. Calls `_update_position()`: `new_pos = start_pos ± (elapsed / travel_time) * 100`.
3. If `_target_position` is set and the computed position reaches it:
   - Sends `CMD_STOP` if target is not 0 or 100 (endstops auto-stop).
   - Clears all movement state.
4. Reports state to HA every 1 s (every 5 ticks).
5. Terminates when position reaches 0% or 100% without a partial target.

---

## Calibration Persistence

Calibration data (open and close travel times in seconds) is stored in
`.storage/schellenberg_usb_calibration` via HA's `Store` API.

### Store structure

```json
{
  "<config_entry_id>": {
    "<device_id>": {
      "open_time": 25.40,
      "close_time": 23.15
    }
  }
}
```

### Load path (`cover.py:async_setup_entry`)

1. `_get_cal_store(hass)` initializes a single `Store` instance per HA session
   (cached in `hass.data[_HASS_DATA_KEY]`).
2. Calibration data is merged into `device_data` using `setdefault` — subentry data
   wins over persisted data; persisted data fills in gaps.
3. `SchellenbergCover.__init__` treats `None` or `0.0` travel times as uncalibrated
   and falls back to `DEFAULT_TRAVEL_TIME` (60 s) for the position computation.
   `_is_calibrated` is `False` if either time is `None`.

### Save path

- **Bidirectional path:** `CalibrationFlowHandler._save_calibration_data` writes to
  the legacy `schellenberg_usb_devices` Store and then dispatches
  `SIGNAL_CALIBRATION_COMPLETED`. The cover's `_handle_calibration_completed`
  callback calls `_save_calibration` to also write to the calibration Store.
- **Timed path:** `TimedCalibrationFlowHandler._emit_calibration_signal` dispatches
  `SIGNAL_CALIBRATION_COMPLETED` directly. The cover callback writes to the
  calibration Store.

Both paths pass `(device_id, open_time, close_time, final_position)` on the signal.
`final_position=0` for the bidirectional flow (ends on a close run);
`final_position=100` for the timed flow (ends on an open run, motor at top).

---

## Timed Calibration Flow (`options_flow_timed_calibration.py`)

The `TimedCalibrationFlowHandler` is used for non-bidirectional motors that cannot
report movement events. It is entered via `async_step_reconfigure` when
`CONF_BIDIRECTIONAL` is `False`.

**Flow steps:**

1. `timed_cal_precondition` — Instruction screen; user confirms shutter is fully open. No command sent.
2. `timed_cal_close` — Sends `CMD_DOWN` via `api.control_blind()`, records
   `time.monotonic()` before the `await`. Shows a form. On next submit, records elapsed time.
   - Rejects `elapsed < CAL_MIN_TRAVEL_TIME` (2 s) as a misfire.
   - Rejects `elapsed > CAL_MAX_TRAVEL_TIME` (120 s) as a "walked away" run.
3. `timed_cal_open` — Sends `CMD_UP`, records start time, shows a form. On next submit,
   records elapsed open time with the same guards.
4. `timed_cal_confirm` — Shows measured times. User may confirm or redo. On confirm,
   emits `SIGNAL_CALIBRATION_COMPLETED` with `final_position=100`.

No `CMD_STOP` is ever issued by this handler — motors run to their physical endstops.
`time.monotonic()` is captured before each `await` to avoid inflating timing with
coroutine scheduling latency.

---

## Entry Hierarchy and Device Registry

```
ConfigEntry (hub)
│   data: {serial_port: "/dev/ttyUSB0"}
│   runtime_data: SchellenbergUsbApi
│
├── ConfigSubentry (type: "hub")
│   └── Device: "Schellenberg USB Stick"
│       └── Entities: connection sensor, version sensor, mode sensor, LED switch
│
├── ConfigSubentry (type: "blind", for each paired motor)
│   │   data: {device_id, device_enum, bidirectional, [open_time, close_time], [initial_position]}
│   └── Device: "{device_name}"
│       └── Entity: SchellenbergCover
```

The hub subentry is created automatically on first `async_setup_entry` to keep
hub-level entities (sensors, LED switch) grouped under the hub device. Blind
subentries are created by `SchellenbergPairingSubentryFlow` after pairing or manual
add. When subentries change, `_on_entry_updated` in `__init__.py` detects the diff
via `_SETUP_CALLBACKS[entry_id]["subentry_ids"]` and reloads the config entry.

---

## Key Constraints and Anti-Patterns

### Serial port sanity check is blocking

`config_flow.py` and `options_flow.py` open the serial port with the blocking
`serial.Serial(port)` call to validate connectivity before creating/updating the
entry. This is intentional (documented with `# NOTE: blocking open used only to
sanity-check connectivity`) but means the HA event loop is briefly blocked during
flow validation.

### Device enumerators are allocated sequentially

`api.initialize_next_device_enum()` scans `_registered_devices.values()` for the
highest existing hex enum and adds 1. Enumerators start at `PAIRING_DEVICE_ENUM_START`
(0x10) and are capped at 0xFF with wrap-around.

### Stick-busy retry

When the stick responds `tE`, the last command is stored in `_pending_retry_command`
and re-sent after 100 ms. Only one pending retry exists at a time; a new `tE` cancels
any in-flight retry task before scheduling a fresh one.

### Ignore unknown signals

The `CONF_IGNORE_UNKNOWN` hub option demotes log lines for unknown device IDs from
`WARNING` to `DEBUG`. It is live-applied to `api.ignore_unknown` without a reload
when the port path is unchanged.

---

## Directory Structure

```
custom_components/schellenberg_usb/
├── __init__.py                    — integration setup, subentry tracking
├── api.py                         — serial layer (SchellenbergUsbApi, SchellenbergProtocol)
├── config_flow.py                 — hub config + blind subentry flows
├── const.py                       — DOMAIN, CMD_*, CONF_*, SIGNAL_*, type aliases
├── cover.py                       — SchellenbergCover entity + calibration store helpers
├── options_flow.py                — hub options (serial port, ignore_unknown toggle)
├── options_flow_calibration.py    — event-driven calibration (bidirectional motors)
├── options_flow_pairing.py        — PairingFlowHandler (legacy options-flow helper)
├── options_flow_timed_calibration.py — button-press timing calibration (timed motors)
├── sensor.py                      — USB stick status sensors
├── switch.py                      — LED switch entity
├── manifest.json                  — integration metadata, pyserial-asyncio dependency
└── strings.json / translations/   — UI strings for config/options flows
```

<!-- generated-by: gsd-doc-writer -->
# Getting Started

This guide walks you from a fresh install to a working, calibrated Schellenberg roller-shutter motor in Home Assistant. Read it top-to-bottom the first time; experienced users can jump to the step they need.

---

## Prerequisites

- **Home Assistant** 2025.1.0 or later
- **Schellenberg USB Funk-Stick** plugged into the machine running Home Assistant (USB VID `16C0` / PID `05E1`, manufacturer string `van ooijen`)
- Serial port access — the HA host user must be able to open the serial device (typically `/dev/ttyUSB0` or `/dev/ttyACM0` on Linux)
- **HACS** installed (for the recommended install path)

---

## Step 1 — Install the integration

**Via HACS (recommended)**

1. Open HACS in your Home Assistant sidebar.
2. Go to **Integrations** and click the **+** button.
3. Search for `Schellenberg USB`, select it, and click **Download**.
4. Restart Home Assistant when prompted.

**Manual install**

Copy the `custom_components/schellenberg_usb/` folder from this repository into your HA `config/custom_components/` directory, then restart Home Assistant.

---

## Step 2 — Add the integration (hub setup)

1. Go to **Settings → Devices & Services → Add Integration**.
2. Search for **Schellenberg USB** and select it.
3. If the stick was already plugged in, Home Assistant may have auto-discovered it and will ask you to confirm the serial port. Otherwise, enter the port path (default `/dev/ttyUSB0`) and click **Submit**.

The hub entry is now created and the stick connects automatically. For serial port options and USB auto-discovery details see [docs/CONFIGURATION.md](CONFIGURATION.md).

---

## Step 3 — Add a motor

Go to **Settings → Devices & Services**, find **Schellenberg USB**, and click **+ Add device**. A menu offers two paths:

### Option A — Auto-pair (motor is nearby and reachable)

1. Choose **Pair automatically**.
2. Put your motor into pairing mode (see [README — Device Pairing Instructions](../README.md#device-pairing-instructions) for button combinations by model).
3. The integration listens for up to 10 seconds. When the motor responds, you are prompted to give it a friendly name.
4. Bidirectional motors proceed straight to calibration (Step 4A). No extra steps needed here.

### Option B — Manual add (motor is already paired to the stick)

Use this when the motor was paired by hand before you installed this integration, or when the motor never sends events back (non-bidirectional / timed motors).

1. Choose **Add manually**.
2. Enter the motor's two-character hexadecimal enum (e.g. `10`, `11`, `12` — check your stick's pairing log or increment from `10` for each motor added).
3. Choose the motor type:
   - **Bidirectional** — motor sends movement events back to the stick (most ROLLODRIVE PREMIUM motors). Leave this toggled on.
   - **Timed (non-bidirectional)** — motor never confirms movement; drive-to-position relies on button-press timing. Toggle this off.
4. Optionally enter a friendly name; if left blank, the name defaults to `Blind <enum>`.
5. For timed motors only: set an **initial position** (0 = fully closed, 100 = fully open) that reflects where the shutter physically is right now. This seeds position tracking until calibration completes.

The motor appears as a cover entity immediately after this step.

---

## Step 4 — Calibrate

Calibration records how many seconds the motor takes to travel from fully closed to fully open (and back). Without it, position tracking and drive-to-percentage commands are unavailable.

> **Note:** Calibration does NOT set motor end-stops. Physical travel limits must be configured on the motor itself using its built-in adjustment features or a Schellenberg remote before you calibrate here.

### 4A — Bidirectional motors (event-based)

The integration detects movement automatically — you control the motor with your physical remote during calibration.

Full step-by-step instructions are in [README — Calibration Steps](../README.md#calibration-steps).

### 4B — Timed (non-bidirectional) motors (button-press timing)

The integration drives the motor itself and measures elapsed time between your button presses — no motor events are required.

1. Open the device page for your motor and click the **Calibrate** (gear) icon.
2. **Precondition step:** Confirm that the shutter is fully open (at the top) before proceeding.
3. **Close run:** The integration sends a close command automatically. Wait until the motor reaches the bottom endstop and stops on its own, then press **Next**. (Valid travel: 2 – 120 seconds.)
4. **Open run:** The integration sends an open command automatically. Wait until the motor reaches the top endstop and stops on its own, then press **Next**.
5. **Confirm:** The measured open and close times are shown. Press **Done** to save, or check **Redo** to repeat the measurements.

After calibration the shutter position is set to 100 % (fully open), matching where the motor ended up.

---

## Step 5 — Control the motor

Once calibrated, your motor appears in Home Assistant as a standard cover entity with:

- **Open / Close / Stop** buttons
- **Position slider** — drag to any percentage; the integration calculates travel time automatically

From this point the entity works like any other HA cover: use it in automations, dashboards, and voice assistants.

---

## Recalibrating

If travel times change (motor replaced, mechanical adjustment, etc.), open the motor's device page and click the **Calibrate** gear icon to run calibration again. Existing times are overwritten on confirmation.

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| Integration not found after install | HA not restarted, or browser cache | Restart HA; clear browser cache |
| "Cannot connect" on serial port | Wrong path or permission denied | Verify the path with `ls /dev/tty*`; add the HA user to the `dialout` group |
| Auto-pair times out | Motor not in pairing mode, or out of range | Move the stick closer; retry pairing mode on the motor |
| Timed calibration rejects "too short" | Submitted before motor reached endstop | Wait for the motor to stop completely before pressing Next |
| Position drifts over time | Calibration times no longer accurate | Recalibrate from the device page |

For configuration details (serial port, baud rate, subentry data) see [docs/CONFIGURATION.md](CONFIGURATION.md).

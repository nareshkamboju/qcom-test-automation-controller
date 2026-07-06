# Onboarding a New Device Type (QTAC + LAVA)

This guide explains how to bring up a **new Qualcomm device type** end‑to‑end:
from the QTAC GPIO configuration (`.tcnf`) that drives the debug board, through
the boot/state scripts (`powerOn`, `powerOff`, `bootToEDL`), out to the LAVA
worker that automates the board in CI.

It exists because the git history in this repo is intentionally terse
(e.g. `reorder flow`, `Fix configs (#11)`, `update docs`). Those messages do
**not** explain *why* a pin moved or *why* a delay changed, so this document
captures the model you need instead of relying on `git log`.

## Table of Contents
- [1. Mental Model: Config → Script → GPIO → Boot](#1-mental-model-config--script--gpio--boot)
- [2. How a Device Boots (and where GPIO fits)](#2-how-a-device-boots-and-where-gpio-fits)
- [3. Anatomy of a `.tcnf` Config](#3-anatomy-of-a-tcnf-config)
- [4. The Standard Boot/State Scripts](#4-the-standard-boot-state-scripts)
- [5. GPIO ↔ Boot-Mode Relationship](#5-gpio--boot-mode-relationship)
- [6. Debug Board Families (PSoC / FTDI / PIC32CX)](#6-debug-board-families-psoc--ftdi--pic32cx)
- [7. Onboarding a New Device — Step by Step](#7-onboarding-a-new-device--step-by-step)
- [8. From QTAC to LAVA (CI automation)](#8-from-qtac-to-lava-ci-automation)
- [9. Extracting Values From a Boot Log](#9-extracting-values-from-a-boot-log)
- [10. Checklist](#10-checklist)
- [11. Troubleshooting](#11-troubleshooting)

## 1. Mental Model: Config → Script → GPIO → Boot

QTAC never talks to the Qualcomm SoC directly. It toggles **GPIO lines on a
debug board**, which are physically wired to the device's buttons, power rails,
and mode-select straps. The chain is:

```
QTAC UI / tac-api  ─▶  named command (pkey, pedl, battery…)
                        │  (mapping lives in the .tcnf "pins" array)
                        ▼
                   Debug-board GPIO pin  ─▶  device pad/button/strap
                        │
                        ▼
                   SoC samples the strap at power-on ─▶ chooses boot mode
```

- A `.tcnf` file maps **logical command names** (like `pedl`, `pkey`, `battery`)
  to **physical GPIO pins** on a specific debug board.
- An embedded **script** (also inside the `.tcnf`, mirrored by
  `configurations/DefaultScript.txt`) sequences those commands with timing to
  produce a boot mode.
- `configurations/devicelist.json` maps a **platform_id** and
  **debugBoardType** to the correct `.tcnf`.

So "onboarding a new device" = pick a debug board family, wire the pins, write
the pin→command map, and tune the boot script timing.

## 2. How a Device Boots (and where GPIO fits)

Qualcomm SoCs decide their boot path from **strap/button state sampled at the
moment power is applied**. That is why every boot script first forces a known
OFF state, sets the mode straps, and *then* powers on. Order matters more than
almost anything else.

Common target modes:

| Mode | Purpose | Strap/keys held at power-on |
| :-- | :-- | :-- |
| Normal boot | Boot to OS | none (just power key) |
| **EDL** (Emergency Download) | Low-level flashing via QDL/firehose | `pedl` (EDL strap) high |
| Secondary EDL | Alternate EDL path | `sedl` high |
| Fastboot | `fastboot` flashing | `voldn` (volume down) held |
| UEFI menu | Firmware menu | `volup` (volume up) held |

`bootToEDL` is the one you almost always need for automated flashing, because
LAVA uses it to enter QDL and push a firehose programmer.

## 3. Anatomy of a `.tcnf` Config

A `.tcnf` is JSON. The fields that matter for onboarding:

| Field | Meaning |
| :-- | :-- |
| `name` / `description` | Human label shown in the Device Catalog |
| `platform_id` | Numeric ID that ties this config to `devicelist.json` |
| `platform_type` | `FTDI`, `PSOC`, or `PIC32CX` (must match the debug board) |
| `bus` | (FTDI only) bus lanes `A/B/C/D` and their `bus_function` |
| `pins` | The **GPIO → command map** (see below) |
| `buttons` | UI buttons and which `command` each one runs |
| `variables` | Tunable timings (`edl`, `fastboot`, `uefi`, in ms) |
| `script` | The embedded boot/state script (see section 4) |
| `supportedFirmwareVer` | Debug-board firmware compatibility |

### The `pins` array — the GPIO map

Each enabled pin binds a **physical pin** to a **logical command**:

```jsonc
{
  "bus": "D",            // FTDI lane (PSoC uses a flat pin_number, no bus)
  "pin_number": "5",     // physical GPIO on the debug board
  "command": "pedl",     // logical name used by the script/tac-api
  "name": "EDL (Emergency Download Mode)",
  "enabled": true,
  "inverted": false,     // true = active-low line (e.g. battery, usb0)
  "initial_value": false,// state applied at config load
  "input": false         // false = output (drive), true = read/sense
}
```

Key rules learned from the shipping configs:

- `command` is the **contract** between the config and the script. Renaming a
  pin's `command` breaks every `def` in the script that references it.
- `inverted: true` means the electrical line is active-low. In the FTDI/PSoC
  Hawi configs, `battery` and `usb0` are inverted — the script writes `1` to
  *cut* power and `0` to *apply* it.
- FTDI configs address pins by `bus` + `pin_number`; PSoC configs use a flat
  `pin_number` with `bus = null`.

Example — Hawi MTP8975 (`TAC_FTDI_65.tcnf`) enabled outputs:

| Command | Bus/Pin | Inverted | Wired to |
| :-- | :-- | :-- | :-- |
| `pshold` | C/0 | no | Force PS_HOLD high |
| `pkey` | D/3 | no | Power key |
| `volup` | D/0 | no | Volume up (UEFI strap) |
| `voldn` | D/2 | no | Volume down (fastboot strap) |
| `pedl` | D/5 | no | EDL strap |
| `battery` | D/1 | **yes** | Battery / main rail |
| `usb0` | D/7 | **yes** | USB VBUS |
| `eud` | D/4 | no | Embedded USB Debug |

## 4. The Standard Boot/State Scripts

The script language is line-based. Each line is either a `def <name>()` block,
a `<command> <value>` GPIO write, a `delay <ms>`, a call to another `def`, or a
`logComment`. `$edl`, `$fastboot`, `$uefi` expand to the `variables` values.

The canonical building blocks (from `configurations/DefaultScript.txt`):

```text
def powerOffTheDevice()      # cut the rails
    battery 1                # (inverted) 1 = battery removed
    usb0 1                   # (inverted) 1 = VBUS off
    usb1 1

def powerOnTheDevice()       # apply the rails, in order
    delay 500
    battery 0                # apply battery
    delay 900
    usb0 0                   # then VBUS
    usb1 0
    delay 30
```

Everything else composes these two:

```text
def powerOff()               # rails off + release all buttons/straps
    powerOffTheDevice
    pkey 0
    volup 0
    voldn 0
    pedl 0

def powerOn()                # clean off, then rails on, then pulse power key
    powerOff
    powerOnTheDevice
    pkey 1
    delay 800
    pkey 0

def bootToEDL()              # THE flashing entry point
    powerOffTheDevice        # 1) guarantee OFF
    pkey 0
    volup 0
    voldn 0
    pedl 1                   # 2) assert EDL strap BEFORE power
    powerOnTheDevice         # 3) apply power (SoC samples pedl here)
    pkey 1
    delay $edl               # 4) hold power key long enough for EDL
    pkey 0
    pedl 0                   # 5) release strap
```

The invariant to remember: **assert the mode strap while the device is OFF,
then power on, then release.** If `pedl` is asserted after power-on, the SoC has
already chosen its boot path and EDL will not be entered.

## 5. GPIO ↔ Boot-Mode Relationship

This is the core of what you asked: how the GPIO configuration relates to the
device boot load.

| Boot mode (`def`) | Straps/keys set before `powerOnTheDevice` | GPIO commands used |
| :-- | :-- | :-- |
| `powerOn` | none | `battery`, `usb0`, `pkey` |
| `bootToEDL` | `pedl 1` | + `pedl` |
| `bootToSecondaryEDL` | `sedl 1` | + `sedl` |
| `bootToFastboot` | `voldn 1` | + `voldn` |
| `bootToUEFI` | `volup 1` | + `volup` |

Reading the table right-to-left: **each extra boot mode you need requires an
extra GPIO line** on the debug board, mapped to the right `command` name in the
`.tcnf` `pins` array, and a matching `def` in the script. If the board has no
pin wired to the EDL strap, no amount of scripting produces `bootToEDL`.

Timing (`$edl`, `$fastboot`, `$uefi`) is per-SoC and is tuned empirically by
watching the boot log — see section 9. Battery-only boards add
`bootToBatteryEDL` / `batteryReset` variants (see `TAC_PSOC_66.tcnf`) because
they cannot cut the main rail and must reset via long button holds instead.

## 6. Debug Board Families (PSoC / FTDI / PIC32CX)

`devicelist.json` uses `debugBoardType` to select the family:

| `debugBoardType` | Family | Config prefix | Pin addressing |
| :-- | :-- | :-- | :-- |
| 2 | Cypress PSoC 5LP | `TAC_PSOC_*.tcnf` | flat `pin_number`, `bus = null` |
| 3 | FTDI | `TAC_FTDI_*.tcnf` | `bus` (A/B/C/D) + `pin_number` |
| 4 | Microchip PIC32CX | `TAC_PIC32CXAuto_*.tcnf` | flat, uses `DefaultPIC32CXScript.txt` |

Notes:
- PSoC boards must be **factory-programmed** with firmware or they won't
  enumerate (see the top-level `README.md`).
- PIC32CX (automotive) scripts differ: `powerOn` toggles `battery` rather than a
  power key, and modes like `bootToFastboot` use dedicated `fastboot`/`uefi`
  pins. Start from `DefaultPIC32CXScript.txt`, not `DefaultScript.txt`.
- The same physical device can have **two catalog entries** (e.g. Hawi MTP8975
  is both `platform_id 65` FTDI and `platform_id 66` PSoC) — one per board type.

## 7. Onboarding a New Device — Step by Step

1. **Identify the debug board family.** FTDI, PSoC, or PIC32CX? This decides the
   config prefix, pin addressing, and which default script to copy.

2. **Get the pin map from hardware.** From the board schematic / cable strip,
   list which GPIO pin drives each function you need: at minimum `pkey`,
   `battery`, `usb0`, and `pedl` (for EDL). Add `volup`/`voldn` if you need
   UEFI/fastboot.

3. **Copy a starting config.** Either the closest existing config for the same
   board family and a similar SoC (e.g. `TAC_FTDI_65.tcnf` for a new FTDI phone),
   or a fill-in-the-blank skeleton from `configurations/templates/`. Save as
   `TAC_<FAMILY>_<newid>.tcnf` in `configurations/`.

4. **Edit metadata.** Set `name`, `description`, a unique `platform_id`, and
   confirm `platform_type` matches the family.

5. **Rewrite the `pins` array.** For each function, set `bus`/`pin_number`,
   `command`, `name`, `enabled: true`, and `inverted` (true for active-low rails
   like `battery`/`usb0`). Disable pins you don't use.

6. **Adjust the embedded `script`.** Keep the standard `def` structure. Only add
   or remove GPIO writes that correspond to pins you actually wired. Ensure any
   `command` referenced in the script exists as an enabled pin.

7. **Set timing `variables`.** Start from the copied values (`edl`, `fastboot`,
   `uefi`) and refine after testing against the boot log.

8. **Register in `devicelist.json`.** Add a `catalog` entry with the new
   `platform_id`, `debugBoardType` (2/3/4), `name`, `description`, and
   `configPath` pointing at your new `.tcnf`.

9. **Validate in QTAC.** Build (`build.sh` / `build.bat`), open the Device
   Catalog, load the config, and exercise `powerOn`, `powerOff`, `bootToEDL`
   while watching the serial log.

10. **Wire up automation.** Once the buttons work by hand, expose them to CI via
    `tac-api` and LAVA (section 8).

## 8. From QTAC to LAVA (CI automation)

In CI, the same `.tcnf` commands are invoked headless through `tac-api.py`,
addressed by the debug board's **serial number** (not the platform_id):

```bash
/usr/local/bin/tac-api.py --serial <BOARD_SERIAL> --command bootToEDL
/usr/local/bin/tac-api.py --serial <BOARD_SERIAL> --command powerOn
/usr/local/bin/tac-api.py --serial <BOARD_SERIAL> --command powerOff
```

A LAVA device `.jinja2` wires these into power/flash hooks. For a QDL/firehose
flow the important keys are:

```jinja2
{% set power_off_command = '.../tac-api.py --serial <SER> --command powerOff' %}
{% set power_on_command  = '.../tac-api.py --serial <SER> --command powerOn'  %}
{% set qdl_enter_commands = ['.../tac-api.py --serial <SER> --command bootToEDL'] %}
{% set board_qdl_id = '<8-CHAR-EDL-ID>' %}                 # SoC serial, hex
{% set board_id = "QUSB_BULK_CID:<CID>_SN:" + board_qdl_id %}  # CID is per board-family
{% set fastboot_serial_number = '<usb serial>' %}
```

Two field notes that are easy to get wrong:
- **`board_qdl_id`** is the SoC's 8‑char serial from the boot log, uppercased.
- **`CID`** in `board_id` is **per board-family** (hamoa=`0446`,
  kaanapali=`046A`, glymur=`046E`, lemans/rb8=`0440`, qcs8300/rb4=`0455`,
  rb1/unoq=`0420`, rb3g2=`042F`). Do **not** copy it from another device type —
  read it from `lsusb` while the board is in EDL/QDL mode. This exact mistake is
  called out as a TODO in the Shikra onboarding notes.

The board serial used by `tac-api` is the FTDI/PSoC serial that also appears in
the worker's `tac-config` file (`<board-serial> <friendly-name>`) and in the
`ser2net.yaml` connector path.

## 9. Extracting Values From a Boot Log

When onboarding a real board you rarely have a schematic handy for every value.
Capture a serial boot log (via ser2net/PuTTY) and mine it. Using a Shikra IQS
EVK log as the worked example:

| You need | Grep for | Example value |
| :-- | :-- | :-- |
| device-type / model | `Machine model:` | `Qualcomm ... Shikra IQS EVK` |
| storage type | `Boot Device` | `eMMC` |
| chip name | `Chip Name` | `IQ2390S` |
| `board_qdl_id` | `Chip Serial Number` | `0xDB827C8D` → `DB827C8D` |
| `fastboot_serial_number` | `usb: Serial -` | `3a356369` |
| serial connector | `Port /dev/serial/by-id/...` | `usb-FTDI_Quad_RS232-HS-if00-port0` |

The `Machine model:` string should match the DTS `compatible` you name the
device-type after; the `Chip Serial Number` (uppercased, 0x stripped) becomes
`board_qdl_id`. Boot timings for `$edl`/`$fastboot`/`$uefi` are refined by
watching how long the log takes to reach the relevant prompt.

## 10. Checklist

- [ ] Debug board family identified (FTDI / PSoC / PIC32CX)
- [ ] Pin → command map obtained from schematic or existing sibling config
- [ ] New `TAC_<FAMILY>_<id>.tcnf` created with unique `platform_id`
- [ ] `pins` array: `command`, `bus`/`pin_number`, `enabled`, `inverted` set
- [ ] Embedded `script` references only enabled pins; strap-before-power order kept
- [ ] `variables` timings set (`edl`, `fastboot`, `uefi`)
- [ ] `devicelist.json` catalog entry added with correct `debugBoardType` + `configPath`
- [ ] `powerOn` / `powerOff` / `bootToEDL` verified by hand in QTAC
- [ ] `tac-api --command bootToEDL` verified headless
- [ ] LAVA `.jinja2` created; `board_qdl_id` + per-family `CID` correct
- [ ] `tac-config`, `ser2net.yaml`, worker config updated

## 11. Troubleshooting

- **EDL never entered.** Strap asserted after power-on, or `pedl` pin not wired /
  wrong `pin_number`. Confirm the pin toggles in QTAC and that `pedl 1` runs
  *before* `powerOnTheDevice`.
- **Device won't power off.** `battery`/`usb0` `inverted` flag wrong — the rail
  is active-low; `1` should cut power. Flip `inverted` in the `.tcnf`.
- **Config doesn't appear in catalog.** Missing/incorrect `devicelist.json`
  entry, or `debugBoardType` doesn't match `platform_type` in the `.tcnf`.
- **QDL fails to find board.** `board_id` CID copied from another family; read
  the real `QUSB_BULK_CID:xxxx` from `lsusb` in EDL mode.
- **Board doesn't enumerate at all (PSoC).** Board not factory-programmed.
- **Commit history is unhelpful.** Expected — prefer this doc and the config
  contents over `git log`; describe *why* in future commits and update this file.

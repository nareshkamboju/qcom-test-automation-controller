# Rules & Steps: Onboarding a New Device Type (QTAC)

A concise, checklist-style companion to
[Onboarding a New Device Type (QTAC + LAVA)](./02-Onboarding-New-Device.md).
Use this page as the authoritative rulebook when adding a new device to
`configurations/`. The other guide explains *why*; this one tells you *what to do
and what not to break*.

## Table of Contents
- [Rules (must follow)](#rules-must-follow)
- [Step-by-Step](#step-by-step)
- [Field Reference](#field-reference)
- [Boot-Mode Rules](#boot-mode-rules)
- [Validation Gate](#validation-gate)
- [Definition of Done](#definition-of-done)

## Rules (must follow)

1. **One config per debug board family.** Pick exactly one of FTDI, PSoC, or
   PIC32CX. The file prefix, pin addressing, and default script all follow from
   this choice. The same device on two board families = two separate configs
   (e.g. Hawi MTP8975 is both `platform_id 65` FTDI and `66` PSoC).
2. **`platform_type` must match `debugBoardType`.** `FTDI`↔`3`, `PSOC`↔`2`,
   `PIC32CX`↔`4`. A mismatch means the config never loads in the catalog.
3. **`platform_id` is unique and never reused.** Pick the next free integer
   (currently the highest in use is `74`; choose `75+`). Renumbering an existing
   ID breaks every worker that references it.
4. **The pin `command` name is a contract.** Every `command` used in the
   `script` MUST exist as an `enabled: true` pin. Never rename a `command`
   without updating the script, and vice-versa.
5. **Copy, don't hand-write.** Start from the closest existing `.tcnf` of the
   same family/SoC. Do not create a config from scratch.
6. **Assert straps while OFF, then power on, then release.** Any boot mode
   (`bootToEDL`, `bootToFastboot`, `bootToUEFI`) sets its strap during the OFF
   state before `powerOnTheDevice`. Straps set after power-on do nothing.
7. **Respect `inverted` (active-low) rails.** For `battery`/`usb0`/`usb1`,
   `1` = power removed, `0` = power applied. Keep the flag; don't invert logic in
   the script instead.
8. **Only wire what exists.** Do not enable a pin or add a boot-mode `def` unless
   the debug board physically drives that line.
9. **Base scripts by family.** FTDI/PSoC copy `DefaultScript.txt`; PIC32CX copies
   `DefaultPIC32CXScript.txt` (different power model — `battery` instead of a
   power-key pulse).
10. **Register in `devicelist.json`.** A `.tcnf` with no catalog entry (correct
    `platform_id`, `debugBoardType`, `configPath`) is invisible to QTAC.
11. **Tune timings via `variables`, not magic numbers.** Keep `$edl`,
    `$fastboot`, `$uefi` as `variables`; refine against a serial boot log.
12. **Write a real commit message.** History here is terse and unhelpful. State
    the device, board family, `platform_id`, and *why* timings/pins are set —
    don't rely on `git log` later.

## Step-by-Step

1. **Choose the board family** (FTDI / PSoC / PIC32CX) → decides prefix and
   default script.
2. **Collect the pin map** from the schematic or a sibling config: at minimum
   `pkey`, `battery`, `usb0`, `pedl`; add `volup`/`voldn`/`sedl`/`eud` as wired.
3. **Copy a starting config** to `configurations/TAC_<FAMILY>_<newid>.tcnf` —
   either the closest existing sibling, or a fill-in-the-blank skeleton from
   [`configurations/templates/`](../../configurations/templates/README.md).
4. **Edit metadata:** `name`, `description`, unique `platform_id`, correct
   `platform_type`.
5. **Rewrite the `pins` array:** set `bus`/`pin_number`, `command`, `name`,
   `enabled: true`, `inverted`; disable unused pins (`enabled: false`).
6. **Edit the embedded `script`:** keep the standard `def` blocks; add/remove
   only GPIO writes matching pins you actually wired.
7. **Set `variables`:** start from copied `edl`/`fastboot`/`uefi` values.
8. **Add the `devicelist.json` catalog entry:** `platform_id`, `debugBoardType`,
   `name`, `description`, `configPath` → your new file.
9. **Build QTAC** (`./build.sh` or `build.bat`) and load the config from the
   Device Catalog.
10. **Verify by hand:** exercise `powerOn`, `powerOff`, `bootToEDL` while
    watching the serial log; refine timings.
11. **Wire automation** (optional): expose via `tac-api`/`pytac` and LAVA — see
    the companion guide, section 8.
12. **Commit** with a descriptive message (Rule 12).

## Field Reference

Config-level keys:

| Key | Rule |
| :-- | :-- |
| `name` / `description` | Human label in the catalog |
| `platform_id` | Unique integer; never reuse (see Rule 3) |
| `platform_type` | `FTDI` / `PSOC` / `PIC32CX`; match `debugBoardType` |
| `bus` | FTDI only: lanes `A/B/C/D` + `bus_function` (2 = GPIO output lane) |
| `variables` | `edl`, `fastboot`, `uefi` timings in ms |
| `script` | Boot/state `def` blocks; only reference enabled pins |

Per-pin keys (in `pins[]`):

| Key | Rule |
| :-- | :-- |
| `command` | Logical name; must match every use in `script` (Rule 4) |
| `bus` + `pin_number` | FTDI: both required. PSoC/PIC32CX: `bus = null`, flat `pin_number` |
| `enabled` | `true` to activate; `false` pins are ignored |
| `inverted` | `true` for active-low rails (`battery`, `usb0`) — see Rule 7 |
| `initial_value` | State applied at config load (FTDI/PSoC drive it; PIC32CX ignores) |
| `input` | `false` = output (drive), `true` = sense/read |

## Boot-Mode Rules

Each mode = a strap held during the OFF→ON transition. Add the mode only if the
matching pin is wired.

| Mode (`def`) | Required pin | Strap set before power-on |
| :-- | :-- | :-- |
| `powerOn` | `pkey`, `battery`, `usb0` | none |
| `bootToEDL` | `pedl` | `pedl 1` |
| `bootToSecondaryEDL` | `sedl` | `sedl 1` |
| `bootToFastboot` | `voldn` | `voldn 1` |
| `bootToUEFI` | `volup` | `volup 1` |

Battery-only boards (cannot cut the main rail) use `bootToBatteryEDL` /
`batteryReset` with long button holds instead of a rail cut — see
`TAC_PSOC_66.tcnf`.

## Validation Gate

Do not consider the device onboarded until all pass:

- [ ] Config loads in the QTAC Device Catalog without error.
- [ ] `powerOff` fully powers the device down.
- [ ] `powerOn` boots to the OS.
- [ ] `bootToEDL` reaches EDL/QDL (confirm with `lsusb` / host tooling).
- [ ] Any added mode (`bootToFastboot`/`bootToUEFI`) reaches its target.
- [ ] Every `command` in the `script` maps to an enabled pin (no `NameError`).
- [ ] Timings stable across 3+ repeated runs.

## Definition of Done

- [ ] `configurations/TAC_<FAMILY>_<id>.tcnf` added, unique `platform_id`.
- [ ] `platform_type` matches `debugBoardType`.
- [ ] `pins` map complete: `command`, `bus`/`pin_number`, `enabled`, `inverted`.
- [ ] `script` references only enabled pins; strap-before-power order kept.
- [ ] `variables` timings set and verified.
- [ ] `devicelist.json` catalog entry added with correct `configPath`.
- [ ] Validation Gate fully green.
- [ ] Descriptive commit message written (Rule 12).

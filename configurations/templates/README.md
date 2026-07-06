# Device-Type Config Templates

Fill-in-the-blank starting points for onboarding a **new device type**. Copy the
template that matches your debug board family, rename it, and replace every
`<...>` placeholder. Then follow the rules in
[../../docs/getting-started/03-Device-Type-Onboarding-Rules.md](../../docs/getting-started/03-Device-Type-Onboarding-Rules.md).

| Template | Board family | Pin addressing |
| :-- | :-- | :-- |
| `TAC_FTDI_TEMPLATE.tcnf` | FTDI (`debugBoardType: 3`) | `bus` (A/B/C/D) + `pin_number` |
| `TAC_PSOC_TEMPLATE.tcnf` | Cypress PSoC (`debugBoardType: 2`) | flat `pin_number`, `bus: null` |

> PIC32CX: start from a real `TAC_PIC32CXAuto_*.tcnf` and
> `configurations/DefaultPIC32CXScript.txt` (its power model differs — no
> power-key pulse), then apply the same rules.

## How to use

1. **Copy and rename** to `configurations/TAC_<FAMILY>_<newid>.tcnf`
   (e.g. `TAC_FTDI_75.tcnf`). Use the next free `platform_id` — never reuse one.
2. **Fill placeholders:** `<Device Type Name>`, `<... description>`, `author`,
   `creation_date`/`modification_date`, `usb_descriptor`, and `platform_id`
   (currently `0`).
3. **Edit the `pins` array** to match the schematic:
   - Set the real `bus`/`pin_number` for each function.
   - `enabled: true` only for lines the board actually drives.
   - Keep `inverted: true` on active-low rails (`battery`, `usb0`): `1` cuts
     power, `0` applies it.
4. **Keep the `script` and `pins` in sync** — every `command` referenced in the
   `script` must map to an `enabled: true` pin. The template ships a
   self-consistent `powerOn` / `powerOff` / `bootToEDL` set with
   `pshold`/`battery`/`usb0`/`pkey`/`pedl`/`volup`/`voldn` enabled. Disable pins
   your board doesn't drive **and** remove their uses from the script; add
   `bootToUEFI`/`bootToFastboot` only if `volup`/`voldn` are wired.
5. **Register** the file in `configurations/devicelist.json` with a matching
   `platform_id`, `debugBoardType`, `name`, `description`, and `configPath`.
6. **Validate** the JSON before loading:
   ```bash
   python3 -c "import json,sys; json.load(open(sys.argv[1]))" configurations/TAC_<FAMILY>_<newid>.tcnf
   ```

## Notes

- The `script` value is a single JSON string using `\r\n` line breaks and tabs,
  exactly as QTAC writes it. Editing in the QTAC Configuration Editor is the
  safest way to change it; hand-editing the string is possible but error-prone.
- These templates are **not** shipped as usable configs and have `platform_id: 0`
  on purpose so they can't collide with real entries or be picked up by tooling.

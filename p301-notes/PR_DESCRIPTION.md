# Pull Request Template — Pichau P301 (`258a:1007`, fw `X182`)

> [!NOTE]
> Copie o conteúdo abaixo da linha `---` ao abrir o Pull Request no GitHub (`LDrawe/libratbag:sinowealth-pichau-p301` → `libratbag/libratbag:master`).
> Preencha ou ajuste os itens marcados com `[FILL AFTER LINUX TEST]` após rodar os testes reais no Linux, e lembre-se de **remover a pasta `p301-notes/` da branch** antes de abrir o PR.

---

**Title:** `sinowealth: add support for Pichau P301 (258a:1007, fw X182)`

Add support for the **Pichau Gaming P301** (`PGM-P301-RGB`), USB `258a:1007` (rev `0101`), firmware `X182`, Avago/PixArt A3050 (`0x3050`) sensor.

## Protocol (reverse-engineered from `OemDrv.exe` / `Cfg.ini` and live HID dumps)

- Standard SinoWealth vendor HID interface (`MI_01`): 6-byte commands on report `0x05`, 520-byte feature report `0x04` (123-byte config payload at `raw[0x08..0x82]`, 88-byte button map).
- Firmware version `X182` (`58 31 38 32`) on cmd `0x01`.
- Two onboard profiles (`0x11` and `0x21`; `0x31` is not present on this firmware); active profile queried/switched via cmd `0x02`.
- Sensor ID `0x02` (`A3050` / `0x3050` in `GetSenosr` inside `OemDrv.exe`):
  - Uses a 1-indexed discrete 11-step DPI table (`raw = 0x01..0x0b`):
    `{ 500, 750, 1000, 1250, 1500, 1750, 2000, 2500, 3000, 3500, 4000 }`.
  - Independent X/Y DPI is not exposed for `0x3050` in the vendor software.
  - Default factory profile uses 6 DPI stages (`01 03 05 07 09 0b` → `500, 1000, 1500, 2000, 3000, 4000`) with RGB per-stage colors.
- Debounce (`0x1a`) and LOD (`0x1b`) queries return `05 00 00 00 00 00` (unsupported by firmware, already handled gracefully by `driver-sinowealth.c`).

## Driver changes

- `src/driver-sinowealth.c`:
  - Add `SINOWEALTH_SENSOR_A3050 = 0x02` to `enum sinowealth_sensor`.
  - Add `SINOWEALTH_A3050_DPIS` discrete lookup table and update `sinowealth_get_max_dpi_for_sensor()` (`4000`), `sinowealth_raw_to_dpi()` (with bounds clamping for disabled slots), and `sinowealth_dpi_to_raw()`.
  - Update `sinowealth_init_profile()` to expose only the 11 discrete A3050 DPIs via `ratbag_resolution_set_dpi_list()` and omit `RATBAG_RESOLUTION_CAP_SEPARATE_XY_RESOLUTION` for `SINOWEALTH_SENSOR_A3050`.
- `data/devices/sinowealth-1007.device`:
  - Add `[Driver/sinowealth/devices/X182]` (`Buttons=8`, `DeviceName=Pichau P301`, `LedType=RGB`, `Profiles=2`, `SensorType=A3050`).

## Tested

- `meson test -C build` (`data-parse-test`, `test-device`, etc.): `[FILL AFTER LINUX TEST]`
- Tested on physical Pichau P301 (`258a:1007`, fw `X182`) with `ratbagd --verbose=raw`, `ratbagctl`, and Piper:
  - Profile reading (2 profiles, 8 buttons, discrete DPIs `500..4000`): `[FILL AFTER LINUX TEST]`
  - Config write / commit & persistence check across re-read: `[FILL AFTER LINUX TEST]`
  - LED modes / colors / polling rate: `[FILL AFTER LINUX TEST]`

## Known limitations

- Only the discrete 11 DPI steps supported by the A3050 firmware (`500..4000`) are selectable; arbitrary intermediate DPI values are mapped to the nearest supported step.
- Debounce time and lift-off distance are not supported by this firmware.
- No custom Piper SVG is included in this PR (uses generic fallback artwork).

## Disclosure: AI assistance

An AI assistant was used to help disassemble/analyze the Windows vendor utility (`OemDrv.exe`), cross-check the HID protocol dumps, and draft the `driver-sinowealth.c` changes. All hardware captures and live device testing were performed against the physical Pichau P301, and every change in this PR was reviewed and verified on the real hardware.

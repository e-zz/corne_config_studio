# ZMK Firmware Config

ZMK firmware configuration for two split ergonomic keyboards:

| Keyboard | Board | MCU | Shield |
|----------|-------|-----|--------|
| **Corner** (Corne CRKBD) | nice_nano_v2 | nRF52840 | built-in ZMK Corne |
| **Chipper42** | seeeduino_xiao_ble | nRF52840 | custom `chipper42` shield |

## Keymap

Both keyboards share a single keymap at `config/corne.keymap` (`config/chipper42.keymap` just includes it).

Six layers:

| Layer | Purpose |
|-------|---------|
| 0 default | QWERTY base |
| 1 symbol | Symbols (hold BSLH/ENTER) |
| 2 num | Numbers (hold `;`) |
| 3 fun | F-keys, Bluetooth, media |
| 4 sys | Ext power, RGB, bootloader, ZMK Studio unlock |
| 5 kp | Numpad + mouse keys |

Custom behaviors: mod-morph esctab/bsdel, tap-dance sftcaps/lralt, hold-tap u_lt, LM macro.

Mouse keys on layer 5 with per-layer speed scaling via `&zip_xy_scaler`.

## ZMK Studio

Both keyboards support runtime keymap editing via ZMK Studio. Left-side firmware includes `studio-rpc-usb-uart` snippet. Unlock via `&studio_unlock` on SYS layer (combo pos 0+24+12).

## Build

CI builds on push/PR via GitHub Actions (`.github/workflows/build.yml`). See `build.yaml` for the build matrix.

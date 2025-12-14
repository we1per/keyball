# Keyball Codebase Guide for AI Agents

## Project Overview

Keyball is a split keyboard with trackball series built on QMK firmware. The firmware is modular, with platform-specific keyboard definitions (39, 44, 46, 61 keys) and a shared core library for trackball functionality.

## Architecture

### Directory Structure
- **qmk_firmware/keyboards/keyball/**: QMK keyboard definitions
  - **lib/keyball/**: Core library with trackball logic (`keyball.c/h`), PMW3360 optical sensor driver
  - **keyballXX/**: Individual keyboard models (39, 44, 46, 61 keys)
  - **drivers/pmw3360/**: Optical motion sensor driver (SPI-based)
- **bin/**: Build scripts (e.g., `build-keyball-all.sh`)
- **.github/workflows/**: CI/CD for firmware building

### Key Components

**Split Keyboard Communication** (`SPLIT_TRANSACTION_IDS_KB`):
- Uses QMK's split transaction system with three custom transactions:
  - `KEYBALL_GET_INFO`: Sync trackball detection/capability status
  - `KEYBALL_GET_MOTION`: Transfer motion data (x/y) from right to left side
  - `KEYBALL_SET_CPI`: Update CPI settings across split sides
- Defined in each keyboard's `config.h` (e.g., [keyball39/config.h](../qmk_firmware/keyboards/keyball/keyball39/config.h#L42))

**Trackball Processing** ([keyball.c](../qmk_firmware/keyboards/keyball/lib/keyball/keyball.c)):
- PMW3360 polling (4ms interval via `KEYBALL_TX_GETMOTION_INTERVAL`)
- Motion → mouse movement with acceleration (`movement_size_of()` function)
- Scroll snap modes: vertical (default), horizontal, free (toggled via `SSNP_VRT`, `SSNP_HOR`, `SSNP_FRE`)
- CPI adjustment: `CPI_I100`, `CPI_D100`, `CPI_I1K`, `CPI_D1K` keycodes
- EEPROM persistence: `KBC_SAVE` to persist settings

**Keymap Pattern** ([example default keymap](../qmk_firmware/keyboards/keyball/keyball39/keymaps/default/keymap.c)):
- Layer 0: Base QWERTY with IME keys (LNG1/LNG2)
- Layer 1: Function keys, mouse buttons (BTN1–BTN3), arrow keys
- Layer 2: Numpad with mouse navigation
- Layer 3: Configuration layer (RGB, scroll/CPI adjustment, trackball modes)
- Auto-enable scroll mode when on layer 3: `keyball_set_scroll_mode(get_highest_layer(state) == 3)`

## Build & Development Workflow

### Local Build
```bash
# Clone QMK (0.22.14 verified)
git clone https://github.com/qmk/qmk_firmware.git --depth 1 -b 0.22.14 qmk

# Symlink this repo into QMK
cd qmk/keyboards && ln -s ../../keyball/qmk_firmware/keyboards/keyball keyball && cd ..

# Build specific model + keymap
make SKIP_GIT=yes keyball/keyball39:default
```

### Batch Build (All Models)
```bash
# Build all models and keymaps (defined in bin/build-keyball-all.sh)
./bin/build-keyball-all.sh
# Outputs .hex files and size report to tmp/build_log/
```

### GitHub Actions Workflow
- **build-firmware.yml**: Template for single keyboard:keymap build
- **build-user.yml**: On-demand workflow for fork builds (select keyboard + keymap)
- **build-all.yml**: Batch build all models
- Artifacts: `.hex` files in workflow outputs

### Test Build Checklist
- Use `test` keymap to verify operation before merging changes
- Compare firmware sizes: `./bin/hexsize.sh` generates TSV for tracking

## Project-Specific Conventions

### Configuration Patterns
- Compile-time toggles use `#define` guards: `#ifndef KEYBALL_SCROLLSNAP_ENABLE`
- Examples in [keyball.h](../qmk_firmware/keyboards/keyball/lib/keyball/keyball.h#L21-L54):
  - `KEYBALL_CPI_DEFAULT`: Default cursor speed (500)
  - `KEYBALL_SCROLLSNAP_ENABLE`: Scroll snap feature toggle
  - `KEYBALL_PMW3360_UPLOAD_SROM_ID`: High-CPI SROM (adds 4KB+)

### Model Detection
- Model determined via `PRODUCT_ID` bitmask in [keyball.h](../qmk_firmware/keyboards/keyball/lib/keyball/keyball.h#L77-L85)
- Pattern: `(PRODUCT_ID & 0xff00) == 0x0200` → Keyball39

### OLED Display Integration
- Custom font: `keyboards/keyball/lib/logofont/logofont.c`
- Format utilities: `format_4d()` for aligned numbers, `to_1x()` for hex chars

### IME/Language Keys
- Japanese-specific: `KC_LNG1` (Hiragana), `KC_LNG2` (Katakana)
- Mapped in default keymaps (Layer 0)

## Common Modifications

### Adding a New Configuration Option
1. Define in [keyball.h](../qmk_firmware/keyboards/keyball/lib/keyball/keyball.h) with default value
2. Declare in keyboard's `config.h` to override
3. Use `#ifdef` guards in [keyball.c](../qmk_firmware/keyboards/keyball/lib/keyball/keyball.c) for conditional code

### Adding a Custom Keymap
1. Create directory: `qmk_firmware/keyboards/keyball/keyballXX/keymaps/your_keymap/`
2. Copy `keymap.c` from `default` and modify
3. Build: `make SKIP_GIT=yes keyball/keyballXX:your_keymap`

### Debugging Split Sync Issues
- Check `SPLIT_TRANSACTION_IDS_KB` matches between both sides
- Monitor via console if `CONSOLE_ENABLE` is set
- Verify `SOFT_SERIAL_PIN` alignment (D2 by default)

## References

- QMK Documentation: https://docs.qmk.fm/
- PMW3360 Datasheet: Referenced in driver comments
- Build Guide: [Japanese](../keyball61/doc/rev1/buildguide_jp.md), [English](../keyball61/doc/rev1/buildguide_en.md)

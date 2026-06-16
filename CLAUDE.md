# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A fork of [Marlin 2.0.x](https://github.com/MarlinFirmware/Marlin) firmware customized for the **MKS Robin Nano** (V1.x and V2.x, STM32F103VE) 3D printer board. The fork's main addition over upstream Marlin is a full-color touchscreen UI built on [LittlevGL/LVGL](https://github.com/littlevgl/lvgl) (`Marlin/src/lcd/extui/lib/mks_ui/`), plus board-specific pins/HAL glue and bitmap/font assets under `Firmware/`.

## Build

Build system is PlatformIO (same as upstream Marlin). `platformio.ini` defines `default_envs = mks_robin_nano35`.

```bash
pio run                      # build the default env (mks_robin_nano35)
pio run -e mks_robin_nano35  # build explicitly
pio run -t clean             # clean
```

Output binary and the `assets` folder (LVGL images/fonts) land in `.pio/build/mks_robin_nano35/`. Both must be copied to an SD card together to flash the board — the bootloader on the LCD looks for `Robin_nano35.bin` plus `assets/` at SD root.

### Selecting V1.x vs V2.x hardware

Both variants share the `mks_robin_nano35` PlatformIO env; the board revision and display bus are chosen via `#define`s in `Marlin/Configuration.h`, not via separate envs:

| Board | `MOTHERBOARD` | UI bus |
|---|---|---|
| Robin Nano V1.x | `BOARD_MKS_ROBIN_NANO` | `#define TFT_LVGL_UI_FSMC` |
| Robin Nano V2.x | `BOARD_MKS_ROBIN_NANO_V2` | `#define TFT_LVGL_UI_SPI` |

Only one of `TFT_LVGL_UI_FSMC` / `TFT_LVGL_UI_SPI` should be active at a time (see `Marlin/Configuration.h` around `MOTHERBOARD` and the `TFT_LVGL_UI_*` block).

## Tests (CI test-builds)

There are no unit tests; "tests" means compiling many `Configuration.h`/`Configuration_adv.h` permutations to catch build breaks, mirroring `.github/workflows/test-builds.yml`. Test recipes live in `buildroot/tests/<env>-tests` (e.g. `buildroot/tests/mks_robin_nano35-tests`) and are shell scripts that call helpers in `buildroot/bin/` (`use_example_configs`, `opt_set`, `opt_enable`, `opt_disable`, `exec_test`, `restore_configs`). `use_example_configs` pulls example `Configuration*.h` files from the separate `MarlinFirmware/Configurations` repo, so running these locally requires network access.

```bash
chmod +x buildroot/bin/* buildroot/tests/*
export PATH=./buildroot/bin/:./buildroot/tests/:$PATH
run_tests . mks_robin_nano35   # runs buildroot/tests/mks_robin_nano35-tests
```

`restore_configs` (run automatically at the end of each test script) does `git checkout Marlin/Configuration*.h` — so it requires the repo to actually be a git checkout with those files tracked/committed, otherwise it's a no-op.

## Architecture

- **Entry point**: `Marlin/Marlin.ino` → `Marlin/src/MarlinCore.cpp`/`.h` (setup/loop, mirrors stock Marlin).
- **`Marlin/Configuration.h` / `Configuration_adv.h`**: the single source of build-time config (board selection, kinematics, thermal settings, enabled features, UI bus). Almost all conditional compilation in the codebase keys off `#define`s set here. `config/README.md` is a stub pointing at upstream's separate `Configurations` repo — it is *not* where this fork's active config lives.
- **`Marlin/src/HAL/`**: per-MCU hardware abstraction (timers, ADC, serial, EEPROM emulation, etc). Robin Nano boards build against `HAL/STM32F1` (`board = genericSTM32F103VE`, set in `platformio.ini`).
- **`Marlin/src/pins/`**: per-board pin-mapping headers, selected by the `MOTHERBOARD` define. Relevant ones for this fork: `pins/stm32f1/pins_MKS_ROBIN_NANO.h` and `pins_MKS_ROBIN_NANO_V2.h`.
- **`Marlin/src/module/`**: core machine subsystems independent of UI/G-code parsing — `planner`, `stepper`, `temperature`, `endstops`, `motion`, `settings` (EEPROM load/save), etc. Most physical-motion/thermal logic lives here.
- **`Marlin/src/gcode/`**: G-code dispatch. `gcode.cpp`/`parser.cpp` route incoming commands to per-command-group handlers in subfolders (`motion/`, `temp/`, `feature/`, `sd/`, `eeprom/`, …).
- **`Marlin/src/feature/`**: optional, independently togglable features (bed leveling, filament runout, babystepping, power loss recovery, etc.), each compiled in only when its config flag is set.
- **`Marlin/src/lcd/`**: all display/UI code. Subfolders split by display family: `dogm` (graphical LCD), `dwin`, `HD44780` (character LCD), `tft`, `menu` (shared menu logic used by the above), and `extui` (the external-UI abstraction layer used to host fully custom UIs).
- **`Marlin/src/lcd/extui/lib/mks_ui/`** — **this fork's centerpiece**: the LVGL-based color touchscreen UI for the Robin Nano's TFT. `tft_lvgl_configuration.cpp` is the UI entry point/init; each screen is its own `draw_*.cpp`/`.h` pair (e.g. `draw_machine_settings.cpp`, `draw_filament_change.cpp`). When adding/changing a screen, follow the existing `draw_*` file-per-screen pattern.
- **`Firmware/mks_font/`, `Firmware/mks_pic/`**: source bitmap assets for the LVGL UI. Per the README, these are converted to RGB565 binary via the [LVGL online image converter](https://lvgl.io/tools/imageconverter), the resulting `.bin` files go in the build's `assets` folder, and that folder ships on the SD card alongside the firmware binary for the on-screen update flow.
- **`platformio.ini`**: the `[common]` section's `build_src_filter` determines which `src/` subtrees are compiled for most envs (excludes other display families' code, e.g. `HD44780`, `dwin`, unused menu games). Per-env sections (e.g. `[env:mks_robin_nano35]`) `extend` shared sections like `[common_stm32f1]` and layer on board-specific `build_flags`/`extra_scripts`.

# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Purpose

This is a documentation-first project: it records the process and steps for JTAG-debugging an ESP32-S3 through the chip's built-in USB-Serial-JTAG peripheral (GPIO19 = D-, GPIO20 = D+), with no UART bridge and no external JTAG probe. [README.md](README.md) is the main deliverable and contains the step-by-step procedure.

The firmware in `main/hello_world_main.c` is the stock ESP-IDF `hello_world` example and exists only as a debug target. It calls `esp_restart()` after a 10-second countdown, which interrupts long debug sessions.

## Language and encoding

- README and other prose docs are written in English. Screenshots referenced by the README live in `img/`.
- All files are UTF-8 (`files.encoding` is set in `.vscode/settings.json`). When writing files from PowerShell 5.1, pass `-Encoding utf8` explicitly.

## Environment

- ESP-IDF v6.1 at `C:\esp\v6.1\esp-idf`, target `esp32s3`, Windows 11, VS Code with the ESP-IDF extension.
- `idf.py` only works in a shell where the IDF environment is loaded: use the ESP-IDF terminal, or run `C:\esp\v6.1\esp-idf\export.ps1` first.
- OpenOCD board config: `board/esp32s3-builtin.cfg` (set in `idf.openOcdConfigs`).

## Commands

```powershell
idf.py set-target esp32s3
idf.py build
idf.py -p COMx flash monitor     # COMx = the USB-Serial-JTAG CDC port; exit monitor with Ctrl+]
idf.py menuconfig

idf.py openocd                   # start OpenOCD (GDB on 3333, telnet 4444, Tcl 6666)
idf.py gdb                       # in a second terminal, attach GDB to the running OpenOCD

# Flash over JTAG instead of the serial port
openocd -f board/esp32s3-builtin.cfg -c "program_esp_bins build flasher_args.json verify reset exit"
```

In VS Code, `F5` uses the `gdbtarget` attach configuration in `.vscode/launch.json`; the ESP-IDF extension starts OpenOCD itself.

`pytest_hello_world.py` is left over from the upstream example. It needs `pytest-embedded-idf` / `pytest-embedded-qemu` and is parametrized for ESP-IDF's CI targets, so it is not part of the workflow here.

## Things that are easy to miss

- `sdkconfig` is gitignored and there is no `sdkconfig.defaults`, so configuration is not persisted in the repo. Logs already reach the USB port with the default config, because `CONFIG_ESP_CONSOLE_SECONDARY_USB_SERIAL_JTAG=y` mirrors the UART0 console there (output only, non-blocking); `CONFIG_ESP_CONSOLE_USB_SERIAL_JTAG=y` is only needed for input over USB. Put settings that must survive in `sdkconfig.defaults`.
- The top-level `CMakeLists.txt` sets `MINIMAL_BUILD ON`: only `main` and its declared dependencies are built. A new component include fails until it is added to `REQUIRES`/`PRIV_REQUIRES` in `main/CMakeLists.txt`.
- `clangd.arguments` in `.vscode/settings.json` hardcodes the absolute `build` path and the clangd binary path; update them if the project or toolchain moves.
- Only one process can own the JTAG interface. Stop any running OpenOCD before starting another session or flashing over JTAG.
- ESP32-S3 has 2 hardware breakpoints and 2 watchpoints.
- Never burn eFuses (`DIS_USB_JTAG`, `DIS_USB_SERIAL_JTAG`, `STRAP_JTAG_SEL`) without explicit confirmation — it is irreversible and can permanently disable this debug path.

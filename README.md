# Termux-ESP-Flasher

Flash ESP32/ESP8266 firmware from Android using Termux. No root required.

![License](https://img.shields.io/badge/license-Apache--2.0-blue.svg)
![Platform](https://img.shields.io/badge/platform-Android%20%2F%20Termux-green.svg)
![ESP32](https://img.shields.io/badge/chip-ESP32%20%7C%20ESP8266-orange.svg)
![No Root](https://img.shields.io/badge/root-not%20required-brightgreen.svg)

A Termux-native `.bin` flasher that talks directly to the chip's USB endpoints. No `esptool.py` subprocess, no pyserial.

If Web Serial doesn't work in Chrome for Android (it's desktop-only) and `esptool.py` can't find a `/dev/ttyUSB*` node because Termux has no root, this is the tool built for exactly that gap.

Two device families are supported, auto-detected from the USB VID:PID:

- **Native USB CDC** - ESP32-S3 / C3 / S2 (`303A:1001` / `303A:0002`). Talks directly to the chip's USB-Serial-JTAG peripheral. Bootloader entry/exit is handled by `cdc_reset.py`.
- **UART bridge** - classic ESP32 and ESP8266 devkits behind a CP2102, CH340/CH340G, CH9102, or FTDI FT232. The bridge is opened over raw USB and its DTR/RTS lines are pulsed in the classic GPIO0+EN pattern by `uart_reset.py`.

Both paths upload the real ESP-IDF RAM stub for faster block writes, and both fall back to the plain ROM bootloader if the stub fails to load.

## Why not just use esptool.py?

`esptool.py` talks over pyserial, which expects a `/dev/ttyUSB*` or `/dev/ttyACM*` node. On stock no-root Termux there usually isn't one. Android hands USB access to apps as a raw file descriptor via `termux-usb`, not a serial device node. This tool talks straight to the USB endpoints instead of assuming a tty exists.

> If you're on a desktop or laptop Linux box with a real `/dev/ttyUSB*` node, just use real `esptool.py`. It's more battle-tested. This tool exists for the no-root Termux gap that esptool's pyserial transport can't reach.

## What's supported

- **Chip auto-detection.** `--chip` is optional on `probe`, `write`, and `verify`. Omit it and the tool syncs with the ROM bootloader, reads the chip's magic-value register, and picks the right chip automatically.
- **Stub loader.** Both native-USB and UART-bridge sessions upload the real ESP-IDF RAM stub after syncing, switching from 1 KiB ROM-only blocks to 16 KiB stub blocks. Falls back to ROM-only if the stub fails. Pass `--no-stub` to skip it entirely and stay on the plain ROM loader. Recommended on boards with no auto-reset circuit (e.g. native-USB S2 boards with only a BOOT button) since a failed stub handshake kicks the chip out of the ROM bootloader with no way back in except physically re-holding BOOT.
- **Automatic baud renegotiation on UART-bridge boards.** Steps up through 921600 -> 460800 -> 230400 baud, verifying each one actually holds before trusting it. If a rate doesn't hold, it re-enters the ROM bootloader and retries at 115200. Native USB CDC boards skip this entirely since baud rate doesn't mean anything over USB CDC.
- **Live progress.** The flash progress bar streams in real time.
- **Multi-file writes in one session.** `write` accepts one or more `OFFSET:FILE` pairs and flashes all of them under a single USB permission prompt.

## Scope and limitations

- **No full-chip erase command.** Flash a full image (bootloader + partition table + app) at the correct offsets instead of erasing first. `write --erase` will erase exactly the bytes about to be overwritten if the ROM supports it.
- **Auto-detection is best-effort.** An unrecognized chip value means you need to pass `--chip` explicitly.
- Stub data is vendored for more chips than are currently wired up. Open an issue if you need a specific chip added.

## Install

```bash
pkg update && pkg install python termux-api libusb
pip install nrflash
```

`pip install nrflash` pulls in `pyusb` and `espbridge` automatically. `termux-api` and `libusb` are system packages that still need `pkg install`. The **Termux:API** app from [F-Droid](https://f-droid.org) (not the Play Store) must also be installed for the no-root USB permission flow to work.

<details>
<summary>Installing from source</summary>

```bash
git clone https://github.com/7wp81x/Termux-ESP-Flasher
cd Termux-ESP-Flasher
pip install .
```
</details>

## Updating

```bash
pip install --upgrade nrflash
```

If pip reinstalls the same old version, force it to skip the cache:

```bash
pip install --upgrade --no-cache-dir nrflash
```

Check your installed version with `nrflash --version`.

## Usage

```bash
# --chip is optional everywhere except erase-info
nrflash probe
nrflash write --offset 0x0 firmware.bin

# Force a specific chip
nrflash write --chip esp32c3 --offset 0x0 firmware.bin

# Flash + MD5-verify against the device afterward
nrflash write --chip esp32s3 --offset 0x0 firmware.bin --verify

# Flash bootloader + partition table + app in one session
nrflash write 0x0:bootloader.bin 0x8000:partitions.bin 0x10000:app.bin

# Erase the write range first, then flash
nrflash write --offset 0x0 firmware.bin --erase

# Stay in the bootloader after flashing instead of rebooting
nrflash write --offset 0x0 firmware.bin --no-reboot

# Skip the RAM stub and use the plain ROM loader
nrflash write --offset 0x0 firmware.bin --no-stub
```

First run on no-root Termux pops a USB permission dialog. Tap **OK**. The permission persists until you unplug.

## Single binary vs. three images

Real `esptool.py write_flash` usually takes three files at three offsets (bootloader, partition table, app):

```
0x0      bootloader.bin
0x8000   partition-table.bin
0x10000  app.bin
```

`nrflash write` handles all three in one call, or one file per invocation if you prefer. If your build produces a single merged `.bin` (PlatformIO can do this, check for `firmware.factory.bin` after `pio run`), one call at `0x0` is all you need.

## How bootloader entry works

- **UART-bridge boards** (CP2102/CH340/CH9102/FTDI) have DTR/RTS lines wired into EN/BOOT. `uart_reset.py` pulses them in the classic auto-reset pattern.
- **Native USB CDC boards** have no bridge chip. `cdc_reset.py` uses the CDC `SET_CONTROL_LINE_STATE` DTR/RTS bits to trigger an internal EN/BOOT reset.

## Files

```
src/nrflash/
  __init__.py           package version
  __main__.py           enables `python3 -m nrflash.cli`
  cli.py                CLI: argv parsing, fd bootstrap, flash/verify/probe commands
  rom_loader.py         SLIP framing + ROM bootloader protocol, chip auto-detection
  cdc_reset.py          native-USB-CDC bootloader entry/exit
  uart_reset.py         UART-bridge bootloader entry/exit
  stub_flasher_data.py  vendored ESP-IDF RAM stub binaries (Apache-2.0/MIT, Espressif)
```

USB backend detection, fd wrapping, and UART-bridge register access come from the [espbridge](https://github.com/7wp81x/ESP-Bridge) package, installed automatically as a dependency.

## Troubleshooting

**`probe` reports no response from bootloader**
Some clone boards need the physical BOOT button held while plugging in. Try unplugging and replugging the OTG cable before assuming it's a protocol problem.

**Flashing is slow or the progress bar looks stuck**
Check the log for a `Baud rate raised to ... and verified` line. If it's missing, your board or cable couldn't hold a higher rate and the tool already fell back to 115200 automatically.

**`Verify MISMATCH` after a successful-looking write**
Re-run `write --verify`. A flaky OTG link or unstable baud rate can drop bytes mid-transfer in a way that still returns "success" on a given block.

## Legal

For use on hardware you own. Flashing arbitrary firmware to a device you don't own or have written permission to modify can violate warranty terms or applicable law.

## Related projects

- [Termux-PlatformIO](https://github.com/7wp81x/Termux-PlatformIO) - compile PlatformIO projects in Termux, uses nrflash for flashing
- [ESP-Bridge](https://github.com/7wp81x/ESP-Bridge) - the USB backend package nrflash depends on
- [NRSuite](https://github.com/7wp81x/NRSuite) - no-root wireless security toolkit powered by ESP32

## License

Apache-2.0 - see [`LICENSE-APACHE`](LICENSE-APACHE)
Stub binaries: Apache-2.0 + MIT - see [`STUB_LICENSE-APACHE`](STUB_LICENSE-APACHE) and [`STUB_LICENSE-MIT`](STUB_LICENSE-MIT)

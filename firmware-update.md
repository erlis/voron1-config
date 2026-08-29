# Voron 1 — Klipper & Firmware Update Notes

- Printer: **voron1** (host `voronpi`, user `pi`)
- Layout: **BTT Octopus STM32F446** over **USB** only. No CAN, no toolhead MCU. Beacon on USB. Nevermore/StealthMax over Bluetooth.
- Klipper firmware repo: `~/klipper` = **Kalico** (https://github.com/KalicoCrew/kalico), branch `bleeding-edge-v2`
- Config repo: https://github.com/erlis/voron1-config
- Guide: **YELLOW** USB flash — https://usb.esoterical.online/updating.html  
  Do **not** use the green CAN-bridge mainboard pages.

No need to flip the printer or press RESET/BOOT for a normal update. Flash from SSH using the existing `usb-Klipper_...` serial device.

## Hardware identity

| Role | Hardware | How Linux sees it |
|---|---|---|
| Mainboard | BTT **Octopus STM32F446** (pins: X `PF13` / MOTOR_0) | `/dev/serial/by-id/usb-Klipper_stm32f446xx_0E0029001950534841313020-if00` |
| Probe | Beacon RevH | `/dev/serial/by-id/usb-Beacon_Beacon_RevH_7FDAC2E15154364134202020FF142809-if00` |
| Nevermore | StealthMax, Bluetooth | `bt_address: 28:CD:C1:0C:94:E0` — not a Klipper MCU flash |

`[mcu]` in `config.d/main.cfg` uses `serial:` (USB) and `restart_method: command`. There is no `canbus_uuid` and no `can0`.

## Always first: update Kalico on the Pi

Do this **before** `make menuconfig` / flashing. Moonraker’s UPDATE button is unreliable for Kalico (it expects official Klipper `master`). Build MCU firmware only after `~/klipper` is on `origin/bleeding-edge-v2`.

```bash
cd ~/klipper
git fetch origin
git reset --hard origin/bleeding-edge-v2
git log -1 --oneline
```

Confirm `HEAD -> bleeding-edge-v2, origin/bleeding-edge-v2`. Then flash the Octopus so MCU firmware matches the host.

Example (2026-08-29): `e0a57a0b Fixed config checks for ringing test (#755)` → host version `v2026.07.00-48-ge0a57a0b`.

## menuconfig — Octopus USB (verified 2026-08-29)

```
[*] Enable extra low-level configuration options
    Micro-controller Architecture (STMicroelectronics STM32)
    Processor model (STM32F446)
    Bootloader offset (32KiB bootloader)
    Clock Reference (12 MHz crystal)
    Communication interface (USB (on PA11/PA12))
[*] Optimize stepper code for 'step on both edges'
()  GPIO pins to set at micro-controller startup
[ ] High-precision stepping support (slow)
```

Do **not** change 32KiB to 8KiB. Katapult on this Octopus reports application start `0x8008000` (32 KiB).

## Updating the Octopus (USB, no button)

https://usb.esoterical.online/updating.html

```bash
# FIRST: git fetch / reset --hard origin/bleeding-edge-v2  (see "Always first" above)
sudo service klipper stop
cd ~/klipper
make menuconfig
make clean && make
make flash FLASH_DEVICE=/dev/serial/by-id/usb-Klipper_stm32f446xx_0E0029001950534841313020-if00
ls /dev/serial/by-id/    # usb-Klipper_stm32f446xx_0E0029001950534841313020-if00 should return
sudo service klipper start
```

`make flash` talking about “CanBoot” / CAN is noise — this is USB.

If `make flash` cannot find a bootloader, **stop**. Do not open the printer. Next options are Katapult USB (`flashtool.py -r -d …`) or DFU (BOOT jumper) only if the software path fails.

## After a Kalico update

- Deprecated config: https://docs.kalico.gg/Config_Changes.html
- Mainsail should show `mcu` = `stm32f446xx` at the **same** Kalico version as Host, plus Beacon. No `can0` / `can1`.

## Last successful MCU flash

- Date: 2026-08-29
- Kalico: `e0a57a0b` (`v2026.07.00-48-ge0a57a0b`) on host and Octopus
- Octopus: USB F446 / 32 KiB Katapult (`0x8008000`) / `make flash FLASH_DEVICE=/dev/serial/by-id/usb-Klipper_stm32f446xx_0E0029001950534841313020-if00`
- SHA: `D5F392C5B50C7B203E36CBCDA96B7A09A5022093`
- No button, no CAN. “CAN Flash Success” is Klipper’s flash_can.py talking to USB Katapult.

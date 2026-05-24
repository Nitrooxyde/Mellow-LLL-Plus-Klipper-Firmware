# Mellow FLY LLL Buffer Plus — Klipper MCU Firmware (autonomous auto-feed)

[![License: GPL v3](https://img.shields.io/badge/License-GPLv3-blue.svg)](LICENSE)
![MCU](https://img.shields.io/badge/MCU-STM32F072CB-green)
![Klipper](https://img.shields.io/badge/Klipper-MCU%20firmware-orange)

Custom **Klipper MCU firmware** for the Mellow **FLY LLL Buffer Plus** that brings together the **best of both worlds**:

- 🟢 the **autonomous smart-buffer auto-feed** of Mellow's standalone firmware (hall sensors drive the feed motor *in firmware*, real-time), **and**
- 🟢 the board running as a **standard Klipper USB MCU**, so the filament-entrance sensor can trigger a **print pause** through Klipper.

Both at the same time, over a single USB cable (+ 24 V motor power) — something neither the stock standalone firmware nor the community host-macro configs can do on their own.

---

## Why this exists

The official Mellow firmware is **standalone**: the buffer auto-feeds on its own but the board does **not** talk Klipper (no USB MCU; runout has to go through a separate signal wire).

The community Klipper integrations ([river29](https://github.com/river29/Mellow-LLLBufferPLUS-klipper), [ss1gohan13](https://github.com/ss1gohan13/BufferPLUS-klipper)) flash **stock Klipper** and re-implement the buffer with **host-side g-code macros**. That works, but feed bursts go through the **g-code queue** (latency, can lag during long `G1 E` moves) and they switch the active extruder, which can **misdirect extrusion during a print**.

This project runs the buffer state machine **inside the MCU firmware** as a Klipper task — zero g-code-queue latency, no active-extruder juggling, can't stall a print. The host only reads the entrance sensor on `PB7`.

| | Standalone (Mellow) | Stock Klipper + host macros | **This firmware** |
|---|---|---|---|
| Auto-feed | ✅ in-firmware | ⚠️ host macros (queue latency) | ✅ **in-firmware** |
| Klipper MCU / USB sensor | ❌ | ✅ | ✅ |
| Risk of disrupting a print | n/a | ⚠️ `ACTIVATE_EXTRUDER` | ✅ none |
| Runout → print pause | signal wire | USB (PB7) | USB (PB7) |

---

## How it works

A small Klipper MCU task (`src/buffer.c`, `DECL_INIT` + `DECL_TASK`) ports the state machine of Mellow's open firmware ([Fly3DTeam/Buffer](https://github.com/Fly3DTeam/Buffer)) and drives the on-board TMC2208 over its single-wire UART (`VACTUAL` velocity mode):

```
HALL3 (PB4) blocked  → motor FORWARD (push filament)
HALL2 (PB3) blocked  → motor STOP
HALL1 (PB2) blocked  → motor BACK (retract)
ENDSTOP_3 (PB7) = no filament → STOP   (+ host pauses the print)
```

Photo-sensor convention is taken **verbatim** from the source: *blocked = 1, clear = 0*. Manual **feed/retract buttons** (KEY1/KEY2) and the **mainboard control signals** (PB5/PB6, active-low) are supported too. A 60 s continuous-feed timeout acts as a jam safety.

> The only change versus the standalone is that the button "hold" is handled **event-driven** (non-blocking) instead of the original blocking `while()` loop, because blocking for seconds would break Klipper host comms. Behaviour is identical.

### Pins (STM32F072CB)

| Function | Pin | Function | Pin |
|---|---|---|---|
| HALL1 | PB2 | KEY1 (retract) | PB13 |
| HALL2 | PB3 | KEY2 (feed) | PB12 |
| HALL3 | PB4 | FRONT signal | PB5 |
| Entrance switch | PB7 | BACK signal | PB6 |
| TMC EN | PA6 | TMC UART | PB1 |

---

## Installation

> Done from your Klipper host (e.g. a Raspberry Pi). Commands assume `~/klipper` and `~/katapult` ([Arksine/katapult](https://github.com/Arksine/katapult)) are cloned, the Klipper build toolchain is installed, and `dfu-util` is available (`sudo apt install dfu-util`).

### Step 0 — Find your buffer & power it

Connect the buffer's USB to the host **and** give it **24 V on VIN** (the motor needs it; the firmware configures the TMC at first boot, so 24 V must be present then).

List USB serial devices to identify the board and note your **unique chip ID** (you'll reuse it in `printer.cfg`):

```bash
ls /dev/serial/by-id/
# e.g. usb-Klipper_stm32f072xb_3C003A...3720-if00   (or a Mellow / katapult id)
#                            └──────── your XXXXXX ────────┘
```

### Step 1 — Enter DFU mode (to install Katapult)

On the buffer board: **hold BOOT**, tap **RESET**, release **RESET**, then release **BOOT**. Confirm (needs `sudo`):

```bash
sudo dfu-util -l
# → Found DFU: [0483:df11] ... name="@Internal Flash  /0x08000000/064*0002Kg"
```

### Step 2 — Back up the existing firmware (recommended)

```bash
sudo dfu-util -a 0 -d 0483:df11 -U ~/lll_backup_flash.bin       -s 0x08000000:0x20000
sudo dfu-util -a 1 -d 0483:df11 -U ~/lll_backup_optionbytes.bin            # option bytes
```

### Step 3 — Build & flash Katapult (one-time bootloader)

Katapult lets you flash all **future** updates over USB without opening the case.

```bash
cd ~/katapult
make menuconfig
```

- Micro-controller Architecture: **STMicroelectronics STM32**
- Processor model: **STM32F072**
- Clock Reference: **8 MHz crystal**
- Communication interface: **USB (on PA11/PA12)**
- Application start offset: **8 KiB**
- Support bootloader entry on rapid double click of reset: **OFF** ⚠️ *(the LLL Plus has no usable reset button; leaving this ON makes the board fail to enumerate)*

```bash
make clean && make
# board still in DFU (Step 1); flash with mass-erase so no stale app remains:
sudo dfu-util -a 0 -d 0483:df11 -D out/katapult.bin -s 0x08000000:force:mass-erase:leave
```

Re-plug USB and confirm Katapult enumerates:

```bash
ls /dev/serial/by-id/
# → usb-katapult_stm32f072xb_XXXXXX-if00
```

### Step 4 — Add this firmware to Klipper

```bash
cp src/buffer.c ~/klipper/src/buffer.c
```

Add **one line** to `~/klipper/src/Makefile` (only compiled for the F072 buffer, never for your main board):

```make
src-$(CONFIG_MACH_STM32F072) += buffer.c
```

> ⚠️ This line is reverted by `git pull` on the Klipper tree — re-add it after updating Klipper. `buffer.c` itself is untracked and survives.

### Step 5 — Build Klipper

> ⚠️ `make menuconfig` overwrites `~/klipper/.config`. If you build other MCUs (your main board, toolhead…) from this same tree, **back up its config first** and restore it afterwards:
> ```bash
> cp ~/klipper/.config ~/klipper/.config.mainboard   # backup
> # ...build & flash the buffer...
> cp ~/klipper/.config.mainboard ~/klipper/.config    # restore later
> ```

```bash
cd ~/klipper
make menuconfig
```

- Micro-controller Architecture: **STMicroelectronics STM32**
- Processor model: **STM32F072**
- Bootloader offset: **8 KiB bootloader** *(must match Katapult)*
- Clock Reference: **8 MHz crystal**
- Communication interface: **USB (on PA11/PA12)**

```bash
make clean && make
```

### Step 6 — Flash Klipper over USB (via Katapult)

```bash
~/klippy-env/bin/python3 ~/katapult/scripts/flashtool.py \
  -f ~/klipper/out/klipper.bin \
  -d /dev/serial/by-id/usb-katapult_stm32f072xb_XXXXXX-if00
```

> Use the **klippy-env** Python — it has `pyserial` (the system Python usually doesn't). After flashing, the board re-enumerates as `usb-Klipper_stm32f072xb_XXXXXX-if00`.

### Step 7 — printer.cfg

```ini
[mcu LLL_PLUS]
serial: /dev/serial/by-id/usb-Klipper_stm32f072xb_XXXXXX-if00
restart_method: command

[filament_switch_sensor lll_entrance]
switch_pin: ^!LLL_PLUS:PB7
pause_on_runout: True
```

`pause_on_runout: True` pauses the print automatically on runout — it uses Klipper's standard `[pause_resume]` (the same `PAUSE`/`RESUME` your slicer and `M600` rely on, present in virtually every config). **Don't** also add `runout_gcode: PAUSE`: that double-pauses and assumes the macro exists. If you'd rather a custom action, set `pause_on_runout: False` and add your own `runout_gcode:` (e.g. `M118 ...`).

**No motor/stepper/TMC section** — the firmware owns the motor; the host only reads `PB7`. Then `FIRMWARE_RESTART`.

---

## Updating later (no case opening)

```bash
cd ~/klipper && make clean && make          # (re-add the Makefile line if a git pull removed it)
~/klippy-env/bin/python3 ~/katapult/scripts/flashtool.py \
  -f ~/klipper/out/klipper.bin \
  -d /dev/serial/by-id/usb-Klipper_stm32f072xb_XXXXXX-if00
# then FIRMWARE_RESTART
```

`flashtool` reboots the running app into Katapult over USB automatically — you can target the **Klipper** serial directly, no buttons.

---

## TMC2208 register values

Derived directly from Mellow's firmware parameters (via the `TMCStepper` library it uses), not guessed:

| Register | Value | Notes |
|---|---|---|
| GCONF | `0x000001C4` | spreadCycle, pdn_disable, mstep_reg_select (shaft bit set for FORWARD) |
| CHOPCONF | `0x12020055` | toff=5, vsense, 64 µsteps, intpol |
| IHOLD_IRUN | `0x00010F07` | IRUN=15, IHOLD=7 (≈ `rms_current(500)` @ Rsense 0.11) |
| PWMCONF | `0xC10D0024` | pwm_autoscale |
| VACTUAL | `77575` | 260 rpm, 64 µsteps |

Speed/current can be changed in `src/buffer.c` (`SPEED_RPM`, the register values) and re-flashed.

---

## Status / disclaimer

Validated on **one setup** (FLY LLL Buffer Plus, STM32F072CB): auto-feed, manual buttons and runout-pause all working. Pins and register values are specific to this board. Use at your own risk — it's firmware on your hardware.

## Credits

- **[Fly3DTeam / Mellow](https://github.com/Fly3DTeam/Buffer)** — original FLY Buffer firmware & hardware (state machine ported from their open source).
- **[Klipper](https://github.com/Klipper3d/klipper)** — MCU firmware framework.
- **[Katapult](https://github.com/Arksine/katapult)** — USB bootloader.
- Community Klipper integrations: [river29](https://github.com/river29/Mellow-LLLBufferPLUS-klipper), [ss1gohan13](https://github.com/ss1gohan13/BufferPLUS-klipper).

## License

[GPL-3.0](LICENSE) — derived from GPL-3.0 sources (Fly3DTeam/Buffer, Klipper).

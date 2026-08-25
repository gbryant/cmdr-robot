# cmdr-robot

A [commander](https://github.com/gbryant/commander) consumer: a Raspberry Pi Pico 2 W
that drives a Roomba from a Bluetooth game controller, with WiFi + telnet + OTA. It's
the master half of a two-board robot — the other half is
[cmdr-oi-bridge](https://github.com/gbryant/cmdr-oi-bridge), an Arduino R4 sitting
between this board and the Roomba as an I2C→Open Interface bridge. You need both.

Naming: commander-based projects use a `cmdr-` prefix so they don't need to live under
one umbrella folder.

![Two boards mounted on top of a green Roomba: a Raspberry Pi Pico 2 W on a Grove
carrier at the top, an Arduino Uno R4 WiFi in a clear case below it, joined by
ribbon cables through a small Grove hub.](docs/img/robot.jpg)

*The two halves in place. Top: the Pico 2 W — Bluetooth pad in, drive commands out.
Bottom: the Uno R4 WiFi running [cmdr-oi-bridge](https://github.com/gbryant/cmdr-oi-bridge),
which turns those into Roomba Open Interface commands on `Serial1`. The cables between
them are the I2C link (Qwiic, 3.3 V) — the Pico is master, the R4 is the slave at
address 66.*

## Setup

Prereqs — four checkouts: the **Pico SDK**, the **FreeRTOS kernel**, **Bluepad32**
(Bluetooth) and **pico_fota_bootloader** (OTA). Commander's
[getting-started guide](https://github.com/gbryant/commander/blob/main/docs/getting-started.md)
bootstraps all four into `~/u-developer` with one script.

```bash
cp secrets.h.example secrets.h     # then fill in your WiFi
export PICO_SDK_PATH=~/u-developer/pico-sdk
export BLUEPAD32_PATH=~/u-developer/bluepad32
# PFB_PATH defaults to ~/u-developer/pico_fota_bootloader; FreeRTOS is found as a
# sibling of PICO_SDK_PATH. Set FREERTOS_KERNEL_PATH / PFB_PATH only if yours
# live somewhere other than the setup-sdks.sh layout.
cmake -B build-pico2 -S . -DPICO_BOARD=pico2_w     # also writes the dev scripts
./bum                                              # build + upload + monitor
./bum-ota                                          # wireless OTA (cmdr-robot.local)
```

The dev scripts (`bum`/`build`/`upload`/`monitor`/`bum-ota`) are gitignored, so a
fresh clone starts without them. On pico they come from the **cmake configure step
above**, not from `cmdr regen` as on the other targets — commander generates them
via `commander_generate_scripts()` at configure time, which is also why they hold
absolute paths and shouldn't be committed.

### Updating the commander framework

This project pins commander to a release tag (the `GIT_TAG` in `CMakeLists.txt`).
`cmdr pull` re-fetches that same tag, so on its own it changes nothing — to adopt a
newer release, **`cmdr pin <tag>` then `cmdr pull`** (rebuild after). `cmdr unpin`
floats on `main` instead; `cmdr update` updates the cmdr tool itself.

> Don't build against a local commander checkout as a normal workflow — it makes this
> project silently depend on unpublished commander state. (`cmdr link <path>`
> exists as a deliberate, temporary exception for framework development.)

## Control scheme (Wii U Pro / compatible pad)

- **Left stick Y** — throttle.
- **Right stick X** — steering (arcs only; never spins).
- **Hold L2/R2** — spin in place: left stick forward = CW, back = CCW. **R2** = normal
  speed, **L2** = slow (half) for fine course corrections.
- **`calibrate`** — interactive stick calibration. **`btforget`** — clear BT pairings.
  **`drivedbg`** — stream raw→calibrated→mixer values (temporary diagnostic).

Drive/spin smoothing, calibration, and the controller stack live in commander
(`modules/locomotion/DriveMixer.h`, `modules/controller/`); this project is the glue.

Hardware-confirmed: a paired pad drives a real Roomba through the I2C bridge, with
telnet live at the same time — WiFi and Bluetooth share the one CYW43 radio, which
works because the Pico runner owns a single `cyw43_arch_init()`.

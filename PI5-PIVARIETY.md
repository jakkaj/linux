# Arducam CamArray (Pivariety) on a Raspberry Pi 5: what we found, and the driver fix on this branch

> **What rots:** the mechanisms, the fix and the reproduce commands are durable for this driver and HAT firmware.
> The numbers are single-rig readings from 2026-10-17 (one Pi 5, one HAT, one OV9281 fitted), and the kernel and
> firmware versions will move on. If the upstream driver changes, re-check before relying on any of it.

Branch `pi5-pivariety-streamon` of a fork of `raspberrypi/linux`. Its base is `5ec7749803e9` (rpi-6.12.y,
"arducam-pivariety: Fix mutex init and NULL pointer"), the exact source of the module shipped in Raspberry Pi OS
kernel `6.12.20+rpt-rpi-2712`. That was verified by building it out of tree and matching the running module's
`srcversion` (`2F0090E2099B7A8E84B740D`); the commit before it gives a different one.

## Setup
- Raspberry Pi 5 (8 GB), Raspberry Pi OS Bookworm, kernel `6.12.20+rpt-rpi-2712` (`CONFIG_HZ=250`).
- Arducam CamArray stereo HAT (2x OV9281 mono). HAT MCU firmware `0x0003`. Only **one** camera was fitted, so the
  2560-wide composite has a black right half.
- `config.txt`: `camera_auto_detect=0` and `dtoverlay=arducam-pivariety` (CAM/DISP 1). The HAT has its own USB-C power.
- Capture by raw V4L2 on `rp1-cfe-csi2_ch0`, with no libcamera (see "libcamera" below):
  ```
  media-ctl -d /dev/mediaN -l '"csi2":4 -> "rp1-cfe-csi2_ch0":0 [1]'
  media-ctl -d /dev/mediaN -V '"csi2":0 [fmt:Y8_1X8/2560x800 field:none]'
  media-ctl -d /dev/mediaN -V '"csi2":4 [fmt:Y8_1X8/2560x800 field:none]'
  v4l2-ctl -d /dev/video0 --set-fmt-video=width=2560,height=800,pixelformat=GREY
  v4l2-ctl -d /dev/video0 --stream-mmap=4 --stream-count=300 --stream-to=/dev/null --verbose
  ```
  (Find N with `media-ctl -d /dev/mediaN --print-dot | grep rp1-cfe-csi2_ch0`.)

## Symptom: a ~32 s stall at every stream start (fixed on this branch)
With N buffers queued, frames 0..N-1 arrive at once, then nothing for ~32 s, then a steady stream. The buffer
sequence numbers show **no gaps**, although every frame in those 32 s is lost.

**Root cause.** `VIDIOC_STREAMON` itself blocks for 32.26 s (`strace -T -e trace=ioctl`). Kernel-stack samples
(`/proc/<pid>/stack` every 0.5 s) put all of it in `wait_for_free()`:

| phase | where | ≈ time |
|---|---|---|
| after the stream-on write | `pivariety_set_stream → wait_for_free` | 3.5 s |
| re-applying controls | `__v4l2_ctrl_handler_setup → pivariety_s_ctrl → wait_for_free` | 3 × 8 s |
| final wait | `pivariety_set_stream → wait_for_free` | 3.5 s |

- `wait_for_free()` polls `SYSTEM_IDLE_REG` (0x0107) until it reads 0, up to `1000 / interval` times with
  `msleep(interval)`, and returns 0 even on timeout.
- On this firmware the register reads **1 in standby and while streaming** (raw read:
  `i2ctransfer -f -y 11 w2@0x0c 0x01 0x07 r4` → `0x00 0x00 0x00 0x01`), so every call runs to the timeout.
- At `HZ=250`, `msleep(1)` and `msleep(2)` sleep ~8 ms, so "1 s" becomes 3.5–8 s per call.
- The HAT streams from the stream-on write. rp1-cfe fills the pre-queued buffers, then has no buffer while the
  application is still inside STREAMON, so it drops every frame. Those drops do not advance the sequence number.
  **Detect drops by timestamp period, not by sequence**, on this receiver.
- The same waits cost **~40 s at probe** (control enumeration): the sensor was only usable ~42 s after boot.

**Fix (commit `9d022c5`, verified: building this recipe reproduces srcversion `1970970E7777078574BD704`).** Bound `wait_for_free()` by wall-clock time (1 s, `usleep_range()`). After the first
timeout, set `idle_unsupported` and pace by delay from then on (one `v4l2_warn` says so). Devices that do clear the
flag are unaffected.

| measured (Pi 5, 2560x800 Y8, one camera) | stock | this branch |
|---|---|---|
| `VIDIOC_STREAMON` | 32.26 s | **0.024 s** |
| frames dropped at start (by timestamp period) | 2079 | **0** |
| sensor in use after boot | 42.4 s | **2.9 s** |
| 2 min soak | — | 7800/7800, 0 dropped, interval median 15.379 ms, max 15.405 ms |
| 10 min soak (stock, after its stall) | 39,500/39,500, p99.9 15.389 ms | — |

## Build it out of tree (no kernel rebuild)
```
sudo apt install linux-headers-rpi-2712          # Pi 5 on Bookworm; usually present: ls /lib/modules/$(uname -r)/build
mkdir pivariety && cd pivariety
for f in arducam-pivariety.c arducam-pivariety.h; do
  curl -fLO https://raw.githubusercontent.com/jakkaj/linux/pi5-pivariety-streamon/drivers/media/i2c/$f; done
echo 'obj-m := arducam-pivariety.o' > Makefile
make -C /lib/modules/$(uname -r)/build M=$PWD modules
sudo install -D -m 644 arducam-pivariety.ko /lib/modules/$(uname -r)/updates/arducam-pivariety.ko
sudo depmod -a && sudo reboot
cat /sys/module/arducam_pivariety/srcversion      # should no longer be the stock 2F0090E2...
```
Undo: `sudo rm /lib/modules/$(uname -r)/updates/arducam-pivariety.ko && sudo depmod -a && sudo reboot`.
A new kernel package does not carry the module over: rebuild after each kernel update. First check that the
upstream driver is still at `5ec7749803e9`; if it has moved, rebase this patch.
**Do not hot-reload the module** (`modprobe -r` / `modprobe`) under rp1-cfe. It re-binds into stale state
("Rejecting subdev ... Already set!!", a second capture node), and the next shutdown hung in its final stage.

## Known issues not fixed here
1. **The first stream after each boot delivers no frames; an immediate retry works.** It is not settling time
   (it still fails at 60 s uptime), not runtime PM (streams from the suspended state work), not fixed by a
   3.5 s delay after stream-on, and not fixed by writing standby first. The sensor driver logs the same
   sequence on the failing and the working attempt. Workaround in the application: if no frame arrives within
   1 s, STREAMOFF and STREAMON again. The retry's first frame comes at ~0.44 s. The old ~40 s probe delay used to
   hide this.
2. **The crop query wedges the HAT and then oopses.** `VIDIOC_SUBDEV_G_SELECTION` with `V4L2_SEL_TGT_CROP` makes
   the MCU stop answering (it holds the camera I2C SCL low, as seen with GPIO41 switched to a plain input). The
   driver then dereferences NULL with its mutex held (`pivariety_get_selection+0xec`), and every later call blocks
   in D state. Only a Pi power cycle recovers it. `media-ctl -p` and libcamera both issue this query.
3. **libcamera.** `rpicam-hello --list-cameras` / `cam -l` issue that crop query, wedge the HAT, and then report
   "No cameras available!". WirePlumber's libcamera monitor (`pipewire-libcamera`) probes cameras at every login,
   so disable it: `/etc/wireplumber/main.lua.d/51-no-libcamera.lua` containing `libcamera_monitor.enabled = false`.
4. **Back-power through Qwiic.** With the HAT's Qwiic lead joined to the Pi's I2C chain, cycling the HAT's USB-C
   does not reset its MCU. Keep the HAT off Qwiic.
5. **Controls had no visible effect** on this HAT with one camera: exposure 50/100/681/5000 and gain 1/15 gave the
   same image mean, before and during streaming.
6. **Frame rate:** 65.016 fps at 2560x800 (CLOCK_MONOTONIC buffer stamps, `ts-src-eof`), against a published 60 fps
   for that mode.
7. `rp1-cfe: Unable to determine sensor link rate, using 999 Mbps`: the driver exposes no `V4L2_CID_LINK_FREQ`.
   Harmless here.

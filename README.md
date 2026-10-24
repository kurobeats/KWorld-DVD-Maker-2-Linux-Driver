# KWorld DVD Maker 2 (`1b80:e600`) on Linux

Linux support for the KWorld DVD Maker 2 / Jaycar XC4867 RCA capture stick, which
enumerates as `1b80:e600` but reports an **unknown chip ID (146, EM28286/EM2980)**.
Mainline `em28xx` does not bind it. This repo contains the evidence trail and a
3-hunk patch that adds support.

## TL;DR

**Status: patch works.** Device binds as `KWorld DVD Maker 2 (card=113)`, registers
`/dev/video2`, and streams `YUYV 720x576` frames. Not yet tested with an actual analog
video source.


The device is *not* some unsupported alien design. It is an **EM2828x-family part
with the integrated video decoder**, and mainline `em28xx` already speaks that
protocol. The gap is three tiny things:

1. chip ID `146` is not in the `em28xx_chip_id` enum / switch,
2. USB ID `1b80:e600` is not in the device table,
3. there is no board entry.

## Evidence

| Fact | Where it came from |
|---|---|
| USB `1b80:e600`, product string "USB 28286 Device" | `lsusb` / `dmesg` on the capture host |
| Product is bound by eMPIA driver as `USB\VID_1B80&PID_E600&MI_00 ; HU319-E EM2980 Iso` | `docs/win-driver-inf-excerpt.txt` |
| Device reports **chip ID 146** (`0x92`) | Windows USB trace, `traces/` |
| Mainline knows `CHIP_ID_EM28281 = 145` and `CHIP_ID_EM2828X = 148`, not 146 | `em28xx-reg.h` |
| Mainline's `EM2828X_BUILTIN` decoder path exists and is used by 5 boards | `em28xx-cards.c` |
| `EM28281_BOARD_STARTECH_SVID2USB232` is "EM28281 with integrated TVP5150-compatible video decoder" | `em28xx-cards.c` |

### The decisive finding

The Windows driver's init sequence and mainline's `em2828X_decoder_vmux()` write the
**same 16-bit `0x7Axx` register bank with the same values**:

| Register | Windows trace | Mainline `em2828X_decoder_vmux()` |
|---|---|---|
| `0x7A0B` | `0x00` | `0x00` (composite/S-video) |
| `0x7A1C..0x7A1F` | `0x1E, 0x99, 0x99, 0x9A` | `0x1E, 0x99, 0x99, 0x9A` |
| `0x7A20 / 0x7A21` | `0x3D / 0x3E` | `0x3D / 0x3E` |
| `0x7A29`, `0x7A2F` | `0x00`, `0x52` | `0x00`, `0x52` |
| `0x7A40`, `0x7A51`, `0x7AC1` | `0x05`, `0x00`, `0x1B` | `0x05`, `0x00`, `0x1B` |
| `0x7A3F` | `0x01` then `0x00` (trigger) | `0x01` then `0x00` |
| `0xB6`, `0x38`, `0xB1`, `0xB3`, `0xB5` | `0x8F`, `0x01`, `0x70`, `0x00`, `0x00` | `0x8F`, `0x01`, `0x70`, `0x00`, `0x00` |

Same trigger-latch register, same ordering, same magic numbers. Also matching: the
16-bit EEPROM subaddressing (`eeprom_addrwidth_16bit`) and the I2C clock default
(`WAIT_ENABLE | FREQ_100_KHZ`).

These registers are *not* in upstream before the EM2828x support was added, so this
is not a coincidence — it is the same silicon generation.

## The patch

`patch/0001-media-em28xx-rtl-add-KWorld-DVD-Maker-2-EM28286.patch` — 3 hunks, 33 added lines:

- `em28xx-reg.h`: `CHIP_ID_EM28286 = 146`
- `em28xx.h`: `EM28286_BOARD_KWORLD_DVD_MAKER_2 = 114`
- `em28xx-cards.c`: chip-ID case (`wait_after_write = 0`, `eeprom_addrwidth_16bit = 1`),
  board entry (`EM28XX_BUILTIN` decoder, composite + S-Video, line-in audio), USB ID.

Board entry deliberately inherits the default XCLK/I2C settings, which match what the
Windows trace does (`I2C_CLK = WAIT_ENABLE | FREQ_100_KHZ`).

### Status: WORKS (binding + streaming proven)

Tested on Fedora 44, kernel `7.2.11-200.fc44.x86_64`. The device binds, registers
`/dev/video2` + `/dev/vbi0` + `/dev/media0`, and delivers frames:

```
em28xx 1-1:1.0: New device  USB 28286 Device @ 480 Mbps (1b80:e600, interface 0, class 0)
em28xx 1-1:1.0: Video interface 0 found: bulk isoc
em28xx 1-1:1.0: chip ID is em28286
em28xx 1-1:1.0: EEPROM ID = 26 00 01 00, EEPROM hash = 0x674caada
em28xx 1-1:1.0: 	AC97 audio (5 sample rates)
em28xx 1-1:1.0: 	500mA max power
em28xx 1-1:1.0: Identified as KWorld DVD Maker 2 (card=113)
em28xx 1-1:1.0: analog set to isoc mode.
em28xx 1-1:1.0: Config register raw data: 0x6a
em28xx 1-1:1.0: I2S Audio (3 sample rate(s))
em28xx 1-1:1.0: No AC97 audio processor
em28xx 1-1:1.0: V4L2 video device registered as video2
em28xx 1-1:1.0: V4L2 VBI device registered as vbi0
```

Inputs and formats as reported:

```
Input 0: Composite      (SDTV standards, autoswitch)
Input 1: S-Video        (SDTV standards, autoswitch)
Formats: YUYV, RGB565, 411P, and the Bayer variants (webcam path)
```

Capture verified with ffmpeg (`frames:v 20`): `rawvideo YUY2, yuyv422, 720x576,
29.97 fps` -> 20 frames written to FFV1. Frames are delivered and timestamps advance.

### Still unverified: an actual analog signal

The capture above was with nothing attached to the RCA jack — every frame was flat
`Y=16`, `YLOW=YLOW=16` (zero noise), `UAVG` alternating 127.8/179.7, i.e. the chip's own
blue/black background rather than digitised video. Nobody has yet fed it a working
composite source, so colour, AGC, VBI data and audio remain unconfirmed.

What *is* proven about the driver path: switching video standard flips the output
geometry (`--set-standard ntsc` -> 720x480, `pal` -> 720x576), so the register writes
reach the chip and the scaler responds. The driver is doing its job.

Two driver-side hypotheses were tested against a camcorder on the composite input and
**both produced no change**, so they are not the cause:

| Experiment | Result |
|---|---|
| Invert the `0x7A02` input mux (`0x4F` <-> `0x4E`, matching the Windows trace) | still flat blue, reverted |
| Force `XCLK = 0x0F` instead of the driver default `0x27` (trace value) | still flat blue, reverted |

Next physical checks, in order: camcorder in VCR/Play with A/V-out enabled; the OEM
4-pole breakout (generic 3.5mm TRRS cables misroute video); camcorder into the device's
*white* jack in case the silkscreen is wrong; and a known-good source such as a DVD
player with a plain RCA lead.

If a known-good source is confirmed on the wire, the next driver-side step is a debug
build that reads back `0x7A3A`/`0x7A3C` (the chip's status/lock registers, `0x0e`/`0x05`
in the Windows trace) during capture, to tell "no signal at the pin" apart from "signal
present but decoder not locking".

## Known issues

### `WARN_ON` on the VBI node (pre-existing upstream bug, not this patch)

Querying `/dev/vbi0` (udev's `v4l_id` does this on every plug) trips:

```
WARNING: drivers/media/v4l2-core/v4l2-ioctl.c:1117 at v4l_querycap+0x115/0x130
```

Cause is in `em28xx-video.c`, not in the added board entry:

- `querycap` only reports `V4L2_CAP_TUNER` when `tuner_type != TUNER_ABSENT`,
- but the VBI node's `device_caps` gets `V4L2_CAP_TUNER` added whenever the *video*
  node lacks it (`if ((v4l2->vdev.device_caps & V4L2_CAP_TUNER) == 0)`),
- so for any `TUNER_ABSENT` board with VBI, `capabilities` is not a superset of
  `device_caps` and the core's `WARN_ON` fires.

Video capture is unaffected (`/dev/video2`: 0 warnings). The Hauppauge USB Live2 and the
StarTech SVID2USB232 have the same shape (`TUNER_ABSENT` + builtin decoder + VBI), so this
predates the patch. Worth fixing separately before an upstream submission, since a driver
that warns on every device plug gets flagged in review.

### Other open questions

- Does the device need a firmware upload (`merlinFW.rom` ships in the driver package)?
  No firmware message appeared in dmesg and streaming works, so apparently not — but
  unverified against a real signal.
- Audio: the EEPROM advertises AC97, but chipcfg `0x6a` selects I2S and the driver
  concludes "No AC97 audio processor". So `em28xx-alsa` will expose an I2S card. Whether
  the I2S path actually carries the RCA audio is untested.
- Sibling PIDs (`e606`, `e607`, `e608`, `e60A`, `e60B`, `e60E`, `e60F`, `e610`) are also
  "EM2980 Iso" per the Windows INF and would likely work with the same board entry.
  Not added, because untested.

### Previously suspected, now disproved

- ~~Input setup ordering / `0x7A02` mismatch~~ — no probe failure; the board entry as
  written initialises cleanly.
- ~~`CHIP_ID_EM2828X` being in `em28xx_audio_setup()`'s digital-only list causes audio
  problems~~ — `EM28286` is deliberately excluded and the device correctly reports I2S.

## Testing

Verified working build recipe, against a Fedora 44 kernel `7.2.11-200.fc44.x86_64`.
The patch is generated against **v7.2.11** (matching that kernel). Master has since gained
`CHIP_ID_EM28281` / the StarTech board, so it needs a rebase before upstream submission.

### 1. Fetch the driver sources for your exact kernel

```
B="https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/plain/drivers/media"
mkdir -p ~/em28xx-build/inc ~/em28xx-build/drivers/media/usb/em28xx
cd ~/em28xx-build/drivers/media/usb/em28xx
for f in em28xx-core.c em28xx-i2c.c em28xx-cards.c em28xx-camera.c em28xx-video.c \
         em28xx-vbi.c em28xx-audio.c em28xx.h em28xx-reg.h em28xx-v4l.h; do
  curl -sO "$B/usb/em28xx/$f?h=v7.2.11"
done
cd ~/em28xx-build/inc
for t in xc2028 xc4000 xc5000; do curl -sO "$B/tuners/$t.h?h=v7.2.11"; done
```

### 2. Apply the patch and build

```
cd ~/em28xx-build
patch -p1 < 0001-media-em28xx-add-KWorld-DVD-Maker-2-EM28286.patch
```

`Makefile` — `obj-m`, not `obj-$(CONFIG_...)`, since `CONFIG_` is not set out-of-tree:

```make
obj-m += em28xx.o
obj-m += em28xx-v4l.o
obj-m += em28xx-alsa.o

em28xx-y      := em28xx-core.o em28xx-i2c.o em28xx-cards.o em28xx-camera.o
em28xx-v4l-y  := em28xx-video.o em28xx-vbi.o
em28xx-alsa-y := em28xx-audio.o

# em28xx.h needs tuner headers; em28xx-video.c includes em28xx-v4l.h next to itself
ccflags-y += -I$(src)/inc -I$(src)/drivers/media/usb/em28xx

KDIR := /lib/modules/$(shell uname -r)/build
all:
	$(MAKE) -C $(KDIR) M=$(PWD) modules
clean:
	$(MAKE) -C $(KDIR) M=$(PWD) clean
```

```
ln -s drivers/media/usb/em28xx/em28xx-*.c .
ln -s drivers/media/usb/em28xx/em28xx.h .
ln -s drivers/media/usb/em28xx/em28xx-reg.h .
sudo dnf install -y kernel-devel      # must match `uname -r`
make
```

`Skipping BTF generation ... due to unavailability of vmlinux` is expected and harmless.

### 3. Swap in the patched modules

`insmod` does **not** resolve dependencies — the deps have to be loaded first, or you get
`Unknown symbol in module`. This is the sequence that works:

```
# em28xx.ko
sudo modprobe mc videodev tveeprom
# em28xx-alsa.ko
sudo modprobe snd snd-pcm
# em28xx-v4l.ko
sudo modprobe videobuf2-common videobuf2-v4l2 videobuf2-vmalloc

sudo modprobe -r em28xx-v4l em28xx-alsa em28xx 2>/dev/null
cd ~/em28xx-build
sudo insmod em28xx.ko
sudo insmod em28xx-alsa.ko
sudo insmod em28xx-v4l.ko
```

Expected on success:

```
usbcore: registered new interface driver em28xx
em28xx: Registered (Em28xx Audio Extension) extension
em28xx: Registered (Em28xx v4l2 Extension) extension
```

If a dep is still missing, `dmesg` names it exactly:
`em28xx: Unknown symbol tveeprom_hauppauge_analog (err -2)` → `modprobe tveeprom`.

Now plug the device in. `udev`'s `modprobe em28xx` is a no-op because our module of the
same name is already resident — the stock driver will not steal the device.

### 4. Check it bound

```
dmesg | tail -40
```

Success looks like:

```
em28xx 1-1:1.0: New device KWorld DVD Maker 2 @ 480 Mbps (1b80:e600) ...
em28xx 1-1:1.0: chip ID is em28286
```

Failure modes and what they mean:

| dmesg | meaning |
|---|---|
| `unknown em28xx chip ID (146)` | patch not loaded (stock module got the device) |
| `em28xx: probe of ... failed with error -22` | board/init mismatch, inspect the failing write |
| nothing at all, `error -110`/`-62` | USB enumeration problem, not driver — powered hub / different port |

### 5. Feed it a signal and check formats

Connect the camcorder's **yellow composite** plug (or S-Video). Set the standard to match
the camcorder (most are PAL 625/50 or NTSC 525/60 — get this wrong and you get rolling
black-and-white).

```
v4l2-ctl --list-devices
v4l2-ctl -d /dev/video2 --list-inputs
v4l2-ctl -d /dev/video2 --list-formats-ext
v4l2-ctl -d /dev/video2 --set-input 0        # 0 = composite
v4l2-ctl -d /dev/video2 --set-standard pal
v4l2-ctl -d /dev/video2 --set-fmt-video=width=720,height=576,pixelformat=YUYV
```

### 6. Grab frames before touching OBS

ffmpeg gives far better error messages than OBS. Do this first:

```
ffmpeg -f v4l2 -i /dev/video2 -t 10 -c:v ffv1 out.mkv
ffplay out.mkv
```

If ffmpeg fails here, OBS will fail too, with less information.

### 7. OBS

Only once ffmpeg produces a picture:

1. `sudo dnf install -y obs-studio`
2. Add source → **Video Capture Device (V4L2)** → pick the `em28xx` device
3. Set resolution/format, and the input + video standard (OBS's V4L2 plugin exposes both)
4. Audio is a *separate* source: **Audio Input Capture (PulseAudio/ALSA)** → the `em28xx`
   card. This needs `em28xx-alsa.ko`; video will work without it, audio will not.

Recording tip for tape digitisation: OBS is fine for tests but for real captures prefer
`ffmpeg -f v4l2` or `v4l2-ctl --stream-mmap` — OBS re-encodes and drops frames under load.

### Getting back to stock

```
sudo rmmod em28xx-alsa em28xx-v4l em28xx    # or just reboot
```

## Host-side note: USB enumeration instability

On the machine this was first probed on (test machine, `xhci_hcd`), the device enumerated
once, then subsequent probes failed with `error -110` / `device not accepting address`
/ `error -62` and a failed port power-cycle. That is a power/enumeration problem, not a
driver problem — test on a powered hub or a desktop port before concluding the stick is
dead. The driver cannot help with that.

## Host-side note: USB enumeration instability (device failing)

The stick is not electrically reliable. Observed on the test machine across sessions, on
the same port, with everything else working:

```
[first session]  usb 1-1: device descriptor read/64, error -110
                 usb usb1-port1: attempt power cycle
                 usb 1-1: device not accepting address 11, error -62
                 usb usb1-port1: unable to enumerate USB device
[later session]  usb 1-1: device descriptor read/64, error -71
                 usb 1-1: new full-speed USB device number 21 using xhci_hcd
                 usb 1-1: Device not responding to setup address.
                 usb 1-1: device not accepting address 21, error -71
                 usb usb1-port1: unable to enumerate USB device
```

Two things make this a hardware verdict rather than a driver or port one:

- it is a **high-speed** device (`@ 480 Mbps` when it works) that intermittently falls
  back to **full-speed** (12 Mbps) before failing — a worn connector or failing power
  path does exactly this;
- it enumerates and binds correctly when it *does* link (see above), then drops off the
  bus mid-session.

Worth trying before condemning it: a **powered** hub (the EEPROM reports 500 mA max), a
different port, and no extension cable. It was never tested on a port other than the
one failing port.

## Debug log: chasing the missing analog picture

With a camcorder confirmed working on a TV, over the same cable, the device still emits a
perfectly uniform blue field. Observed values are identical across every configuration:

```
YAVG=15.9728  YLOW=16  YHIGH=16  UAVG=179.688  SATAVG=52.22
```

`YLOW == YHIGH` means zero noise in the luma plane. Real analog video always carries
noise, so this is the chip's internally generated "no signal" background, not a failed
digitisation of a real signal.

Experiments run (each rebuilt, redeployed, and re-captured):

| Experiment | Source | Result |
|---|---|---|
| Invert `0x7A02` input mux (`0x4F` <-> `0x4E`) | Windows trace | no change |
| `XCLK = 0x0F` instead of driver default `0x27` | Windows trace | no change |
| `0x48 = 0x01` at capture start instead of `0x00` | Windows trace | no change |
| Replay 8 registers mainline never writes + 3 differing values | Windows trace | no change |
| Both RCA jacks, both inputs (composite mux / S-Video mux) | - | no change |
| All standards: PAL, PAL-M, NTSC, NTSC-M, SECAM | - | no change (PAL-M alone showed `YLOW=0`, `YAVG=11.68`) |

The replay set, for anyone retrying: `0x7A08=5D 0x7A09=16 0x7A0A=7A 0x7A28=5E
0x7A30=2A 0x7A53=CC 0x7A5D=16 0x7A5E=50 0x7A60=B4 0x7A67=64 0xB0=3A 0xB2=38` then the
`0x7A3F` `01`->`00` trigger. Verified on the wire with `reg_debug=1` that all of them
reached the device.

### Register-level comparison, Linux vs Windows

Captured with `insmod em28xx.ko reg_debug=1`, which logs every vendor control transfer as
`OUT: 40 00 00 00 <wIndex LE> <wLen LE> >>> <data>` — so `wIndex` is the register. 229
writes, 67 distinct registers. Mainline already matches Windows on the large majority,
including the entire `0x7Axx` setup and the `0x7A3F` trigger. The gaps were the 8 registers
above plus `0x7A30`, `0x7A08/09/0A`, `0x7A02`, `0x48`. Closing the gap changed nothing.

### Conclusion

Register state now closely matches the Windows driver and the input still produces
nothing, on hardware whose USB link is independently unreliable (see below). The most
probable remaining explanation is a faulty analog input path on this unit. The decisive
test is to run the stick under Windows with its own driver: if it shows no picture there
either, the unit is dead and no further driver work will help.

Note the driver *is* demonstrably controlling this chip: changing the video standard
flips the output geometry between 720x576 and 720x480.

## SELinux: loading test modules from a home directory fails

On Fedora (Enforcing), `modprobe` runs as `kmod_t` and cannot load a module from a home
directory:

```
avc: denied { module_load } for comm="insmod"
  path="/home/user/em28xx-build/em28xx-v4l.ko"
  scontext=system_u:system_r:kmod_t tcontext=unconfined_u:object_r:user_home_t
  tclass=system permissive=0
```

The symptom is a silent load failure (`modprobe` returns 0-ish, no `/dev/video2`). Fix —
install under `/lib/modules` and label it:

```
D=/lib/modules/$(uname -r)/extra/em28xx-test
sudo mkdir -p $D
sudo cp em28xx.ko em28xx-alsa.ko em28xx-v4l.ko $D/
sudo restorecon $D/*.ko                 # -> modules_object_t
sudo chcon -u system_u $D/*.ko
```

Then pin the load path so a `udev`-triggered `modprobe` uses the patched modules instead
of fighting over the stock ones:

```
printf 'install em28xx /sbin/insmod %s/em28xx.ko\ninstall em28xx-alsa /sbin/insmod %s/em28xx-alsa.ko\ninstall em28xx-v4l /sbin/insmod %s/em28xx-v4l.ko\n' $D $D $D \
  | sudo tee /etc/modprobe.d/zz-em28xx-test.conf
sudo restorecon /etc/modprobe.d/zz-em28xx-test.conf
```

Remove that file and the directory to get back to stock.

## Provenance of `traces/`

`traces/usb-trace-windows-gusfxmgk.txt` is a USB capture of this exact device
(`1b80:e600`) under the Windows driver, published by a user in
<https://superuser.com/questions/1407808/using-the-dvd-maker-2-em2980-chip-on-xenialpup>
(original: <https://pastebin.com/GUSfxmgK>). Registers are already annotated with
Linux `em28xx` register names by that user. Included here as it is the ground truth the
patch was derived from. Credit to them.

## Licence

The patch is kernel code and is `GPL-2.0-only`, matching
`drivers/media/usb/em28xx/`. Kernel patches also require a `Signed-off-by:` trailer
(`git commit -s`) before submission to linux-media.

Documentation and analysis text: CC0 / public domain, do whatever helps.

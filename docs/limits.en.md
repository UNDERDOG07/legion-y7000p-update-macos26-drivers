# What this machine cannot do

Four of these are not "not configured yet". They are blocked below the config layer, and the reason
matters more than the workaround.

## Sleep

The GPU does not come back. Waking produces a black internal panel and, on the external output, either
nothing or a corrupted frame.

No `hibernatemode` value, USB power setting or `pmset` flag fixes this, because the fault is in the
driver's reinit path rather than in power management. `/var/vm/sleepimage` is irrelevant — hibernation
never succeeds here either.

Practical answer: turn sleep off and use the lid as a screen-off switch.

```bash
sudo pmset -a sleep 0
```

## AirDrop, Handoff, Continuity

All three ride on AWDL, the peer-to-peer Wi-Fi interface. AirportItlwm does not implement it, and no
EFI setting or kext brings it back.

Bluetooth is not the cause. That is worth stating plainly, because the symptom (nearby devices that
will not exchange files) reads like a Bluetooth problem and sends people down the wrong path.

## Secure Boot

Two routes, both closed by this firmware.

**Self-signed `db`.** Generate a key, sign `BOOTx64.efi`, `OpenCore.efi` and every driver, enrol the
certificate. This machine's BIOS only offers a destructive setup mode that clears existing keys; there
is no append path. Enrolling would mean reinstalling Microsoft's PK/KEK/db by hand and putting the
Windows boot chain at risk.

**shim.** Boot Microsoft-signed `shim`, let it chain to a signed OpenCore, enrol our certificate into
MOK. The chain itself works — shim loads, Bootstrap runs, OpenCore starts. What fails is `MokList`:
after enrolling `db.cer`, every `Mok*` variable reads back absent through `IODeviceTree`. The BIOS
reports `Device Guard` active and refuses writes to that store.

So Secure Boot stays off, and the EFI on the stick is unsigned by design. **Turning Secure Boot on will
stop this machine from booting.**

One diagnostic detail worth keeping: `nvram -p` only lists Apple's own variables, so `Mok*` looks absent
even when it is not. Read them through `ioreg -p IODeviceTree -n options -r -l -w0` instead.

## Hardware video decode

No NVDEC on this GPU under macOS, and no iGPU driver, so there is no decode engine in the path.

VideoToolbox has a Metal-backed software route that is 2.5–4× faster than ffmpeg's CPU decoder for
HEVC, but it is not hardware decode, and H.264 gets nothing from it. Numbers and method in
[gpu.md](gpu.md).

## Also broken

- **DRM playback** — Apple TV, Netflix. No HDCP path in the driver. Not worth investigating.
- **Wi-Fi 6** — the card supports 802.11ax, the link stays at ac/80 MHz. See [wireless.md](wireless.md).
- **Some BLE peripherals** — pair, then drop. Firmware-side, no config fix.
- **RGB keyboard zones** — the controller is a USB HID device with no macOS driver. Backlight works,
  colours do not.

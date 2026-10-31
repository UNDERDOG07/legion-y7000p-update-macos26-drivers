# Audio

Speakers and the internal microphone both work, with two kexts doing the work and one device property
tying it to the codec.

## What is running

| Piece | Version | Where |
|---|---|---|
| [`AppleALC.kext`](https://github.com/acidanthera/AppleALC) | 1.9.9 | `EFI/OC/Kexts` |
| [`SimpleHDA.kext`](https://github.com/laobamac/SimpleHDA) | 1.0.0 | `EFI/OC/Kexts` |
| `layout-id` | `99` | `DeviceProperties → Add → PciRoot(0x0)/Pci(0x1f,0x3)` |

`SimpleHDA` is scoped to `MinKernel 25.0.0` / `MaxKernel 25.99.99`, which is Darwin 25 — macOS 26
only. It is the one audio component here that is tied to the OS version.

Its purpose is stated plainly in the project description: **restore native AppleHDA on macOS Tahoe
through OpenCore and Lilu.** macOS 26 changed how `AppleHDA` loads, and AppleALC alone does not cover
that — this kext puts the framework back first.

Version comparison in [sources.md](sources.md).

## Verifying

```bash
system_profiler SPAudioDataType | grep -E "Default (Output|Input)"
```

Should return both. If output is missing, look at the codec first:

```bash
ioreg -l | grep -i "IOAudioDeviceName"
```

If the HDA device is present but no audio device is created, the layout is wrong — try the standard
ALC257 layouts (`11`, `13`, `99`) before anything else. The codec on this board is ALC257, driven
through Intel SST.

## The part that is not audio

Permissions for the microphone are a separate problem and a much nastier one. The hardware records
fine from the command line while every GUI app gets refused, because the TCC layer never shows a
prompt. If the microphone "does not work" in one app but `ffmpeg` can record, the audio stack is fine
and you are in [permissions.md](permissions.md).

## Known quirks

- The input source reports as `内置麦克风` (internal microphone). Switching that name is not configurable
  through `layout-id`.
- Volume keys work through the standard HID path; the OSD appears normally.
- Headphone jack detection has not been tested. The machine is used with the internal speakers and a
  USB audio interface (FRANSUN KT02H20), which enumerates as a separate device and needs no kext.

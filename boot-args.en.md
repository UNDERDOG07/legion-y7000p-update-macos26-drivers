# Boot arguments

Fourteen arguments, in the order they sit in `boot-args`:

Thirty-two. The first fourteen are the base, the last eighteen are the AirDrop increment, in
`boot-args` order:

```
debug=0x100 keepsyms=1 -vi2c-force-polling ctrsmt=full
nvfb=1 nvaccel=1 nvfbheads=4 -nvkmsnosmooth
amfi_get_out_of_my_way=0x1 amfi=0x80 ipc_control_port_options=0
itlwm_he=1 revpatch=sbvmm,cpuname -ibtcompatbeta

# added for AirDrop (ForbiddenItlwm):
-btleadv -itlawdlcoexprobe -itlawdliface -itlawdlnativeup -itlawdlcoexrx
-itlawdlcoextx -itlawdlcoexmulti -itlawdlcoexdata -itlawdlmitx -itlawdlelicit
-itlawdlcaps -itlawdlsvcmerge -itlawdlawsync itlawdlintcoal=4
-itlawdlcoexbatx -itlawdlcoexpeerpause -itlawdlcoexbatrace itlawdltxqueue=4096
```

They fall into groups, and the confidence level differs per group. Some were tested by taking them out
and watching the machine break. Others are carried over from the driver package's own instructions and
have never been isolated one at a time. The difference is marked below.

## The two everything else depends on

**`amfi=0x80`** turns AMFI off. The NVIDIA kexts are unsigned, so without it they do not load and there
is no picture at all. `amfi_get_out_of_my_way=0x1` relaxes the same check one layer up, and
`ipc_control_port_options=0` keeps the Mach IPC restrictions added in Ventura from interfering with
the same path. This trio is not optional on this machine.

It also has a cost that took a while to trace: with AMFI off, macOS stops showing privacy prompts for
camera and microphone. See [permissions.md](docs/permissions.md).

**`ctrsmt=full`** is read by `CpuTopologyRebuild`. Left out, the machine exposes 24 cores and macOS
schedules the 16 E-cores as if they were P-cores. With it, E-cores are presented as SMT siblings:
8 physical, 32 logical, which is what `hw.ncpu` reports now.

## GPU

`nvfb=1`, `nvaccel=1`, `nvfbheads=4`, `-nvkmsnosmooth`.

These come straight out of the NullMoth package's own instructions, and this repo has never tried
removing them individually. In order: enable the framebuffer, enable acceleration, keep four
framebuffer heads available, and turn off the KMS smoothing transition.

The names describe the intent; they are not a claim that each one was verified in isolation. If you
are debugging a picture problem, these are the first four to experiment with, but change one at a time
and keep the working set in a text file.

## Boot path

`debug=0x100` and `keepsyms=1` keep panic information and symbol names available after a crash. They
cost a little memory and nothing else. Keep them: the one panic this machine had was diagnosed entirely
from those two.

`-vi2c-force-polling` makes VoodooI2C poll instead of waiting for interrupts. Without it the trackpad
does not respond on this machine — the kext stack loads, the device never reports. See
[input.md](docs/input.md).

## Wi-Fi and Bluetooth

`itlwm_he=1` tells AirportItlwm to advertise HE (802.11ax) support. The card supports ax in hardware
and reports it in `Supported PHY Modes`. In practice this machine still associates at 802.11ac, so the
argument is present for the capability rather than for a measurable gain — see
[wireless.md](docs/wireless.md) for the measurements.

`-ibtcompatbeta` is read by `IntelBTPatcher`, not by the firmware driver. **In the current
configuration it does nothing**: `IntelBTPatcher` is disabled to make room for `BTLEAdvFixup`, so
nothing parses the flag. It stays only so that reverting needs one less edit.

### The eighteen AirDrop flags

This group only means anything under `ForbiddenItlwm`. It is copied verbatim from the "tested set" in
the author's `docs/BOOT-FLAGS.md`; no removal test has been run on any of them, so **what each one
claims to do is documented, but which of them this machine actually needs is not known.**

| Group | Flags | Claimed effect |
|---|---|---|
| Interface and listener | `-itlawdliface` `-itlawdlcoexprobe` `-itlawdlnativeup` `-itlawdlcoexrx` `-itlawdlcoextx` | creates `awdl0`, opens a second firmware context, lets macOS's AWDL peer manager start, receives frames, gives the listener its own queue |
| Coexistence scheduling | `-itlawdlcoexmulti` `itlawdlintcoal=4` `-itlawdlcoexpeerpause` `-itlawdlcoexbatx` `itlawdltxqueue=4096` | puts the listener on its own channel, shortens interrupt coalescing, pauses the peer queue while it is away, outbound Block Ack, transmit queue size |
| Announcement and discovery | `-itlawdlmitx` `-itlawdlelicit` `-itlawdlcaps` `-itlawdlsvcmerge` `-itlawdlawsync` | sends the Mac's own AWDL announcement, adds the fields Apple devices expect, reports two spatial streams and VHT, merges service records, aligns with the peer's availability windows |
| Data path | `-itlawdlcoexdata` `-itlawdlcoexbatrace` | carries AirDrop data both directions; the second only records Block Ack events in the I/O Registry, for diagnosis |
| Bluetooth | `-btleadv` | turns on `BTLEAdvFixup`, which is how the iPhone finds this Mac over BLE |

Two consequences:

- **Remove `-btleadv` before updating macOS**, or the delta update may fail. See
  [docs/updates.en.md](docs/updates.en.md)
- Going back to `AirportItlwm`, remove all eighteen. They do no harm, but they do nothing there and
  only make the configuration look more complex than it is

## Updates and the CPU name

`revpatch=sbvmm,cpuname` is read by `RestrictEvents` and does two unrelated things at once:

- `sbvmm` reports a VMM secure-boot model, which is what makes delta updates possible on an unsupported
  board. See [updates.md](docs/updates.md).
- `cpuname` allows the CPU name to be overridden, which is what puts `i9-14900HX` back in About This
  Mac. Without it the name is derived from SMBIOS and comes out empty.

Both live in the same token, and that is worth remembering: anything that rewrites `revpatch=` in
`boot-args` silently replaces both. That is exactly how the CPU name was lost for a week — see
[power.md](docs/power.md).

There is a second copy of `revpatch` in `NVRAM → Add` under the RestrictEvents GUID, with `cpuname`
already in it. **`boot-args` wins.** Keeping both is harmless but confusing; the NVRAM copy is a
backup, not a config.

## Trimming the list

If you want to see how much of this is actually needed, remove in this order and keep notes:

1. `debug=0x100`, `keepsyms=1` — safe, only affects diagnostics
2. `itlwm_he=1` — expect no visible change
3. `-ibtcompatbeta` — expect no visible change
4. `nvfbheads=4`, `-nvkmsnosmooth` — do these one at a time, watch for flicker on the internal panel

Do not remove `amfi=0x80`, `ctrsmt=full`, or `revpatch=sbvmm` unless you are deliberately giving up
the GPU driver, the CPU topology, or delta updates.

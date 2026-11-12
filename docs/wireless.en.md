# Wi-Fi and Bluetooth

Two separate stacks, two separate results: Wi-Fi is good enough to use, Bluetooth works but is the
weaker half.

## Wi-Fi

| Piece | Value |
|---|---|
| Card | Intel **AX1690i**, `8086:1692` |
| Driver | [`AirportItlwm.kext`](https://github.com/laobamac/itlwm) 2.4.0 (**fork**) |
| Firmware | `itlwm: 2.4.0_738d4c6 fw: 68.01d30b0c.0` |
| Boot argument | `itlwm_he=1` |
| Interface | `en0` |

This machine runs `AirportItlwm`. The old config used `itlwm.kext`; the difference between the two:

| | `itlwm` | `AirportItlwm` |
|---|---|---|
| Presents as | an Ethernet device | a native Airport interface |
| System Wi-Fi menu | none | yes |
| Scan / associate | through a private tool such as HeliPort | handled by the system |

The "Wi-Fi ❌" entry in the old notes is a state from the `itlwm` period.

Upstream [OpenIntelWireless/itlwm](https://github.com/OpenIntelWireless/itlwm) stops at **v2.3.0
(2024-06)** and does not support macOS 26. The 2.4.0 here comes from **`v2.4.0-alpha`** in
[laobamac/itlwm](https://github.com/laobamac/itlwm); the bundle identifier is still
`com.zxystd.AirportItlwm`. **Downloading 2.3.0 from upstream will not work here** — comparison in
[sources.en.md](sources.en.md).

### What it actually negotiates

Measured here at `-34 dBm` — a very strong signal, so the link rate is not distance-limited:

| Field | Value |
|---|---|
| PHY mode | **802.11ac** |
| Channel | 157 (5 GHz, **80 MHz**) |
| Tx rate | 867 Mbps |
| MCS / NSS | 9 / 2 |
| Country code | CN |

The card advertises `802.11 a/b/g/n/ac/ax`, and `itlwm_he=1` is set, but the link still comes up as
802.11ac at 80 MHz. On the same access point an ax/160 MHz link would be roughly three times faster.

Before blaming the driver, check two things on the router: whether 802.11ax is enabled for the 5 GHz
band (many default to ac compatibility) and whether 160 MHz is permitted. If both are open and it still
negotiates ac, the limit is in the driver's HE handling.

### Checking

```bash
system_profiler SPAirPortDataType | grep -E "PHY Mode|Channel"
sudo wdutil info
```

`wdutil` reports more than the menu bar. Holding Option and clicking the Wi-Fi icon gives the same
fields in a dialog, including RSSI and noise — with the caveat that this driver reports noise as
`0 dBm`, so that field is meaningless here.

### AirDrop

**It works, both directions, slowly.** It requires a different driver: `AirportItlwm` has no AWDL
interface, so the Wi-Fi kext is replaced with
[ForbiddenItlwm](https://github.com/retropineaple/ForbiddenItlwm).

| Piece | Version | Role |
|---|---|---|
| `ForbiddenItlwm.kext` | 0.4.1-beta1 | replaces AirportItlwm |
| `BTLEAdvFixup.kext` | 0.8.3 | Lilu plugin, takes over IntelBTPatcher's hook |
| Boot arguments | 18 of them | see [boot-args.en.md](../boot-args.en.md) |

Three hard requirements; miss one and it does not work:

1. **The two Wi-Fi drivers cannot both be enabled.** They share class names. `AirportItlwm.kext` stays
   in `Kexts` for rollback, with its entry set to `Enabled=false`
2. **`IntelBTPatcher` must be disabled.** It and `BTLEAdvFixup` hook the same function, and having both
   loaded makes Lilu panic
3. **Both new kexts are limited to Tahoe** — `MinKernel 25.0.0` / `MaxKernel 25.99.99` — so booting
   another macOS version from this stick cannot load the wrong build

Measured state (26.7.2, country CN):

```
ItlwmAWDLBuildCheck            verified
ITLAWDLCoexAdded               Yes
ITLAWDLCoexCountryLearned      Yes
ITLAWDLCoexChannel             6      expected 149
ITLAWDLCoexOwnWidth            20     expected 80
ITLAWDLCoexPeerPresentEntries  1      of 16 slots
ITLAWDLCoexTxSubmitErrors      0
BTLEAdvHookRouted              1
```

The listener sits on channel 6, which is the direct cause of the slow transfers. Details in
[limits.en.md](limits.en.md).

**Before updating macOS, boot once without `-btleadv`.** `BTLEAdvFixup` patches `bluetoothd`'s code
pages; an update that verifies that file will fail, and the delta update falls back to the full
installer.

**Until the first transfer completes, `TransportReady` reads `No` and `DataCopyAttempts` stays at 0.**
That is normal and does not mean the driver is idle.

## Bluetooth

| Piece | Version | State |
|---|---|---|
| `IntelBluetoothFirmware.kext` | 2.5.0 (fork) | enabled |
| `IntelBTPatcher.kext` | 2.5.0 (fork) | **disabled** (conflicts with BTLEAdvFixup) |
| `BTLEAdvFixup.kext` | 0.8.3 | enabled (AirDrop needs it) |
| `BlueToolFixup.kext` | 2.7.3 | enabled |
| Boot arguments | `-ibtcompatbeta`, `-btleadv` | see below |

Three resident kexts, three jobs. The firmware driver uploads the blob; `BTLEAdvFixup` patches
`bluetoothd` (a job `IntelBTPatcher` used to do); `BlueToolFixup`, a Lilu plugin, makes the controller
present itself as an Apple one.

That last piece explains why System Information shows `Chipset: THIRD_PARTY_DONGLE` and
`Vendor 0x004C` rather than Intel. It is deliberate — without the disguise macOS 26 does not attach the
controller at all.

`-ibtcompatbeta` is read by `IntelBTPatcher`, not by the firmware driver. **Right now the flag does
nothing** — `IntelBTPatcher` is disabled, so nothing parses it. It is left in place so that reverting to
the old configuration does not require another edit.

`-btleadv` is read by `BTLEAdvFixup` and is the prerequisite for AirDrop discovery. Without it the
iPhone never sees this Mac.

### Checking

```bash
system_profiler SPBluetoothDataType | head -20
```

`State: On` plus nearby devices means the stack is up. The firmware upload shows in the kernel log:

```
IOUSBHostDevice: setConfigurationGated: IntelBluetoothFirmware selected configuration 1
```

A `kernelmanager_helper: Could not find identifier (com.zxystd.IntelBluetoothFirmware)` line is normal
for an injected kext and is not an error.

### What connects, and what does not

Controllers and headphones work. Other classes of device do not — pairing either fails outright or the
device drops straight after pairing.

"Bluetooth works" on its own is misleading. Two of the most common
peripheral types are fine, and most of the rest of the category is not. A controller (GameSir G7 Pro)
and headphones are what this machine is actually used with, and both are stable.

2.4 GHz coexistence is a factor too — both antennas are in the lid, so a busy Wi-Fi link hurts
Bluetooth range.

Handoff and Continuity were not tested. They use AWDL as well, so the conditions should match AirDrop.

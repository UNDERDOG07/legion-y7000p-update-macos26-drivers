# Wi-Fi and Bluetooth

Two separate stacks, two separate results: Wi-Fi is good enough to use, Bluetooth works but is the
weaker half.

## Wi-Fi

| Piece | Value |
|---|---|
| Card | Intel CNVi part, `8086:1692` — factory, never replaced |
| Driver | [`AirportItlwm.kext`](https://github.com/laobamac/itlwm) 2.4.0 (**fork**) |
| Firmware | `itlwm: 2.4.0_738d4c6 fw: 68.01d30b0c.0` |
| Boot argument | `itlwm_he=1` |
| Interface | `en0` |

The older notes list an AX1690i at `8086:7A70` and describe it as swapped in by the owner. **Neither is
correct** — the card is the factory one, and `7A70` is not this machine's ID. What the system reports
is what to go by; that row in the old notes can be ignored entirely.

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

### AirDrop does not work

Not Bluetooth, and not fixable with kexts. AirportItlwm has no AWDL interface, which is what AirDrop,
Handoff and Continuity all use. See [limits.md](limits.md).

## Bluetooth

| Piece | Version |
|---|---|
| `IntelBluetoothFirmware.kext` | 2.5.0 |
| `IntelBTPatcher.kext` | 2.5.0 |
| `BlueToolFixup.kext` | 2.7.3 |
| Boot argument | `-ibtcompatbeta` |

Three kexts, three jobs. The firmware driver uploads the blob; `IntelBTPatcher` applies compatibility
patches for newer macOS builds; `BlueToolFixup`, a Lilu plugin, makes the controller present itself as
an Apple one.

That last piece explains why System Information shows `Chipset: THIRD_PARTY_DONGLE` and
`Vendor 0x004C` rather than Intel. It is deliberate — without the disguise macOS 26 does not attach the
controller at all.

`-ibtcompatbeta` is read by `IntelBTPatcher`, not by the firmware driver. Whether it helps here is not
settled: Bluetooth connects with it present, some BLE devices still drop, and no A/B test has been run
with it removed.

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

That split matters, because "Bluetooth works" on its own is misleading. Two of the most common
peripheral types are fine, and most of the rest of the category is not. A controller (GameSir G7 Pro)
and headphones are what this machine is actually used with, and both are stable.

2.4 GHz coexistence is a factor too — both antennas are in the lid, so a busy Wi-Fi link hurts
Bluetooth range.

Handoff and Continuity do not work, for the same AWDL reason as AirDrop.

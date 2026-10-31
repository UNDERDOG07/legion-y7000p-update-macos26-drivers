# Model and serial number

## Current identity

| Field | Value |
|---|---|
| `SystemProductName` | `MacBookPro16,4` |
| Board ID | `Mac-A61BADE1FDAD7B05` |
| CPUID spoof | family 6 / model `0xA5` / stepping 5 |

`MacBookPro16,4` is the 2019 16-inch with the Radeon Pro 5600M — the newest Intel large-screen machine
Apple shipped, 8-core, with a discrete GPU. For a laptop whose panel has to be driven by a dGPU running
macOS 26, this is the closest match that exists.

`Kernel → Emulate → Cpuid1Data = 55 06 0A 00` reports model `0xA5` (Raptor Lake) rather than the real
`0xB7` (Raptor Lake Refresh). The `0xA5` power-management path is better exercised, and this has been
stable on this machine.

## The serial decides what About This Mac says

This is the trap that cost an afternoon.

About This Mac does not read the model name from SMBIOS. It takes the **last four characters** of the
serial number and asks Apple:

```bash
curl "https://support-sp.apple.com/sp/product?cc=ML7H&lang=en"
```

Whatever that returns is what the UI shows. A serial whose last four characters belong to a different
model produces a machine that is internally consistent and displays the wrong thing:

| Field | Value | Points at |
|---|---|---|
| SMBIOS product name | `MacBookPro16,4` | 16-inch 2019 |
| Board ID | `Mac-A61BADE1FDAD7B05` | 16-inch 2019 |
| Serial (old) | ended `ML7H` | **13-inch 2020, four Thunderbolt ports** |

Everything except the serial was right, and the display was wrong. The old serial was also a real
machine's number, which is its own problem — iServices keys devices by serial.

## Fixing it

Generate serials for the target model, then check the config code rather than trusting the output:

```bash
./macserial -m MacBookPro16,4 -n 10
```

For `MacBookPro16,4` the config code is `MD6T`. Candidates ending in `MD6T` resolve to
"MacBook Pro (16-inch, 2019)":

```bash
curl "https://support-sp.apple.com/sp/product?cc=MD6T&lang=en"
```

Change all three fields together:

- `SystemSerialNumber`
- `MLB` — must come from the same generation as the serial
- `SystemUUID` — regenerate, a fresh v4 is fine

`ROM` was left alone. Ideally it is the `en0` MAC, but changing it is another identity change, and
services were working.

## Before you change it

Sign out of Apple ID first, including Find My Mac. Changing the serial while signed in leaves a ghost
device on the account that is tedious to remove.

Expect iServices to ask for verification afterwards. If it refuses outright, wait 24 hours instead of
retrying — repeated attempts raise the risk score.

`~/Library/Preferences/ByHost/` entries keyed to the old `IOPlatformUUID` stop applying, so display and
sound preferences may reset once. LaunchAgents are not affected, since they are keyed by label.

## Reading the SMBIOS table

`ioreg` does not expose the strings in plain text; the table is a binary blob. To read it:

```bash
ioreg -c AppleSMBIOS -r -l -w0 | grep '"SMBIOS" = <'
```

Decode the hex, walk the structures, and Type 1 holds the product name and serial, Type 4 the processor
string. That is how the `Unknown` CPU name was traced — SMBIOS had the correct string the whole time,
and macOS simply was not using it.

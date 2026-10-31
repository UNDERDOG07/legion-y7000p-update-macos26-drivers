# Keyboard and trackpad

## Keyboard

Two keyboards, handled differently.

| Device | Bus | Driver |
|---|---|---|
| Internal | PS/2 | `VoodooPS2Controller` |
| External (MCHOSE A7 V2 Pro+) | USB HID | native, no kext |

The external board is a Windows-layout keyboard, so the modifiers sit one position off from a Mac
keyboard: `Ctrl / Win / Alt` where macOS expects `Ctrl / Option / Command`.

### Modifiers

Nothing needs remapping. Both keyboards send the modifiers where macOS expects them, and
`hidutil property --get UserKeyMapping` returns `null` — that is the correct state here, not a missing
configuration.

Swapping Option and Command is the first thing most people do on a Windows keyboard. Here it would be a
regression — the external board's `Win` key is already Command and `Alt` is already Option.

If you do end up with a board that needs the swap, **System Settings → Keyboard → Keyboard Shortcuts →
Modifier Keys** has a panel that lists each keyboard separately. It is the better option when it is
needed, since it survives updates and needs no login item.

`hidutil` is the alternative when the panel cannot express what you want — mapping Caps Lock to
Control, for instance, or applying a rule to one vendor/product ID only:

```bash
hidutil property --matching '{"VendorID":0x3837,"ProductID":0x100a}' \
  --set '{"UserKeyMapping":[{"HIDKeyboardModifierMappingSrc":0x7000000E6,
                            "HIDKeyboardModifierMappingDst":0x7000000E7}]}'
```

It does not persist across reboot. A LaunchAgent is needed to reapply it.

### Function keys

The F-row sends standard function keys; brightness and volume sit on the Fn layer. That is normal for
this chassis and matches how the machine behaves under Windows.

## Trackpad

It works. **That is not an OS update side effect — it started working when a dead reference in the
config was removed.**

`VoodooInput` is the trackpad input protocol layer. **VoodooI2C 2.9.1 ships only `VoodooGPIO` and
`VoodooI2CServices`, not `VoodooInput`** — it depends on the copy that comes with VoodooPS2.

The old config carried two `VoodooInput` entries:

- one pointing at `VoodooI2C.kext/Contents/PlugIns/VoodooInput.kext` — **enabled**, but there is no file
  at that path
- one pointing at `VoodooPS2Controller.kext/Contents/PlugIns/VoodooInput.kext` — **disabled**

So the layer never loaded at all. Every kext was up, the device was present, and the trackpad did
nothing — the state the old notes describe as "the stack is there, it never responded".

The current combination:

| Piece | Version | State |
|---|---|---|
| [`VoodooI2C.kext`](https://github.com/VoodooI2C/VoodooI2C) | 2.9.1 | enabled |
| `VoodooI2CHID.kext` | 1.0 | enabled |
| `VoodooGPIO.kext` | 1.1 | enabled |
| `VoodooI2CServices.kext` | 1.0 | enabled |
| `VoodooInput.kext` from VoodooPS2Controller | 1.1.7 | **enabled** (the key one) |
| `VoodooPS2Mouse.kext` / `VoodooPS2Trackpad.kext` | 2.3.8 | **disabled** |
| Boot argument | | `-vi2c-force-polling` |

The changes: the entry pointing at a file that does not exist is gone, VoodooPS2's `VoodooInput` is on,
and the PS2 mouse and trackpad plugins are off.

The device is I2C HID on GPIO, `INTC1085`. One `VoodooI2CHIDDevice` instance attaches.

`-vi2c-force-polling` was already in the old config and never left. It makes the driver poll instead of
waiting on the GPIO interrupt — remove it and the kexts still load, the device just never reports input.
It is necessary, not sufficient: the old config had it and the trackpad still did nothing.

### Checking

```bash
ioreg -c VoodooI2CHIDDevice | grep -c "class VoodooI2CHIDDevice"
```

`1` means the device attached. `0` means the stack is not reaching the hardware — check first that
`SSDT-GPI0.aml` is loaded and that the `GPI0 _STA → XSTA` rename in ACPI is enabled, since the GPIO
controller has to be handed to the kext rather than to the firmware.

## RGB lighting

The 4-zone RGB controller is a separate USB HID device (`048d:c995`) with no macOS driver. The keyboard
backlight itself works, because that is firmware-controlled and survives the OS not knowing about it.

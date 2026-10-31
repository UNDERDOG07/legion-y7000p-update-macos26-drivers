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

This is worth stating plainly, because swapping Option and Command is the first thing most people do on
a Windows keyboard, and on this machine it would be a regression. The external board's `Win` key
behaves as Command and `Alt` as Option without any change.

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

It works — which it did not on macOS 15. The older notes for this machine record the kext stack as
present with the touchpad never responding.

| Piece | Version |
|---|---|
| `VoodooI2C.kext` | 2.9.1 |
| `VoodooI2CHID.kext` | 2.9.1 (plugin) |
| `VoodooGPIO.kext` | plugin of VoodooI2C |
| Boot argument | `-vi2c-force-polling` |

The device is I2C HID on GPIO, `INTC1085`. One `VoodooI2CHIDDevice` instance attaches.

`-vi2c-force-polling` is doing real work. Without it the kexts load, the device never reports input,
and the whole stack looks installed-but-dead. It forces polling instead of waiting on the GPIO
interrupt, at a small power cost.

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

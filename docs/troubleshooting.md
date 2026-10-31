# Troubleshooting

Ordered by how often each one comes up.

## Black screen after a driver install

1. Power off, boot to the picker.
2. Press **space** to reveal the auxiliary entries.
3. Pick `1401: Start once without NVIDIA driver`.
4. Once the desktop is up, reinstall from `1401.app` and reboot.

If the entry is missing, the `Misc → Tools` entries were probably reset by `1401.app` — see
[gpu.md](gpu.md). `1401: Remove NVIDIA driver` is the heavier option: it sets an NVRAM flag, uninstalls
after macOS starts, and reboots by itself.

Note that the driver also rolls itself back. If a new install fails twice, `lastgood.sh` restores the
previous kexts from `/Library/NullMoth/backup-<timestamp>/`. Check `lastgood.log` before assuming
anything is permanently broken.

## Desktop comes up, but slow and low resolution

The driver is not loaded.

```bash
kmutil showloaded | grep -i nullmoth      # expect 4 lines
system_profiler SPDisplaysDataType | grep -i metal
```

Four kexts plus `Metal 3` means it is loaded and the problem is somewhere else.

## No picture, but the machine is clearly running

Fans spinning, keyboard backlight on, nothing on screen. The driver loaded and failed to bring up a
display. Boot with `Start once without NVIDIA driver`.

## An app cannot use the microphone or camera

Look at the permission list first:

- the app is listed → toggle it off and on
- the list is empty → [permissions.md](permissions.md); this needs a database write, and toggling cannot
  help because there is nothing to toggle

A quick way to separate the hardware from the permission layer:

```bash
ffmpeg -f avfoundation -i ":0" -t 1 -y /tmp/mic.wav
```

If that records, the audio stack is fine and the problem is TCC.

## Wi-Fi connects but is slow

Check the negotiated mode, not the signal strength:

```bash
system_profiler SPAirPortDataType | grep -E "PHY Mode|Channel"
```

`802.11ac` at 80 MHz is what this machine does today, even with `itlwm_he=1` and a strong signal. See
[wireless.md](wireless.md).

## Trackpad does not respond

```bash
ioreg -c VoodooI2CHIDDevice | grep -c "class VoodooI2CHIDDevice"
```

`0` means the stack is not reaching the device. Usually `-vi2c-force-polling` is missing, or
`SSDT-GPI0.aml` is not loaded.

## Updates are always the full installer

`revpatch=sbvmm` is missing or was overwritten by another change. See [updates.md](updates.md).

## `Processor Name: Unknown`

`cpuname` is not active inside `revpatch`. See [power.md](power.md) — this is almost always the
`boot-args` versus NVRAM shadowing, not a broken SMBIOS.

## The picker shows entries it should not

`1401.app` rewrites `config.plist` and resets `HideAuxiliary` and the tool entries' `Auxiliary` flags.
Re-apply both after any change made in that app.

## Reading the logs

```bash
# kernel errors since boot
log show --last 5m --predicate 'process == "kernel" AND messageType == error' --style compact

# tccd decisions
log show --last 1h --predicate 'process == "tccd"' --style compact | grep -i policy

# driver load state
kmutil showloaded | grep -v com.apple

# OpenCore log, if file logging is on
cat /Volumes/1401/oc.log 2>/dev/null
```

`Misc → Debug → Target` controls OpenCore logging. `3` writes to the console only; adding `0x40`
writes a file to the EFI partition. That file is the only record available when a boot fails before
macOS starts — worth enabling before you change anything risky.

One caveat on `nvram`: it only lists Apple's own variables. Anything under a third-party GUID (Mok,
RestrictEvents) has to be read through the device tree:

```bash
ioreg -p IODeviceTree -n options -r -l -w0 | tr ',' '\n' | grep -i <name>
```

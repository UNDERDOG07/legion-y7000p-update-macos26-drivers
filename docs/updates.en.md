# Delta updates

macOS offers a delta update (a few hundred MB to ~3 GB) instead of a full installer (10–15 GB) only when
it can verify the secure-boot chain. On an unsupported board that check fails, so the system has to be
made to report a VMM secure-boot model.

## What is required

All three, together:

| Requirement | Value here |
|---|---|
| `Misc → Security → SecureBootModel` | `Disabled` |
| `RestrictEvents.kext` | 1.1.7, enabled |
| `boot-args` | `revpatch=sbvmm` |

`sbvmm` is described upstream as *forces VMM SB model, allowing OTA updates for unsupported models*.

## Why it is easy to lose

`revpatch` is a single token. This config also carried a copy in `NVRAM → Add` under the RestrictEvents
GUID, with `cpuname` already in it:

```
boot-args   revpatch = sbvmm              ← wins
NVRAM       revpatch = sbvmm,cpuname      ← ignored
```

Adding `sbvmm` to `boot-args` to fix updates silently replaced the value that carried `cpuname`, and the
CPU name disappeared as a side effect. The two changes looked unrelated at the time. See
[power.md](power.md).

When you touch `revpatch`, write the whole list in one go:

```
revpatch=sbvmm,cpuname
```

## Checking that it worked

The only reliable test is the update itself:

- a few hundred MB to ~3 GB → delta, working
- 10–15 GB → full installer, the check is still failing

If it is still full, read `install.log` and the `softwareupdated` entries around the download for the
actual rejection reason rather than guessing.

## What breaks after an update

- **GPU driver.** The kernel collection is rebuilt; check `kmutil showloaded | grep -i nullmoth` and
  reinstall from `1401.app` if fewer than four lines come back.
- **Anything in `/Library/Extensions`.** Everything installed outside the EFI is at risk.
- **`hidutil` mappings**, if you used them instead of System Settings.
- **LaunchAgents survive.** The brightness job keeps working.

## This is not Secure Boot

`revpatch=sbvmm` makes the system *claim* a VM secure-boot model. It does not enable firmware Secure
Boot and does not make the boot stick signed. Firmware Secure Boot must stay off — see
[limits.md](limits.md).

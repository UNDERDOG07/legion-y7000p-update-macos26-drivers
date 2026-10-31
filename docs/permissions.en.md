# Privacy permissions when the prompt never appears

This is the one problem on this machine with no upstream fix, and the one that took longest to
understand. The diagnosis is written out in full because the symptom points in the wrong direction.

## The symptom

System Settings → Privacy & Security → Microphone is **empty**. Not "everything is off" — the list has
no entries at all. An app asks for the microphone, gets refused without any dialog appearing, and then
tells the user to enable it in Settings, where there is nothing to enable.

Camera behaves the same way, though less consistently.

## What the log actually says

```
tccd: Policy disallows prompt for Sub:{com.tencent.qq}
tccd: access to kTCCServiceMicrophone denied
```

`Policy disallows prompt` means tccd decided this request may not be shown to the user and returned a
denial without asking. No prompt means no record; no record means an empty list; an empty list means
there is no switch to turn on. The loop closes on itself.

Three things that look like the cause and are not:

- **The app's signature.** `codesign --verify --deep --strict` passes, Gatekeeper reports `accepted`,
  and the designated requirement matches what is on record.
- **The `csreq` blob.** Decoded with `csreq -r file -t`, it is exactly the app's requirement —
  `identifier "com.tencent.qq"`, matching team ID.
- **`Policy disallows prompt` as a message.** It also appears for `UserEventAgent`, `routined` and
  `managedcorespotlightd`. Background daemons are genuinely not supposed to prompt. The message is
  normal; what is abnormal is seeing it for a foreground app.

## The likely cause

AMFI is off (`amfi=0x80`) because the NVIDIA kexts are unsigned. tccd's decision about whether a prompt
is permitted runs through code-signing validation, and with AMFI disabled that path is degraded. The
conservative outcome is a refusal.

That is inference, not proof. What is proven is that turning AMFI back on is not available here: the
driver does not load without it, and without the driver there is no picture. Permissions are the cheaper
loss.

One test that was tried and disproved a different theory: removing `revpatch=sbvmm` and rebooting
changed nothing. The VMM secure-boot claim is not involved.

## The part that matters: there are two databases

```
/Library/Application Support/com.apple.TCC/TCC.db          system
~/Library/Application Support/com.apple.TCC/TCC.db         user
```

**Application-level grants live in the user database.** Writing a `kTCCServiceMicrophone` row into the
system database produces a record that looks correct, reads back correctly, and does nothing — the app
is still refused and Settings still shows an empty list, because both read the user one.

The clue that settled it: the user database already contained a `kTCCServiceAudioCapture` row for
`com.tencent.qq`, dated before the upgrade, while the system database had no microphone row for
anything.

## The row that works

| Column | Value | Note |
|---|---|---|
| `service` | `kTCCServiceMicrophone` | or `kTCCServiceCamera`, `kTCCServiceScreenCapture` |
| `client` | bundle id | |
| `auth_value` | `2` | allowed |
| `auth_reason` | **`1`** | user-set. `4` (system-set) is not treated the same |
| `csreq` | the app's designated requirement | copy from an existing row for the same app if one exists |
| `last_modified` | now | |

Write it to **both** databases, with `tccd` stopped:

```bash
sudo killall tccd
# insert into system db, then the same row into the user db
sudo killall tccd
```

The user database is what takes effect; the system one keeps `tccgrant list` honest.

## `tccgrant`

The helper used here wraps all of that. It resolves `csreq` in three steps — reuse an existing row for
the same bundle id, otherwise compute it from `codesign -d -r-` piped through `csreq`, otherwise leave
it null — backs both databases up, and writes them in one transaction.

```bash
tccgrant list                              # everything, both databases
tccgrant check mic com.tencent.qq          # one app, one service
tccgrant mic com.tencent.qq                # grant
tccgrant mic,camera /Applications/QQ.app   # several services, path accepted
tccgrant revoke mic com.tencent.qq         # remove
```

Services: `mic`, `camera`, `screen`.

After granting, the app must be **fully quit** — ⌘Q, not closing the window — and reopened. tccd reads
the grant at process start.

### A shell trap worth remembering

The first version printed `授予 ��`. Cause: in bash, `"$client："` with a full-width character directly
after the variable lets the shell swallow that character's first byte into the variable name. The
variable becomes empty and the character becomes a fragment. Write `"${client}："`.

## What is still broken

Every new app that needs the microphone will hit the same wall. There is no way to make the prompt
appear short of re-enabling AMFI, which costs the GPU driver.

The workflow is therefore: install the app, try it once, watch it fail, then `tccgrant mic <bundle-id>`
and relaunch.

Camera and screen recording have behaved better — some grants were created through a real prompt, which
suggests the restriction is applied per service rather than globally. Try the normal path first; reach
for the script when the list stays empty.

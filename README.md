*(日本語版: [README.ja.md](README.ja.md))*

# Hisense A6L (HLTE730T) — Root, Permanent Unlock, and VoLTE Enablement

Field notes and reusable tools from reverse-engineering VoLTE support on the **Hisense A6L (HLTE730T)**, a China-market Snapdragon 660 (SDM660) phone with an unusual dual-display design (a regular color LCD on the front, a full E Ink/electronic-paper display on the back) and zero VoLTE/IMS support out of the box, for use on Japanese MVNO/MNO SIMs (mineo/KDDI and Rakuten Mobile tested).

This is not a polished how-to for beginners — it's a record of what was actually tried, what failed, and what worked, so the next person (or future me) doesn't have to re-derive it. Contributions / corrections welcome.

**Standard disclaimer:** this involves permanently unlocking the bootloader (irreversible), flashing a patched boot image, and poking at live modem firmware over the Qualcomm diagnostic port. You can brick your device. Back up your `boot.img`, `vbmeta.img`, and full system/modem partitions via EDL *before* you start. None of this is Hisense- or Qualcomm-endorsed.

## Table of contents

- [Device background](#device-background)
- [Permanent root / bootloader unlock](#permanent-root--bootloader-unlock)
- [Brick recovery: EDL and a no-teardown trigger cable](#brick-recovery-edl-and-a-no-teardown-trigger-cable)
- [The VoLTE problem](#the-volte-problem)
- [Technique 1: patching ModemTestMode to drive QMI directly](#technique-1-patching-modemtestmode-to-drive-qmi-directly)
- [Technique 2: live-editing modem EFS via QPST](#technique-2-live-editing-modem-efs-via-qpst)
- [Technique 3 (the one that actually worked for Rakuten): try every stock carrier profile](#technique-3-the-one-that-actually-worked-for-rakuten-try-every-stock-carrier-profile)
- [Why offline-edited mcfg_sw.mbn files get rejected](#why-offline-edited-mcfg_swmbn-files-get-rejected)
- [Tools included in this repo](#tools-included-in-this-repo)
- [Open questions / not yet solved](#open-questions--not-yet-solved)

## Device background

- **Model:** Hisense A6L, model number `HLTE730T` (also seen as `HLTE730T.B1`, `.B6`)
- **Form factor:** dual-display — a conventional color LCD on the front, a full E Ink (electronic paper) display on the back, both independently drivable. The framework carries E Ink-specific plumbing (e.g. an `mPreEInkStatus` flag surfaced in `NetworkController.MobileSignalController` logs, a step-counter readout on the lock screen) that isn't present on a normal single-screen device — worth knowing about before assuming any generic Android/Qualcomm guide applies as-is.
- **SoC:** Qualcomm Snapdragon 660 (SDM660)
- **Modem firmware baseline:** `MPSS.AT.3.1-00819-SDM660_1.2` (visible in QPST), Policy Manager XML header shows `mmcp.mpss/8.1.1`
- **Android:** 9.0, China-market firmware, no Google apps, VoLTE gated at the Android framework layer regardless of modem capability
- Firehose loader for EDL: `prog_emmc_ufs_firehose_Sdm660_ddr_30060000.elf`

## Permanent root / bootloader unlock

1. Temp-unlock via the OEM `fastboot Hisense unlock` command. **Stock platform-tools `fastboot` does not recognize the `Hisense` OEM subcommand** — you need a custom/patched fastboot binary that supports it (search for Hisense-specific fastboot tools; not redistributed here).
2. `fastboot erase avb_custom_key` — this is the actual irreversible unlock step. It does **not** by itself wipe userdata or show a confirmation dialog on this device (contrary to some guides for other Hisense models) — issue `fastboot erase userdata` yourself too, or be ready to do a factory reset from the resulting "Decryption Unsuccessful" recovery screen.
3. Flash a Magisk-patched `boot.img` and a `vbmeta.img` flashed with `--disable-verity --disable-verification`.
4. This survives a genuine cold boot. (A temp-unlock-only path, without step 2, only survives a single `fastboot continue` and reverts after a real reboot — this cost significant time to figure out.)

## Brick recovery: EDL and a no-teardown trigger cable

**You don't need any of this if the unlock procedure above goes smoothly** — EDL is only the fallback for when something goes wrong (a bad flash, a bootloop, etc). Read this section before you start so you're prepared, but the happy-path root/unlock flow never touches EDL.

Emergency Download Mode (EDL) requires shorting test points inside the phone (teardown), normally. Look for a **"deep flash cable" / "EDL cable" / "test point cable"** for Snapdragon devices — these are USB cables with a resistor wired into a spare pin that forces the phone into EDL on connection, no disassembly required. Cheap, widely sold for exactly this purpose, and worth buying before you start rather than after you need it.

Entry procedure that reliably works on this device with such a cable:

1. Power the phone off completely.
2. Hold the cable's inline button down (a clothespin/binder clip works well for this) and plug the cable into the **phone** — leave the other end **unplugged from the PC** for now.
3. Get your left hand ready on **Volume Down** and **Power**, don't press yet.
4. With your right hand, plug the USB end into the **PC** — this should land *just* before, or essentially simultaneously with, pressing Volume Down + Power together with your left hand.
5. Wait 2–3 seconds. **No vibration and a black screen means it worked** (a vibration means it booted normally instead — power off and retry).
6. Release the buttons immediately, and don't forget to remove the clothespin/clip from the cable button afterward.
7. The device should now enumerate and be reachable from the PC (Sahara/Firehose). Note: **while the cable's button is held down, the device cannot communicate** — it must be released for the PC-side tooling to actually talk to it.

Once in EDL, use `edl.py` (or Qualcomm's own `QSaharaServer`/`fh_loader` from QPST) with the firehose loader listed above to read/write `boot`, `vbmeta`, `system`, and modem partitions raw. Constraints we hit:
- One read/write operation per EDL session on this device — power-cycle and re-enter EDL between operations.
- Kill any stray Python processes holding the port before retrying.
- Set `PYTHONIOENCODING=utf-8` / `PYTHONUTF8=1` if using `edl.py` on Windows.
- May need the WinUSB driver via Zadig.

## The VoLTE problem

This device ships with zero VoLTE/IMS on a China SKU. The Android-side gates (`ImsManager.isVolteEnabledByPlatform()`, the per-SIM `volte_vt_enabled` column in `content://telephony/siminfo`, `QtiVolteSwitchController`'s China-carrier-only default) can all be forced open with root:

```sh
setprop persist.dbg.volte_avail_ovr 1
# per-SIM, once you know the subId:
content update --uri content://telephony/siminfo --bind volte_vt_enabled:i:1 --where "_id=<subid>"
```

But that alone isn't enough — the **modem's own MCFG (Modem Config) carrier policy** also has to permit VoLTE for your network's MCC, and separately has to not block general data PDN establishment for a SIM whose home PLMN doesn't match the loaded carrier profile. Getting both right is the bulk of this repo.

## Technique 1: patching ModemTestMode to drive QMI directly

The real, Qualcomm-intended way to select/activate a carrier's MCFG config at runtime (not just at boot) is `com.qualcomm.qti.modemtestmode` (package name "ModemTestMode" / "MBN Test"), which drives the modem over QMI via `com.qualcomm.qti.qcrilhook.QcRilHook`. Two activities matter:

- **`MbnFileLoad`** — browse `/data/vendor/modem_config/mcfg_sw/generic/<region>/<carrier>/.../mcfg_sw.mbn` (this directory has the *full* unrestricted carrier pool, regardless of anything in `oem_sw.txt`) and load a file into modem EFS (`setupMbnConfig()` → `qcRilSetConfig()`).
- **`MbnFileActivate`** — list configs currently registered in EFS, select one, and activate it (`selectConfig()` → `qcRilSelectConfig()`), which triggers an automatic reboot ~15s later.

On this device, stock `MbnFileActivate` refuses to proceed with a hard error ("Device is not configured properly") because `getMbnInfo()` bails out as soon as the HW-side MBN config query returns null — which it always does, since this modem's HW config was never separately activated. This is an unrelated check blocking the SW/carrier flow we actually want.

**Fix:** `patches/ModemTestMode-MbnFileActivate.smali.diff` — deodex `ModemTestMode.apk` (baksmali against the full boot classpath), change the `return v1` early-exit to `goto :cond_83` (fall through past the failed check), reassemble, and repackage.

Gotchas that cost real time:
- The stock APK has **no `classes.dex` entry at all** — real code lives only in the paired `.odex`/`.vdex`. You must *add* `classes.dex` as a new zip entry, not "replace" one (a naive replace-if-exists repack script will silently produce a dex-less APK).
- Deploy via a **Magisk module** overlay (`/data/adb/modules/<id>/system/app/ModemTestMode/ModemTestMode.apk`) — this is a Legacy SAR device (`/` itself is `mmcblk0p*`, no separate `/system` mount), so `mount -o rw,remount /` fails and you don't want to touch the raw partition for an app-level change anyway.
- You must **also** whiteout the original `.odex`/`.vdex` (`mknod <module>/system/app/ModemTestMode/oat/arm64/ModemTestMode.{odex,vdex} c 0 0`) **and** delete the separate ART cache at `/data/dalvik-cache/arm64/system@app@ModemTestMode@ModemTestMode.apk@classes.{dex,vdex}`. Missing any one of these three locations means the old code keeps running and you'll waste time debugging a "patch that didn't take."
- `MbnFileLoad`'s file browser needs SELinux permissive (`setenforce 0`) — the `radio` domain lacks `search` on `vendor_mbn_data_file` (`/data/vendor/modem_config`) by policy, so the browser silently shows an empty list otherwise. Resets to enforcing on reboot; no persistent policy change needed, just re-run it each session.

## Technique 2: live-editing modem EFS via QPST

Once you're root + diag-port-capable, **QPST's EFS Explorer** (`C:\Program Files (x86)\Qualcomm\QPST\bin\EFSExplorer.exe`) lets you browse and directly overwrite files inside the modem's own EFS filesystem — a completely different code path from "import a new MCFG package," and (on this device at least) it is **not** subject to the whole-package digest check described below.

Enable the diag port:
```sh
adb shell setprop persist.usb.eng 1
```
Windows should then enumerate a `Diagnostics Interface (COMx)` device (`USB\VID_109B&PID_90FE&MI_02\...` on this device — a "9091"-class Qualcomm diag composite). Add it in `QPSTConfig.exe`, then connect from `EFSExplorer.exe`.

What worked:
- Editing `/policyman/carrier_policy.xml` for an **already-Activated** carrier config (e.g. adding your SIM's PLMN to an existing `<plmn_list name="unrestricted_operators">` in that carrier's policy) — this took effect after a plain `adb reboot`, no re-Activate needed.
- Editing individual already-declared NV items the same way (right-click → "Copy Item File from PC..." for `Item File` type entries — note this is a *different* context-menu action than "Copy Data File from PC" used for plain `File` type entries like the XML above; the local source file's *name* must exactly match the EFS item name).

What did **not** work:
- Writing a **new**, previously-undeclared file into a carrier's `/policyman/` directory (e.g. dropping a `carrier_policy.xml` into the generic `common/row` profile, which never shipped one). The write itself succeeds silently, but Policy Manager never picks it up after reboot. Best guess: Policy Manager consumes a per-config item manifest built at import time, not a live directory scan — so a file has to have been part of the *original* imported package to ever be read, even though you can freely rewrite its *contents* afterward.

The Windows automation helper used throughout (`scripts/efs-explorer-automation-helpers.ps1`) is a small set of `SetCursorPos`/`mouse_event`/`SendKeys`-based functions for driving EFS Explorer's dialogs without a proper Win32 automation ID model (it's an old MFC app; standard UI Automation mostly can't see its list items). Screenshot the dialog, read pixel coordinates off the *displayed* image, and this repo's functions do the rest. Clipboard paste (`Set-Clipboard` + `^v`) is more reliable than `SendKeys` for typing file paths — direct `SendKeys` of a path intermittently mangled through an active IME on this system.

## Technique 3 (the one that actually worked for Rakuten): try every stock carrier profile

After a lot of patching effort, the thing that actually solved VoLTE **and** general data for a Rakuten Mobile SIM (MCC/MNC `440-11`) was embarrassingly simple: **just Activate a different unmodified stock carrier profile and see what happens.** This mirrors a trick documented independently by others (e.g. an XDA guide for a different Snapdragon device recommends the same "try every profile in the pack, note which carrier gets VoLTE from which" approach — MCFG carrier policies are often scoped by *country* (MCC), not by the specific operator, so an unrelated carrier's profile for your country frequently unlocks VoLTE for you too).

Concretely, on this device with a Rakuten SIM:

| Profile loaded (all stock, unmodified) | VoLTE / IMS PDN | General data (default APN) |
|---|---|---|
| `apac/kddi` | works | **fails**, `OEM_DCFAILCAUSE_4` |
| `apac/dcm` (NTT docomo) | works | **fails**, same cause |
| `common/row` (generic ROW) | doesn't work | works |
| **`apac/sbm` (SoftBank)** | **works** | **works** |

The pattern: the two Japanese-MNO-specific profiles we tried (KDDI, docomo) both carry a `carrier_policy.xml` that whitelists their own PLMNs for VoLTE eligibility *and*, empirically, also blocks general data PDN establishment for SIMs outside that whitelist — this looks like deliberate anti-fraud/roaming-abuse behavior baked into real MNO carrier policies, not a device bug. Generic ROW has no such whitelist (so data always works) but also no country-level VoLTE-enable directive. SoftBank's profile apparently has neither restriction and Just Works for a Rakuten SIM. We did not reverse-engineer *why* — verified purely empirically (`dumpsys telephony.registry` for `mVoLteServiceState`, `dumpsys connectivity` for `NetworkAgentInfo` showing both `extra: ims` and `extra: <apn>` as `CONNECTED/CONNECTED` + `everValidated=true`, and an actual `ping` test).

**If you're chasing VoLTE for a specific SIM on a device with several stock carrier profiles available: try all of them before writing any patches.** It's free, fast, fully reversible (just re-Activate `MbnFileLoad`/`MbnFileActivate` on another profile, or the original), and may just work.

## Why offline-edited mcfg_sw.mbn files get rejected

If profile-swapping doesn't get you there and you actually need to edit a `carrier_policy.xml`, know what you're up against first.

`mcfg_sw.mbn` is an ELF-wrapped container (`readelf`/`pyelftools` parse it fine): segment 0 is a hash-table/secure-boot-style segment, segment 1 holds an embedded X.509-style certificate chain (OEM attestation, unrelated to the payload), segment 2 (`PT_LOAD`) holds the actual NV item data as a flat sequence of `type,path-len,path,data-len,data` TLV records ending in a trailer item containing a **32-byte digest** (SHA-256) of the other segments.

We confirmed (via [`sbaresearch/mbn-mcfg-tools`](https://github.com/sbaresearch/mbn-mcfg-tools), a Python tool that genuinely recomputes this hash rather than copying it — verified by a round-trip no-op extract→repack producing a byte-identical file) that **a correctly-recomputed internal hash is not sufficient** to get a modified file accepted by `MbnFileLoad` on this modem: any edited file, hash-valid or not, comes back `error code:-1` from the QMI import path. Both this tool's own author and [`fenrir-naru/mbn_utils`](https://github.com/fenrir-naru/mbn_utils) (an earlier, less complete tool with the same goal) independently document hitting the same wall, suspected to be either an external reference digest checked outside the file (`sbaresearch`'s README notes `mcfg_sw_config_digest_version` stored separately in EFS) or a genuine cryptographic signature the tools don't attempt to forge.

Net effect: **don't bother hand-editing an `.mbn` and trying to re-import it via `MbnFileLoad`.** Either (a) live-edit an already-declared file/item inside an already-Activated config via EFS Explorer (Technique 2), or (b) find a stock profile that already does what you want (Technique 3).

`patches/mbn-mcfg-tools-windows-path-fix.patch` is unrelated to the above — it's a small fix for a Windows-specific bug in that tool (leading-slash item paths like `/policyman/carrier_policy.xml` get passed straight to `pathlib.Path()`, which Windows resolves to the current drive root instead of a relative path, silently dropping most extracted files). Useful if you want to use the tool on Windows at all, independent of the hash-wall issue above.

## Tools included in this repo

- `patches/ModemTestMode-MbnFileActivate.smali.diff` — the HW-check bypass (Technique 1). This is a diff against baksmali output of the stock APK's own code, not a redistributed binary — you regenerate the patched dex yourself.
- `patches/mbn-mcfg-tools-windows-path-fix.patch` — Windows path-handling fix for `sbaresearch/mbn-mcfg-tools`.
- `scripts/efs-explorer-automation-helpers.ps1` — PowerShell mouse/keyboard automation for driving QPST EFS Explorer's dialogs (Technique 2).

Not included: any Hisense/Qualcomm-copyrighted binaries (stock or patched APK, `.mbn` files, boot images). Regenerate those yourself from your own device's firmware using the patches above.

## Open questions / not yet solved

- Root cause of why KDDI/docomo profiles specifically reject non-home-PLMN data PDN (`OEM_DCFAILCAUSE_4`) — never isolated to a specific NV item; one candidate (`/nv/item_files/modem/mmode/is_plmn_block_req_in_lte_only_mode`) was tried and ruled out empirically.
- Whether SoftBank's profile working for Rakuten is coincidence-of-lenient-policy or something more specific to how these two carriers' MCC-440 policies happen to be written — untouched, black-box result.
- 5GHz WiFi hotspot: France/Japan-style DFS-channel regulatory gating exists on this device for real reasons (not just an artificial restriction), and a first attempt at a jar-swap patch bootlooped the device (recovered via EDL). Root cause of the bootloop: SystemServerClasspath jars need proper deodex/re-encode, not a naive dex swap — same discipline as the ModemTestMode patch above, just not yet re-attempted with the correct technique.
- `subMask` ended up `1` (single-SIM) rather than `3` (DSDS) during one KDDI Activate, as a side effect of bypassing the HW check — impact on dual-SIM behavior unverified.

# Alioth Pengu/AOSPA A17 — Agent Findings

Last updated: 2026-11-07
Device: Xiaomi Poco F3 (`alioth`, SM8250)
ROM: Pengu / AOSPA Android 17
Main tree: local Pengu/AOSPA checkout (absolute path intentionally omitted)

This file records the state, decisions, hard-won fixes, and safety notes from the
Dolby Stage 1, thermal-default, torch-strength, and AW8697 haptics work. Read this
before changing the vibrator, audio, or thermal stack again.

---

## Repositories and branches

| Component | Remote | Branch |
|---|---|---|
| Device tree | `pa-sanji/device_xiaomi_aliothCLO` | `alioth-a17-bringup` |
| Vendor blobs | `pa-sanji/proprietary_vendor_xiaomi` | `alioth-a17-bringup` |
| Hardware / HALs | `pa-sanji/android_hardware_xiaomi` | current work branch |
| Kernel | `pa-sanji/xiaomi_sm8250_kernel_e404` | `alioth-a17-bringup` (4.19.404) |
| AOSPA vendor | `pa-sanji/android_vendor_aospapengu` | `sync/alioth-only` |

Upstream references used for the haptics port:

- `PocoF3Releases/hardware_xiaomi` branch `cnb` (`vibrator/aw8697`, `vibrator/effect`)
- `PocoF3Releases/kernel_xiaomi_sm8250` branch `aosp-17`
  (`drivers/input/misc/aw8697_haptic`)
- `PocoF3Releases/device_xiaomi_alioth` branch `aosp-17`
  (`aw8697_waveform_fixup.py`, `proprietary-files.txt`)
- `PocoF3Releases/vendor_xiaomi_alioth` branch `aosp-17`
  (`proprietary/vendor/etc/vibrator/*.bin`)

Do not mix in the older Sanjis A16 HAL or the original QTI AIDL vibrator service
for the AW8697 path.

---

## Work completed / in tree

### 1. Dolby Stage 1

- Kept Stage 1 only: `libswdap`, `libdlbvol`, `libdlbpreg`, DAX 3.7,
  `LunarisDolby`, `DSPVolumeSynchronizer`.
- Framework `DolbyDapController` is disabled.
- Build 108 succeeded with RBE and was verified by a tester as "much louder".
- Preserved fallback zip:
  `PenguinOS-108-stage1-dolby.zip`
  SHA-256 `15e4fb6adc93629fee0f0b145f54087f9d5d024023faed2aa4e7a0f0bd5c41ab`.
- Latest Dolby log: software DAP active (`DlbDap2Process`,
  `ProcessingMode=1`), no `DlbQdspGefDriver`, no `-22`, no Dolby denials, no
  crashes. Remaining EQ parameter/log churn is app-side and non-blocking.
- Dolby Vision / C2 is deliberately not part of Stage 1.

### 2. AOSPA-default thermals

- Removed the XiaomiParts thermal UI and thermal profile writers.
- Removed the custom thermal configs from the device tree.
- Added the four missing AOSPA vendor thermal blobs.
- Rely on vendor `mi_thermald`.
- Kernel fix: `thermal: Restore the thermal_message sconfig store`
  (`a8ac93262a66934b397bda89b1116e016e9b9c4f`):
  - `ATOMIC_INIT(-1)`
  - accept `0`
  - reject negatives
- With no XiaomiParts writer, AOSPA default behavior is `sconfig=-1` until a
  profile is written.
- Kernel fix: `arm64: dts: alioth: Remove apollo battery profile`
  (`2026020d2ab9134d61bd83bedc75a44335c5a1b2`).

### 3. Torch strength control

- Device commit `bca40b1` `alioth: Wire up torch strength control`.
- AOSPA vendor commit `8bde35de` `soong: Wire TARGET_CAMERA_SERVICE_EXT_LIB
  into libcameraservice`.
- `led:switch_2` used; ueventd permissions; `switch_2` labeled
  `vendor_sysfs_torch`.

### 4. AW8697 haptics

Ported the current PocoF3Releases implementation, not the older A16 HAL.

**HAL / libraries**

- `hardware/xiaomi/vibrator/aw8697/`
  - `android.hardware.vibrator-service.xiaomi_aw8697`
  - `PrimitiveSynth.cpp/.h`, `Vibrator.cpp/.h`, `service.cpp`
  - `vibrator.rc`, `vibrator.xml`
- `hardware/xiaomi/vibrator/effect/`
  - `libaw8697effect` (renamed from upstream `libqtivibratoreffect.xiaomi`
    to avoid colliding with the existing QTI library)
- AIDL V2 service registered as
  `android.hardware.vibrator.IVibrator/default`.

**Kernel driver**

- Replaced `drivers/input/misc/aw8697_haptic/` with the PocoF3Releases
  `aosp-17` stock driver plus its safety fixes.
- New/changed files include `aw8697_gain.h`; the Makefile is intentionally
  minimal because `USE_CONT_F0_CALI` is only for `lmi`.
- Exposes the new ABI used by the HAL:
  `abi_version=2`, `ram_gain_abi=1`, `playback_active`, bounded `custom_wave`.
- `CONFIG_INPUT_AW8697_HAPTIC=y`; munch driver remains disabled.

**DTS**

- `alioth-sm8250.dtsi` AW8697 node:
  - `vib_cont_drv_lev = <73>`
  - `vib_cont_drv_lvl_ov = <73>`
  - `vib_bst_vol_default/ram/rtp = <0x11>`
- `xiaomi-sm8250-common.dtsi` base node holds the rest (10 effect children,
  `vib_effect_id_boundary = 10`, `vib_effect_max = 193`).

**Firmware**

- `vendor/firmware/aw8697_haptic.bin`
  - SHA-256 `7e31b22b591d5f45dcf262529b71fd2b5b3277f98553414b54d3b69d7255d4c4`
  - SHA-1 `37721cb00ef2e50e6051de104605c061c9ef5f90`
- `vendor/firmware/aw8697_rtp_1.bin`
  - SHA-1 `9f32c9bf0776a566d3254129a028dae9bed76b81`

**Generated primitive waveforms** (`vendor/etc/vibrator/`)

All are unchanged 24 kHz stock RAM slices (verified against upstream vendor
files):

| File | Stock slot | SHA-256 |
|---|---|---|
| `primitive_effect_1.bin` (CLICK) | 0 `0_click` | `dcb60279196fcd747381e41c162ad0a745bc0c685e80b7d04d801486f0d1d248` |
| `primitive_effect_2.bin` (THUD) | 1 | `297b62d8a93dc6cea9e29794db7fe34334d973557bdb779096d356c08b9526f6` |
| `primitive_effect_7.bin` (LIGHT_TICK) | 2 `2_tick` | `15b1a3b72399ea4ebe96f71bae6a60e1e7fc705a831f367f0957942e43fda1c0` |

Extraction is SHA-gated by `device/xiaomi/alioth/aw8697_waveform_fixup.py`.
`extract-files.sh` calls it for those three destinations.

**Init / permissions**

- `init.alioth.rc`:
  - `custom_wave` and `gain`: `system:system 0660`
  - Removed the stale `high_vol` chown (the new stock driver has no such node)
- SELinux:
  - `vendor_sysfs_aw8697_custom_wave`
  - `gain` reuses `sysfs_vibrator`
  - `f0_value` reuses the existing `sysfs_f0_value` type from the audio
    policy (read-only for `hal_vibrator_default`)
  - `hal_vibrator_default` gets input device access, gain/custom-wave rw,
    and read-only `f0_value`
  - executable labeled `hal_vibrator_default_exec`

---

## Critical findings (do not regress these)

### A. Base DTS must have `rtp_time_count > vib_effect_max`

The stock driver's DT parse does:

```c
if (aw8697->info.effect_max >= aw8697->info.rtp_time_count ||
    aw8697->info.effect_id_boundary > aw8697->info.effect_max)
        return -EINVAL;
```

Our original `xiaomi-sm8250-common.dtsi` had only **175** `vib_rtp_time`
entries while `vib_effect_max = 193`. The driver probe therefore returned
`-EINVAL`, and the whole vibrator would be dead (HAL readiness wait would fail).

Fix applied: append 19 zeros (reserved IDs 175–192 and streamed effect 193),
giving **194** entries, matching upstream. Always re-check this if the DTS or
`effect_max` changes.

### B. Waveform slot mapping was wrong on the first attempt

Correct mapping (matches upstream comments and actual files):

- `primitive_effect_1.bin` = `CLICK` = stock slot **0**
- `primitive_effect_2.bin` = `THUD` = stock slot **1** (two close pulses)
- `primitive_effect_7.bin` = `LIGHT_TICK` = stock slot **2**

The first generated `primitive_effect_2.bin` used slot 3 and had the wrong
hash. Fixed and verified byte-for-byte against
`PocoF3Releases/vendor_xiaomi_alioth/aosp-17`.

### C. Stale sysfs nodes from the old driver

- `high_vol` no longer exists in the stock driver; its `chown` was removed
  from `init.alioth.rc`.
- The stock driver also has no `vov` node. A common-tree `chown .../vov`
  (if inherited) is a harmless no-op, but do not depend on it.
- `gain`/`custom_wave` are the only AW8697 nodes intentionally exposed to the
  HAL; everything else stays root-only.

### D. Do not rename the audio `sysfs_f0_value` type

The audio policy already defines `sysfs_f0_value` and labels the same
`2-005a/f0_value` path. Reusing it avoids breaking `hal_audio_default` and
`audioserver`. The vibrator HAL only needs read access.

### E. `effect_id_boundary = 10`, `effects_count = 10`

The HAL probes RAM effects 0..9 at startup and disables itself if any upload
fails. The base DTS provides exactly 10 `wf_*` children. Do not reduce them.

### F. HAL expects the new ABI

Readiness requires `period_size > 0`, `max_size >= 4096`, `period_size <= 2048`,
`abi_version=2`, `ram_gain_abi=1`, `playback_active`, and
`custom_wave_id=193`. The old local driver lacks these; do not revert the
kernel driver unless reverting the HAL too.

---

## Safety review notes

The user explicitly asked for a double-check that this cannot damage the
device. Static review found:

- **Gain:** HAL clamps to 1..128. Driver `gain` sysfs accepts up to 0xff, but
  only root/system can write it; SELinux only grants the vibrator HAL.
- **Boost voltage:** `vmax` is root-only and the driver masks it to the
  hardware field; `aw8697_haptic_set_bst_vol()` clamps `0xe0` to `0x1f`.
- **Drive levels:** stock Alioth values (73/73, boost 0x11), not increased.
- **Custom wave:** ring buffer is 8191 bytes; HAL writes only when `free_size`
  allows; stream length capped at 240000 samples (10 s nominal); final
  unaligned write sets EOF; cancellation uses `rb_force_exit()` and wakes the
  writer.
- **Composition:** max 256 steps, delay 0..1000 ms, scale 0..1; samples are
  scaled in software and stay inside int8.
- **Amplitude:** `setAmplitude` accepts finite `(0, 1]`; constant vibration
  uses the sysfs gain path; predefined effects ignore amplitude changes while
  active (as upstream).
- **SELinux:** only `hal_vibrator_default` can write `gain`/`custom_wave`;
  `f0_value` is read-only; no `vmax`/calibration access is granted.
- **AAC/RichTap intentionally not packaged.** Production effects use
  `PrimitiveSynth`, not `libaachaptics.so` or
  `xiaomi-alioth-richtap-render`.
- **Known theoretical risk:** if the AW8697 IRQ never completes, the driver's
  `flush_work()` could block an erase. This is a hardware-fault path, not
  observed; on-device testing is still required.
- **On-device testing has not happened yet.** Do not claim verified safety
  until `logcat`, `dmesg`, `dumpsys vibrator_manager`, and a physical
  vibration/temperature check pass.

---

## Build notes

- Full build command used: `./rom-build.sh alioth -i` with RBE environment
  variables (BuildBuddy). Do not commit the API key.
- Build 108 = verified Dolby Stage 1 fallback zip.
- Builds 111/112 were restarted after static-review fixes:
  - wrong waveform slot mapping
  - stale `high_vol` chown
  - `vib_rtp_time` length bug
- Build 113 is the first build containing all of the above fixes and it
  succeeded (`BUILD113_EXIT=0`).
  - Result: `PenguinOS-celerity-20261107-gms-alioth.zip` (2.7 GiB)
  - SHA-256 `121ad77c5113b5cbf146d693ba246a9c0f1cdfe39224bef2117144a515cdd16b`
  - The build was briefly stuck because an orphaned `siso` process from the
    killed build 112 held the build FIFO; killing that process resumed it.
  - Verified in the built images:
    - `dtbo.img` entry contains the AW8697 node with `vib_rtp_time` count
      **194** and `vib_effect_max = 193`.
    - `vmlinux` contains `aw8697_i2c_probe`, `aw8697_haptics_upload_effect`,
      `aw8697_custom_wave_store`, and the
      `custom_wave_id=%d;abi_version=2;ram_gain_abi=1;` ABI string.
    - `vendor.img` contains
      `bin/hw/android.hardware.vibrator-service.xiaomi_aw8697` (0755),
      `lib64/libaw8697effect.so`, all three waveforms (hashes match),
      `etc/init/vibrator.rc`, and `etc/vintf/manifest/vibrator.xml`.
    - Vendor policy contains the `vendor_sysfs_aw8697_custom_wave` genfscon,
      the `2-005a/gain` `sysfs_vibrator` genfscon, and the
      `hal_vibrator_default` allow rules.
    - No QTI vibrator service is installed in `/vendor/bin/hw`.
  - On-device flashing/testing: build 113 flashed; haptics and torch
    confirmed working.

### Useful post-flash checks

```sh
adb shell getprop | grep -i vibrator
adb shell dumpsys vibrator_manager
adb shell ls -l /vendor/bin/hw/android.hardware.vibrator-service.xiaomi_aw8697
adb shell ls -l /vendor/lib64/libaw8697effect.so
adb shell ls -l /vendor/etc/vibrator/
adb shell ls -l /vendor/etc/vintf/manifest/ | grep -i vibrator
adb shell cat /sys/devices/platform/soc/a8c000.i2c/i2c-2/2-005a/custom_wave
adb shell dmesg | grep -i aw8697
adb shell logcat -d | grep -iE 'aw8697|vibrator|haptic'
```

Expected `custom_wave` output includes:

```text
period_size=512;max_size=8191;free_size=...;custom_wave_id=193;abi_version=2;ram_gain_abi=1;playback_active=0;
```

Things to watch on device:

- service stays up (no restart loop)
- short click, notification, and long `on()` vibration work
- no stuck vibration after cancel
- no abnormal noise or heat
- no SELinux `avc: denied` for `hal_vibrator_default`
- `dumpsys vibrator_manager` shows amplitude/primitives support

---

### On-device log review (2026-11-07, build 113)

User flashed build 113 and confirmed haptics and torch work. `sconfig` reads
`-1`, which is the intended AOSPA default.

Verified from the captured `dmesg`/`logcat`:

- **Haptics:** 1687 `aw8697_rtp_work_routine` messages, all from the normal
  short final custom-wave chunk (304/381 bytes vs the 512-byte period). No
  i2c, probe, firmware, checksum, or upload errors.
- **Torch:** `CameraProviderManager` reports `AVAILABLE_ON`/`AVAILABLE_OFF`
  and `FlashlightController` changes strength 0–200. No `switch_2`/torch
  SELinux denials.
- **Thermal:** `sconfig=-1` is expected. Charger thermal levels and DRM HBM
  toggles show mitigation is active; no thermal trip/overheat logs.
- **Kernel:** no panic, Oops, WARNING, hung task, or watchdog.
- **SELinux:** no denials for `hal_vibrator_default`, AW8697, `switch_2`,
  torch, `thermal_message`, or Dolby.
- **Dolby Stage 1:** `DlbDlbEffect`/`DolbyManagerService` active; no hard
  Dolby failures.

Benign log noise / optional cleanup:

- `aw8697_rtp_work_routine buffer size 512, availbe size 304/381` is printed
  at `pr_err` for the normal short final RTP chunk. Demote to `pr_debug` if a
  clean dmesg is wanted.
- `flashv2: ... led short/open fault detected! led_status1=8` is printed at
  `pr_emerg`. The newer `leds-qti-flash.c` driver logs the same condition at
  `pr_debug`; it occurs while the LED is off/open-circuit-detected and torch
  works. Not a functional fault.
- CamX `WriteFlashStateToThermalNode() open .../flash_state failed`: the
  kernel `thermal_message` node has no `flash_state` attribute. Torch works;
  this is a camera thermal-report log only.

Pre-existing / unrelated findings:

- `vendor.qti.gnss.ILocAidlGnss/default` is missing, so `vendor_locservice`
  retries it every second. HIDL GNSS services are installed; GPS was not
  tested. Likely a pre-existing AIDL/HIDL packaging mismatch.
- `ACDB-LOADER` `-100` and `send_afe_cal_type ... ret -22` calibration noise,
  plus `audio_hw_extn` perf-lock errors. Audio/Dolby still work.
- `WifiHAL getUsableChannels Error -3` / `WifiChipAidlImpl code 4`; likely
  pre-existing and unrelated.
- Keybox certificate expired 2026-11-06 (Play Integrity/keybox only).
- App crashes: Banglalink (credential-encrypted prefs before unlock) and
  Google Keep (sync assertion); app-side.
- 305 SELinux denials in the capture, mostly KernelSU (`ksu`) and AOSP/QTI
  policy gaps; none for the features above.

---

## Next steps

1. Build 113 is flashed; haptics and torch are confirmed working and
   `sconfig=-1` is expected.
2. Optional cleanup build: demote the AW8697 short-buffer and flashv2
   fault logs to debug level.
3. Commit/push vibrator changes to the existing `pa-sanji` branches.
4. Commit/push Dolby Stage 1 and thermal-removal changes once validated.
5. Investigate the missing AIDL GNSS service only if GPS is required.
6. Only then consider AAC/RichTap, Dolby Vision/C2, or further haptic tuning.

---

## Explicit non-goals / things not to do

- Do not re-add the XiaomiParts thermal UI or custom thermal profiles.
- Do not package the QTI vibrator service for the AW8697 path.
- Do not write `vmax` or calibration nodes from the HAL.
- Do not regenerate waveforms from the wrong stock slot.
- Do not shorten `vib_rtp_time` below `vib_effect_max + 1`.
- Do not expose `custom_wave` to `system_server` or apps.

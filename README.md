# itel RS4 (S666LN) → AxionOS (Android 16)

Build **recipe** for **AxionOS** (`AxionAOSP/android`, branch `lineage-23.2` = LineageOS 23.2 + AOSP
`android-16.0.0_r4`) on the **itel RS4** (`S666LN`, MediaTek **MT6789 / Helio G99**). The same design as
[`itel_rs4_AICP`](https://github.com/riza220201/itel_rs4_AICP), whose scripts these are: a local manifest plus
idempotent scripts, not the Android source tree. Everything ROM-specific is in **`rom.conf`**.

## The device stack (branch `lineage-23.2` in each)

| repo | role |
|---|---|
| `device_itel_S666LN` | device glue, sepolicy, overlays; AxionOS builds its `lineage_S666LN` product |
| `device_itel_S666LN-kernel` | dtb, dtbo, 198 + 206 modules, **`prebuilt/` = the published Neutron-built vanilla kernel** (imported, committed) |
| `vendor_itel_S666LN` | blobs from stock rev 28 via 23.2's extract-utils |

The kernel is never compiled in the ROM. The build presents the Pixel 10 identity (`S666LN_IDENTITY=pixel10`, set
in `rom.conf`): itel never shipped Android 16, and e-KYC rejects a device+OS pair that does not exist. AxionOS's
attestation hooks only act on a keybox the user imports, so the device's own keybox still attests.

## Layout

| path | what |
|---|---|
| `rom.conf` | the ROM: manifest, product + release, who owns `build/soong`, the lunch helper, artifact names, identity |
| `local_manifests/S666LN.xml` | the tree's `.repo/local_manifests/S666LN.xml` (the build refuses to run if they differ) |
| `sync.sh` | flock'd, shallow, stall-timeout `repo sync` |
| `tools/verify-sync.sh` | read-only: worktrees, shallowness, LFS payloads, `tmp_pack_*`, the ROM owns build/soong, `repo status` |
| `apply-overlays.sh` | **ROM configuration only**, numbered idempotent steps (1: imported kernel; 3–6: framework fixes for this SoC) |
| `rom-build.sh` | preflight (RAM, out-dir disk, sync lock, LFS, manifest, device tree clean + on `lineage-23.2`) → `axion S666LN <variant> va` → `m bacon` |
| `build-and-sign.sh` | the detached driver: verify-sync → apply-overlays → build → adb-mode gate → release signing → VERDICT |
| `sign-release.sh` | `sign_target_files_apks` with your own keys, then `ota_from_target_files -k`, then proof the signer is yours |

AxionOS is built **vanilla** (`va`: no GMS); microG is flashed on top separately. AxionOS's Game Space has bypass
charging built in through LineageOS Health charging control, so step 7 (AICP's Game Space tile) does not apply here.

## Reproduce

```sh
mkdir -p ~/axion && cd ~/axion
repo init -u https://github.com/AxionAOSP/android.git -b lineage-23.2 --git-lfs --depth=1 --no-clone-bundle
mkdir -p .repo/local_manifests && cp ~/itel_rs4_Axion/local_manifests/S666LN.xml .repo/local_manifests/
TOP=~/axion ~/itel_rs4_Axion/sync.sh && TOP=~/axion ~/itel_rs4_Axion/tools/verify-sync.sh
# the build refuses a device tree that is not on its branch (detached HEAD after a sync)
repo start lineage-23.2 device/itel/S666LN device/itel/S666LN-kernel vendor/itel/S666LN

TOP=~/axion OUT_TARGET=~/axion/out VARIANT=user SIGN=1 \
  setsid nohup bash ~/itel_rs4_Axion/build-and-sign.sh > /dev/null 2>&1 < /dev/null &
```

`SIGN=1` needs your release keys in `~/rs4-release-keys` (mode 700); they never go into a repo.

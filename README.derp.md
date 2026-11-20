# android_hardware_samsung_slsi_scsc_wifibt_wifi_hal — Derp fork

Samsung SLSI (Exynos7885) SCSC WiFi HAL, as used by the **Samsung A30s (a30s)**
crDroid 16 / LineageOS 23.2 build.

| | |
|---|---|
| Upstream | https://github.com/LineageOS/android_hardware_samsung_slsi_scsc_wifibt_wifi_hal |
| Upstream branch tracked | `lineage-23.2` (upstream tip at import: `033d400` — *"wifi_hal: update to latest linaro source"*) |
| Our branch | `lineage-23.2`, forked from that tip |
| Remotes in the build tree | `lineage` = upstream, `hyperramzey` = this fork |
| Build integration | `local_manifests/a30s.xml` → `path="hardware/samsung_slsi/scsc_wifibt/wifi_hal"`, installs as `vendor/lib64/libwifi-hal.so` under the Soong module name **`libwifi-hal-slsi`** |

## Local changes

| Commit | Issue | What |
|---|---|---|
| `17e2349` | device_samsung_a30s **#8 bug 6**, **#14.4** | `wifi_logger.cpp`: the vendor debug/logger `start()` path logged `ALOGE` for every failed `requestResponse()`. On this driver `-EOPNOTSUPP` is permanent (the SCSC firmware does not implement the vendor debug command) while the caller keeps retrying, so it emitted one error line per retry on every boot — measured 49 lines on a normal boot, 287 in the capture attached to #14, whose report attributes ~3 min of `logd` CPU in 5.5 h to this class of spam. Demoted to `ALOGV`, which is compiled out unless `LOG_NDEBUG` is 0, so the per-retry formatting cost goes too. Control flow and the returned value are unchanged. |

Nothing else in this repository differs from upstream `033d400`; the whole delta is
`git diff 033d400..lineage-23.2`.

## Why this fork exists

The A30s ships SCSC WiFi, and this HAL is the only component that emits that
particular flood — the property set and `/vendor/build.prop` carry no knob for it
(`strings` over the built library returns only the format string itself), so the log
call site is the lever. Keeping the change here rather than as an uncommitted edit in
a build directory is what makes it survive a `repo sync` and makes it reviewable.

## Verifying the flood is gone

```bash
# the built library carries no trace of the string, because ALOGV compiles out
grep -ac 'Failed to register debug response' \
  out/target/product/a30s/vendor/lib64/libwifi-hal.so      # -> 0  (0.8.1)
# the 0.8.0 library on a device still has it               # -> 1
# on device, after a boot:
adb shell su 0 sh -c 'logcat -d -b all | grep -c "Failed to register debug response"'   # -> 0
```

## Rebasing onto a newer upstream

```bash
git fetch lineage
git log --oneline lineage/lineage-23.2 ^lineage-23.2      # what is new upstream
git rebase --onto lineage/lineage-23.2 033d400 lineage-23.2
git push --force-with-lease hyperramzey lineage-23.2
```
The rebase is safe to replay blindly only because the delta is one log-level change;
re-read the `start()` path if upstream ever implements the debug command for real.

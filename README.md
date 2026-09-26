# Quaderno Gen 2 app unlock research

Research toward a reproducible, open source way to enable third-party Android apps on Fujitsu QUADERNO Gen 2 devices. The target is the Android 9 and Google Play setup shown in [Good e-Reader's video](https://www.youtube.com/watch?v=xUSnAtFd9_c). The video advertises the result and says stock note/PDF functions remain; it does not explain the unlock procedure. This repository is licensed MIT. Fujitsu firmware and other third-party materials retain their own licenses.

## Device under study

- Model: QUADERNO A4 Gen 2, USB product name `FMVDP41`.
- Firmware reported by the authenticated device API: `2.2.09.11030FP`.
- Host sees MTP plus a USB CDC ACM serial interface.
- On the same LAN, the device advertises `_dp_fujitsu._tcp` as `Android.local`.
- The public `/register/information` endpoint reports model and serial only. The firmware-version endpoint requires an authenticated client.
- No ADB interface is exposed in normal mode.

## Existing open source work

- [`A4_fw_unpacker`](https://github.com/ygjsz/A4_fw_unpacker) (MIT) extracts/decrypts Gen 2 update packages. It does not repack or install modified firmware.
- [`dpt-tools`](https://github.com/HappyZ/dpt-tools) documents root/update methods for Sony Digital Paper devices. Its [Gen 2 A4 issue](https://github.com/HappyZ/dpt-tools/issues/195) reports the update method failing on Quaderno Gen 2; no working fix is documented there.
- [`dpt-rp1-py`](https://github.com/janten/dpt-rp1-py) supports document/device APIs for Quaderno Gen 2. It is not a root or app-install method.

## Verified limits so far

- The host can identify and communicate with the device over MTP and the local device API.
- Pairing over the local device API succeeded; the client key is stored locally with owner-only permissions.
- A temporary USB serial mode switch was not performed. The current account cannot open `/dev/ttyACM0`.
- No write, update, root, or app-install operation has been attempted.
- Fujitsu's 797,327,290-byte update package for this firmware was downloaded to `/tmp` and unpacked with `A4_fw_unpacker`; both package signatures verified successfully. SHA-256: `e9d9a34f1a6154e12ec22fd1fe14c09b1924b258adedbd377964bc6645190e1a`.
- The decrypted package contains a recovery root filesystem plus `system.raw`, `vendor.raw`, `boot.img`, `dtbo`, `vbmeta`, and `rawdata.img` update images.
- Its recovery updater verifies the contents signature using the public key from `rawdata`, then runs an installer that checks the current version and writes those images to device partitions. The RSA private key bundled with the unpacker matches the key in `rawdata.img` and decrypts the package key; its public fingerprint differs from the package-signing key. The signing private key has not been found in the inspected files.
- The included i.MX8MM U-Boot binary contains Android Fastboot support, AVB verification, and the commands `flashing unlock` and `flashing get_unlock_ability`. This makes bootloader unlock status the next software-only check; the device has not yet been put into Fastboot mode.
- The existing package is verified and extractable; that does not establish that modified contents can be accepted by the device.
- No A4 Gen 2 Fastboot key sequence has been found in public documentation. Home + Power is only an unverified experiment from other Sony hardware, not a confirmed Quaderno procedure.
- A community reply reports that the commercial unlock service desolders and reprograms a flash chip, then resolders it. Treat this as an unverified report, not confirmed service documentation: [discussion](https://www.reddit.com/r/FujitsuQuaderno/comments/11y6exi/has_anyone_loaded_other_software_on_the_quaderno/). An earlier specialist forum discussion likewise says no software root method was known and points to chip-level access: [MobileRead thread](https://www.mobileread.com/forums/showthread.php?t=346817).
- Therefore, the presence of Fastboot commands in U-Boot is not evidence that this device exposes Fastboot or permits unlocking. A read-only Fastboot query is still useful if we can enter that mode safely.

## Next steps

1. Try a non-destructive boot-key entry into Fastboot only with the owner operating the device; the suggested Home + Power sequence is unverified. If it appears, run only `fastboot flashing get_unlock_ability` and `fastboot getvar unlocked`. The official Android Platform-Tools binary is staged under `/tmp/quaderno-platform-tools`; no unlock or flash command has been run.
2. If Fastboot is inaccessible or locked, obtain a board/chip identification and map a read-only dump/recovery method. Do not apply unknown images to the device.
3. Compare any acquired flash dump with the official package, determine the mechanism behind the commercial modification, and then document a reproducible procedure with A4 Gen 2/firmware coverage and a tested recovery path.

The project is exploratory and is not yet a working unlock tool or procedure. The hardware modification path is based on community reports and remains to be confirmed. No public reproduction, tested recovery process, or Google Play installation has been achieved.

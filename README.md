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

## Reproduce the stock firmware inspection

This downloads and unpacks the official package on the computer. It does not connect to or modify the Quaderno. The extractor is [`A4_fw_unpacker`](https://github.com/ygjsz/A4_fw_unpacker); run it only on the downloaded package, never on the device.

```sh
mkdir -p quaderno-work
cd quaderno-work
curl -fL 'https://www.fmworld.net/download/digital-paper/sw/FwUpdater_gen2_2.2.09.11030FP.pkg' -o FwUpdater_gen2_2.2.09.11030FP.pkg
printf '%s  %s\n' 'e9d9a34f1a6154e12ec22fd1fe14c09b1924b258adedbd377964bc6645190e1a' 'FwUpdater_gen2_2.2.09.11030FP.pkg' | sha256sum -c -
git clone https://github.com/ygjsz/A4_fw_unpacker.git
cd A4_fw_unpacker
./unpacker.sh ../FwUpdater_gen2_2.2.09.11030FP.pkg ../unpacked
unzip -l ../unpacked/contents_archive.zip
```

The package and extracted firmware images remain Fujitsu materials; the MIT license in this repository applies only to this repository's original notes and code.

## Verified limits so far

- The host can identify and communicate with the device over MTP and the local device API.
- Pairing over the local device API succeeded; the client key is stored locally with owner-only permissions.
- A read-only check confirms the normal paired client can request `/auth/nonce/{client_id}` (HTTP 200), but its request to `/testmode/auth/nonce/{client_id}` gets HTTP 401. The Sony-oriented `dpt-tools` source says test mode needs `K_PRIV_DT`; regular device pairing does not grant that credential.
- A temporary USB serial mode switch was not performed. The current account cannot open `/dev/ttyACM0`.
- No write, update, root, or app-install operation has been attempted.
- Fujitsu's 797,327,290-byte update package for this firmware was downloaded to `/tmp` and unpacked with `A4_fw_unpacker`; both package signatures verified successfully. SHA-256: `e9d9a34f1a6154e12ec22fd1fe14c09b1924b258adedbd377964bc6645190e1a`.
- The decrypted package contains a recovery root filesystem plus `system.raw`, `vendor.raw`, `boot.img`, `dtbo`, `vbmeta`, and `rawdata.img` update images.
- Inspection of `system.raw:/system/build.prop` confirms Android 9 / SDK 28, build type `user`, tags `dev-keys`, and security patch `2019-04-05` (fingerprint `Android/evk_8mm/evk_8mm:9/PQ2A.190405.003/11030448:user/dev-keys`). No Google/GMS/Play packages appear in `/system/app` or `/system/priv-app`. The video's result therefore needs extra system components and a supported way to boot the modified image; the stock firmware does not merely hide a preinstalled Play Store icon.
- ADB is disabled by stock configuration: `/system/etc/prop.default` sets `ro.debuggable=0`, `ro.secure=1`, `ro.adb.secure=1`, and `persist.sys.usb.config=none`. The vendor USB gadget script only adds ADB when `ro.debuggable` is nonzero. The standard `adbd` executable is absent from `/system/bin` and `/vendor/bin`, so `adb reboot bootloader` is not a stock path. The working USB profile exposes MTP.
- Its recovery updater verifies the contents signature using the public key from `rawdata`, then runs an installer that checks the current version and writes those images to device partitions. The RSA private key bundled with the unpacker matches the key in `rawdata.img` and decrypts the package key; its public fingerprint differs from the package-signing key. The signing private key has not been found in the inspected files.
- The included i.MX8MM U-Boot binary contains Android Fastboot support, AVB verification, and the commands `flashing unlock` and `flashing get_unlock_ability`. This does not establish that the device exposes Fastboot; the device has not yet been put into that mode.
- The same U-Boot binary includes `bootcmd_mfg`, `usb_dnl_sdp`, `usb_dnl_fastboot`, the message `Detect USB boot. Will enter fastboot mode!`, and a recovery-key check. These are candidate boot paths, not proof that they are user-accessible on the QUADERNO. NXP's generic i.MX instructions rely on volume keys or a U-Boot console; the A4 has no volume buttons, and no A4-specific key mapping or USB-boot trigger has been found. See [NXP's i.MX Android FAQ](https://community.nxp.com/t5/i-MX-Processors-Knowledge-Base/i-MX-Android-Frequently-Asked-Questions/ta-p/1112713).
- The existing package is verified and extractable; that does not establish that modified contents can be accepted by the device.
- No A4 Gen 2 Fastboot key sequence has been found in public documentation. Home + Power is only an unverified experiment from other Sony hardware, not a confirmed Quaderno procedure.
- A community reply reports that the commercial unlock service desolders and reprograms a flash chip, then resolders it. Treat this as an unverified report, not confirmed service documentation: [discussion](https://www.reddit.com/r/FujitsuQuaderno/comments/11y6exi/has_anyone_loaded_other_software_on_the_quaderno/). An earlier specialist forum discussion likewise says no software root method was known and points to chip-level access: [MobileRead thread](https://www.mobileread.com/forums/showthread.php?t=346817).
- A public A5 Gen 2 repair post includes an internal board photo, but does not identify the storage chip, test pads, or a dump method. It is FMVDP51, so the photo cannot establish A4 FMVDP41 board details: [teardown and recovery post](https://note.com/kanfu0303/n/n4251d670295f).
- Therefore, the presence of Fastboot commands in U-Boot is not evidence that this device exposes Fastboot or permits unlocking. A read-only Fastboot query is still useful if we can enter that mode safely.

## Next steps

1. Find an A4-specific way to invoke the U-Boot recovery-key or USB-boot path. Home + Power remains an unverified experiment. If Fastboot appears, run only `fastboot flashing get_unlock_ability` and `fastboot getvar unlocked`. The official Android Platform-Tools binary is staged under `/tmp/quaderno-platform-tools`; no unlock or flash command has been run.
2. If those paths are unavailable, determine whether a legitimate test-mode credential exists or obtain board/chip identification and map a read-only dump/recovery method. Do not apply unknown images to the device.
3. Compare any acquired flash dump with the official package, determine the mechanism behind the commercial modification, and then document a reproducible procedure with A4 Gen 2/firmware coverage and a tested recovery path.

The project is exploratory and is not yet a working unlock tool or procedure. The hardware modification path is based on community reports and remains to be confirmed. No public reproduction, tested recovery process, or Google Play installation has been achieved.

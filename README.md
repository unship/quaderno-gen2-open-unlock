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
- The [unlock video](https://www.youtube.com/watch?v=xUSnAtFd9_c) links to Good e-Reader's [A4 Gen 2 unlock-service listing](https://goodereader.com/blog/product/fujitsu-quaderno-a4-gen-2-android-9-0-unlock-service). The video presents the result and product offer, not a reproducible procedure.

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
- The stock `/system/app/EbookTestMode/EbookTestMode.apk` is a factory diagnostic UI (battery, pen/touch, NFC, and test-file tools). It does not document a bootloader unlock or substitute for the rejected `/testmode` API credential.
- Read-only access to `/dev/ttyACM0` was authorized, but the current account lacks the `uucp` group permission; the attempted unprivileged read failed and sent no data.
- No write, update, root, or app-install operation has been attempted.
- Fujitsu's 797,327,290-byte update package for this firmware was downloaded to `/tmp` and unpacked with `A4_fw_unpacker`; both package signatures verified successfully. SHA-256: `e9d9a34f1a6154e12ec22fd1fe14c09b1924b258adedbd377964bc6645190e1a`.
- The decrypted package contains a recovery root filesystem plus `system.raw`, `vendor.raw`, `boot.img`, `dtbo`, `vbmeta`, and `rawdata.img` update images.
- Inspection of `system.raw:/system/build.prop` confirms Android 9 / SDK 28, build type `user`, tags `dev-keys`, and security patch `2019-04-05` (fingerprint `Android/evk_8mm/evk_8mm:9/PQ2A.190405.003/11030448:user/dev-keys`). No Google/GMS/Play packages appear in `/system/app` or `/system/priv-app`. The video's result therefore needs extra system components and a supported way to boot the modified image; the stock firmware does not merely hide a preinstalled Play Store icon.
- ADB is disabled by stock configuration: `/system/etc/prop.default` sets `ro.debuggable=0`, `ro.secure=1`, `ro.adb.secure=1`, and `persist.sys.usb.config=none`. The vendor USB gadget script only adds ADB when `ro.debuggable` is nonzero. The standard `adbd` executable is absent from `/system/bin` and `/vendor/bin`, so `adb reboot bootloader` is not a stock path. The working USB profile exposes MTP.
- Its recovery updater verifies the contents signature using the public key from `rawdata`, then runs an installer that checks the current version and writes those images to device partitions. The RSA private key bundled with the unpacker matches the key in `rawdata.img` and decrypts the package key; its public fingerprint differs from the package-signing key. The signing private key has not been found in the inspected files.
- The included i.MX8MM U-Boot binary contains Android Fastboot support, AVB verification, and the commands `flashing unlock` and `flashing get_unlock_ability`. This does not establish that the device exposes Fastboot; the device has not yet been put into that mode.
- The U-Boot binary identifies its board as `fsl,imx8mm-evk` and reports version `2018.03-gdb8aa62-dirty`. In NXP's matching [i.MX8MM EVK source branch](https://github.com/nxp-imx/uboot-imx/blob/imx_v2018.03_4.14.98_2.3.0/board/freescale/imx8mm_evk/imx8mm_evk.c#L750), `is_recovery_key_pressing()` is a TODO stub that returns 0. The Fastboot code uses that hook for recovery-key detection. This strongly suggests the stock reference path has no hardware recovery-key combo; `-dirty` means Fujitsu could have changed that code, so the binary's exact function has not yet been proven. The binary also contains `Detect USB boot. Will enter fastboot mode!`. In [NXP's matching `common/autoboot.c`](https://github.com/nxp-imx/uboot-imx/blob/imx_v2018.03_4.14.98_2.3.0/common/autoboot.c), this message is conditional on `is_boot_from_usb()` and routes into the USB manufacturing/fastboot path. This points to the i.MX ROM USB serial-download boot source, not the normal Android USB serial interface; it does not reveal how to select that source on the QUADERNO. The binary's `usb_dnl_sdp` and `usb_dnl_fastboot` strings are consistent with that path. NXP's generic i.MX instructions use Power + Volume Down for recovery and Power + Volume Up to open recovery controls; the A4 has no volume buttons. The generic `Fastboot: Got Recovery key pressing or recovery commands!` log also appears on unrelated i.MX boards, so it does not identify the A4 key mapping. See [NXP's i.MX Android FAQ](https://community.nxp.com/t5/i-MX-Processors-Knowledge-Base/i-MX-Android-Frequently-Asked-Questions/ta-p/1112713) and [an unrelated i.MX boot log](https://community.nxp.com/t5/i-MX-Processors/iMX6q-sabresd-OTA-update-for-Android-9/td-p/935986).
- The existing package is verified and extractable; that does not establish that modified contents can be accepted by the device.
- No A4 Gen 2 Fastboot key sequence has been found in public documentation. Sony DPT's [`dpt-tools` diagnostic-mode code](https://github.com/HappyZ/dpt-tools/blob/228688ed8069d6a618670c104fd66e9235a56616/python_api/libInteractive.py#L397-L411) describes powering off, holding HOME, tapping POWER, then releasing HOME when a black square appears. That enters Sony DPT diagnostic mode, not Fastboot; Quaderno A4 Gen 2 behavior is unverified. The user authorized trying this sequence, but it requires pressing the device's physical buttons.
- During the authorized Home + Power attempt, the user reported a yellow blink followed by normal startup, with no black square. The host logged a USB disconnect and transient descriptor-read errors, then rediscovered `FMVDP41` (`04c5:1656`) with the same MTP and CDC ACM interfaces. `fastboot devices` and `adb devices` remained empty.
- A community reply reports that the commercial unlock service desolders and reprograms a flash chip, then resolders it. Treat this as an unverified report, not confirmed service documentation: [discussion](https://www.reddit.com/r/FujitsuQuaderno/comments/11y6exi/has_anyone_loaded_other_software_on_the_quaderno/). An earlier specialist forum discussion likewise says no software root method was known and points to chip-level access: [MobileRead thread](https://www.mobileread.com/forums/showthread.php?t=346817).
- A public A5 Gen 2 repair post includes an internal board photo, but does not identify the storage chip, test pads, or a dump method. It is FMVDP51, so the photo cannot establish A4 FMVDP41 board details: [teardown and recovery post](https://note.com/kanfu0303/n/n4251d670295f).
- A commercial [FMVDP41 teardown report](https://www.chip1stop.com/POL/en/products/Fomalhaut-Techno-Solutions/Tablet-teardown-report%3AFMVDP41/FHTS%2A0000883) is listed by Fomalhaut Techno Solutions through Chip One Stop. The public listing identifies a 5,113 KB report but exposes no board photos or boot-mode details; no report has been purchased or redistributed here.
- Therefore, the presence of Fastboot commands in U-Boot is not evidence that this device exposes Fastboot or permits unlocking. A read-only Fastboot query is still useful if we can enter that mode safely.

## Next steps

1. Identify whether Fujitsu changed the U-Boot recovery-key stub or determine the board-specific way to invoke the i.MX ROM USB serial-download path. No button combination or boot strap is verified. If Fastboot becomes available, run only `fastboot flashing get_unlock_ability` and `fastboot getvar unlocked`. The official Android Platform-Tools binary is staged under `/tmp/quaderno-platform-tools`; no unlock or flash command has been run.
2. If those paths are unavailable, determine whether a legitimate test-mode credential exists or obtain board/chip identification and map a read-only dump/recovery method. Do not apply unknown images to the device.
3. Compare any acquired flash dump with the official package, determine the mechanism behind the commercial modification, and then document a reproducible procedure with A4 Gen 2/firmware coverage and a tested recovery path.

The project is exploratory and is not yet a working unlock tool or procedure. The hardware modification path is based on community reports and remains to be confirmed. No public reproduction, tested recovery process, or Google Play installation has been achieved.

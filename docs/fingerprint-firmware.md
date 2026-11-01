# Fingerprint TA firmware under an unpinned stock ROM

`liuqin-fpc-stable-module.service` provisions ten FPC TA files
(`fpcliu.mdt`, `fpcliu.b00` .. `fpcliu.b08`) from the read-only `modem_a`
partition and validates each one against
`/usr/local/lib/liuqin-fpc-oem/firmware/SHA256.json`. Those pinned values
identify the `OS2.0.6.0.VMYCNXM` release. With `OS3.0.7.0.VMYCNXM` flashed, the
service refuses to start:

```
RuntimeError: stock FPC firmware hash mismatch: fpcliu.mdt
```

## What actually differs

| file | pinned (OS2.0.6.0.VMYCNXM) | OS3.0.7.0.VMYCNXM on the test unit |
|---|---|---|
| `fpcliu.mdt` | `90a1b317…` | `0869455d…` |
| `fpcliu.b08` | `9da1e79a…` | `11b671c7…` |
| `fpcliu.b00` .. `fpcliu.b07` | — | byte-identical to the pinned values |

So only two of the ten files changed between the two stock releases.

## Method A — accept the firmware that is on the tablet (unpinned)

Rewrite the two entries in `firmware/SHA256.json` with the hashes of the local
`modem_a` files, and keep the upper-level manifests
(`/usr/local/lib/liuqin-fpc-oem/SHA256.json` and `SHA256SUMS`) consistent so the
build-integrity records still match.

Observed on the test unit:

- service `active (exited)`, `fpc_firmware=READY source=local_modem_read_only`;
- driver reads HWID `0x2012` and identifies the sensor;
- `fprintd` enumerates one device and an enrolment session starts
  (`OEM enroll sample status=… remaining=…`).

The enrolment did **not** complete on this combination: the session aborted with
`pipeline_incomplete=finger_lost_wait`, so end-to-end fingerprint login was not
verified with the OS3.0.7.0 TA.

Advantages: no extra download, small change.
Drawbacks: the TA / driver combination is not the one upstream validated.

## Method B — restore the pinned (OS2.0.6.0) firmware (used on the test unit)

Extract `NON-HLOS.bin` from the official `OS2.0.6.0.VMYCNXM` fastboot package,
mount it read-only, and copy the ten `fpcliu.*` files into the firmware cache:

```sh
tar -xzf liuqin_images_OS2.0.6.0.VMYCNXM_*.tgz "*/images/NON-HLOS.bin"
mnt=$(mktemp -d) && mount -t vfat -o ro,loop,nodev,nosuid,noexec NON-HLOS.bin "$mnt"
install -m 0600 -o root -g root "$mnt"/image/fpcliu.* /usr/local/lib/liuqin-fpc-oem/firmware/
umount "$mnt" && rmdir "$mnt"
systemctl restart liuqin-fpc-stable-module.service
```

All ten hashes then match the pinned manifest and the service reports
`fpc_firmware=READY source=verified_cache`. The kernel identifies the sensor
(`0x2012`) and `fprintd` enumerates the device. No fingerprint was enrolled on
the test unit, so login itself was not exercised.

Advantages: exactly the firmware combination upstream validated.
Drawbacks: needs the 6.7 GB `OS2.0.6.0` package (and ~110 MB extracted
`NON-HLOS.bin`).

## Which one to use

| | Method A (local TA) | Method B (pinned TA) |
|---|---|---|
| download needed | none | 6.7 GB stock ROM |
| firmware identity | OS3.0.7.0 TA | upstream-validated OS2.0.6.0 TA |
| service / sensor / fprintd | works | works |
| enrolment verified | no (aborted at `finger_lost_wait`) | not attempted |
| upstream support | no | yes (same firmware as the supported ROM) |

Method B keeps the device on the firmware the driver and userspace were validated
against and is the safer choice for a daily driver. Method A is a reasonable
alternative if re-downloading the older ROM is not practical; treat it as
unvalidated and expect possible enrolment quality issues.

## Warning

Do not clear `/usr/local/lib/liuqin-fpc-oem/firmware/` on an `OS3.0.7.0` device:
the next service start re-provisions from `modem_a` and, with the pinned
manifest, fails again with the hash mismatch. If the cache must be recreated,
re-run the Method B copy first.

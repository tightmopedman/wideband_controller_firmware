# 3.1.0, ready to sign and upload

Both units move, for the first time since the update path was finished:

- the **display** goes 3.0.6 to 3.1.0, and this is the first published image with the finished update UI: the progress
  screen, the failure verdict with its reason, and the version guard;
- the **controller** goes 3.0.1 to 3.1.0, and this is the first published controller image carrying the trial-boot
  confirm (`verifyRollbackLater`) as well as the 250 ms warm-up poll, the 10 s heartbeat, the three-miss error and the
  calibration abort.

| File | Size | sha256 (first 16) |
|---|---|---|
| `manifest.json` | 437 B | unsigned |
| `controller_3.1.0.bin` | 928,160 B | `aa7c92ca30b24e5f` |
| `display_3.1.0.bin` | 3,411,792 B | `dceedff3c6e5600e` |

Both are app images for the ESP32-S3: `esptool image_info` reports `Checksum (valid)` and `Validation Hash (valid)` on each.
They fit their slots: the controller's app partition is 0x330000 (3,342,336 B) and the display's is 0x640000 (6,553,600 B).

## Sign it

The key is not on the bench machine, so this manifest is unsigned on purpose:

```bash
python3 common/tools/sign_manifest.py sign --dir firmware/3.1.0 --key <path to private.pem>
python3 common/tools/sign_manifest.py verify --dir firmware/3.1.0 --key <path to public.pem>
```

`verify` prints `signature verifies against <key>`. Do not upload until it does.

## What to upload

To `github.com/tightmopedman/wideband_controller_firmware`, branch `main`, as **one commit** so the manifest and its
signature never disagree for a second. The hosted repository is flat, and the firmware resolves each url in the manifest
against it:

- `manifest.json`
- `manifest.sig`
- `controller_3.1.0.bin`
- `display_3.1.0.bin`

Nothing else changes. Both images have new names, so nothing already hosted is replaced.

## The order to run the updates in

**Controller first, display second.** The display's own update installs the *release* image, which has no bench console, so
doing it last leaves the console available for the controller step and for anything that goes wrong. The controller step is
driven from the Updates screen either way.

What each step should show, and what would mean something went wrong:

| Step | Expected |
|---|---|
| Controller check | row goes to `Ready to Update`; 3.1.0 is newer than 3.0.1 |
| Controller update | progress screen counts, then `Update Successful!`; the unit restarts into the new image and its row then reads 3.1.0 |
| if it rolls back | `Failed: the controller came back on 3.0.1, not 3.1.0`; the old image is what is running and the panel says so |
| Display check | row goes to `Ready to Update`; 3.1.0 is newer than 3.0.6 |
| Display update | downloads, installs into the other slot, restarts, and comes back reporting 3.1.0 |

The controller step can only be run once per publish: after it, the unit reports 3.1.0 and the same manifest correctly
refuses to install it again (`Failed: the controller already runs 3.1.0, which the manifest does not beat`). That refusal
is the guard working, not a fault.

After the display takes the release, the bench console is gone from that unit. Re-flash the bench build over USB
(`pio run -e wideband_v3 -t upload`) when it is needed again; it reports 3.1.0 as well.

## The trap this release exists to avoid

The hosted `controller_3.0.1.bin` is 787,840 bytes and predates the trial-boot confirm, so a unit that installed it would
boot a new image that confirms itself without ever testing it: a bad image would never roll back. It is also older code
than the unit is running, which is why the version guard was added before this publish rather than after it.

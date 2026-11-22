# postmarketOS for Xiaomi Mi 6 (sagit)

Experimental aarch64 port based on the archived postmarketOS device recipes.
The r14 kernel was tested on a Mi 6 with Phosh and OpenRC. It includes the
firmware memory map, battery gauge and charging fixes, plus the ath10k
disconnect fixes. The last test completed five 2.4 GHz and three 5 GHz
reconnects without a firmware crash. QC fast charging is still experimental
and is not included.

The r14 release contains the kernel APK for an existing compatible
installation. It is not a Fastboot image. The older r8 full image predates
the r14 fixes and contains device-specific login settings. Full images also
contain Xiaomi/Qualcomm firmware whose redistribution terms have not been
verified.

## Source and build inputs

- pmbootstrap: https://gitlab.postmarketos.org/postmarketOS/pmbootstrap
  at `7d23010a125ca7386fec4e390d14128b6c24d266`.
- pmaports: https://gitlab.postmarketos.org/postmarketOS/pmaports
  at `0e458a2417d6f5ed61c7b70621d5632e30136091`.
- `ports/` contains the package recipes and the exact kernel patches used
  for r14. The kernel recipe pins the source archive and its SHA-512 checksum.
- `config/pmbootstrap.example.cfg` records the target, UI and development
  package selection. `config/rootfs/` contains the shareable overlay.

Use a separate Linux build workspace. Check out the pinned revisions,
move the four archived sagit/MSM8998 recipes out of `device/archived/`
in that checkout (device-xiaomi-sagit, firmware-xiaomi-sagit,
linux-postmarketos-qcom-msm8998 and soc-qcom-msm8998), then copy the five
directories in `ports/` into `device/testing/`.

Initialize pmbootstrap with the local pmaports checkout, device
`xiaomi-sagit`, and Phosh. Use the example for package and OpenRC options.
Build the local kernel and dependency packages, then generate a fresh
rootfs with credentials chosen for your device:

```sh
pmbootstrap build --arch aarch64 linux-postmarketos-qcom-msm8998
pmbootstrap build --arch aarch64 soc-qcom-msm8998 firmware-xiaomi-sagit pd-mapper
pmbootstrap install
```

For the desktop settings and network defaults, apply `config/rootfs/`
to the build rootfs before image generation and run
`glib-compile-schemas --strict /usr/share/glib-2.0/schemas` in that rootfs.
Review image size against your device's system partition and verify the
exported artifacts before flashing. Package resolution from edge mirrors
may differ from the original October 2026 build.

## Optional: fix WCN3990 5 GHz download stalls

[`mi6-split-amsdu.patch`](ports/linux-postmarketos-qcom-msm8998/mi6-split-amsdu.patch)
is the Linux 6.0 backport used in the tested r20 kernel, based on the
[upstream v4 proposal](https://lkml.rescloud.iu.edu/2610.2/11670.html).
WCN3990 can deliver the subframes of one received A-MSDU in consecutive HTT
notifications. The old `ath10k_snoc` path treated the first incomplete batch
as a fatal error (`failed to extract amsdu: -11`) and stopped processing
that receive path. The patch retains the incomplete batch until the next
matching notification; it bounds the queue and purges it on error or teardown.
It does not change firmware, transmit rate, or the 2.4 GHz settings.

The patch is opt-in; the checked-in APKBUILD still reproduces r14. To build
with the fix, use the separate Linux pmaports checkout configured above:

1. Copy the port recipes as above. In the **test checkout's**
   `device/testing/linux-postmarketos-qcom-msm8998/APKBUILD`, add
   `mi6-split-amsdu.patch` at the end of the `source` list. Set `pkgrel=21`
   (or a higher unused revision if needed). `default_prepare` applies the patch.
2. With pmbootstrap pointed at that checkout, refresh checksums and build:

   ```sh
   pmbootstrap checksum linux-postmarketos-qcom-msm8998
   pmbootstrap build --arch aarch64 linux-postmarketos-qcom-msm8998
   ```

3. For a fresh `xiaomi-sagit` image, run `pmbootstrap install` using the
   same checkout and confirm the image contains the new kernel package. For an
   existing compatible installation, copy the resulting aarch64 kernel APK
   to the phone, verify its SHA-256 against the builder's copy, and run:

   ```sh
   sudo apk add --no-network ./linux-postmarketos-qcom-msm8998-6.0-r21.apk
   ```

   Adjust the filename to match `pkgrel`. For a locally built package whose
   signature the phone does not trust, add `--allow-untrusted` only after
   checking its checksum. Keep the previous kernel APK and `boot.img` and
   confirm USB/Fastboot recovery first: installation can write the boot
   partition. Do not reboot if installing or updating the boot image fails.

After reboot, verify the installed package and `uname -v`. Repeat the same
5 GHz Mac-to-phone download against the same BSSID and check `dmesg` for
`failed to extract amsdu: -11` or `split amsdu did not resume immediately`.
The r20 diagnostic build on one Mi 6 reached about 200 Mbps in both directions
without reproducing the earlier disconnect in that test. The clean r14-based
recipe plus this patch is a separate build and must be verified on-device.

## Optional: trace 2.4 GHz transmit stalls

The r14 ath10k patches address key removal and disassociation, not data-plane
throughput. `config/ath10k-tx-trace.config` enables the existing ath10k trace
events in a separate diagnostic build without changing rate control, firmware,
or the release config. In the pinned kernel source tree on the Linux builder,
merge it into the release config, using a separate pmaports checkout for testing:

```sh
port=/path/to/postmarketos-mi6
aport=/path/to/test-pmaports/device/testing/linux-postmarketos-qcom-msm8998
cfg=config-postmarketos-qcom-msm8998.aarch64
scripts/kconfig/merge_config.sh -m \
    "$port/ports/linux-postmarketos-qcom-msm8998/$cfg" \
    "$port/config/ath10k-tx-trace.config"
cp .config "$aport/$cfg"
pmbootstrap pkgrel_bump linux-postmarketos-qcom-msm8998
pmbootstrap checksum linux-postmarketos-qcom-msm8998
pmbootstrap build --arch aarch64 linux-postmarketos-qcom-msm8998
```

Point pmbootstrap at the test pmaports checkout first. The device recipe can
update the boot partition on kernel package installation; preserve the current
boot image and confirm a USB/Fastboot recovery path before installing.

On the test kernel, run the block below in a root console or over USB SSH.
While it is waiting, run `iperf3 -c <phone-ip> -R -t 15` over 2.4 GHz from
the other endpoint, then press Enter to stop and print the trace. Only the
two TX metadata events and the filtered completion log are collected, not
packet payloads.

```sh
(
    set -eu
    base=/sys/kernel/tracing
    mountpoint -q "$base" || mount -t tracefs tracefs "$base"
    trace="$base/instances/mi6-tx-$$"
    mkdir "$trace"
    trap 'rmdir "$trace"' EXIT
    echo 0 > "$trace/tracing_on"
    echo mono > "$trace/trace_clock"
    echo 1024 > "$trace/buffer_size_kb"
    events="$trace/events/ath10k"
    echo 'msg ~ "htt tx completion msdu_id *"' > "$events/ath10k_log_dbg/filter"
    for event in ath10k_htt_tx ath10k_txrx_tx_unref ath10k_log_dbg; do
        echo 1 > "$events/$event/enable"
    done
    echo 1 > "$trace/tracing_on"
    printf '%s\n' 'Run the 2.4 GHz test now, then press Enter.'
    read -r _
    echo 0 > "$trace/tracing_on"
    cat "$trace"/per_cpu/cpu*/stats
    cat "$trace/trace"
)
```

Compare TX preparation and completion by timestamp and `msdu_id` (IDs are
reused); check the per-CPU overrun counters before interpreting unmatched
IDs. The preparation event precedes the host-interface submission, so a
missing completion alone does not prove that firmware received the frame.
Check `dmesg` for host submission failures too. The completion log contains
the driver's normalized state (`1` ACK, `2` no ACK, `3` discarded), not the
raw firmware status. This capture is a diagnostic, not a throughput fix.


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


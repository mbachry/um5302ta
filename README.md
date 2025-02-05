# Zenbook S 13 OLED (UM5302TA)

## TL;DR

Using a kernel version ≥ 6.0.9 and applying the DSDT patch specified in the [speaker](#speaker) section.

## Keyboard

✔️ Work (kernel version ≥ 5.19.10 or ≥ 5.15.69 (longterm))

<details>
<summary>
<del>Patch: <a href="./patches/kernel/ACPI-skip-IRQ-override-on-AMD-Zen-platforms.patch">kernel/ACPI-skip-IRQ-override-on-AMD-Zen-platforms.patch</a></del> (included in kernel 5.15.69 / 5.19.10 / 6.0, <a href="https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=9946e39fe8d0a5da9eb947d8e40a7ef204ba016e">source</a>)
</summary>

```diff
diff --git a/drivers/acpi/resource.c b/drivers/acpi/resource.c
index c2d4947844250..510cdec375c4d 100644
--- a/drivers/acpi/resource.c
+++ b/drivers/acpi/resource.c
@@ -416,6 +416,16 @@ static bool acpi_dev_irq_override(u32 gsi, u8 triggering, u8 polarity,
 {
 	int i;

+#ifdef CONFIG_X86
+	/*
+	 * IRQ override isn't needed on modern AMD Zen systems and
+	 * this override breaks active low IRQs on AMD Ryzen 6000 and
+	 * newer systems. Skip it.
+	 */
+	if (boot_cpu_has(X86_FEATURE_ZEN))
+		return false;
+#endif
+
 	for (i = 0; i < ARRAY_SIZE(skip_override_table); i++) {
 		const struct irq_override_cmp *entry = &skip_override_table[i];

```

</details>

## Speaker

⚠️ DSDT patch required (kernel version ≥ 6.0.9, [details](https://bugzilla.kernel.org/show_bug.cgi?id=216194))

<details>
<summary>
Patch: <a href="./patches/dsdt/spkr-dsd.patch">dsdt/spkr-dsd.patch</a>
</summary>

```diff
diff --git a/dsdt.dsl b/dsdt.dsl
index 663aa79..f485c41 100644
--- a/dsdt.dsl
+++ b/dsdt.dsl
@@ -18,7 +18,7 @@
  *     Compiler ID      "INTL"
  *     Compiler Version 0x20200717 (538969879)
  */
-DefinitionBlock ("", "DSDT", 2, "_ASUS_", "Notebook", 0x01072009)
+DefinitionBlock ("", "DSDT", 2, "_ASUS_", "Notebook", 0x0107200A)
 {
     External (_SB_.ALIB, MethodObj)    // 2 Arguments
     External (_SB_.APTS, MethodObj)    // 1 Arguments
@@ -14734,6 +14734,85 @@ DefinitionBlock ("", "DSDT", 2, "_ASUS_", "Notebook", 0x01072009)
             Method (_DIS, 0, NotSerialized)  // _DIS: Disable Device
             {
             }
+
+            Name (_DSD, Package (0x02)  // _DSD: Device-Specific Data
+            {
+                ToUUID ("daffd814-6eba-4d8c-8a91-bc9bbf4aa301") /* Device Properties for _DSD */,
+                Package (0x06)
+                {
+                    Package (0x02)
+                    {
+                        "cirrus,dev-index",
+                        Package (0x02)
+                        {
+                            0x40,
+                            0x41
+                        }
+                    },
+
+                    Package (0x02)
+                    {
+                        "reset-gpios",
+                        Package (0x08)
+                        {
+                            SPKR,
+                            Zero,
+                            Zero,
+                            Zero,
+                            SPKR,
+                            Zero,
+                            Zero,
+                            Zero
+                        }
+                    },
+
+                    Package (0x02)
+                    {
+                        "spk-id-gpios",
+                        Package (0x08)
+                        {
+                            SPKR,
+                            0x02,
+                            Zero,
+                            Zero,
+                            SPKR,
+                            0x02,
+                            Zero,
+                            Zero
+                        }
+                    },
+
+                    Package (0x02)
+                    {
+                        "cirrus,speaker-position",
+                        Package (0x02)
+                        {
+                            Zero,
+                            One
+                        }
+                    },
+
+                    Package (0x02)
+                    {
+                        "cirrus,gpio1-func",
+                        Package (0x02)
+                        {
+                            Zero,
+                            One
+                        }
+                    },
+
+                    Package (0x02)
+                    {
+                        "cirrus,gpio2-func",
+                        Package (0x02)
+                        {
+                            0x02,
+                            0x02
+                        }
+                    }
+                }
+            })
         }
     }

```

See also: [DSDT - ArchWiki](https://wiki.archlinux.org/title/DSDT)

</details>

<details>
<summary>
<del>Patch: <a href="./patches/kernel/ALSA-hda-realtek-Add-quirk-for-ASUS-Zenbook-using-CS35L41.patch">kernel/ALSA-hda-realtek-Add-quirk-for-ASUS-Zenbook-using-CS35L41.patch</a></del> (included in kernel 6.0.9 / 6.1, <a href="https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=8d06679b25fc6813eb2438fac7fa13f4f3c2ef37">source</a>)
</summary>

```diff
diff --git a/sound/pci/hda/patch_realtek.c b/sound/pci/hda/patch_realtek.c
index 701a72ec5629a..b4f7ff8cfe41b 100644
--- a/sound/pci/hda/patch_realtek.c
+++ b/sound/pci/hda/patch_realtek.c
@@ -9404,6 +9404,7 @@ static const struct snd_pci_quirk alc269_fixup_tbl[] = {
 	SND_PCI_QUIRK(0x1043, 0x1e8e, "ASUS Zephyrus G15", ALC289_FIXUP_ASUS_GA401),
 	SND_PCI_QUIRK(0x1043, 0x1c52, "ASUS Zephyrus G15 2022", ALC289_FIXUP_ASUS_GA401),
 	SND_PCI_QUIRK(0x1043, 0x1f11, "ASUS Zephyrus G14", ALC289_FIXUP_ASUS_GA401),
+	SND_PCI_QUIRK(0x1043, 0x1f12, "ASUS UM5302", ALC287_FIXUP_CS35L41_I2C_2),
 	SND_PCI_QUIRK(0x1043, 0x1f92, "ASUS ROG Flow X16", ALC289_FIXUP_ASUS_GA401),
 	SND_PCI_QUIRK(0x1043, 0x3030, "ASUS ZN270IE", ALC256_FIXUP_ASUS_AIO_GPIO2),
 	SND_PCI_QUIRK(0x1043, 0x831a, "ASUS P901", ALC269_FIXUP_STEREO_DMIC),
```

</details>

<details>
<summary>
<del>Patch: <a href="./patches/kernel/cs35l42-hda-no-acpi-dsd-csc3551.patch">kernel/cs35l42-hda-no-acpi-dsd-csc3551.patch</a></del> (rejected, <a href="https://patchwork.kernel.org/project/alsa-devel/patch/20220703053225.2203-1-xw897002528@gmail.com/">source</a>)
</summary>

```diff
diff --git a/sound/pci/hda/cs35l41_hda.c b/sound/pci/hda/cs35l41_hda.c
index e5f0549bf06d..3917f398334d 100644
--- a/sound/pci/hda/cs35l41_hda.c
+++ b/sound/pci/hda/cs35l41_hda.c
@@ -1231,7 +1231,7 @@ static int cs35l41_no_acpi_dsd(struct cs35l41_hda *cs35l41, struct device *physd

 	if (strncmp(hid, "CLSA0100", 8) == 0) {
 		hw_cfg->bst_type = CS35L41_EXT_BOOST_NO_VSPK_SWITCH;
-	} else if (strncmp(hid, "CLSA0101", 8) == 0) {
+	} else if (strncmp(hid, "CLSA0101", 8) == 0 || strncmp(hid, "CSC3551", 7) == 0) {
 		hw_cfg->bst_type = CS35L41_EXT_BOOST;
 		hw_cfg->gpio1.func = CS35l41_VSPK_SWITCH;
 		hw_cfg->gpio1.valid = true;
```

</details>

## Microphone

✔️ Work (kernel version ≥ 6.0.3)

<details>
<summary>
<del>Patch: <a href="patches/kernel/ASoC-amd-yc-Add-ASUS-UM5302TA-into-DMI-table.patch">kernel/ASoC-amd-yc-Add-ASUS-UM5302TA-into-DMI-table.patch</a></del> (included in kernel 6.0.3 / 6.1, <a href="https://git.kernel.org/pub/scm/linux/kernel/git/next/linux-next.git/commit/?id=4df5b13dec9e1b5a12db47ee92eb3f7da5c3deb5">source</a>)
</summary>

```diff
diff --git a/sound/soc/amd/yc/acp6x-mach.c b/sound/soc/amd/yc/acp6x-mach.c
index e0b24e1daef3d..5eab3baf3573d 100644
--- a/sound/soc/amd/yc/acp6x-mach.c
+++ b/sound/soc/amd/yc/acp6x-mach.c
@@ -171,6 +171,13 @@ static const struct dmi_system_id yc_acp_quirk_table[] = {
 			DMI_MATCH(DMI_PRODUCT_NAME, "21J6"),
 		}
 	},
+	{
+		.driver_data = &acp6x_card,
+		.matches = {
+			DMI_MATCH(DMI_BOARD_VENDOR, "ASUSTeK COMPUTER INC."),
+			DMI_MATCH(DMI_PRODUCT_NAME, "UM5302TA"),
+		}
+	},
 	{}
 };

```

</details>

### Known Issues

- Volume is quite low: <https://bugzilla.kernel.org/show_bug.cgi?id=216495>

## Bluetooth

✔️ Work (kernel version ≥ 6.0)

<details>
<summary>
<del>Patch: <a href="./patches/kernel/Bluetooth-btusb-Add-a-new-VID-PID-0489-e0e2-for-MT7922.patch">kernel/Bluetooth-btusb-Add-a-new-VID-PID-0489-e0e2-for-MT7922.patch</a></del> (included in kernel 6.0, <a href="https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=57117d7234dadfba2a83615b2a9369f6f2f9914f">source</a>)
</summary>

```diff
diff --git a/drivers/bluetooth/btusb.c b/drivers/bluetooth/btusb.c
index 205b7d3b1cc3a..21135a419bcc3 100644
--- a/drivers/bluetooth/btusb.c
+++ b/drivers/bluetooth/btusb.c
@@ -492,6 +492,9 @@ static const struct usb_device_id blacklist_table[] = {
 	{ USB_DEVICE(0x13d3, 0x3568), .driver_info = BTUSB_MEDIATEK |
 						     BTUSB_WIDEBAND_SPEECH |
 						     BTUSB_VALID_LE_STATES },
+	{ USB_DEVICE(0x0489, 0xe0e2), .driver_info = BTUSB_MEDIATEK |
+						     BTUSB_WIDEBAND_SPEECH |
+						     BTUSB_VALID_LE_STATES },

 	/* Additional Realtek 8723AE Bluetooth devices */
 	{ USB_DEVICE(0x0930, 0x021d), .driver_info = BTUSB_REALTEK },
```

</details>

## Suspend

✔️ Work

### Modern Standby (S0ix, s2idle)

Modern standby should work out-of-box.

### S3 Sleep (deep, not recommended)

S3 sleep may be unstable and cause freezing.

```
options mem_sleep_default=deep
```

<details>
<summary>
Patch: <a href="./patches/dsdt/s3.patch">dsdt/s3.patch</a>
</summary>

```diff
diff --git a/dsdt.dsl b/dsdt.dsl
index 01b8c57..fa83d84 100644
--- a/dsdt.dsl
+++ b/dsdt.dsl
@@ -18,7 +18,7 @@
  *     Compiler ID      "INTL"
  *     Compiler Version 0x20200717 (538969879)
  */
-DefinitionBlock ("", "DSDT", 2, "_ASUS_", "Notebook", 0x01072009)
+DefinitionBlock ("", "DSDT", 2, "_ASUS_", "Notebook", 0x0107200A)
 {
     External (_SB_.ALIB, MethodObj)    // 2 Arguments
     External (_SB_.APTS, MethodObj)    // 1 Arguments
@@ -413,7 +413,7 @@ DefinitionBlock ("", "DSDT", 2, "_ASUS_", "Notebook", 0x01072009)

     Name (SS1, Zero)
     Name (SS2, Zero)
-    Name (SS3, Zero)
+    Name (SS3, One)
     Name (SS4, One)
     Name (IOST, 0xFFFF)
     Name (TOPM, 0x00000000)
@@ -3298,7 +3298,7 @@ DefinitionBlock ("", "DSDT", 2, "_ASUS_", "Notebook", 0x01072009)
         Zero,
         Zero
     })
-    Name (XS3, Package (0x04)
+    Name (_S3, Package (0x04)
     {
         0x03,
         Zero,
```

See also: [DSDT - ArchWiki](https://wiki.archlinux.org/title/DSDT)

</details>

## Fingerprint

❌ Not work ([bug](https://gitlab.freedesktop.org/libfprint/libfprint/-/issues/402))

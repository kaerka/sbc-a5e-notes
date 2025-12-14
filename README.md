# Radxa Cubie A5E – Platform Notes & Findings

This document captures real-world findings from hands-on evaluation of the **Radxa Cubie A5E (Allwinner A523 / sun55i)** using both the **Radxa vendor OS** and **Armbian (community nightly)**.

These notes exist to document **actual subsystem behavior**, limitations, and architectural tradeoffs — not marketing claims.  This is just a set of my own notes for later reference.

---

## TL;DR (Executive Summary)

- **Radxa OS**: Hardware mostly “works”, but the OS is fragile, noisy, and not trustworthy.
- **Armbian**: Clean, stable, and correct — but **missing PCIe, NVMe, CSI, and secondary Ethernet** due to upstream enablement gaps.
- **Wi-Fi works** because it is **SDIO-based**, not PCIe.
- **CSI camera support is completely absent in Armbian** (not broken — not implemented).
- **NVMe works only on Radxa OS**, via vendor BSP hacks.
- **Best use today**: A5E as a **network / controller appliance**, not a camera or storage node.
- **Camera + NVMe projects should use Rock5-class or NIO boards instead**.

---

## Hardware Overview

- **SoC**: Allwinner A523 (sun55i)
- **CPU**: ARM Cortex-A55
- **Memory**: 4 GB
- **Storage**:
  - microSD (boot/root)
  - M.2 2230 NVMe (PCIe, limited support)
- **Networking**:
  - 1× Gigabit Ethernet (working)
  - 1× secondary Ethernet (DT-dependent)
  - Onboard Wi-Fi (AIC, SDIO)
- **Camera**:
  - MIPI CSI (4-lane on paper)

---

## OS Images Evaluated

### 1. Radxa OS (Vendor Image)

**Status**: Functional but deeply flawed.

#### What works
- PCIe link trains
- NVMe enumerates and works
- CSI camera works (via BSP)
- Wi-Fi works
- Ethernet works

#### Major issues
- Extremely noisy and fragile boot
- Old, patched (“dirty”) U-Boot
- Runtime device tree mutation
- Invalid phandles, clock/regulator errors
- PCIe link instability and retries
- CSI works only via BSP (not upstream)
- systemd behavior overridden:
  - `multi-user.target` ignored
  - GUI starts anyway
  - `/etc/fstab` inconsistently honored
- Unnecessary packages (e.g. **CUPS installed by default**)
- General feeling: *boots because errors are ignored*

#### Summary
> Radxa OS is a **vendor BSP survival stack**, not a maintainable Linux system.  
> It is acceptable only when hardware enablement (NVMe, CSI) is required and correctness is secondary.

---

### 2. Armbian (Community Nightly, Debian Trixie)

**Status**: Clean, stable, correct — but incomplete.

Example:
```
CPU:   Allwinner A523 (SUN55I)
Model: Radxa Cubie A5E
DRAM:  4 GiB
sunxi_set_gate: (CLK#35) unhandled
Core:  77 devices, 22 uclasses, devicetree: separate
WDT:   Not starting watchdog@2050000
MMC:   mmc@4020000: 0
Loading Environment from FAT... Unable to read "uboot.env" from mmc0:1...
In:    serial@2500000
Out:   serial@2500000
Err:   serial@2500000
Net:   eth0: ethernet@4500000
starting USB...
sun4i_usb_phy phy@4100400: External vbus detected, not enabling our own vbus
USB EHCI 1.00
USB OHCI 1.0
USB EHCI 1.00
USB OHCI 1.0
Bus usb@4101000: 1 USB Device(s) found
Bus usb@4101400: 1 USB Device(s) found
Welcome to Armbian_community!
```

#### What works
- Clean SPL → TF-A → U-Boot → kernel boot chain
- Stable serial console (115200 8N1)
- SSH
- systemd behaves correctly
- `/etc/fstab` honored
- USB works
- Ethernet (primary) works
- Wi-Fi works (SDIO)

#### What does NOT work (by design)
- **PCIe Root Complex (sun55i)** not enabled
  - `lspci` empty
  - No PCIe controller driver bound
  - No NVMe possible
- **NVMe absent**
- **CSI / MIPI camera stack absent**
  - No `/dev/media*`
  - No CSI platform devices
  - No libcamera graph
- **Secondary Ethernet not present**
- No sun55i PCIe or CSI DT overlays

#### Why this is expected
- sun55i (A523) is very new upstream
- Armbian avoids shipping BSP-only hacks
- PCIe + CSI require significant kernel + DT work
- Armbian prefers correctness over partial enablement

#### Summary
> Armbian is *honest Linux*: stable, predictable, minimal — but missing major subsystems due to upstream gaps.

---

## Wi-Fi Details (Why It Works Without PCIe)

Wi-Fi on the A5E is **NOT PCIe-based**.

- **Chipset**: AIC Semiconductor AIC8800 series
- **Bus**: **SDIO** (Wi-Fi) + UART (Bluetooth)
- **Driver**: `aicbsp` (vendor BSP)
- **SDIO clock**: 150 MHz
- **Expected throughput**: ~200–300 Mbps (realistic)

Example dmesg (with armbian):
```
root@radxa-cubie-a5e:~# dmesg | grep -i sdio
[    1.931740] sdiohal:sdiohal_parse_dt adma_tx:1, adma_rx:1, pwrseq:0, irq type:data, gpio_num:0, blksize:840
[    1.934701] sdiohal:sdiohal_init ok
[    1.973935] mmc1: new high speed SDIO card at address 390b
[    7.942718] aicbsp: aicbsp_sdio_probe:1 vid:0xC8A1  did:0x0082
[    7.943084] aicbsp: aicbsp_sdio_probe:2 vid:0xC8A1  did:0x0182
[    7.943104] aicbsp: aicbsp_sdio_probe after replace:1
[    7.943115] aicbsp: aicbsp_get_feature, set FEATURE_SDIO_CLOCK 150 MHz
[    7.943124] aicbsp: aicwf_sdio_reg_init
[    7.947160] rwnx_load_firmware :firmware path = /lib/firmware/aic8800_fw/SDIO/aic8800D80/fw_patch_table_8800d80_u02.bin
[    8.027713] rwnx_load_firmware :firmware path = /lib/firmware/aic8800_fw/SDIO/aic8800D80/fw_adid_8800d80_u02.bin
[    8.029535] rwnx_load_firmware :firmware path = /lib/firmware/aic8800_fw/SDIO/aic8800D80/fw_patch_8800d80_u02.bin
[    8.054475] rwnx_load_firmware :firmware path = /lib/firmware/aic8800_fw/SDIO/aic8800D80/fw_patch_8800d80_u02_ext0.bin
[    8.168914] rwnx_load_firmware :firmware path = /lib/firmware/aic8800_fw/SDIO/aic8800D80/fmacfw_8800d80_u02.bin
[    8.374062] aicbsp: sdio_err:<aicwf_sdio_bus_pwrctl,1404>: bus down
[    8.438127] aicbsp: aicbsp_get_feature, set FEATURE_SDIO_CLOCK 150 MHz
[    8.438140] aicsdio: aicwf_sdio_reg_init
[    8.444484] aicbsp: aicbsp_get_feature, set FEATURE_SDIO_CLOCK 150 MHz
root@radxa-cubie-a5e:~#
```
### Implications
- Wi-Fi works even with PCIe completely disabled
- Performance is lower than PCIe-based Wi-Fi
- Vendor driver, not upstream
- Acceptable for control traffic, not for high-bandwidth streaming

---

## CSI Camera Status

### Radxa OS
- CSI works via Allwinner BSP
- Vendor camera stack
- Not upstream-compatible
- Not libcamera-friendly

### Armbian
- **CSI does not exist**
- No CSI controller registered
- No media graph
- No sensor binding
- This is a **hard stop** for MIPI camera work today

Example:
```
root@radxa-cubie-a5e:~# dmesg |egrep -i 'csi|enc|video'
[    0.000665] Calibrating delay loop (skipped), value calculated using timer frequency.. 48.00 BogoMIPS (lpj=96000)
[    0.012030] CPU features: detected: Data cache clean to the PoU not required for I/D coherence
[    0.035197] /soc/clock-controller@2001000: Fixed dependency cycle(s) with /soc/rtc@7090000
[    0.035266] /soc/interrupt-controller@3400000: Fixed dependency cycle(s) with /soc/interrupt-controller@3400000
[    0.035525] /soc/clock-controller@7010000: Fixed dependency cycle(s) with /soc/clock-controller@2001000
[    0.035546] /soc/clock-controller@7010000: Fixed dependency cycle(s) with /soc/rtc@7090000
[    0.035643] /soc/rtc@7090000: Fixed dependency cycle(s) with /soc/clock-controller@7010000
[    0.036360] /soc/clock-controller@2001000: Fixed dependency cycle(s) with /soc/rtc@7090000
[    0.040432] /soc/clock-controller@7010000: Fixed dependency cycle(s) with /soc/clock-controller@2001000
[    0.040528] /soc/clock-controller@7010000: Fixed dependency cycle(s) with /soc/rtc@7090000
[    0.041317] /soc/i2c@7081400: Fixed dependency cycle(s) with /soc/pinctrl@7022000/r-i2c-pins
[    0.041662] /soc/clock-controller@7010000: Fixed dependency cycle(s) with /soc/rtc@7090000
[    0.041821] /soc/clock-controller@2001000: Fixed dependency cycle(s) with /soc/rtc@7090000
[    0.041974] /soc/rtc@7090000: Fixed dependency cycle(s) with /soc/clock-controller@7010000
[    0.658874] SCSI subsystem initialized
[    0.659607] videodev: Linux video capture interface: v2.00
[    0.841574] Block layer SCSI generic (bsg) driver version 0.4 loaded (major 242)
[    0.878103] usbcore: registered new interface driver uvcvideo
[    1.638670] Key type encrypted registered
[    5.126735] systemd[1]: systemd 257.9-1~deb13u1 running in system mode (+PAM +AUDIT +SELINUX +APPARMOR +IMA +IPE +SMACK +SECCOMP +GCRYPT -GNUTLS +OPENSSL +ACL +BLKID +CURL +ELFUTILS +FIDO2 +IDN2 -IDN +IPTC +KMOD +LIBCRYPTSETUP +LIBCRYPTSETUP_PLUGINS +LIBFDISK +PCRE2 +PWQUALITY +P11KIT +QRENCODE +TPM2 +BZIP2 +LZ4 +XZ +ZLIB +ZSTD +BPF_FRAMEWORK +BTF -XKBCOMMON -UTMP +SYSVINIT +LIBARCHIVE)
[    6.678919] systemd[1]: Listening on systemd-creds.socket - Credential Encryption/Decryption.
[   25.567821] sun55i-gmac200 4510000.ethernet: deferred probe timeout, ignoring dependency
[   25.569114] panfrost 1800000.gpu: deferred probe timeout, ignoring dependency
root@radxa-cubie-a5e:~#
```

---

## NVMe / PCIe Status

### Radxa OS
- PCIe RC enabled via BSP
- NVMe works
- Link instability and retries observed
- Not upstreamable

### Armbian
- PCI core enabled
- **No sun55i PCIe Root Complex driver**
- No platform device
- No DT node
- NVMe cannot work without upstream PCIe RC support

Reference:
- Armbian forum: NVMe on A5E not supported (still accurate)

Example:
```
root@radxa-cubie-a5e:~# lsblk
NAME        MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
mmcblk0     179:0    0 14.9G  0 disk
├─mmcblk0p1 179:1    0  512M  0 part /boot
└─mmcblk0p2 179:2    0 14.2G  0 part /var/log.hdd
                                     /
zram0       254:0    0  1.9G  0 disk [SWAP]
zram1       254:1    0   50M  0 disk /var/log
zram2       254:2    0    0B  0 disk
root@radxa-cubie-a5e:~# df -h
Filesystem      Size  Used Avail Use% Mounted on
udev            1.9G     0  1.9G   0% /dev
tmpfs           393M  5.6M  387M   2% /run
/dev/mmcblk0p2   14G  1.6G   13G  11% /
tmpfs           2.0G     0  2.0G   0% /dev/shm
tmpfs           5.0M     0  5.0M   0% /run/lock
tmpfs           1.0M     0  1.0M   0% /run/credentials/systemd-resolved.service
tmpfs           1.0M     0  1.0M   0% /run/credentials/systemd-networkd.service
tmpfs           2.0G  4.0K  2.0G   1% /tmp
/dev/mmcblk0p1  511M   65M  447M  13% /boot
/dev/zram1       47M  1.2M   43M   3% /var/log
tmpfs           1.0M     0  1.0M   0% /run/credentials/systemd-journald.service
tmpfs           1.0M     0  1.0M   0% /run/credentials/getty@tty1.service
tmpfs           393M  4.0K  393M   1% /run/user/1000
tmpfs           1.0M     0  1.0M   0% /run/credentials/serial-getty@ttyS0.service
tmpfs           393M  4.0K  393M   1% /run/user/0
root@radxa-cubie-a5e:~# lspci
root@radxa-cubie-a5e:~#
root@radxa-cubie-a5e:~#
root@radxa-cubie-a5e:~# dmesg |egrep -i 'sunxi|pcie'
[    0.879258] sunxi-wdt 2050000.watchdog: Watchdog enabled (timeout=16 sec, nowayout=0)
[    1.657238] sun55i-a523-r-pinctrl 7022000.pinctrl: initialized sunXi PIO driver
[    1.696671] sun55i-a523-pinctrl 2000000.pinctrl: initialized sunXi PIO driver
[    1.928641] sunxi-mmc 4020000.mmc: Got CD GPIO
[    1.954085] sunxi-mmc 4021000.mmc: initialized, max. request size: 2048 KB, uses new timings mode
[    1.954434] sunxi-mmc 4020000.mmc: initialized, max. request size: 2048 KB, uses new timings mode
root@radxa-cubie-a5e:~#
```

---

## Practical Conclusions

### What this board is good for (today)
- Network / controller appliance
- SSH / orchestration node (k3s, control plane)
- Spectra display controller
- Experimentation with:
  - kernel builds
  - DT work
  - Armbian contributions

### What this board is NOT good for (today)
- CSI / MIPI camera projects (on Armbian)
- NVMe-heavy workloads (on Armbian)
- High-performance Wi-Fi
- Storage + camera combined roles

---

## Recommended Architecture

- **Use A5E**:
  - As a controller
  - Possibly with Radxa OS if NVMe is required
  - Treat OS as *appliance firmware*
- **Use Rock5 / NIO boards**:
  - For CSI cameras
  - For NVMe storage
  - For upstream-friendly work
- **Standardize on Armbian everywhere possible**
- Accept Radxa OS only where hardware enablement forces it

---

## Final Verdict

> The Radxa Cubie A5E is not a bad board — but it is an **early-upstream board**.
> I've hit a nearly impossible wall with this A5E... one I could probably solve, but it would be weeks or months of work, and it just isn't worth it.  
> So while it has nearly perfect hardware to become a camera controller, it will get relegated to a lesser role for now.
Hardware:
1. 8 core arm64 processor
2. 4gb ram
3. sdcard + NVMe SSD
4. built-in wifi
5. CSI MIPI interface (camera high speed interface) - but no support for the Radxa 4k camera yet
6. Built in 4k video decoder/encoder
7. Small size and runs on 5volt 3amp
8. Working serial console access - with caveats (doesn't have same chipset as other radxa boards, slower console)
9. Dual 1GbE ethernet, and one supports PoE (power over ethernet)

> So... in short, a cool board, but nearly useless for the primary functions I'd use it for.  So now, it probably will live on as one of these:
1. Spectra controller, using the radxa messy image with modifications (especially to remove much of the extra junk services)
2. Maybe a network routing device, using the two ethernet interfaces + wifi
3. Possibly as a GPIO controller of some sort, though noting it has a weird gpio chipset

> This wasn't meant to be a review of this board... but as I can't use it right now for it's primary function, well, that's what it's become.

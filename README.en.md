# OnePlus 6 Resurrection: From Brick to Debian Server

> **OnePlus 6 (enchilada) · OxygenOS 11.1.2.2 · Mobian 13.0 (Debian) · 2026-10-03**

[中文版 / Chinese version](README.md)

How a bricked OnePlus 6 went from "completely unbootable" to "a compact server running Debian" in a single session.

This document records the full process, **every trap I fell into**, and the steps that actually solved the problem.

---

## TL;DR

```bash
# Three things you must remember

# 1. EDL (Firehose) writes to sectors that already hold data SILENTLY FAIL
#    You must erase first.
edl e tz_a && edl w tz_a tz.img     # ✅ Correct
edl w tz_a tz.img                    # ❌ Reports success, data unchanged

# 2. Expand sparse rootfs images with the official simg2img - do not write your own parser
simg2img rootfs.img rootfs.raw

# 3. ⭐ ERASE the dtbo partition before booting Linux - this is the decisive step
fastboot erase dtbo
```

---

## Table of Contents

- [1. Device Information](#1-device-information)
- [2. Starting Point: A Brick](#2-starting-point-a-brick)
- [3. Five Key Discoveries](#3-five-key-discoveries)
- [4. Complete Reproducible Procedure](#4-complete-reproducible-procedure)
- [5. Diagnostic Methodology](#5-diagnostic-methodology)
- [6. Tools and Files](#6-tools-and-files)
- [7. Timeline and Effort](#7-timeline-and-effort)
- [8. Lessons Learned](#8-lessons-learned)
- [9. TWRP Rescue Procedures](#9-twrp-rescue-procedures) ← **Look here when things break**
  - [9.1 Do This Before You Flash Anything](#91-do-this-before-you-flash-anything)
  - [9.2 Why TWRP Is the Strongest Diagnostic Tool](#92-why-twrp-is-the-strongest-diagnostic-tool)
  - [9.3 Switching to TWRP (Rescue)](#93-switching-to-twrp-rescue)
  - [9.4 Scenario A: System Won't Boot - Diagnose First](#94-scenario-a-system-wont-boot---diagnose-first)
  - [9.5 Scenario B: Corrupted rootfs - Rewrite It](#95-scenario-b-corrupted-rootfs---rewrite-it)
  - [9.6 Scenario C: Going Back from Debian to Android](#96-scenario-c-going-back-from-debian-to-android)
  - [9.7 Scenario D: Completely Bricked - Use EDL](#97-scenario-d-completely-bricked---use-edl)
  - [9.8 TWRP Gotchas (Traps I Actually Hit)](#98-twrp-gotchas-traps-i-actually-hit)
  - [9.9 Rescue Command Reference](#99-rescue-command-reference)
  - [9.10 Rescue Decision Tree](#910-rescue-decision-tree)
- [Appendix A: Command Cheat Sheet](#appendix-a-command-cheat-sheet)
- [Appendix B: References](#appendix-b-references)

---

## 1. Device Information

| Item | Value |
|---|---|
| Model | OnePlus 6 (A6003) |
| Codename | `oneplus-enchilada` |
| SoC | Qualcomm Snapdragon 845 (SDM845) |
| Storage | 128 GB UFS 2.1 (SAMSUNG KLUDG4U1EA-B0C1) |
| Serial | `085ba2e7` |
| Chip serial | `0xc650672c` (3327158060) |
| HWID | `0x0008b0e10051459b` |
| PK_HASH | `dd7c5f2e53176bee91747b53900ccec33dd30fa4ded0dfe9baf9156e6910862f` |
| Stock firmware | OxygenOS 11.1.2.2 (Android 11, SDK 30) |
| Final system | **Mobian 13.0 (Debian stable, Phosh)** |

### Why HWID and PK_HASH Matter

**The EDL Firehose loader must match both the device's HWID and PK_HASH.** Generic SDM845 loaders fail silently:

```
✅ Correct: oneplus/0008b0e10051459b_dd7c5f2e53176bee_fhprg_op6t.bin  (680,769 B)
❌ Generic: prog_firehose_ddr.elf etc. - no error, but does nothing
```

---

## 2. Starting Point: A Brick

### Symptoms

- Device enters EDL (`05c6:9008`) and fastboot (`18d1:d00d`)
- Shows the OnePlus logo for ~75 seconds, then falls back to fastboot
- The `boot_a` partition's first 64 bytes are **all zeros** - no kernel
- Slot state: `slot-unbootable:a: yes`, `slot-retry-count:a: 0`

### Approaches That Had Already Failed

| Method | Result |
|---|---|
| `fastboot flash boot_a boot.img` | Reported `OKAY [0.254s]`, but `boot_a` remained zeroed |
| `fastboot flash system_a system.img` | Reported `OKAY [83.807s]`, but wrote nothing |
| `edl qfil` on 24 partitions | All reported 100%, but **not one actually wrote** |
| `edl w` single partition | Reported success, data unchanged |
| MSM Download Tool | Crashed on Windows 11 (a 32-bit app from 2018) |
| WSL 2 + usbipd | Cannot bypass the Windows driver signing problem |

**The common thread: every tool reported success, and none of them actually wrote data.**

---

## 3. Five Key Discoveries

### 3.1 ⭐ Firehose Silent Write Failure - The Biggest Trap

**This is the single most important discovery and it explains every earlier failure.**

#### The Symptom

Issuing a Firehose `program` command against a sector that **already holds data** returns an ACK, but **the content does not change**.

#### Experimental Evidence

```bash
# Step 1: read the current contents of tz_a
edl r tz_a /tmp/tz_before.img
# -> b2d1236c... (has data)

# Step 2: write the new content directly
edl w tz_a tz.img
# -> "Wrote tz.img to sector 134"  ✅ reports success

# Step 3: read back and verify
edl r tz_a /tmp/tz_after.img
# -> b2d1236c... ❌ IDENTICAL to before! Nothing changed

# Step 4: erase first, then write
edl e tz_a        # -> "Erased tz_a starting at sector 134"
edl w tz_a tz.img # -> "Wrote tz.img to sector 134"

# Step 5: read back and verify
edl r tz_a /tmp/tz_final.img
# -> 100% byte-identical to tz.img ✅
```

#### Conclusion

> **In EDL/Firehose mode you must `erase` before you `write`.**
> Writing directly onto a sector that already holds data fails silently - it returns success without writing.

#### Why This Misleads You

Every tool in the chain (`fastboot`, `edl w`, `edl qfil`, MSM Download Tool) will:
1. Report `OKAY` / `100%` / `Wrote ... to sector N`
2. Actually do nothing

**The only reliable verification is to read back and compare bytes.**

---

### 3.2 Firmware Must Match the System Version

All firmware partitions (`tz`, `xbl`, `abl`, `hyp`, `modem`, etc.) **must come from the same OTA release**.

#### Symptom

With mismatched firmware, ABL (the Android Bootloader) starts, fastboot works, but **the kernel cannot complete hardware initialisation** → logo appears, then a crash.

#### The Correct Firmware Source

```
OxygenOS 11.1.2.2 OTA
  OnePlus6Oxygen_22.J.62_GLO_0620_2111252336
  Download: https://otafsg1.h2os.com/patch/amazone2/GLO/OnePlus6Oxygen/
            OnePlus6Oxygen_22.J.62_GLO_0620_2111252336/
            OnePlus6Oxygen_22.J.62_OTA_0620_all_2111252336_14afec75dd6fa.zip
```

#### The 18 Firmware Partitions to Flash

```
abl  aop  bluetooth  cmnlib  cmnlib64  devcfg  dsp  fw_4j1ed
fw_4u1ea  hyp  keymaster  LOGO  oem_stanvbk  qupfw  storsec
tz  xbl  xbl_config
modem  (requires --slot=all)
```

#### Verification

```
16/16 firmware partitions verified byte-exact:
  abl_a            MATCH
  aop_a            MATCH
  bluetooth_a      MATCH
  cmnlib64_a       MATCH
  cmnlib_a         MATCH
  devcfg_a         MATCH
  fw_4j1ed_a       MATCH
  fw_4u1ea_a       MATCH
  hyp_a            MATCH
  keymaster_a      MATCH
  oem_stanvbk      MATCH
  qupfw_a          MATCH
  storsec_a        MATCH
  tz_a             MATCH
  xbl_a            MATCH
  xbl_config_a     MATCH
```

---

### 3.3 Hand-Written Sparse Parsers Are Unreliable

Mobian's `rootfs.img` is in Android sparse format (magic `ED26FF3A`).

#### The Mistake I Made

I wrote a Python parser to expand the sparse image:

```python
# ❌ The buggy version
elif ctype == CHUNK_FILL:
    fill = f.read(4)
    o.write(fill * (csz * blk_sz))
    ...
```

**Symptoms:**
```
File size: 11,331,903,488 bytes   ← should be 5,998,936,064
e2fsck: "Superblock has an invalid journal (inode 8)"
        "The journal superblock is corrupt"
```

The expanded image had a **corrupted filesystem** and could not be mounted.

#### The Correct Approach

```bash
sudo apt install android-sdk-libsparse-utils
simg2img rootfs.img rootfs.raw
```

**Result:**
```
Size: 5,998,936,064 bytes = 1,464,584 blocks  ✅ exactly as expected
e2fsck: Passes 1-5 all clean, zero errors
```

#### Lesson

> **Do not implement parsers for standard formats yourself.** Use the official tool, then **verify the result with `e2fsck`**.

---

### 3.4 Boot Image Header Addresses Must Match the Device

Mobian's boot image uses postmarketOS generic addresses:

| Field | Mobian original | OnePlus stock / TWRP |
|---|---|---|
| `kernel_addr` | `0x10008000` | **`0x8000`** |
| `ramdisk_addr` | `0x11000000` | **`0x1000000`** |
| `tags_addr` | `0x10000100` | **`0x100`** |
| `header_version` | `0` | **`1`** |

#### Repacking

Repack Mobian's kernel + ramdisk using the stock header layout, preserving Mobian's cmdline:

```python
# Key fields
hdr[12] = 0x8000        # kernel_addr   (stock value)
hdr[20] = 0x1000000     # ramdisk_addr  (stock value)
hdr[32] = 0x100         # tags_addr     (stock value)
hdr[40] = 1             # header_version
hdr[44] = 0x1600015b    # os_version (taken from stock, avoids anti-rollback rejection)
hdr[64:576] = mobian_cmdline   # keeps mobile.root=UUID=...
```

#### cmdline (Must Be Preserved)

```
mobile.root=UUID=6cad760a-7957-4d99-896d-7adf1845e68b
mobile.qcomsoc=qcom/sdm845
mobile.vendor=oneplus
mobile.model=enchilada
console=ttyMSM0,115200
init=/sbin/init
ro loglevel=7 splash
```

---

### 3.5 ⭐⭐ `fastboot erase dtbo` - The Decisive Step

**This is the step that finally made the device boot.**

#### The Wrong Path (The Detour I Took)

My reasoning was: Mobian's boot image is `kernel (11.4 MB) + ramdisk (18.4 MB) = 29.8 MB`, which is the entire file, so **there is no room for a device tree**, therefore the DTB must come from the `dtbo` partition.

So I:
1. Extracted the mainline DTB from the rootfs (`sdm845-oneplus-enchilada.dtb`, 118,165 bytes)
2. Analysed the stock dtbo format (**discovering it uses big-endian, not standard AOSP little-endian**)
3. Built a correct dtbo image and flashed it into `dtbo_a`

**Result: still failed, back to fastboot in 5 seconds.**

#### The Correct Answer (Official Documentation)

From the [postmarketOS/Nura wiki](https://wiki.nura.eco/wiki/OnePlus_6_(oneplus-enchilada)):

> ### Erase DTBO
>
> **The dtbo partition includes information that will cause the bootloader
> to break the Nura image, so it must be erased before flashing.**
>
> ```
> $ fastboot erase dtbo
> ```

**The dtbo partition must be ERASED, not filled with the right content.**

#### Why

The device tree **is already appended to the kernel image**:

```
vmlinuz-6.12-sdm845          11,239,833 bytes
boot.img kernel_size         11,357,998 bytes
Difference                      118,165 bytes  ← exactly the DTB size!
```

**The kernel carries its own device tree.** A bootloader-provided dtbo **overrides or interferes** with it, breaking the boot.

#### Execution

```bash
fastboot erase dtbo_a
fastboot erase dtbo_b
fastboot erase dtbo      # generic name (acts on the current slot)
```

#### Result

```
[17:25:56] Linux 6.12-sdm845 kernel starts
           USB gadget registers: 0525:a4a2 Netchip Linux-USB Ethernet/RNDIS Gadget
           *** KERNEL IS RUNNING ***
```

**From that second on, the device was running Linux.**

---

## 4. Complete Reproducible Procedure

### Prerequisites

- OnePlus 6 with an unlocked bootloader
- A Linux host (Ubuntu Server used here)
- A data cable (a USB 2.0 port is more reliable)

### Step 1: Prepare Tools

```bash
# EDL tool
git clone https://github.com/bkerler/edl
pip install -r edl/requirements.txt

# Official sparse tools
sudo apt install android-sdk-libsparse-utils android-tools-adb android-tools-fastboot

# Note: distro-packaged fastboot can be flaky; official platform-tools are recommended
```

### Step 2: Get the Matching Firehose Loader

```bash
mkdir -p ~/Loaders/oneplus
# Must match HWID + PK_HASH
# Filename format: <HWID>_<first 24 chars of PK_HASH>_fhprg_<device>.bin
# This case: 0008b0e10051459b_dd7c5f2e53176bee_fhprg_op6t.bin
```

**How to obtain it**: read HWID/PK_HASH from the device in EDL mode, then match against a loader collection.

### Step 3: Download and Extract Official Firmware

```bash
# Download the OOS 11.1.2.2 OTA (~2 GB)
curl -O 'https://otafsg1.h2os.com/patch/amazone2/GLO/OnePlus6Oxygen/OnePlus6Oxygen_22.J.62_GLO_0620_2111252336/OnePlus6Oxygen_22.J.62_OTA_0620_all_2111252336_14afec75dd6fa.zip'

# Extract payload.bin
unzip -o *.zip payload.bin

# Extract partitions with payload_dumper
pip install payload_dumper
payload_dumper payload.bin
```

What you need:
```
Firmware (18): abl aop bluetooth cmnlib cmnlib64 devcfg dsp fw_4j1ed
               fw_4u1ea hyp keymaster LOGO oem_stanvbk qupfw storsec
               tz xbl xbl_config modem
System:        boot system vendor dtbo vbmeta
```

### Step 4: Enter EDL Mode

```
1. Unplug the USB cable
2. Hold Power for 20 seconds (full shutdown)
3. Hold Volume Up + Volume Down
4. Plug the USB cable in
5. Verify: lsusb | grep 05c6:9008
```

### Step 5: Flash Firmware (Erase + Write)

```bash
LOADER=~/Loaders/oneplus/0008b0e10051459b_dd7c5f2e53176bee_fhprg_op6t.bin

for P in xbl xbl_config aop tz hyp bluetooth abl keymaster \
         cmnlib cmnlib64 devcfg qupfw storsec fw_4j1ed fw_4u1ea oem_stanvbk modem; do
    echo "=== ${P}_a ==="
    edl --loader=$LOADER e ${P}_a          # ← MUST erase first!
    edl --loader=$LOADER w ${P}_a ${P}.img
done
```

**⚠️ Note**: the `edl` CLI accepts only **one operation per invocation**. `edl w a b c` just prints Usage.

### Step 6: Prepare the Mobian Images

```bash
# Download Mobian 13.0 (Debian stable)
curl -O https://images.mobian.org/qcom/mobian-sdm845-phosh-13.0.tar.xz

# Extract (nested: tar.xz -> tar -> files)
7z x mobian-sdm845-phosh-13.0.tar.xz
7z x mobian-sdm845-phosh-13.0.tar
# Yields:
#   mobian-sdm845-phosh-20251002.boot-enchilada.img   (29,798,400 B)
#   mobian-sdm845-phosh-20251002.rootfs.img           (4,221,440,856 B, sparse)
```

### Step 7: Expand the Sparse rootfs (Official Tool)

```bash
simg2img mobian-sdm845-phosh-20251002.rootfs.img rootfs.raw

# Verify (important!)
e2fsck -fn rootfs.raw
# Should print: Passes 1-5 all clean

# Confirm the UUID matches
dd if=rootfs.raw bs=1 skip=1024 count=1024 2>/dev/null | xxd -s 104 -l 16 -p
# Should print: 6cad760a79574d99896d7adf1845e68b
```

### Step 8: Repack the Boot Image

Mobian's boot image header uses postmarketOS generic addresses and must be changed to OnePlus values.

**Method**: take the stock boot.img header layout and apply Mobian's kernel + ramdisk + cmdline.

Key fields:
```
kernel_addr    = 0x8000
ramdisk_addr   = 0x1000000
tags_addr      = 0x100
page_size      = 4096
header_version = 1
os_version     = 0x1600015b    (from stock, avoids anti-rollback rejection)
cmdline        = <Mobian's, must contain mobile.root=UUID=...>
id             = SHA1(kernel + kernel_size + ramdisk + ramdisk_size + ...)
```

### Step 9: Flash and Boot ⭐

```bash
# 1. Flash the Linux kernel
fastboot flash boot_a mobian-repacked.img

# 2. Put TWRP in slot B as a rescue fallback
fastboot flash boot_b twrp-3.7.0_11-0-enchilada.img

# 3. Write the rootfs into userdata
fastboot flash userdata rootfs.raw

# 4. ⭐⭐⭐ ERASE dtbo - the decisive step
fastboot erase dtbo_a
fastboot erase dtbo_b

# 5. Disable AVB verification (required for a third-party OS)
fastboot --disable-verity --disable-verification flash vbmeta_a vbmeta.img
fastboot --disable-verity --disable-verification flash vbmeta_b vbmeta.img

# 6. Activate slot A and boot
fastboot --set-active=a
fastboot reboot
```

### Step 10: Log In

The first boot takes 1-2 minutes (you may see a black screen or the logo).

**Default credentials:**

| Item | Value |
|---|---|
| Username | `mobian` |
| Password | `1234` |

> ⚠️ **Change the password immediately**: `passwd`

### Step 11: USB Network Access (Recommended)

Mobian provides an RNDIS/EEM network over USB:

```bash
# Host side
IF=$(ip -brief link | awk '/enp4s0f3u1u2|usb0/ {print $1; exit}')
sudo ip link set $IF up
sudo ip addr add 172.16.42.2/24 dev $IF
ssh mobian@172.16.42.1
```

---

## 5. Diagnostic Methodology

The techniques that actually helped, ranked by value.

### 5.1 Always Verify Write Results

```bash
# After writing, always read back and compare
edl --loader=$LOADER w tz_a tz.img
edl --loader=$LOADER r tz_a /tmp/verify.img
cmp tz.img /tmp/verify.img && echo "OK" || echo "WRITE FAILED"
```

**No tool's success report can be trusted.** Only byte comparison can.

### 5.2 Test the Hardware with an Independent Kernel

**Use TWRP to separate "hardware failure" from "software problem":**

```bash
fastboot flash boot_a twrp.img
fastboot reboot
# If TWRP boots  -> hardware, boot chain and the boot partition are all fine
# If TWRP fails  -> the problem is at the hardware level
```

**In this case that step was the turning point**: TWRP booted successfully, proving the hardware was sound and redirecting the investigation from "hardware" to "image configuration".

### 5.3 Read Partition Contents to Establish Ground Truth

```bash
# e.g. confirm boot_a really holds a kernel (rather than being all zeros)
edl --loader=$LOADER rs 51334 8 /tmp/boot_head.bin --lun=4
xxd /tmp/boot_head.bin
# Should show: 414e4452 4f494421 = "ANDROID!"
```

**⚠️ Watch the LUN**: `boot_a` lives on LUN 4. When reading raw sectors with `rs` you must pass `--lun=4`, otherwise you read unrelated data from LUN 0.

### 5.4 Inspect Filesystems from a Recovery Environment

Once TWRP is running:

```bash
adb shell 'mount -t ext4 -o ro /dev/block/bootdevice/by-name/system_a /mnt/systest'
adb shell 'ls /mnt/systest'
adb shell 'cat /mnt/systest/build.prop | grep ro.build.version'
```

### 5.5 Use the Slot Retry Counter to Tell Whether a Boot Was Even Attempted

```bash
fastboot getvar slot-retry-count:a
```

- **7 → 6** (decrements by one) → the bootloader tried once and gave up (usually a verification failure)
- **7 → 0** (drains to zero) → the bootloader retried until exhausted (usually the kernel started then crashed)

This distinction separates "image rejected" from "kernel crashed".

---

## 6. Tools and Files

### Required Tools

| Tool | Purpose | Source |
|---|---|---|
| `edl` | EDL/Firehose flashing | [bkerler/edl](https://github.com/bkerler/edl) |
| `simg2img` | Expand Android sparse images | `apt install android-sdk-libsparse-utils` |
| `payload_dumper` | Extract partitions from an OTA | [vm03/payload_dumper](https://github.com/vm03/payload_dumper) |
| `fastboot` | fastboot flashing | `apt install android-tools-fastboot` |
| `adb` | Connect to TWRP / the system | `apt install android-tools-adb` |
| `e2fsck` | Verify ext4 images | `apt install e2fsprogs` |
| `7z` | Extract nested archives | `apt install p7zip-full` |

### Key Files

| File | Size | Purpose |
|---|---|---|
| `..._fhprg_op6t.bin` | 680,769 B | Firehose loader (HWID + PK_HASH matched) |
| `boot.img` (stock) | 67,108,864 B | Header layout reference |
| `mobian-...boot-enchilada.img` | 29,798,400 B | Mobian kernel |
| `rootfs.raw` | 5,998,936,064 B | Expanded Debian rootfs |
| `twrp-3.7.0_11-0-enchilada.img` | 67,108,864 B | Recovery environment |

### Key Values

```
rootfs ext4 UUID:   6cad760a-7957-4d99-896d-7adf1845e68b
Stock boot header:  kernel_addr=0x8000  ramdisk_addr=0x1000000  tags_addr=0x100
dtbo table endian:  big-endian
dtbo magic:         0xD7B7AB1E (stored as bytes d7 b7 ab 1e)
userdata partition: 118,112,366,592 B (110 GB)
```

---

## 7. Timeline and Effort

| Phase | Time | Notes |
|---|---|---|
| Trial and error | Hours | Repeated fastboot/EDL flashes, all "successful", none effective |
| **Discovering the erase problem** | - | The turning point |
| Re-flashing all firmware | ~3 min | 18 partitions, erase + write + verify |
| Extracting and transferring system images | ~10 min | 4 GB |
| rootfs expansion (detour) | ~30 min | Hand-written parser → corruption → restart |
| TWRP rescue verification | ~5 min | Proved the hardware was fine |
| Downloading Mobian | ~4 min | 1.31 GB |
| Boot image repack | ~2 min | Header address correction |
| dtbo detour | ~40 min | Built a correct dtbo, but the answer was "erase it" |
| **Boot after erasing dtbo** | **~10 s** | ⭐ Success |

**Total: about one day (including extensive trial and error)**

**Following this document from scratch: about 1-2 hours.**

---

## 8. Lessons Learned

### 8.1 The Most Important Rule

> **Never trust a "success" return value. Read back and compare after every write.**

In this session almost every tool lied:

| Tool | Claimed | Reality |
|---|---|---|
| `fastboot flash boot_a` | `OKAY [0.254s]` | Partition still all zeros |
| `edl w` | `Wrote ... to sector N` | Data unchanged |
| `edl qfil` | 24 partitions at `100%` | Not one byte written |
| MSM Download Tool | `Success` | Also wrote nothing |

**Only `cmp` / MD5 comparison never lies.**

### 8.2 Separate "Hardware Failure" from "Software Configuration"

Testing with an **independent kernel** (TWRP) is the key technique. Do not spend time on software configuration before the hardware is confirmed sound.

### 8.3 Read the Official Documentation

I took a long detour on `dtbo` (analysing the format, building an image, fixing the byte order), while the official wiki answered it in one sentence: **erase it**.

**Before reverse-engineering deeply, search for the official installation guide.**

### 8.4 Do Not Implement Standard Format Parsers Yourself

My hand-written sparse parser corrupted the filesystem and cost about 30 minutes. The official `simg2img` solved it in one command.

**If you must write your own, verify the output with `e2fsck`.**

### 8.5 Always Keep a Rescue Path

**Always keep TWRP in the other slot:**

```bash
fastboot flash boot_b twrp.img      # Slot B as rescue
fastboot flash boot_a linux.img     # Slot A as the main system
```

When things break, `fastboot --set-active=b` returns you to a recovery environment.

### 8.6 Understand A/B Slots

| Variable | Meaning |
|---|---|
| `current-slot` | The active slot |
| `slot-unbootable:X` | Slot marked unbootable |
| `slot-retry-count:X` | Remaining retries (7 = full) |
| `slot-successful:X` | Whether the slot has booted successfully |

```bash
fastboot --set-active=a    # Switch slots and reset the retry counter
```

### 8.7 Specific Technical Traps

| Trap | Correct approach |
|---|---|
| `edl` CLI takes one operation per call | `edl w a b c` just prints Usage |
| Partition names have no leading slash and include the slot suffix | `tz_a` ✅  `/tz_a` ❌ |
| `edl --debugmode` needs `/logs` to exist | `mkdir /logs` first |
| Sahara HELLO_RESP must be 48 bytes | A 16-byte response is rejected |
| Sahara DONE must carry length 8 | Length 0 is ignored |
| `rs` needs an explicit LUN | `boot_a` is on LUN 4 |
| TWRP's `/tmp` is a 3.7 GB tmpfs | Large files will not fit |
| `adb shell dd` swallows stdin | Use `sudo -n` so the password does not consume it |
| OnePlus devices may not support `fastboot boot` | `flash` to the partition instead |
| Non-ASCII `.ps1` files fail to parse | Keep scripts pure ASCII |

### 8.8 Choosing Between fastboot and EDL

| Scenario | Recommended | Reason |
|---|---|---|
| Non-critical partitions (boot/system/vendor) | `fastboot flash` | Simple, verified working here |
| Critical partitions (abl/xbl/tz) | EDL + erase | fastboot refuses them |
| Recovering a bricked device | EDL | The only available channel |
| Writing large images (rootfs) | `fastboot flash` | Native bulk transfer, more reliable than adb stdin |

---

## 9. TWRP Rescue Procedures

> **This is the second most important section in this document** (after `fastboot erase dtbo`).
>
> Having TWRP pre-staged in `boot_b` is exactly what let me prove the **hardware was fine**
> when the device appeared thoroughly bricked, which redirected the investigation from
> "hardware failure" to "image configuration" and ultimately led to success.

### 9.1 Do This Before You Flash Anything

**Before touching the main system, put TWRP in the other slot.**

```bash
# TWRP for enchilada 3.7.0_11-0 (matches Android 11 firmware)
# https://dl.twrp.me/enchilada/twrp-3.7.0_11-0-enchilada.img.html

# Slot A for the main system, slot B for TWRP
fastboot flash boot_a  <your system kernel>.img
fastboot flash boot_b  twrp-3.7.0_11-0-enchilada.img

# Or the other way round - the point is to always have one usable recovery slot
```

**Cost:** zero (`boot_b` was empty anyway)
**Benefit:** one command returns you to a recovery environment whenever anything breaks

### 9.2 Why TWRP Is the Strongest Diagnostic Tool

TWRP is not merely a "recovery mode" - it is a **complete, usable Linux system with an ADB shell**:

| Capability | Use |
|---|---|
| **Mount any partition** | Verify that system/vendor really are valid filesystems |
| **Read/write block devices directly** | Read partitions back to confirm writes actually landed |
| **ADB shell** | A full command-line environment (`dd`, `tar`, `busybox`) |
| **Flash .img files** | Write any partition from the device itself, no host needed |
| **Format / wipe** | Factory reset, format userdata as ext4/f2fs |
| **Backup / restore** | Back partitions up to external storage |

### 9.3 Switching to TWRP (Rescue)

```bash
# 1. Confirm the device is in fastboot
fastboot devices

# 2. Switch to slot B (where TWRP lives)
fastboot --set-active=b
# -> Setting current slot to 'b'   OKAY

# 3. Boot
fastboot reboot
```

**Wait roughly 60-90 seconds**, then:

```bash
# 4. Confirm TWRP is up
lsusb | grep 2a70
# -> Bus 001 Device 064: ID 2a70:9012 OnePlus Technology (Shenzhen) Co., Ltd. ONEPLUS A6003

adb devices
# -> 085ba2e7    recovery
```

**If TWRP does not come up**, TWRP itself is incomplete - go to [9.7 Completely bricked → EDL](#97-scenario-d-completely-bricked---use-edl).

### 9.4 Scenario A: System Won't Boot - Diagnose First

**Do not rush to re-flash. Use TWRP to find out which link in the chain is broken.**

#### Check 1: Does the kernel partition actually hold a kernel?

```bash
adb shell 'dd if=/dev/block/bootdevice/by-name/boot_a bs=1 count=8 2>/dev/null | hexdump -C'
# Should show: 41 4e 44 52 4f 49 44 21   = "ANDROID!"
# If it is all 00 -> the kernel partition is empty (a silent write failure)
```

#### Check 2: Are system / vendor valid filesystems?

```bash
adb shell 'mkdir -p /mnt/systest'
adb shell 'mount -t ext4 -o ro /dev/block/bootdevice/by-name/system_a /mnt/systest; echo rc=$?'
# rc=0  -> mounted, the filesystem is valid
# rc!=0 -> the partition is corrupt or was never written

adb shell 'ls /mnt/systest'
adb shell 'cat /mnt/systest/build.prop | grep -E "ro.build.version"'
# -> ro.build.version.release=11
# -> ro.build.version.sdk=30
```

#### Check 3: Is the rootfs UUID correct?

```bash
adb shell 'dd if=/dev/block/bootdevice/by-name/userdata bs=1 skip=1024 count=1024 2>/dev/null' > /tmp/sb.bin
xxd -s 104 -l 16 -p /tmp/sb.bin
# Must match mobile.root=UUID=... in the kernel cmdline
```

#### Check 4: Did the bootloader actually attempt to boot?

```bash
fastboot getvar slot-retry-count:a
```

| Change | Meaning |
|---|---|
| **7 → 6** (drops by one) | The bootloader tried once and gave up → **image rejected** (verification/format issue) |
| **7 → 0** (drains) | The bootloader retried until exhausted → **the kernel started then crashed** |
| **Unchanged (still 7)** | The slot was never attempted → slot state problem |

**This distinction determines the investigation direction:**

- "Drops by one" → check the boot image header, AVB verification, dtbo
- "Drains to zero" → the kernel is running; check rootfs / drivers / initramfs

### 9.5 Scenario B: Corrupted rootfs - Rewrite It

**Symptoms:** the kernel runs (a USB gadget appears) but the system never comes up and the screen stalls early.

```bash
# Method 1: write directly with fastboot (fastest, recommended)
# Note: the device must be in fastboot, not TWRP
fastboot flash userdata rootfs.raw

# Method 2: stream it in from TWRP using ADB
# ⚠️ sudo swallows stdin, so validate credentials with sudo -n first
echo "$PW" | sudo -S -v          # validate sudo
sudo -n adb shell 'umount /data; umount /sdcard; sync
                   dd of=/dev/block/bootdevice/by-name/userdata bs=4M conv=fsync' < rootfs.raw

# Method 3: TWRP graphical interface
# Install -> Install Image -> select rootfs.img -> select the userdata partition
```

**Always verify after writing:**

```bash
adb shell 'dd if=/dev/block/bootdevice/by-name/userdata bs=1 skip=1024 count=1024 2>/dev/null' > /tmp/sb.bin
xxd -s 56 -l 2 /tmp/sb.bin      # -> 53ef  (ext4 magic)
xxd -s 104 -l 16 -p /tmp/sb.bin # -> UUID, must match the cmdline
```

**⚠️ TWRP's `/tmp` is only a 3.7 GB tmpfs** - a 6 GB rootfs will not fit via `adb push`.

### 9.6 Scenario C: Going Back from Debian to Android

**This is the most commonly overlooked scenario.** After installing Linux, returning to Android requires **restoring dtbo as well** (since it was erased).

```bash
# ===== Step 1: restore dtbo (erased during the Linux install) =====
fastboot flash dtbo_a dtbo.img
fastboot flash dtbo_b dtbo.img

# ===== Step 2: flash the stock kernel back =====
fastboot flash boot_a boot.img
fastboot flash boot_b boot.img

# ===== Step 3: flash the system partitions back =====
fastboot flash system_a system.img
fastboot flash vendor_a vendor.img

# ===== Step 4: restore the signed AVB metadata (undo --disable-verity) =====
fastboot flash vbmeta_a vbmeta.img
fastboot flash vbmeta_b vbmeta.img

# ===== Step 5: wipe userdata (Mobian's rootfs blocks Android boot) =====
fastboot -w
# If fastboot -w fails (missing make_f2fs), use EDL instead:
#   edl --loader=$LOADER e userdata

# ===== Step 6: activate slot A and boot =====
fastboot --set-active=a
fastboot reboot
```

**⚠️ Step 5 cannot be skipped.** Android cannot mount an ext4 partition containing Debian (different UUID, different layout).

**⚠️ If `fastboot flash` reports `Flashing is not allowed for Critical Partitions`:**

```bash
fastboot flashing unlock_critical
# If still refused (some OnePlus models refuse even after unlocking), use EDL:
edl --loader=$LOADER e abl_a
edl --loader=$LOADER w abl_a abl.img
```

### 9.7 Scenario D: Completely Bricked - Use EDL

**If EDL is reachable, the device is recoverable.** Test: `lsusb | grep 05c6:9008` returns something.

```bash
# 1. Enter EDL: unplug -> hold Power 20 s -> hold Vol Up + Vol Down -> plug in
lsusb | grep 05c6:9008

# 2. You must use a HWID + PK_HASH matched loader (generic loaders fail silently)
LOADER=~/Loaders/oneplus/0008b0e10051459b_dd7c5f2e53176bee_fhprg_op6t.bin

# 3. Check the partition table (confirms the loader actually works)
edl --loader=$LOADER printgpt

# 4. Flash the critical partitions - remember: erase first!
for P in xbl xbl_config abl tz hyp aop cmnlib cmnlib64 devcfg \
         keymaster qupfw storsec bluetooth modem dsp; do
    edl --loader=$LOADER e ${P}_a
    edl --loader=$LOADER w ${P}_a ${P}.img
done

# 5. Read back and verify (essential!)
edl --loader=$LOADER r abl_a /tmp/verify.img
cmp abl.img /tmp/verify.img && echo "✅ write OK" || echo "❌ write FAILED"
```

**EDL is the last resort that always works**: as long as the SoC and storage are physically intact, EDL can write any partition.

### 9.8 TWRP Gotchas (Traps I Actually Hit)

| Trap | Explanation | Fix |
|---|---|---|
| **`sudo -S` swallows stdin** | In `echo pw \| sudo -S adb ... < file` the file stream is consumed by the password | Validate with `sudo -S -v`, then use `sudo -n` |
| **`/tmp` is only 3.7 GB** | It is a tmpfs in RAM | Write large files with fastboot, or chunk them |
| **TWRP's `dd` fails on large files** | A 6 GB streamed write reported `Bad address` | Use `fastboot flash` instead |
| **TWRP takes the phone out of fastboot** | After a flash-and-reboot, `fastboot devices` is empty | Run `adb reboot bootloader` first and **confirm `fastboot devices` is non-empty** |
| **fastboot hangs on `< waiting for any device >`** | The phone is not in fastboot but the command was already issued | Add a `timeout`; verify `fastboot devices` first |
| **Interface names are not fixed** | I configured the wrong NIC (`enx00e04c...`) | Find the real name from `dmesg`'s `renamed from usb0` |
| **OnePlus may not support `fastboot boot`** | Reports `Failed to load/authenticate boot image` | `flash` to the partition, then reboot |

### 9.9 Rescue Command Reference

```bash
# ===== Enter / switch =====
fastboot --set-active=b              # Switch to the TWRP slot
fastboot reboot                      # Boot

# ===== Inside TWRP (ADB) =====
adb devices                          # -> 085ba2e7  recovery
adb shell getprop ro.twrp.version    # -> 3.7.0_11-0
adb shell getprop ro.boot.slot_suffix

# ===== Partition operations =====
adb shell 'ls /dev/block/bootdevice/by-name/'                    # List all partitions
adb shell 'mount -t ext4 -o ro /dev/block/bootdevice/by-name/system_a /mnt/x'
adb shell 'umount /mnt/x'
adb shell 'blockdev --getsize64 /dev/block/bootdevice/by-name/userdata'

# ===== Read-back verification (the most important operation) =====
adb shell 'dd if=/dev/block/bootdevice/by-name/boot_a bs=1 count=8 2>/dev/null' | xxd

# ===== Flashing an image from within TWRP =====
adb push image.img /tmp/
adb shell 'dd if=/tmp/image.img of=/dev/block/bootdevice/by-name/boot_a bs=4M'

# ===== Return from TWRP to fastboot =====
adb reboot bootloader
```

### 9.10 Rescue Decision Tree

```
Phone won't boot
    │
    ├─ Can it enter fastboot (18d1:d00d)?
    │      ├─ Yes -> fastboot --set-active=b -> enter TWRP -> diagnose per 9.4
    │      └─ No  ↓
    │
    ├─ Can it enter EDL (05c6:9008)?
    │      ├─ Yes -> flash firmware with the matched loader (9.7)
    │      │         Flash xbl/abl/tz first to restore fastboot capability
    │      └─ No  ↓
    │
    └─ No reaction on USB at all?
           ├─ Try another cable (must be a data cable, not charge-only)
           ├─ Try another port (prefer USB 2.0)
           ├─ Hold Power for 20 s and retry
           ├─ Charge for 30 minutes (the battery may be flat)
           └─ Still nothing -> hardware fault, professional repair needed
```

**Today's actual path** was the leftmost branch: `fastboot` → slot B TWRP → diagnosed that the hardware was fine → corrected the images → success.

---

## Appendix A: Command Cheat Sheet

```bash
# ===== EDL mode =====
lsusb | grep 05c6:9008                          # Confirm EDL mode
edl --loader=$LOADER printgpt                   # Print the partition table
edl --loader=$LOADER r <part> <out.img>         # Read a partition
edl --loader=$LOADER e <part>                   # Erase a partition
edl --loader=$LOADER w <part> <in.img>          # Write a partition
edl --loader=$LOADER reset --resetmode=reset    # Reset

# ===== fastboot =====
fastboot devices
fastboot getvar current-slot
fastboot getvar slot-retry-count:a
fastboot flash boot_a boot.img
fastboot erase dtbo
fastboot --set-active=a
fastboot --disable-verity --disable-verification flash vbmeta_a vbmeta.img
fastboot reboot

# ===== Image processing =====
simg2img sparse.img raw.img                     # Expand a sparse image
e2fsck -fn raw.img                              # Verify ext4
dd if=raw.img bs=1 skip=1024 count=1024 | xxd -s 104 -l 16 -p   # Read the UUID

# ===== TWRP =====
adb devices
adb shell 'mount -t ext4 -o ro /dev/block/bootdevice/by-name/system_a /mnt/x'
adb shell 'ls /dev/block/bootdevice/by-name/'
```

## Appendix B: References

- [postmarketOS/Nura Wiki - OnePlus 6 (enchilada)](https://wiki.nura.eco/wiki/OnePlus_6_(oneplus-enchilada)) ← **most important**
- [Mobian image downloads](https://images.mobian.org/qcom/)
- [bkerler/edl](https://github.com/bkerler/edl)
- [vm03/payload_dumper](https://github.com/vm03/payload_dumper)
- [TWRP for enchilada](https://dl.twrp.me/enchilada/)
- [Debian Wiki - Installing Debian on OnePlus 6](https://wiki.debian.org/InstallingDebianOn/OnePlus/OnePlus6)

---

## Closing Thoughts

This OnePlus 6 was declared end-of-life by OnePlus in December 2021 and was headed for a landfill or, at best, recycling. It now runs **Debian 13 stable** on kernel 6.12 and will keep receiving security updates.

**Snapdragon 845 / 8 GB RAM / 128 GB UFS - as a compact server it beats plenty of VPS instances, and it draws only a few watts.**

Flashing carries risk, but **as long as the bootloader and EDL remain reachable, the device is still recoverable.**

---

*Document status: draft*
*Recorded: 2026-10-03*

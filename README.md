# 一加6 重生记：从变砖到 Debian 服务器

> **OnePlus 6 (enchilada) · OxygenOS 11.1.2.2 · Mobian 13.0 (Debian) · 2026-10-03**

一个被刷成砖的一加6，如何在一次会话中从「完全无法开机」变成「运行 Debian 的紧凑型服务器」。

本文记录完整过程、**踩过的每一个坑**，以及真正解决问题的关键步骤。

---

## TL;DR

```bash
# 核心结论：三个必须记住的点

# 1. EDL (Firehose) 写入已有数据的扇区会静默失败 —— 必须先擦除
edl e tz_a && edl w tz_a tz.img     # ✅ 正确
edl w tz_a tz.img                    # ❌ 报告成功但数据没变

# 2. sparse rootfs 必须用官方 simg2img 展开，别自己写解析器
simg2img rootfs.img rootfs.raw

# 3. ⭐ 刷 Linux 前必须擦除 dtbo 分区 —— 决定性的一步
fastboot erase dtbo
```

---

## 目录

- [1. 设备信息](#1-设备信息)
- [2. 起点：一块砖](#2-起点一块砖)
- [3. 五个关键发现](#3-五个关键发现)
- [4. 完整可复现流程](#4-完整可复现流程)
- [5. 诊断方法论](#5-诊断方法论)
- [6. 工具与文件清单](#6-工具与文件清单)
- [7. 时间线与耗时](#7-时间线与耗时)
- [8. 经验教训](#8-经验教训)

---

## 1. 设备信息

| 项目 | 值 |
|---|---|
| 机型 | OnePlus 6 (A6003) |
| 代号 | `oneplus-enchilada` |
| SoC | Qualcomm Snapdragon 845 (SDM845) |
| 存储 | 128 GB UFS 2.1 (SAMSUNG KLUDG4U1EA-B0C1) |
| 序列号 | `085ba2e7` |
| 芯片序列 | `0xc650672c` (3327158060) |
| HWID | `0x0008b0e10051459b` |
| PK_HASH | `dd7c5f2e53176bee91747b53900ccec33dd30fa4ded0dfe9baf9156e6910862f` |
| 出厂系统 | OxygenOS 11.1.2.2 (Android 11, SDK 30) |
| 最终系统 | **Mobian 13.0 (Debian stable, Phosh)** |

### 为什么 HWID 和 PK_HASH 很重要

**EDL 模式的 Firehose loader 必须与设备的 HWID + PK_HASH 双重匹配。** 通用 SDM845 loader 会静默失败：

```
✅ 正确: oneplus/0008b0e10051459b_dd7c5f2e53176bee_fhprg_op6t.bin  (680,769 B)
❌ 通用: prog_firehose_ddr.elf 等 —— 无报错，但什么都不做
```

---

## 2. 起点：一块砖

### 症状

- 设备能进 EDL（`05c6:9008`）和 fastboot（`18d1:d00d`）
- 显示 OnePlus logo 约 75 秒，然后回落到 fastboot
- `boot_a` 分区前 64 字节**全为零** —— 没有内核
- 槽位状态：`slot-unbootable:a: yes`、`slot-retry-count:a: 0`

### 之前尝试过但失败的方案

| 方法 | 结果 |
|---|---|
| `fastboot flash boot_a boot.img` | 报告 `OKAY [0.254s]`，但 `boot_a` 依然是零 |
| `fastboot flash system_a system.img` | 报告 `OKAY [83.807s]`，但没写入 |
| `edl qfil` 24 个分区 | 全部报告 100%，但**没有一个真正写入** |
| `edl w` 单个分区 | 报告成功，数据没变 |
| MSM Download Tool | 在 Windows 11 上崩溃（2018 年的 32 位程序）|
| WSL 2 + usbipd | 无法绕过 Windows 驱动签名问题 |

**共同点：所有工具都报告成功，但没有一个真正写入数据。**

---

## 3. 五个关键发现

### 3.1 ⭐ Firehose 静默写入失败 —— 最大的坑

**这是整个过程中最重要的发现，解释了之前所有的失败。**

#### 现象

对一个**已经有数据**的扇区执行 Firehose `program` 命令，设备返回 ACK，但**内容不变**。

#### 实验证据

```bash
# 步骤 1: 读取 tz_a 的当前内容
edl r tz_a /tmp/tz_before.img
# → b2d1236c... （有数据）

# 步骤 2: 直接写入新内容
edl w tz_a tz.img
# → "Wrote tz.img to sector 134"  ✅ 报告成功

# 步骤 3: 读回验证
edl r tz_a /tmp/tz_after.img
# → b2d1236c... ❌ 和写入前一模一样！数据没有变化

# 步骤 4: 先擦除，再写入
edl e tz_a        # → "Erased tz_a starting at sector 134"
edl w tz_a tz.img # → "Wrote tz.img to sector 134"

# 步骤 5: 读回验证
edl r tz_a /tmp/tz_final.img
# → 与 tz.img 100% 字节一致 ✅
```

#### 结论

> **在 EDL/Firehose 模式下，必须先 `erase` 再 `write`。**
> 直接 `write` 到有数据的扇区会静默失败——返回成功但不写入。

#### 为什么这会误导人

所有工具链（`fastboot`、`edl w`、`edl qfil`、MSM Download Tool）都会：
1. 报告 `OKAY` / `100%` / `Wrote ... to sector N`
2. 实际什么都没做

**唯一的验证方法是：写完后读回来逐字节比较。**

---

### 3.2 固件必须与系统版本匹配

设备的各个固件分区（`tz`、`xbl`、`abl`、`hyp`、`modem` 等）**必须全部来自同一个 OTA 版本**。

#### 症状

固件版本不匹配时，ABL（Android Bootloader）能启动，fastboot 能工作，但**内核无法完成硬件初始化** → 显示 logo 后崩溃。

#### 正确的固件来源

```
OxygenOS 11.1.2.2 OTA
  OnePlus6Oxygen_22.J.62_GLO_0620_2111252336
  下载: https://otafsg1.h2os.com/patch/amazone2/GLO/OnePlus6Oxygen/
        OnePlus6Oxygen_22.J.62_GLO_0620_2111252336/
        OnePlus6Oxygen_22.J.62_OTA_0620_all_2111252336_14afec75dd6fa.zip
```

#### 需要刷写的 18 个固件分区

```
abl  aop  bluetooth  cmnlib  cmnlib64  devcfg  dsp  fw_4j1ed
fw_4u1ea  hyp  keymaster  LOGO  oem_stanvbk  qupfw  storsec
tz  xbl  xbl_config
modem  (需要 --slot=all)
```

#### 验证

```
16/16 个固件分区逐字节验证通过:
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

### 3.3 手写 sparse 解析器不可靠

Mobian 的 `rootfs.img` 是 Android sparse 格式（magic `ED26FF3A`）。

#### 我犯的错误

自己写了一个 Python 解析器展开 sparse 镜像：

```python
# ❌ 有 bug 的版本
elif ctype == CHUNK_FILL:
    fill = f.read(4)
    o.write(fill * (csz * blk_sz))
    ...
```

**症状：**
```
文件大小: 11,331,903,488 字节  ← 应该是 5,998,936,064
e2fsck: "Superblock has an invalid journal (inode 8)"
        "The journal superblock is corrupt"
```

展开后的镜像**文件系统损坏**，无法挂载。

#### 正确做法

```bash
sudo apt install android-sdk-libsparse-utils
simg2img rootfs.img rootfs.raw
```

**结果：**
```
大小: 5,998,936,064 字节 = 1,464,584 块  ✅ 与预期完全一致
e2fsck: Pass 1-5 全部通过，无任何错误
```

#### 教训

> **不要自己实现标准格式的解析器。** 用官方工具，然后**用 `e2fsck` 验证结果**。

---

### 3.4 boot 镜像头部地址必须匹配设备

Mobian 的 boot 镜像使用 postmarketOS 的通用地址：

| 参数 | Mobian 原始 | 一加原厂 / TWRP | 
|---|---|---|
| `kernel_addr` | `0x10008000` | **`0x8000`** |
| `ramdisk_addr` | `0x11000000` | **`0x1000000`** |
| `tags_addr` | `0x10000100` | **`0x100`** |
| `header_version` | `0` | **`1`** |

#### 重打包

用原厂镜像的头部布局重新打包 Mobian 的内核 + ramdisk，保留 Mobian 的 cmdline：

```python
# 关键字段
hdr[12] = 0x8000        # kernel_addr   (原厂值)
hdr[20] = 0x1000000     # ramdisk_addr  (原厂值)
hdr[32] = 0x100         # tags_addr     (原厂值)
hdr[40] = 1             # header_version
hdr[44] = 0x1600015b    # os_version (取自原厂，避免 anti-rollback 拒绝)
hdr[64:576] = mobian_cmdline   # 保留 mobile.root=UUID=...
```

#### cmdline（必须保留）

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

### 3.5 ⭐⭐ `fastboot erase dtbo` —— 决定性的一步

**这是让设备最终启动的那一步。**

#### 错误的思路（我走的弯路）

我认为：Mobian 的 boot 镜像里 `kernel (11.4 MB) + ramdisk (18.4 MB) = 29.8 MB` 就是文件全部大小，**没有空间放设备树**，所以 DTB 必须来自 `dtbo` 分区。

于是我：
1. 从 rootfs 里提取 mainline DTB（`sdm845-oneplus-enchilada.dtb`，118,165 字节）
2. 分析原厂 dtbo 格式（**发现它用的是大端序，不是标准 AOSP 小端**）
3. 构建了一个正确的 dtbo 镜像，刷进 `dtbo_a`

**结果：仍然失败，5 秒返回 fastboot。**

#### 正确答案（官方文档）

从 [postmarketOS/Nura wiki](https://wiki.nura.eco/wiki/OnePlus_6_(oneplus-enchilada)) 找到：

> ### Erase DTBO
>
> **The dtbo partition includes information that will cause the bootloader
> to break the Nura image, so it must be erased before flashing.**
>
> ```
> $ fastboot erase dtbo
> ```

**dtbo 分区必须被「擦除」，而不是被写入正确内容。**

#### 为什么

设备的差异报告（DTB）**已经附加在内核镜像后面**了：

```
vmlinuz-6.12-sdm845          11,239,833 字节
boot.img 的 kernel_size      11,357,998 字节
差值                            118,165 字节  ← 正好等于 DTB 大小！
```

**内核自己带着设备树。** bootloader 提供的 dtbo 反而会**覆盖/干扰**它，导致启动失败。

#### 执行

```bash
fastboot erase dtbo_a
fastboot erase dtbo_b
fastboot erase dtbo      # 通用名称（作用于当前槽位）
```

#### 结果

```
[17:25:56] Linux 6.12-sdm845 内核启动
           USB gadget 注册: 0525:a4a2 Netchip Linux-USB Ethernet/RNDIS Gadget
           *** 内核运行成功 ***
```

**从这一秒起，设备就在运行 Linux 了。**

---

## 4. 完整可复现流程

### 前置条件

- 一加6，bootloader 已解锁
- 一台 Linux 主机（本例用 Ubuntu Server）
- 数据线（建议 USB 2.0 口，更稳定）

### 步骤 1：准备工具

```bash
# EDL 工具
git clone https://github.com/bkerler/edl
pip install -r edl/requirements.txt

# 官方 sparse 工具
sudo apt install android-sdk-libsparse-utils android-tools-adb android-tools-fastboot

# Firefox/Chrome 驱动的 fastboot 有时不稳定，建议用官方 platform-tools
```

### 步骤 2：获取匹配的 Firehose loader

```bash
mkdir -p ~/Loaders/oneplus
# 必须 HWID + PK_HASH 双重匹配
# 文件名格式: <HWID>_<PK_HASH 前 24 位>_fhprg_<设备>.bin
# 本例: 0008b0e10051459b_dd7c5f2e53176bee_fhprg_op6t.bin
```

**获取方法**：从设备 EDL 模式读取 HWID/PK_HASH，然后在 loader 库中匹配。

### 步骤 3：下载并提取官方固件

```bash
# 下载 OOS 11.1.2.2 OTA (约 2 GB)
curl -O 'https://otafsg1.h2os.com/patch/amazone2/GLO/OnePlus6Oxygen/OnePlus6Oxygen_22.J.62_GLO_0620_2111252336/OnePlus6Oxygen_22.J.62_OTA_0620_all_2111252336_14afec75dd6fa.zip'

# 提取 payload.bin
unzip -o *.zip payload.bin

# 用 payload_dumper 提取分区
pip install payload_dumper
payload_dumper payload.bin
```

需要提取：
```
固件 (18 个): abl aop bluetooth cmnlib cmnlib64 devcfg dsp fw_4j1ed
              fw_4u1ea hyp keymaster LOGO oem_stanvbk qupfw storsec
              tz xbl xbl_config modem
系统:         boot system vendor dtbo vbmeta
```

### 步骤 4：进入 EDL 模式

```
1. 拔掉 USB 线
2. 长按电源键 20 秒（彻底关机）
3. 按住 音量上 + 音量下
4. 插上 USB 线
5. 确认: lsusb | grep 05c6:9008
```

### 步骤 5：刷写固件（擦除 + 写入）

```bash
LOADER=~/Loaders/oneplus/0008b0e10051459b_dd7c5f2e53176bee_fhprg_op6t.bin

for P in xbl xbl_config aop tz hyp bluetooth abl keymaster \
         cmnlib cmnlib64 devcfg qupfw storsec fw_4j1ed fw_4u1ea oem_stanvbk modem; do
    echo "=== $P_a ==="
    edl --loader=$LOADER e ${P}_a          # ← 必须先擦除！
    edl --loader=$LOADER w ${P}_a ${P}.img
done
```

**⚠️ 注意**：`edl` CLI 每次只能执行一个操作。`edl w a b c` 只会打印 Usage。

### 步骤 6：准备 Mobian 镜像

```bash
# 下载 Mobian 13.0 (Debian stable)
curl -O https://images.mobian.org/qcom/mobian-sdm845-phosh-13.0.tar.xz

# 解压（两层嵌套：tar.xz → tar → 文件）
7z x mobian-sdm845-phosh-13.0.tar.xz
7z x mobian-sdm845-phosh-13.0.tar
# 得到:
#   mobian-sdm845-phosh-20251002.boot-enchilada.img   (29,798,400 B)
#   mobian-sdm845-phosh-20251002.rootfs.img           (4,221,440,856 B, sparse)
```

### 步骤 7：展开 sparse rootfs（用官方工具）

```bash
simg2img mobian-sdm845-phosh-20251002.rootfs.img rootfs.raw

# 验证（重要！）
e2fsck -fn rootfs.raw
# 应输出: Pass 1-5 全部通过

# 确认 UUID 匹配
dd if=rootfs.raw bs=1 skip=1024 count=1024 2>/dev/null | xxd -s 104 -l 16 -p
# 应输出: 6cad760a79574d99896d7adf1845e68b
```

### 步骤 8：重打包 boot 镜像

Mobian 的 boot 镜像头部地址是 postmarketOS 通用值，需要改成一加格式。

**方法**：用原厂 boot.img 的头部布局，套上 Mobian 的 kernel + ramdisk + cmdline。

关键字段：
```
kernel_addr    = 0x8000
ramdisk_addr   = 0x1000000
tags_addr      = 0x100
page_size      = 4096
header_version = 1
os_version     = 0x1600015b    (取自原厂，避免 anti-rollback)
cmdline        = <Mobian 的，必须含 mobile.root=UUID=...>
id             = SHA1(kernel + kernel_size + ramdisk + ramdisk_size + ...)
```

### 步骤 9：刷写并启动 ⭐

```bash
# 1. 刷 Linux 内核
fastboot flash boot_a mobian-repacked.img

# 2. 把 TWRP 放到 B 槽作为救援后备
fastboot flash boot_b twrp-3.7.0_11-0-enchilada.img

# 3. 写入 rootfs 到 userdata
fastboot flash userdata rootfs.raw

# 4. ⭐⭐⭐ 擦除 dtbo —— 决定性的一步
fastboot erase dtbo_a
fastboot erase dtbo_b

# 5. 禁用 AVB 校验（第三方系统必需）
fastboot --disable-verity --disable-verification flash vbmeta_a vbmeta.img
fastboot --disable-verity --disable-verification flash vbmeta_b vbmeta.img

# 6. 激活 A 槽并启动
fastboot --set-active=a
fastboot reboot
```

### 步骤 10：登录

首次启动需要 1-2 分钟（会看到黑屏或 logo）。

**默认账号：**

| 项目 | 值 |
|---|---|
| 用户名 | `mobian` |
| 密码 | `1234` |

> ⚠️ **立刻修改密码**：`passwd`

### 步骤 11：USB 网络访问（推荐）

Mobian 通过 USB 提供 RNDIS/EEM 网络：

```bash
# 主机侧
IF=$(ip -brief link | awk '/enp4s0f3u1u2|usb0/ {print $1; exit}')
sudo ip link set $IF up
sudo ip addr add 172.16.42.2/24 dev $IF
ssh mobian@172.16.42.1
```

---

## 5. 诊断方法论

这次真正有用的方法，按价值排序：

### 5.1 永远验证写入结果

```bash
# 写完后必须读回比较
edl --loader=$LOADER w tz_a tz.img
edl --loader=$LOADER r tz_a /tmp/verify.img
cmp tz.img /tmp/verify.img && echo "OK" || echo "写入失败"
```

**所有工具的成功报告都不可信。** 只有字节比较可信。

### 5.2 用独立内核测试硬件

**用 TWRP 区分「硬件故障」和「软件问题」：**

```bash
fastboot flash boot_a twrp.img
fastboot reboot
# 如果 TWRP 启动 → 硬件、引导链、boot 分区全部正常
# 如果 TWRP 也失败 → 硬件层面问题
```

**本例中这一步是转折点**：TWRP 成功启动，证明硬件完好，把排查方向从「硬件」转向「镜像配置」。

### 5.3 读取分区内容确认真实状态

```bash
# 例如：确认 boot_a 真的有内核（而不是全零）
edl --loader=$LOADER rs 51334 8 /tmp/boot_head.bin --lun=4
xxd /tmp/boot_head.bin
# 应看到: 414e4452 4f494421 = "ANDROID!"
```

**⚠️ 注意 LUN**：`boot_a` 在 LUN 4，用 `rs` 直接读扇区时必须指定 `--lun=4`，否则读到的是 LUN 0 的无关数据。

### 5.4 从 recovery 环境检查文件系统

TWRP 启动后可以：

```bash
adb shell 'mount -t ext4 -o ro /dev/block/bootdevice/by-name/system_a /mnt/systest'
adb shell 'ls /mnt/systest'
adb shell 'cat /mnt/systest/build.prop | grep ro.build.version'
```

### 5.5 用槽位重试计数判断是否真的尝试启动

```bash
fastboot getvar slot-retry-count:a
```

- **从 7 减到 6** → bootloader 尝试了 1 次就放弃（通常是校验失败）
- **从 7 减到 0** → bootloader 反复尝试后耗尽重试（通常是内核启动后崩溃）

这个差异能区分「镜像被拒绝」和「内核崩溃」。

---

## 6. 工具与文件清单

### 必需工具

| 工具 | 用途 | 获取 |
|---|---|---|
| `edl` | EDL/Firehose 刷写 | [bkerler/edl](https://github.com/bkerler/edl) |
| `simg2img` | 展开 Android sparse 镜像 | `apt install android-sdk-libsparse-utils` |
| `payload_dumper` | 从 OTA 提取分区 | [vm03/payload_dumper](https://github.com/vm03/payload_dumper) |
| `fastboot` | fastboot 刷写 | `apt install android-tools-fastboot` |
| `adb` | 连接 TWRP / 系统 | `apt install android-tools-adb` |
| `e2fsck` | 验证 ext4 镜像 | `apt install e2fsprogs` |
| `7z` | 解压嵌套归档 | `apt install p7zip-full` |

### 关键文件

| 文件 | 大小 | 说明 |
|---|---|---|
| `..._fhprg_op6t.bin` | 680,769 B | Firehose loader（HWID+PK_HASH 匹配）|
| `boot.img` (原厂) | 67,108,864 B | 头部布局参考 |
| `mobian-...boot-enchilada.img` | 29,798,400 B | Mobian 内核 |
| `rootfs.raw` | 5,998,936,064 B | 展开后的 Debian rootfs |
| `twrp-3.7.0_11-0-enchilada.img` | 67,108,864 B | 恢复环境 |

### 关键数值

```
rootfs ext4 UUID:   6cad760a-7957-4d99-896d-7adf1845e68b
原厂 boot 头部:      kernel_addr=0x8000  ramdisk_addr=0x1000000  tags_addr=0x100
dtbo 表头字节序:     大端 (big-endian)
dtbo magic:          0xD7B7AB1E (大端字节序 d7 b7 ab 1e)
userdata 分区:       118,112,366,592 B (110 GB)
```

---

## 7. 时间线与耗时

| 阶段 | 耗时 | 说明 |
|---|---|---|
| 摸索与失败 | 数小时 | 反复尝试 fastboot/EDL 刷写，全部「成功」但无效 |
| **发现擦除问题** | —— | 转折点 |
| 重新刷写全部固件 | ~3 分钟 | 18 个分区，擦除+写入+验证 |
| 提取并传输系统镜像 | ~10 分钟 | 4 GB |
| rootfs 展开（走弯路）| ~30 分钟 | 手写解析器 → 损坏 → 重来 |
| TWRP 救援验证 | ~5 分钟 | 证明硬件正常 |
| 下载 Mobian | ~4 分钟 | 1.31 GB |
| boot 重打包 | ~2 分钟 | 头部地址修正 |
| dtbo 弯路 | ~40 分钟 | 构建了正确的 dtbo，但答案是「擦除」|
| **擦除 dtbo 后启动** | **~10 秒** | ⭐ 成功 |

**总耗时：约一天（含大量试错）**

**如果重来一次**，按本文档的流程：**约 1-2 小时**。

---

## 8. 经验教训

### 8.1 最重要的一条

> **永远不要相信「成功」的返回值。写完必须读回来比较。**

这次会话中，几乎所有工具都在撒谎：

| 工具 | 谎报 | 真相 |
|---|---|---|
| `fastboot flash boot_a` | `OKAY [0.254s]` | 分区仍是全零 |
| `edl w` | `Wrote ... to sector N` | 数据没变 |
| `edl qfil` | 24 个分区 `100%` | 一个都没写入 |
| MSM Download Tool | `Success` | 同样没写入 |

**只有 `cmp` / MD5 比较不会撒谎。**

### 8.2 区分「硬件故障」和「软件配置」

用**独立内核**（TWRP）测试是关键手段。在确认硬件正常之前，不要花时间在软件配置上。

### 8.3 查官方文档

我在 `dtbo` 上走了一大段弯路（分析格式、构建镜像、修正字节序），而官方 wiki 一句话就给出了答案：**擦除它**。

**在深入逆向之前，先搜索官方安装文档。**

### 8.4 不要自己实现标准格式解析器

手写的 sparse 解析器损坏了文件系统，浪费了约 30 分钟。官方 `simg2img` 一条命令解决。

**如果一定要自己写，必须用 `e2fsck` 等工具验证输出。**

### 8.5 保留救援路径

**始终在另一个槽位保留 TWRP：**

```bash
fastboot flash boot_b twrp.img      # B 槽作为救援
fastboot flash boot_a linux.img     # A 槽作为主系统
```

出问题时 `fastboot --set-active=b` 就能回到恢复环境。

### 8.6 理解 A/B 槽位

| 变量 | 含义 |
|---|---|
| `current-slot` | 当前活动槽位 |
| `slot-unbootable:X` | 该槽位被标记为不可启动 |
| `slot-retry-count:X` | 剩余重试次数（7 = 满）|
| `slot-successful:X` | 该槽位是否成功启动过 |

```bash
fastboot --set-active=a    # 切换槽位并重置重试计数
```

### 8.7 具体的技术坑

| 坑 | 正确做法 |
|---|---|
| `edl` CLI 一次只能一个操作 | `edl w a b c` 只打印 Usage |
| 分区名不带斜杠、带槽位后缀 | `tz_a` ✅  `/tz_a` ❌ |
| `edl --debugmode` 需要 `/logs` 存在 | 先 `mkdir /logs` |
| Sahara HELLO_RESP 必须是 48 字节 | 16 字节会被拒绝 |
| Sahara DONE 必须带长度 8 | 长度 0 会被忽略 |
| `rs` 读扇区需指定 LUN | `boot_a` 在 LUN 4 |
| TWRP 的 `/tmp` 是 3.7 GB tmpfs | 大文件不能放那里 |
| `adb shell dd` 吃 stdin | 用 `sudo -n` 避免密码占用 stdin |
| 一加设备的 `fastboot boot` 可能不支持 | 直接 `flash` 到分区 |
| 中文 .ps1 文件会解析失败 | 保持脚本纯 ASCII |

### 8.8 关于 fastboot 与 EDL 的选择

| 场景 | 推荐 | 原因 |
|---|---|---|
| 非关键分区（boot/system/vendor）| `fastboot flash` | 简单，本例验证有效 |
| 关键分区（abl/xbl/tz）| EDL + 擦除 | fastboot 会拒绝 |
| 恢复变砖设备 | EDL | 唯一可用通道 |
| 写入大镜像（rootfs）| `fastboot flash` | 原生批量传输，比 adb stdin 可靠 |

---

## 附录 A：常用命令速查

```bash
# ===== EDL 模式 =====
lsusb | grep 05c6:9008                          # 确认在 EDL
edl --loader=$LOADER printgpt                   # 打印分区表
edl --loader=$LOADER r <part> <out.img>         # 读取分区
edl --loader=$LOADER e <part>                   # 擦除分区
edl --loader=$LOADER w <part> <in.img>          # 写入分区
edl --loader=$LOADER reset --resetmode=reset    # 复位

# ===== fastboot =====
fastboot devices
fastboot getvar current-slot
fastboot getvar slot-retry-count:a
fastboot flash boot_a boot.img
fastboot erase dtbo
fastboot --set-active=a
fastboot --disable-verity --disable-verification flash vbmeta_a vbmeta.img
fastboot reboot

# ===== 镜像处理 =====
simg2img sparse.img raw.img                     # 展开 sparse
e2fsck -fn raw.img                              # 验证 ext4
dd if=raw.img bs=1 skip=1024 count=1024 | xxd -s 104 -l 16 -p   # 读取 UUID

# ===== TWRP =====
adb devices
adb shell 'mount -t ext4 -o ro /dev/block/bootdevice/by-name/system_a /mnt/x'
adb shell 'ls /dev/block/bootdevice/by-name/'
```

## 附录 B：参考链接

- [postmarketOS/Nura Wiki - OnePlus 6 (enchilada)](https://wiki.nura.eco/wiki/OnePlus_6_(oneplus-enchilada)) ← **最重要**
- [Mobian 镜像下载](https://images.mobian.org/qcom/)
- [bkerler/edl](https://github.com/bkerler/edl)
- [vm03/payload_dumper](https://github.com/vm03/payload_dumper)
- [TWRP for enchilada](https://dl.twrp.me/enchilada/)
- [Debian Wiki - Installing Debian on OnePlus 6](https://wiki.debian.org/InstallingDebianOn/OnePlus/OnePlus6)

---

## 结语

这台一加6 在 2021 年被一加宣布停止支持，本该进填埋场或回收站。现在它运行着 **Debian 13 stable**，内核 6.12，可以持续接收安全更新。

**骁龙 845 / 8 GB RAM / 128 GB UFS —— 作为一台紧凑型服务器，它比很多 VPS 都强，而且功耗只有几瓦。**

刷机有风险，但**只要 bootloader 和 EDL 还能进，设备就还有救**。

---

*文档状态：草稿*
*记录日期：2026-10-03*

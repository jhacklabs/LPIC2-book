---
title: "Topic 203 — Filesystem and Devices"
exam: LPIC-2 / 201-450
weights: "203.1 (4) + 203.2 (3) + 203.3 (2)"
os_target: "Arch Linux / Omarchy"
tags: [lpic2, filesystem, fstab, btrfs, xfs, luks]
---

# Topic 203: Filesystem and Devices

## ۱) مفهوم کلی

این Topic به سه بخش تقسیم می‌شود:
- **203.1** عملیات پایه‌ی فایل‌سیستم (mount کردن، fstab، UUID)
- **203.2** نگهداری فایل‌سیستم (ext2/3/4، Btrfs، XFS، پایش سلامت دیسک با SMART)
- **203.3** پیکربندی گزینه‌های پیشرفته (AutoFS، فایل‌سیستم‌های CD-ROM، رمزنگاری دیسک)

## ۲) چرا این مبحث مهم است؟

فایل‌سیستم قلب داده‌ست. اگر ندانی فایل‌سیستم چطور mount می‌شود، چطور آسیب‌دیدگی‌اش را تشخیص دهی، یا چطور رمزنگاری‌اش کنی، یک سرور به‌راحتی می‌تواند داده از دست بدهد یا در برابر سرقت فیزیکی دیسک آسیب‌پذیر باشد. برای مسیر Cyber Security، بخش **rمزنگاری با LUKS** مستقیماً به «حفاظت داده در حالت سکون» (data-at-rest protection) مربوط است — یکی از اصول پایه امنیت.

## ۳) مثال‌های واقعی + روی سیستم خودم

**واقعی:** یک لپ‌تاپ کاری دزدیده می‌شود؛ چون دیسک با LUKS رمزنگاری شده بود، سارق هیچ داده‌ای نمی‌تواند بخواند، حتی با درآوردن فیزیکی دیسک.

**روی Omarchy:** نصب‌های مدرن Arch اغلب از Btrfs با snapshot استفاده می‌کنند (مخصوصاً برای rollback بعد از یک بروزرسانی خراب) — دقیقاً یکی از مهارت‌های همین Topic.

## ۴) دستورات کامل

### 203.1 — عملیات پایه فایل‌سیستم

```bash
cat /etc/fstab            # جدول mount خودکار در بوت
cat /etc/mtab             # (امروزه معمولاً symlink به /proc/mounts)
cat /proc/mounts          # فایل‌سیستم‌های در حال حاضر mount‌شده
sudo mount /dev/sdb1 /mnt/data
sudo umount /mnt/data
sudo blkid                # نمایش UUID و نوع فایل‌سیستم هر پارتیشن
sync                       # فلاش کردن بافرهای نوشتن به دیسک
sudo swapon /dev/sdb2
sudo swapoff /dev/sdb2
```
استفاده از UUID در fstab (به‌جای `/dev/sdX` که ممکن است ترتیبش عوض شود):
```
UUID=1234-5678  /data  ext4  defaults  0  2
```
واحدهای mount در systemd (جایگزین مدرن fstab، هرچند fstab هنوز رایج‌تر است):
```bash
systemctl list-units --type=mount
```
فایل‌های `.mount` در `/etc/systemd/system/` می‌توانند معادل خط‌های fstab باشند — systemd آن‌ها را خودکار از fstab هم generate می‌کند.

### 203.2 — نگهداری فایل‌سیستم

```bash
sudo mkfs.ext4 /dev/sdb1
sudo mkswap /dev/sdb2
sudo fsck.ext4 -f /dev/sdb1     # اجرای اجباری حتی اگر فایل‌سیستم "clean" باشد
sudo tune2fs -l /dev/sdb1        # نمایش پارامترهای فایل‌سیستم ext
sudo dumpe2fs /dev/sdb1 | less   # اطلاعات سوپربلاک و گروه‌های بلاک
sudo debugfs /dev/sdb1           # ابزار پیشرفته تعاملی برای بازرسی/تعمیر دستی ext
```

> ⚠️ **هشدار جدی:** `mkfs` و `mkswap` تمام داده‌های موجود روی پارتیشن را پاک می‌کنند. همیشه قبل از اجرا `blkid` یا `lsblk` بزن و مطمئن شو دستگاه درست را انتخاب کرده‌ای.

Btrfs (رایج روی نصب‌های مدرن Arch/Omarchy):
```bash
sudo mkfs.btrfs /dev/sdb1
sudo btrfs subvolume create /mnt/@home
sudo btrfs subvolume snapshot /mnt/@home /mnt/@home_snapshot_2026
sudo btrfs filesystem show
```
XFS:
```bash
sudo mkfs.xfs /dev/sdb1
xfs_info /mnt/data
sudo xfs_repair /dev/sdb1     # باید unmount باشد
xfs_check /dev/sdb1
sudo xfsdump -f /backup/xfs.dump /mnt/data
sudo xfsrestore -f /backup/xfs.dump /mnt/restored
```
پایش سلامت دیسک (SMART):
```bash
sudo pacman -S smartmontools
sudo systemctl enable --now smartd
sudo smartctl -a /dev/sda      # گزارش کامل سلامت دیسک
sudo smartctl -t short /dev/sda    # اجرای تست کوتاه self-test
```

### 203.3 — گزینه‌های پیشرفته

AutoFS (mount خودکار فایل‌سیستم فقط هنگام دسترسی، نه در بوت):
```bash
sudo pacman -S autofs
sudo systemctl enable --now autofs
```
فایل اصلی: `/etc/auto.master` که به فایل‌های `/etc/auto.[dir]` اشاره می‌کند. مثال یک خط در `auto.master`:
```
/mnt/nfs  /etc/auto.nfs  --timeout=60
```
رمزنگاری دیسک با LUKS/dm-crypt:
```bash
sudo cryptsetup luksFormat /dev/sdb1
```
> ⚠️ **هشدار بسیار جدی:** این دستور تمام داده‌های روی پارتیشن را برای همیشه غیرقابل‌بازیابی می‌کند و رمز عبور انتخابی را هرگز نمی‌توان بازیابی کرد؛ فراموشی رمز = از دست رفتن کامل داده.

```bash
sudo cryptsetup luksOpen /dev/sdb1 secure_data
sudo mkfs.ext4 /dev/mapper/secure_data
sudo mount /dev/mapper/secure_data /mnt/secure
sudo cryptsetup luksClose secure_data
```
ساخت ایمیج CD-ROM (ISO9660/UDF):
```bash
mkisofs -o output.iso -r /path/to/files/
```

## ۵) نکات مهم آزمون LPIC

- ✅ همیشه بدان `fsck` نباید روی فایل‌سیستم mount‌شده در حالت rw اجرا شود.
- ✅ تفاوت `xfs_repair` (بدون آرگومان خاص، خودکار تعمیر می‌کند) با `fsck.ext4 -f` (اجباری چک کامل) را بدان.
- ✅ ترتیب دستورات LUKS را حفظ کن: `luksFormat` → `luksOpen` → `mkfs` → `mount` ... → `umount` → `luksClose`.
- ✅ `/etc/auto.master` فایل اصلی AutoFS است؛ فایل‌های زیرمجموعه نام دلخواه دارند.
- ✅ افزونه‌های CD-ROM: Joliet (نام فایل طولانی برای ویندوز)، Rock Ridge (متادیتای یونیکس)، El Torito (بوت‌پذیری).
- ✅ ZFS فقط در حد آگاهی لازم است، نه دستورات عملی — آزمون جزئیات آن را نمی‌پرسد.

## ۶) تمرین عملی امن

> ⚠️ برای تمرین‌های `mkfs`، `cryptsetup`، و `luksFormat`، **هرگز روی دیسک اصلی سیستم اجرا نکن**. یا از یک فلش USB اضافه استفاده کن یا یک فایل loop-back بساز (روش زیر کاملاً امن است):

```bash
# ساخت یک فایل ۵۰۰ مگابایتی به‌عنوان "دیسک" مجازی امن
dd if=/dev/zero of=/home/$USER/test.img bs=1M count=500
sudo losetup /dev/loop0 /home/$USER/test.img

# حالا با /dev/loop0 مثل یک دیسک واقعی کار کن
sudo mkfs.ext4 /dev/loop0
sudo mkdir -p /mnt/looptest
sudo mount /dev/loop0 /mnt/looptest
df -h /mnt/looptest

sudo umount /mnt/looptest
sudo losetup -d /dev/loop0
rm /home/$USER/test.img
```

تمرین LUKS روی همین فایل loop:
```bash
dd if=/dev/zero of=/home/$USER/crypt.img bs=1M count=200
sudo losetup /dev/loop1 /home/$USER/crypt.img
sudo cryptsetup luksFormat /dev/loop1
sudo cryptsetup luksOpen /dev/loop1 mytest
sudo mkfs.ext4 /dev/mapper/mytest
sudo mount /dev/mapper/mytest /mnt/looptest
sudo umount /mnt/looptest
sudo cryptsetup luksClose mytest
sudo losetup -d /dev/loop1
rm /home/$USER/crypt.img
```

## ۷) خلاصه جدولی

| فایل‌سیستم/ابزار | کاربرد |
|---|---|
| `/etc/fstab` | جدول mount خودکار |
| `blkid` | نمایش UUID/نوع فایل‌سیستم |
| `tune2fs`, `dumpe2fs`, `debugfs` | ابزارهای ext2/3/4 |
| `btrfs subvolume/snapshot` | مدیریت Btrfs |
| `xfs_repair`, `xfsdump/restore` | مدیریت XFS |
| `smartctl` | سلامت فیزیکی دیسک |
| `/etc/auto.master` | تنظیمات AutoFS |
| `cryptsetup luksFormat/Open/Close` | رمزنگاری دیسک (LUKS) |
| `mkisofs` | ساخت ایمیج ISO |

## ۸) مایندمپ متنی

```
Topic 203: Filesystem and Devices
├── 203.1 Operating
│   ├── /etc/fstab, UUID
│   ├── mount / umount / blkid
│   └── swapon / swapoff
├── 203.2 Maintaining
│   ├── ext2/3/4: tune2fs, dumpe2fs, debugfs
│   ├── Btrfs: subvolume, snapshot
│   ├── XFS: xfs_repair, xfsdump/restore
│   └── SMART: smartctl, smartd
└── 203.3 Advanced Options
    ├── AutoFS: /etc/auto.master
    ├── ISO9660 / UDF: mkisofs
    └── Encryption: cryptsetup (LUKS/dm-crypt)
```

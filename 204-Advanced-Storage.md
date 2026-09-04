---
title: "Topic 204 — Advanced Storage Device Administration"
exam: LPIC-2 / 201-450
weights: "204.1 (3) + 204.2 (2) + 204.3 (3)"
os_target: "Arch Linux / Omarchy"
tags: [lpic2, raid, lvm, iscsi]
---

# Topic 204: Advanced Storage Device Administration

## ۱) مفهوم کلی

سه بخش:
- **204.1** پیکربندی RAID نرم‌افزاری (0، 1، 5)
- **204.2** تنظیم دسترسی به دستگاه‌های ذخیره‌سازی (SSD، NVMe، iSCSI)
- **204.3** LVM — مدیریت حجم منطقی

## ۲) چرا این مبحث مهم است؟

RAID یعنی مقاومت در برابر خرابی فیزیکی دیسک؛ LVM یعنی انعطاف‌پذیری برای تغییر اندازه فضای ذخیره‌سازی بدون خاموش کردن سرور. این دو با هم پایه‌ی تقریباً هر زیرساخت ذخیره‌سازی سازمانی هستند. iSCSI هم دری‌ست به دنیای SAN (Storage Area Network) — یعنی دیسک‌هایی که از طریق شبکه (نه کابل فیزیکی) به سرور متصل می‌شوند.

## ۳) مثال‌های واقعی + روی سیستم خودم

**واقعی:** یک دیتاسنتر از RAID 5 برای دیسک‌های داده استفاده می‌کند (تحمل خرابی یک دیسک بدون از دست دادن داده) و از LVM روی آن، تا بتواند فضای هر سرویس را بدون قطعی رشد دهد.

**روی Omarchy:** روی یک لپ‌تاپ تک‌دیسکی RAID معنا ندارد، اما LVM را می‌توانی حتی روی یک دیسک تک هم تمرین کنی — بسیاری نصب‌های حرفه‌ای Arch از LVM روی یک دیسک استفاده می‌کنند تا بعداً بتوانند پارتیشن‌بندی را انعطاف‌پذیر تغییر دهند.

## ۴) دستورات کامل

### 204.1 — پیکربندی RAID

```bash
sudo pacman -S mdadm
sudo mdadm --create /dev/md0 --level=1 --raid-devices=2 /dev/sdb1 /dev/sdc1
cat /proc/mdstat                        # وضعیت زنده آرایه‌های RAID
sudo mdadm --detail /dev/md0
sudo mdadm --stop /dev/md0
sudo mdadm --assemble /dev/md0 /dev/sdb1 /dev/sdc1
```
فایل تنظیمات: `/etc/mdadm.conf` (یا `/etc/mdadm/mdadm.conf` در برخی توزیع‌ها). نوع پارتیشن سنتی برای RAID: `0xFD` (Linux raid autodetect) در جدول پارتیشن‌بندی MBR.

سطح‌های RAID که باید بشناسی:
- **RAID 0** — striping، سرعت بالا، **بدون تحمل خطا** (خرابی یک دیسک = از دست رفتن همه داده)
- **RAID 1** — mirroring، تحمل خطای کامل یک دیسک، ظرفیت مؤثر نصف
- **RAID 5** — striping + parity توزیع‌شده، تحمل خرابی یک دیسک، نیاز به حداقل ۳ دیسک

### 204.2 — تنظیم دسترسی دستگاه‌های ذخیره‌سازی

```bash
sudo hdparm -I /dev/sda        # اطلاعات کامل دیسک IDE/SATA
sudo hdparm -tT /dev/sda        # تست سرعت خواندن
sdparm --all /dev/sda           # اطلاعات دستگاه‌های SCSI/SATA مدرن
sudo nvme list                  # لیست دستگاه‌های NVMe
sudo nvme smart-log /dev/nvme0
sudo fstrim -v /                # آزادسازی بلاک‌های خالی روی SSD (TRIM)
cat /proc/interrupts | grep nvme
```
پیکربندی TRIM دوره‌ای خودکار روی Arch:
```bash
sudo systemctl enable --now fstrim.timer
```
iSCSI (اتصال دیسک از راه دور از طریق شبکه):
```bash
sudo pacman -S open-iscsi
sudo systemctl enable --now iscsid
sudo iscsiadm -m discovery -t sendtargets -p <target-ip>
sudo iscsiadm -m node --login
sudo iscsiadm -m session          # لیست session های فعال
```
فایل تنظیمات iSCSI initiator: `/etc/iscsi/iscsid.conf`. مفاهیم کلیدی: **WWID/WWN** (شناسه یکتای جهانی دستگاه ذخیره‌سازی)، **LUN** (Logical Unit Number — شماره واحد منطقی روی یک آرایه ذخیره‌سازی مشترک). SAN معمولاً با پروتکل‌های **AoE** (ATA over Ethernet) یا **FCoE** (Fibre Channel over Ethernet) پیاده می‌شود — این‌ها فقط سطح آگاهی لازم دارند.

### 204.3 — LVM (Logical Volume Manager)

```bash
sudo pvcreate /dev/sdb1                  # ساخت physical volume
sudo vgcreate vg_data /dev/sdb1          # ساخت volume group
sudo lvcreate -L 10G -n lv_home vg_data  # ساخت logical volume به اندازه ۱۰ گیگ
sudo mkfs.ext4 /dev/vg_data/lv_home
sudo mount /dev/vg_data/lv_home /mnt/home_new

sudo lvextend -L +5G /dev/vg_data/lv_home    # افزایش اندازه
sudo resize2fs /dev/vg_data/lv_home           # بزرگ‌کردن فایل‌سیستم بعد از extend

sudo lvcreate -L 2G -s -n lv_home_snap /dev/vg_data/lv_home   # snapshot
sudo vgchange -ay vg_data                     # فعال‌سازی volume group
sudo vgs                                       # لیست خلاصه VGها
sudo lvs                                       # لیست خلاصه LVها
sudo pvs                                       # لیست خلاصه PVها
```
فایل تنظیمات: `/etc/lvm/lvm.conf`. دستگاه‌های LVM از طریق `/dev/mapper/vg_data-lv_home` هم قابل‌دسترسی‌اند.

## ۵) نکات مهم آزمون LPIC

- ✅ ترتیب دقیق LVM را حفظ کن: `pvcreate` → `vgcreate` → `lvcreate` → `mkfs` → `mount`.
- ✅ برای بزرگ‌کردن یک LV، هم `lvextend` هم `resize2fs`/`xfs_growfs` لازم است (کوچک‌ کردن برعکس — اول resize فایل‌سیستم، بعد `lvreduce`).
- ✅ نوع پارتیشن RAID legacy: `0xFD`.
- ✅ تفاوت RAID 0/1/5 را با سناریوهای مختلف تمرین کن — سؤال رایج آزمون.
- ✅ ابزار جدید `nvme` مخصوص دستگاه‌های NVMe است، `hdparm` بیشتر برای SATA/IDE قدیمی‌تر.
- ✅ WWID/WWN/LUN فقط باید تعریف‌شان را بشناسی، نه پیکربندی عمیق SAN.

## ۶) تمرین عملی امن

> ⚠️ RAID و LVM واقعی روی دیسک اصلی سیستم بسیار خطرناک است. از فایل‌های loop-back استفاده کن (کاملاً امن، هیچ دیسک فیزیکی درگیر نمی‌شود).

تمرین LVM با فایل‌های loop:
```bash
for i in 1 2; do
  dd if=/dev/zero of=/home/$USER/disk$i.img bs=1M count=300
  sudo losetup /dev/loop$i /home/$USER/disk$i.img
done

sudo pvcreate /dev/loop1 /dev/loop2
sudo vgcreate vg_test /dev/loop1 /dev/loop2
sudo lvcreate -L 200M -n lv_test vg_test
sudo mkfs.ext4 /dev/vg_test/lv_test
sudo mkdir -p /mnt/lvtest
sudo mount /dev/vg_test/lv_test /mnt/lvtest
df -h /mnt/lvtest

# پاک‌سازی
sudo umount /mnt/lvtest
sudo lvremove /dev/vg_test/lv_test
sudo vgremove vg_test
sudo pvremove /dev/loop1 /dev/loop2
sudo losetup -d /dev/loop1
sudo losetup -d /dev/loop2
rm /home/$USER/disk1.img /home/$USER/disk2.img
```

تمرین RAID 1 با فایل‌های loop:
```bash
for i in 3 4; do
  dd if=/dev/zero of=/home/$USER/raid$i.img bs=1M count=200
  sudo losetup /dev/loop$i /home/$USER/raid$i.img
done
sudo mdadm --create /dev/md0 --level=1 --raid-devices=2 /dev/loop3 /dev/loop4
cat /proc/mdstat
sudo mdadm --stop /dev/md0
sudo losetup -d /dev/loop3
sudo losetup -d /dev/loop4
rm /home/$USER/raid3.img /home/$USER/raid4.img
```

## ۷) خلاصه جدولی

| مفهوم | دستور |
|---|---|
| ساخت RAID | `mdadm --create` |
| وضعیت RAID | `cat /proc/mdstat`, `mdadm --detail` |
| اطلاعات دیسک SATA | `hdparm`, `sdparm` |
| اطلاعات NVMe | `nvme list`, `nvme smart-log` |
| آزادسازی TRIM | `fstrim` |
| اتصال iSCSI | `iscsiadm -m discovery/node` |
| PV → VG → LV | `pvcreate` → `vgcreate` → `lvcreate` |
| بزرگ‌کردن LV | `lvextend` + `resize2fs` |
| Snapshot LVM | `lvcreate -s` |

## ۸) مایندمپ متنی

```
Topic 204: Advanced Storage Device Administration
├── 204.1 RAID
│   ├── mdadm.conf, mdadm --create
│   ├── /proc/mdstat
│   └── levels: 0 (speed), 1 (mirror), 5 (parity)
├── 204.2 Storage Access
│   ├── hdparm / sdparm (SATA/IDE)
│   ├── nvme (NVMe)
│   ├── fstrim (SSD)
│   └── iSCSI: iscsiadm, WWID/WWN/LUN, SAN (AoE/FCoE)
└── 204.3 LVM
    ├── pvcreate → vgcreate → lvcreate
    ├── lvextend/lvreduce + resize2fs
    ├── snapshots
    └── /etc/lvm/lvm.conf
```

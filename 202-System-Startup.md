---
title: "Topic 202 — System Startup"
exam: LPIC-2 / 201-450
weights: "202.1 (3) + 202.2 (4) + 202.3 (2)"
os_target: "Arch Linux / Omarchy"
tags: [lpic2, boot, systemd, grub]
---

# Topic 202: System Startup

## ۱) مفهوم کلی

این Topic فرآیند کامل بوت شدن یک سیستم لینوکس را پوشش می‌دهد: از لحظه‌ای که پردازنده روشن می‌شود تا رسیدن به یک سیستم قابل‌استفاده. سه بخش دارد:
- **202.1** شخصی‌سازی startup با systemd و SysV init
- **202.2** بازیابی سیستم (recovery) هنگام خرابی بوت
- **202.3** بوت‌لودرهای جایگزین (SYSLINUX، PXE و…)

## ۲) چرا این مبحث مهم است؟

وقتی سروری بوت نمی‌شود، هیچ SSH ای در کار نیست — فقط تو و کنسول. این لحظه‌ای‌ست که فهم عمیق GRUB، initramfs، و systemd targets نجات‌بخش می‌شود. Omarchy مستقیماً روی systemd ساخته شده، پس این Topic برایت کاملاً کاربردی و روزمره است — حتی مشکلات معمول مثل «سرویس فلان بعد از boot بالا نمی‌آید» دقیقاً همین دانش را می‌طلبد.

## ۳) مثال‌های واقعی + روی سیستم خودم

**واقعی:** یک بروزرسانی اشتباه GRUB باعث می‌شود سرور یک دیتاسنتر بوت نشود؛ مهندس باید از طریق کنسول IPMI وارد `grub shell` شود و دستی کرنل و initrd درست را مشخص کند.

**روی Omarchy:** چون Arch/Omarchy از systemd به‌عنوان init استفاده می‌کند (نه SysV)، بیشترین تمرکز عملی‌ات باید روی `systemctl` و `systemd-analyze` باشد، هرچند برای آزمون باید SysV را هم بشناسی (بسیاری سیستم‌های production هنوز از آن استفاده می‌کنند یا لایه‌ی سازگاری دارند).

## ۴) دستورات کامل

### 202.1 — شخصی‌سازی startup

```bash
systemctl list-units --type=target      # targetهای فعال (معادل runlevel)
systemctl get-default                    # target پیش‌فرض بوت
sudo systemctl set-default multi-user.target
systemctl list-unit-files --type=service # همه سرویس‌ها و وضعیت enable/disable
sudo systemctl enable sshd.service
sudo systemctl disable sshd.service
systemd-analyze blame                    # کدام سرویس بیشترین زمان بوت را گرفته
systemd-analyze critical-chain           # زنجیره وابستگی که بوت را کند کرده
```
معادل‌های SysV (برای آزمون، حتی روی Arch که استفاده نمی‌شوند باید بشناسی‌شان):
```bash
chkconfig --list          # (Red Hat family)
update-rc.d ssh defaults  # (Debian family)
init 3   /  telinit 3     # تغییر runlevel
```
مسیرهای کلیدی systemd: `/usr/lib/systemd/system/` (فایل‌های واحد پیش‌فرض پکیج‌ها)، `/etc/systemd/system/` (override و سفارشی‌سازی محلی — این جایی‌ست که خودت باید تغییرات بدهی)، `/run/systemd/` (واحدهای موقت runtime).

### 202.2 — بازیابی سیستم (Recovery)

> ⚠️ **هشدار جدی:** دستورات این بخش مستقیماً روی فرآیند بوت اثر می‌گذارند. قبل از تغییر GRUB، حتماً یک نسخه پشتیبان از `/boot/grub/grub.cfg` بگیر:
> ```bash
> sudo cp /boot/grub/grub.cfg /boot/grub/grub.cfg.bak
> ```

```bash
sudo grub-install /dev/sda          # نصب GRUB روی MBR یک دیسک (BIOS)
sudo grub-install --target=x86_64-efi --efi-directory=/boot/efi   # روی UEFI
sudo grub-mkconfig -o /boot/grub/grub.cfg
efibootmgr -v                       # لیست ورودی‌های بوت UEFI
```
حالت‌های بازیابی systemd:
```bash
systemctl rescue     # حالت تک‌کاربره با حداقل سرویس‌ها (شبیه runlevel 1)
systemctl emergency  # حداقل‌ترین حالت ممکن، فقط شل روت، بدون mount کامل فایل‌سیستم‌ها
```
برای رسیدن به این حالت‌ها در لحظه بوت، در منوی GRUB روی خط کرنل، پارامتر زیر اضافه می‌شود:
```
systemd.unit=rescue.target
```
یا روش قدیمی‌تر (SysV):
```
single
```
بررسی و تعمیر فایل‌سیستم بعد از خاموشی ناگهانی:
```bash
sudo fsck /dev/sda1
sudo mount -o remount,rw /
```
> ⚠️ هرگز `fsck` را روی یک فایل‌سیستم mount‌شده در حالت read-write اجرا نکن — همیشه یا unmount کن یا read-only mount کن.

### 202.3 — بوت‌لودرهای جایگزین

```bash
extlinux --install /boot/syslinux/     # نصب SYSLINUX روی یک پارتیشن
```
ابزارهای مرتبط با PXE (بوت شبکه‌ای، بدون رسانه فیزیکی): `pxelinux.0`، فایل‌های پیکربندی در `pxelinux.cfg/`. برای ایزو بوت‌شونده: `isolinux.bin`، `isolinux.cfg`، `isohdpfx.bin`. این‌ها معمولاً فقط سطح آگاهی لازم دارند، نه پیکربندی عملی عمیق. **systemd-boot** و **U-Boot** هم فقط باید بشناسی که چه هستند: systemd-boot جایگزین سبک‌تر GRUB برای سیستم‌های UEFI (که خود Arch/Omarchy می‌تواند از آن استفاده کند)، و U-Boot بوت‌لودر رایج سیستم‌های embedded/ARM است.

## ۵) نکات مهم آزمون LPIC

- ✅ تفاوت `systemctl rescue` و `systemctl emergency` را دقیق بدان: rescue فایل‌سیستم‌ها را mount می‌کند، emergency نه.
- ✅ محل صحیح override سرویس‌ها: همیشه در `/etc/systemd/system/`، هرگز مستقیم در `/usr/lib/systemd/system/` تغییر نده (پکیج‌ها آن را overwrite می‌کنند).
- ✅ فرق ESP (EFI System Partition) در UEFI با MBR در BIOS را بشناس.
- ✅ دستور `grub-install` برای BIOS نیاز به دیسک دارد (`/dev/sda`)، برای UEFI نیاز به مسیر ESP.
- ✅ SYSLINUX خانواده کامل دارد: SYSLINUX (دیسک)، ISOLINUX (سی‌دی)، PXELINUX (شبکه) — هرکدام برای رسانه متفاوت.

## ۶) تمرین عملی امن

> ⚠️ **هشدار جدی:** تمرین‌های GRUB واقعی روی سیستم اصلی خطرناک است. این تمرین را در یک ماشین مجازی (مثل QEMU/VirtualBox) با یک نصب Arch/Omarchy تستی انجام بده، نه روی لپ‌تاپ اصلی.

1. targetهای فعال و سرویس‌های کند بوت را ببین:
```bash
systemctl list-units --type=target
systemd-analyze blame | head -10
```
2. یک override امن برای یک سرویس بسازید (بدون تغییر فایل اصلی):
```bash
sudo systemctl edit sshd.service
```
3. در ماشین مجازی تستی، وارد حالت rescue شوید و برگردید:
```bash
sudo systemctl rescue
# بعد از بررسی:
systemctl default
```
4. یک نسخه پشتیبان از grub.cfg بگیرید و پیکربندی مجدد کنید:
```bash
sudo cp /boot/grub/grub.cfg /boot/grub/grub.cfg.bak
sudo grub-mkconfig -o /boot/grub/grub.cfg
```

## ۷) خلاصه جدولی

| مفهوم | ابزار/فایل |
|---|---|
| مدیریت سرویس‌ها | `systemctl enable/disable/start/stop` |
| زمان‌بندی بوت | `systemd-analyze blame/critical-chain` |
| override سرویس | `/etc/systemd/system/` |
| نصب GRUB (BIOS) | `grub-install /dev/sdX` |
| نصب GRUB (UEFI) | `grub-install --target=x86_64-efi` |
| بازسازی کانفیگ GRUB | `grub-mkconfig -o /boot/grub/grub.cfg` |
| حالت تعمیر | `systemctl rescue` / `emergency` |
| بوت شبکه‌ای | PXELINUX, pxelinux.cfg/ |
| بوت‌لودر سبک UEFI | systemd-boot |

## ۸) مایندمپ متنی

```
Topic 202: System Startup
├── 202.1 Customizing Startup
│   ├── systemd targets ↔ SysV runlevels
│   ├── systemctl enable/disable
│   └── /etc/systemd/system/ (override)
├── 202.2 System Recovery
│   ├── BIOS vs UEFI, ESP
│   ├── grub-install, grub-mkconfig
│   ├── rescue.target vs emergency.target
│   └── fsck, mount remount
└── 202.3 Alternate Bootloaders
    ├── SYSLINUX / ISOLINUX / PXELINUX
    ├── PXE boot
    └── systemd-boot, U-Boot (awareness)
```

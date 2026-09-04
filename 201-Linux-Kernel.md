---
title: "Topic 201 — Linux Kernel"
exam: LPIC-2 / 201-450
weights: "201.1 (2) + 201.2 (3) + 201.3 (4)"
os_target: "Arch Linux / Omarchy"
tags: [lpic2, kernel, modules, compiling]
---

# Topic 201: Linux Kernel

## ۱) مفهوم کلی

کرنل قلب لینوکس است: لایه‌ای که مستقیماً با سخت‌افزار حرف می‌زند و منابع (CPU، حافظه، دیسک، شبکه) را بین برنامه‌ها تقسیم می‌کند. این Topic سه جنبه را پوشش می‌دهد:
- **201.1** شناخت اجزای کرنل و مستندات آن
- **201.2** کامپایل کردن یک کرنل سفارشی از سورس
- **201.3** مدیریت کرنل و ماژول‌ها در زمان اجرا (runtime) و عیب‌یابی

## ۲) چرا این مبحث مهم است؟

روی سرورهای production گاهی لازم است کرنل را سفارشی کامپایل کنی — مثلاً برای حذف قابلیت‌های غیرضروری (کاهش سطح حمله امنیتی)، فعال کردن پشتیبانی سخت‌افزاری خاص، یا رفع یک باگ امنیتی که هنوز patch رسمی توزیع نیامده. همچنین وقتی سیستمی بعد از بوت شدن دچار مشکل درایور می‌شود یا سخت‌افزاری شناسایی نمی‌شود، باید بتوانی با ابزارهای runtime علت را پیدا کنی — این دقیقاً کاری‌ست که یک ادمین واقعی هر چند وقت یک‌بار انجام می‌دهد.

از منظر امنیتی: فهم عمیق ماژول‌های کرنل پایه‌ی شناخت rootkit های سطح-kernel هم هست — بسیاری از rootkit های خطرناک دقیقاً با تزریق ماژول مخرب به کرنل کار می‌کنند.

## ۳) مثال‌های واقعی + روی سیستم خودم

**واقعی:** شرکتی که سرورهای embedded می‌سازد، کرنل را با حذف صدها درایور غیرضروری کامپایل می‌کند تا هم حجم initrd کم شود، هم سطح حمله کمتر شود.

**روی Omarchy:** Arch از یک رویکرد متفاوت با Debian/Zorin استفاده می‌کند — بسته `linux` که از pacman نصب می‌شود، از قبل کامپایل‌شده و بهینه است؛ کامپایل دستی کرنل بیشتر برای یادگیری یا نیازهای خیلی خاص (مثل kernel های `linux-hardened` یا `linux-zen` که در AUR/مخازن رسمی موجودند) کاربرد دارد.

## ۴) دستورات کامل

### 201.1 — شناخت اجزای کرنل

```bash
uname -r          # نسخه کرنل در حال اجرا
uname -a           # اطلاعات کامل کرنل و معماری
```
مستندات کرنل معمولاً در `/usr/src/linux/Documentation/` قرار دارد (وقتی سورس دانلود شده باشد). فایل‌های image کرنل با پسوند `zImage` (فشرده با gzip، برای سیستم‌های قدیمی) یا `bzImage` (big zImage، امروز استاندارد) شناخته می‌شوند و امروزه اغلب با فشرده‌سازی `xz` هم می‌آیند (حجم کمتر).

### 201.2 — کامپایل کرنل (روی Arch)

```bash
# دریافت سورس کرنل رسمی آرچ
sudo pacman -S linux-headers
```

> ⚠️ **هشدار جدی:** کامپایل کرنل دستوری کم‌خطر نیست چون در صورت اشتباه پیکربندی، ممکن است سیستم بوت نشود. **همیشه کرنل فعلی را به‌عنوان گزینه‌ی fallback در GRUB نگه دار** و کرنل جدید را با نام دیگر نصب کن، هرگز جایگزین مستقیم.

روش سنتی (شبیه‌سازی محیط LPIC، مستقل از توزیع):

```bash
tar xf linux-<version>.tar.xz
cd linux-<version>/
make menuconfig        # پیکربندی تعاملی متنی (ncurses)
# سایر make targetها:
#   make config      → سؤال‌محور، خط به خط
#   make xconfig     → رابط گرافیکی Qt
#   make gconfig     → رابط گرافیکی GTK
#   make oldconfig   → استفاده از تنظیمات قبلی + سؤال فقط برای گزینه‌های جدید
#   make mrproper    → پاک‌سازی کامل، شامل فایل .config
make -j$(nproc)         # کامپایل با استفاده از همه‌ی هسته‌ها
sudo make modules_install
sudo make install
```
- فایل `.config` در ریشه‌ی سورس، تمام گزینه‌های فعال/غیرفعال کرنل را نگه می‌دارد.
- بعد از کامپایل، ماژول‌ها در `/lib/modules/<kernel-version>/` قرار می‌گیرند.
- برای ساخت initramfs (تصویر ابتدایی رم لازم برای بوت):
```bash
sudo mkinitcpio -p linux-custom     # روش Arch (معادل mkinitrd/mkinitramfs در توزیع‌های دیگر)
```
- ابزار **DKMS** (Dynamic Kernel Module Support) برای کامپایل خودکار ماژول‌های خارج از درخت کرنل (مثل درایورهای انویدیا) هنگام ارتقای کرنل استفاده می‌شود:
```bash
sudo pacman -S dkms
```

### 201.3 — مدیریت runtime و عیب‌یابی

```bash
lsmod                        # لیست ماژول‌های بارگذاری‌شده
modinfo <module_name>        # اطلاعات کامل درباره یک ماژول (پارامترها، وابستگی‌ها، مسیر فایل)
sudo modprobe <module_name>  # بارگذاری هوشمند ماژول (وابستگی‌ها را هم بارگذاری می‌کند)
sudo insmod <path.ko>        # بارگذاری مستقیم فایل ماژول (بدون رفع وابستگی خودکار)
sudo rmmod <module_name>     # حذف ماژول (اگر در حال استفاده نباشد)
sudo depmod -a                # بازسازی نگاشت وابستگی ماژول‌ها (modules.dep)
dmesg | tail -50             # پیام‌های کرنل، از جمله خطاهای بارگذاری درایور
sudo lspci -k                # سخت‌افزار PCI + ماژول کرنل مرتبط با هر کدام
lsusb                        # سخت‌افزار USB
sudo sysctl -a | less        # مشاهده تمام پارامترهای قابل‌تنظیم کرنل در زمان اجرا
sudo sysctl -w net.ipv4.ip_forward=1     # تغییر موقت یک پارامتر
```
تغییرات دائمی در `/etc/sysctl.conf` یا فایل‌های `/etc/sysctl.d/*.conf` ذخیره می‌شوند.

برای عیب‌یابی رویدادهای دستگاه (hotplug و غیره):
```bash
sudo udevadm monitor          # مشاهده زنده‌ی رویدادهای udev
udevadm info -a -n /dev/sda   # اطلاعات کامل درباره یک دستگاه
```
قوانین udev در `/etc/udev/rules.d/` نگهداری می‌شوند.

## ۵) نکات مهم آزمون LPIC

- ✅ تفاوت `modprobe` و `insmod` را دقیق بدان: `modprobe` وابستگی‌ها را خودش حل می‌کند، `insmod` نه.
- ✅ ترتیب صحیح ساخت initramfs بعد از نصب ماژول‌ها را بشناس (اول `modules_install`، بعد initramfs).
- ✅ نام دقیق `make` targetهای مختلف (`oldconfig` در مقابل `menuconfig` در مقابل `mrproper`) سؤال کلاسیک آزمون است.
- ✅ محل‌های کلیدی: `/proc/sys/kernel/`، `/lib/modules/<version>/modules.dep`، `/etc/udev/`.
- ✅ DKMS را فقط در حد آگاهی (چرا لازم است) بشناس، نه پیکربندی عمیق.

## ۶) تمرین عملی امن

> ⚠️ کامپایل کامل کرنل زمان‌بر و پرخطر برای سیستم روزمره است؛ این تمرین را روی یک ماشین مجازی جدا انجام بده، نه سیستم اصلی Omarchy.

1. اطلاعات کرنل فعلی را ببین:
```bash
uname -a
lsmod | head -20
```
2. یک ماژول بی‌خطر را بارگذاری/حذف کن (مثلاً `dummy` که برای تست شبکه است):
```bash
sudo modprobe dummy
lsmod | grep dummy
sudo rmmod dummy
```
3. رویدادهای udev را زنده تماشا کن و یک فلش USB وصل/جدا کن:
```bash
sudo udevadm monitor
```
4. یک پارامتر بی‌خطر sysctl را موقتاً تغییر بده و برگردان:
```bash
sysctl net.ipv4.ip_forward
sudo sysctl -w net.ipv4.ip_forward=1
sudo sysctl -w net.ipv4.ip_forward=0
```

## ۷) خلاصه جدولی

| دستور | کاربرد |
|---|---|
| `lsmod` | لیست ماژول‌های بارگذاری‌شده |
| `modinfo` | جزئیات یک ماژول |
| `modprobe` | بارگذاری هوشمند (با وابستگی) |
| `insmod` / `rmmod` | بارگذاری/حذف مستقیم |
| `depmod -a` | بازسازی نگاشت وابستگی |
| `dmesg` | لاگ پیام‌های کرنل |
| `sysctl` | مشاهده/تغییر پارامترهای runtime |
| `udevadm monitor` | رویدادهای زنده دستگاه |
| `mkinitcpio` (Arch) | ساخت initramfs |
| `dkms` | کامپایل خودکار ماژول‌های خارجی |

## ۸) مایندمپ متنی

```
Topic 201: Linux Kernel
├── 201.1 Kernel Components
│   └── /usr/src/linux/Documentation, zImage, bzImage
├── 201.2 Compiling
│   ├── menuconfig / oldconfig / mrproper
│   ├── make → modules_install → install
│   ├── mkinitcpio / mkinitrd / mkinitramfs
│   └── dkms (external modules)
└── 201.3 Runtime & Troubleshooting
    ├── lsmod / modinfo / modprobe / insmod / rmmod
    ├── /proc/sys/kernel/, sysctl
    ├── dmesg, lspci -k, lsusb
    └── udevadm monitor, /etc/udev/
```

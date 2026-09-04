---
title: "Topic 206 — System Maintenance"
exam: LPIC-2 / 201-450
weights: "206.1 (2) + 206.2 (3) + 206.3 (1)"
os_target: "Arch Linux / Omarchy"
tags: [lpic2, backup, packages, sourcecompile]
---

# Topic 206: System Maintenance

## ۱) مفهوم کلی

سه بخش:
- **206.1** طراحی و اجرای استراتژی پشتیبان‌گیری (backup)
- **206.2** ابزارهای نگهداری سیستم (نصب از سورس، نصب سیستم به‌صورت خودکار، مانیتورینگ)
- **206.3** ابزارهای اطلاع‌رسانی کاربران (notification قبل از maintenance)

## ۲) چرا این مبحث مهم است؟

این دقیقاً همان درسی است که در خلاصه‌ی خودت هم اشاره کردی که یک‌بار Vault‌ات را از دست دادی چون بک‌آپ نداشتی. بک‌آپ بدون استراتژی (نه فقط اجرا، بلکه دانستن full/incremental/differential، دانستن کجا نگه‌داری کنی، و دانستن چطور restore کنی) یکی از حیاتی‌ترین مهارت‌های یک ادمین است — چون تنها زمانی که واقعاً اهمیتش را می‌فهمی، دیگر دیر شده.

## ۳) مثال‌های واقعی + روی سیستم خودم

**واقعی:** یک شرکت هر شب backup می‌گیرد اما هیچ‌وقت restore را تست نمی‌کند؛ روزی که سرور می‌سوزد، می‌فهمند فایل‌های backup خراب بوده‌اند. قانون طلایی: **backup که تست نشده، backup نیست**.

**روی Omarchy:** خودت الان با Obsidian Git این الگو را عملاً پیاده کرده‌ای (commit خودکار هر ۱۰ دقیقه) — این Topic دقیقاً پشت صحنه‌ی نظری همین کاری‌ست که داری انجام می‌دهی، ولی برای کل سیستم‌فایل، نه فقط یک Vault.

## ۴) دستورات کامل

### 206.1 — استراتژی Backup

```bash
tar czf backup_full.tar.gz /home/user/           # backup کامل، فشرده با gzip
tar czf backup_incr.tar.gz --newer-mtime='2026-09-01' /home/user/   # فقط فایل‌های تغییریافته
rsync -avz --delete /home/user/ /mnt/backup/home/    # همگام‌سازی افزایشی سریع
rsync -avz -e ssh /home/user/ user@remote:/backup/home/   # backup روی سرور دیگر
dd if=/dev/sda of=/mnt/backup/disk.img bs=4M status=progress   # ایمیج کامل دیسک
```
> ⚠️ **هشدار بسیار جدی درباره `dd`:** جهت `if=` (ورودی) و `of=` (خروجی) را همیشه دوبار چک کن. برعکس کردن این دو یعنی **پاک شدن کامل دیسک مبدأ**. هرگز این دستور را عجولانه اجرا نکن.

انواع backup که باید مفهومی بلد باشی:
- **Full** — کپی کامل همه‌چیز؛ کندترین اما ساده‌ترین restore
- **Incremental** — فقط تغییرات از آخرین backup (چه full چه incremental قبلی)؛ سریع‌ترین backup، restore پیچیده‌تر (نیاز به زنجیره کامل)
- **Differential** — فقط تغییرات از آخرین backup **full**؛ حد وسط سرعت backup و سادگی restore

ابزار سنتی‌تر: `cpio` (برای آرشیو ساختار خاص، هنوز در برخی اسکریپت‌های legacy دیده می‌شود):
```bash
find /home/user -depth -print | cpio -ov > backup.cpio
```

### 206.2 — ابزارهای نگهداری سیستم

نصب از سورس (وقتی پکیج در مخزن موجود نیست):
```bash
./configure --prefix=/usr/local
make
sudo make install
```
مدیریت پکیج‌ها روی Arch (متفاوت از rpm/dpkg که آزمون هم می‌پرسد):
```bash
sudo pacman -Syu              # بروزرسانی کامل سیستم
sudo pacman -Qdt               # پکیج‌های orphan (بدون وابستگی دیگر)
sudo pacman -Rns $(pacman -Qdtq)   # پاک‌سازی orphanها
pacman -Qi <package>            # اطلاعات کامل یک پکیج نصب‌شده
```
معادل‌های دیگر توزیع‌ها (آزمون‌محور، باید بشناسی):
```bash
rpm -qa            # لیست همه پکیج‌های نصب‌شده (Red Hat)
dpkg -l              # لیست پکیج‌ها (Debian)
apt-get autoremove   # پاک‌سازی وابستگی‌های اضافی (Debian)
```
نصب خودکار سیستم‌عامل (بدون تعامل انسانی — برای دیپلوی انبوه سرورها):
- **Kickstart** (Red Hat/CentOS) — فایل پاسخ خودکار نصب
- **preseed** (Debian/Ubuntu) — معادل Kickstart در دنیای دبیان
این‌ها فقط نیاز به شناخت مفهومی دارند، نه پیاده‌سازی کامل.

مانیتورینگ سیستم (سطح آگاهی؛ عمیقاً در Topic 200 هم دیدیم):
```bash
sudo pacman -S nagios     # فقط نمونه‌ای برای آشنایی، تنظیم آن خارج از محدوده آزمون
```

### 206.3 — اطلاع‌رسانی به کاربران

```bash
sudo wall "سیستم تا ۱۰ دقیقه دیگر برای نگهداری ری‌استارت می‌شود!"
echo "Maintenance at midnight" | sudo tee /etc/motd
sudo shutdown -h +10 "سرور به‌زودی خاموش می‌شود"
sudo shutdown -c            # لغو یک shutdown زمان‌بندی‌شده
```
`/etc/motd` — پیامی که هنگام هر لاگین نمایش داده می‌شود (Message Of The Day). `/etc/issue` و `/etc/issue.net` — پیام قبل از پرامپت لاگین (محلی و از راه شبکه، به ترتیب).

## ۵) نکات مهم آزمون LPIC

- ✅ تفاوت مفهومی full/incremental/differential را حتماً با مثال عددی تمرین کن (سؤال محاسباتی رایج است).
- ✅ `wall` به تمام ترمینال‌های لاگین‌شده پیام می‌فرستد؛ `/etc/motd` فقط هنگام لاگین جدید نمایش داده می‌شود.
- ✅ ترتیب `if=`/`of=` در `dd` را با دقت کامل حفظ کن.
- ✅ Kickstart = Red Hat، preseed = Debian — این جفت‌سازی سؤال رایج آزمون است.
- ✅ فرق `/etc/issue` (محلی) و `/etc/issue.net` (telnet/شبکه) را بدان.

## ۶) تمرین عملی امن

1. یک backup کامل امن از یک پوشه تستی بگیر:
```bash
mkdir -p ~/test_backup_src
echo "test file" > ~/test_backup_src/file1.txt
tar czf ~/test_full.tar.gz ~/test_backup_src/
```
2. یک فایل جدید اضافه کن و incremental بگیر:
```bash
echo "new file" > ~/test_backup_src/file2.txt
tar czf ~/test_incr.tar.gz --newer-mtime="1 minute ago" ~/test_backup_src/
```
3. با rsync یک نسخه همگام‌سازی‌شده بساز:
```bash
mkdir -p ~/test_backup_dst
rsync -avz ~/test_backup_src/ ~/test_backup_dst/
```
4. **تست restore** (مهم‌ترین قدم که خیلی‌ها فراموش می‌کنند!):
```bash
mkdir -p ~/test_restore
tar xzf ~/test_full.tar.gz -C ~/test_restore/
diff -r ~/test_backup_src ~/test_restore/home/$USER/test_backup_src
```
5. یک پیام اطلاع‌رسانی بی‌خطر به خودت بفرست:
```bash
wall "این فقط یک تست بی‌خطر است"
```
6. پاک‌سازی:
```bash
rm -rf ~/test_backup_src ~/test_backup_dst ~/test_restore ~/test_full.tar.gz ~/test_incr.tar.gz
```

## ۷) خلاصه جدولی

| مفهوم | دستور/فایل |
|---|---|
| Backup کامل فشرده | `tar czf` |
| همگام‌سازی افزایشی | `rsync -avz` |
| ایمیج کامل دیسک | `dd if=... of=...` |
| نصب از سورس | `./configure && make && make install` |
| پاک‌سازی orphan (Arch) | `pacman -Rns $(pacman -Qdtq)` |
| نصب خودکار سیستم | Kickstart (RH) / preseed (Debian) |
| پیام به همه کاربران | `wall` |
| پیام لاگین | `/etc/motd`, `/etc/issue`, `/etc/issue.net` |
| خاموشی زمان‌بندی‌شده | `shutdown -h +N`, `shutdown -c` |

## ۸) مایندمپ متنی

```
Topic 206: System Maintenance
├── 206.1 Backup Strategy
│   ├── full / incremental / differential
│   ├── tar, rsync, dd, cpio
│   └── restore testing (حیاتی!)
├── 206.2 Maintenance Tools
│   ├── ./configure && make && make install
│   ├── pacman / rpm / dpkg
│   └── Kickstart / preseed (awareness)
└── 206.3 User Notification
    ├── wall
    ├── /etc/motd
    └── /etc/issue, /etc/issue.net
```

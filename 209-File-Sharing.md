---
title: "Topic 209 — File Sharing"
exam: LPIC-2 / 202-450
weights: "209.1 (5) + 209.2 (3)"
os_target: "Arch Linux / Omarchy"
tags: [lpic2, samba, nfs]
---

# Topic 209: File Sharing

## ۱) مفهوم کلی

دو بخش:
- **209.1** پیکربندی سرور Samba (اشتراک فایل با کلاینت‌های ویندوزی و لینوکسی، پروتکل SMB/CIFS)
- **209.2** پیکربندی سرور NFS (اشتراک فایل استاندارد بین سیستم‌های یونیکس/لینوکس)

## ۲) چرا این مبحث مهم است؟

هر سازمانی که کلاینت‌های ویندوز و لینوکس را با هم دارد، به Samba نیاز دارد — این پل ارتباطی بین دو دنیای کاملاً متفاوت پروتکل فایل است. NFS هم استاندارد صنعتی برای اشتراک فایل بین سرورهای لینوکسی/یونیکسی است (مثلاً یک storage مرکزی که چند وب‌سرور آن را mount می‌کنند). از منظر امنیتی، هر دو پروتکل اگر نادرست پیکربندی شوند، می‌توانند فایل‌های حساس را به هر کسی روی شبکه در دسترس بگذارند — یکی از رایج‌ترین اشتباهات ادمین‌های تازه‌کار، share کردن با دسترسی write برای همه (`guest ok = yes` + `writable = yes` بدون محدودیت).

## ۳) مثال‌های واقعی + روی سیستم خودم

**واقعی:** یک سازمان با ۵۰ کامپیوتر ویندوزی و ۱۰ سرور لینوکسی از Samba استفاده می‌کند تا پوشه‌ی «اسناد مشترک» روی هر دو سیستم‌عامل قابل‌دسترسی باشد؛ همزمان همان سرورهای لینوکسی از NFS برای اشتراک فضای storage با یکدیگر استفاده می‌کنند.

**روی Omarchy:** می‌توانی Samba را نصب کنی و یک share تستی بسازی که حتی از موبایل یا یک ویندوز مجازی (VM) قابل‌مشاهده باشد — تمرین کاملاً امن است چون خودت کنترل کامل روی share را داری.

## ۴) دستورات کامل

### 209.1 — Samba

```bash
sudo pacman -S samba
sudo systemctl enable --now smb nmb
testparm                       # اعتبارسنجی فایل تنظیمات Samba (بسیار مهم قبل از restart)
sudo smbpasswd -a username     # ساخت کاربر Samba (جدا از کاربر سیستم لینوکس)
sudo smbstatus                 # نمایش اتصالات فعال و فایل‌های قفل‌شده
```
فایل تنظیمات اصلی: `/etc/samba/smb.conf`
```ini
[global]
   workgroup = WORKGROUP
   security = user
   map to guest = Bad User

[shared]
   path = /srv/samba/shared
   browsable = yes
   writable = yes
   guest ok = no
   valid users = @staff
```
> ⚠️ **هشدار امنیتی:** `guest ok = yes` یعنی هرکسی بدون رمز عبور می‌تواند به share متصل شود. همیشه `valid users` را برای share های حساس محدود کن.

پارامترهای کلیدی: `security = user` (احراز هویت با کاربر/رمز، رایج‌ترین حالت)، `browsable` (نمایش در لیست شبکه یا نه)، `writable` (اجازه نوشتن).

از سمت کلاینت لینوکسی:
```bash
smbclient -L //server_ip -U username    # لیست share های موجود
smbclient //server_ip/shared -U username
sudo mount -t cifs //server_ip/shared /mnt/samba -o username=user,password=pass
```
در fstab برای mount خودکار:
```
//server_ip/shared  /mnt/samba  cifs  username=user,password=pass,uid=1000  0  0
```
> ⚠️ نوشتن پسورد به‌صورت plain-text در fstab ناامن است؛ روش امن‌تر استفاده از فایل credentials جدا با دسترسی محدود (`chmod 600`) است:
```
//server_ip/shared  /mnt/samba  cifs  credentials=/etc/samba/creds,uid=1000  0  0
```

### 209.2 — NFS

```bash
sudo pacman -S nfs-utils
sudo systemctl enable --now nfs-server
```
فایل تنظیمات export: `/etc/exports`
```
/srv/nfs/shared   192.168.1.0/24(rw,sync,no_subtree_check)
/srv/nfs/readonly 192.168.1.0/24(ro,sync,no_root_squash)
```
> ⚠️ **هشدار امنیتی مهم:** `no_root_squash` یعنی کاربر root روی کلاینت، روی سرور NFS هم root می‌ماند — این یک ریسک امنیتی بزرگ است چون یک کلاینت compromise‌شده می‌تواند با دسترسی root به فایل‌های سرور دستکاری کند. پیش‌فرض امن `root_squash` است (root کلاینت را به یک کاربر بی‌اختیار مثل `nobody` تنزل می‌دهد).

```bash
sudo exportfs -ra              # اعمال مجدد تنظیمات /etc/exports بدون restart کامل
sudo exportfs -v                # نمایش exportهای فعال
showmount -e server_ip          # از سمت کلاینت: دیدن exportهای در دسترس یک سرور
```
از سمت کلاینت:
```bash
sudo mount -t nfs server_ip:/srv/nfs/shared /mnt/nfs
```
در fstab:
```
server_ip:/srv/nfs/shared  /mnt/nfs  nfs  defaults  0  0
```
گزینه‌های کلیدی export: `rw`/`ro` (خواندن/نوشتن)، `sync`/`async` (sync ایمن‌تر ولی کندتر — تغییرات قبل از پاسخ به کلاینت روی دیسک نوشته می‌شوند)، `no_subtree_check` (کاهش overhead بررسی امنیتی زیرشاخه، توصیه‌شده برای اکثر موارد).

## ۵) نکات مهم آزمون LPIC

- ✅ همیشه `testparm` را قبل از restart سرویس Samba اجرا کن — دقیقاً مثل `nginx -t`/`apachectl configtest` در Topic قبل.
- ✅ فرق `root_squash` (امن، پیش‌فرض) و `no_root_squash` (خطرناک) را دقیق بدان — سؤال کلاسیک امنیتی آزمون.
- ✅ کاربران Samba جدا از کاربران سیستم مدیریت می‌شوند (`smbpasswd`)، حتی اگر نام یکسان داشته باشند.
- ✅ `exportfs -ra` برای اعمال تغییرات `/etc/exports` بدون قطع اتصالات فعلی — نکته عملی رایج.
- ✅ `showmount -e` ابزار کلاینت برای کشف exportهای یک سرور NFS است.
- ✅ نسخه‌های NFS (NFSv3 در مقابل NFSv4) از نظر مفهومی متفاوتند: NFSv4 نیازی به `portmapper`/`rpcbind` جداگانه ندارد و امنیت بهتری دارد — فقط باید بدانی این تفاوت وجود دارد.

## ۶) تمرین عملی امن

> این تمرین کاملاً روی `localhost`/شبکه محلی خودت انجام می‌شود.

**Samba:**
```bash
sudo pacman -S samba
mkdir -p ~/samba_test_share
sudo tee -a /etc/samba/smb.conf <<'EOF'

[test]
   path = /home/YOUR_USER/samba_test_share
   browsable = yes
   writable = yes
   guest ok = no
   valid users = YOUR_USER
EOF
testparm
sudo smbpasswd -a $USER
sudo systemctl enable --now smb nmb
smbclient -L //localhost -U $USER
```

**NFS:**
```bash
sudo pacman -S nfs-utils
mkdir -p ~/nfs_test_share
echo "$HOME/nfs_test_share 127.0.0.1(rw,sync,no_subtree_check)" | sudo tee -a /etc/exports
sudo systemctl enable --now nfs-server
sudo exportfs -ra
sudo exportfs -v
showmount -e localhost
sudo mkdir -p /mnt/nfs_test
sudo mount -t nfs localhost:$HOME/nfs_test_share /mnt/nfs_test
ls /mnt/nfs_test
sudo umount /mnt/nfs_test
```

## ۷) خلاصه جدولی

| مفهوم | Samba | NFS |
|---|---|---|
| فایل تنظیمات | `/etc/samba/smb.conf` | `/etc/exports` |
| اعتبارسنجی | `testparm` | (اعتبارسنجی ندارد، مستقیم `exportfs`) |
| اعمال تغییرات بدون قطع | `smbcontrol` یا restart سبک | `exportfs -ra` |
| کاربران | `smbpasswd -a` (جدا از سیستم) | بر اساس UID سیستم |
| مشاهده اتصالات | `smbstatus` | `showmount -e` |
| ریسک امنیتی کلیدی | `guest ok = yes` | `no_root_squash` |
| کلاینت mount | `mount -t cifs` | `mount -t nfs` |

## ۸) مایندمپ متنی

```
Topic 209: File Sharing
├── 209.1 Samba
│   ├── smb.conf: [global], [share]
│   ├── smbpasswd, testparm, smbstatus
│   ├── security = user
│   └── mount -t cifs (fstab: credentials=)
└── 209.2 NFS
    ├── /etc/exports (rw/ro, sync/async, root_squash)
    ├── exportfs -ra / -v
    ├── showmount -e
    └── mount -t nfs
```

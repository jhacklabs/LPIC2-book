---
title: "Topic 210 — Network Client Management"
exam: LPIC-2 / 202-450
weights: "210.1 (2) + 210.2 (3) + 210.3 (2) + 210.4 (4)"
os_target: "Arch Linux / Omarchy"
tags: [lpic2, ldap, pam, nsswitch]
---

# Topic 210: Network Client Management

## ۱) مفهوم کلی

چهار بخش:
- **210.1** مفاهیم پایه‌ی DHCP (که در Topic 205 عمیق دیدیم؛ اینجا از زاویه client management)
- **210.2** پیکربندی PAM (Pluggable Authentication Modules — چارچوب احراز هویت لینوکس)
- **210.3** پیکربندی LDAP کلاینت (اتصال به یک دایرکتوری مرکزی کاربران)
- **210.4** استفاده از LDAP برای احراز هویت متمرکز (NSS + PAM با LDAP)

## ۲) چرا این مبحث مهم است؟

در یک سازمان با ۵۰۰ کارمند، هیچ ادمینی نمی‌خواهد روی هر سرور جداگانه کاربر بسازد. LDAP این مشکل را حل می‌کند: یک دایرکتوری مرکزی کاربران که همه سرورها به آن رجوع می‌کنند. PAM هم چارچوبی است که تعیین می‌کند «چطور» یک کاربر احراز هویت شود — چه از فایل local، چه از LDAP، چه با دو-عاملی. این Topic دقیقاً همان مکانیزمی است که پشت **Single Sign-On** (SSO) سازمانی قرار دارد — یکی از مهم‌ترین مفاهیم امنیتی و عملیاتی دنیای واقعی.

## ۳) مثال‌های واقعی + روی سیستم خودم

**واقعی:** یک شرکت وقتی کارمندی اخراج می‌شود، فقط یک اکانت را در LDAP غیرفعال می‌کند و بلافاصله دسترسی او به تمام ۲۰۰ سرور قطع می‌شود — بدون این مکانیزم، باید روی هر سرور جداگانه اکانت را حذف کنند (فراموشی = ریسک امنیتی باقی‌مانده).

**روی Omarchy:** پیاده‌سازی کامل LDAP روی یک لپ‌تاپ شخصی معمولاً کاربرد عملی روزمره ندارد، اما برای یادگیری می‌توانی OpenLDAP را در یک container/VM نصب کنی و کلاینت لینوکس اصلی‌ات را به آن متصل کنی — دقیقاً شبیه‌سازی محیط سازمانی واقعی.

## ۴) دستورات کامل

### 210.2 — PAM

فایل‌های تنظیمات: `/etc/pam.d/` — یک فایل به‌ازای هر سرویس (`sshd`, `login`, `sudo`, …). ساختار یک خط PAM:
```
type   control    module-path    arguments
```
مثال از `/etc/pam.d/sshd`:
```
auth      required     pam_unix.so
auth      required     pam_env.so
account   required     pam_unix.so
password  required     pam_unix.so
session   required     pam_limits.so
```
انواع `type`: **auth** (احراز هویت هویت کاربر)، **account** (بررسی مجاز بودن اکانت — منقضی نشده باشد)، **password** (مدیریت تغییر رمز)، **session** (تنظیمات قبل/بعد از session، مثل mount کردن home).

انواع `control`: **required** (باید موفق شود، ولی بقیه ماژول‌ها هم اجرا می‌شوند)، **requisite** (باید موفق شود، در صورت شکست فوراً متوقف می‌شود)، **sufficient** (اگر موفق شود کافی‌ست، نیازی به بقیه نیست)، **optional** (فقط در صورت نبود ماژول دیگری تأثیرگذار است).

```bash
sudo pacman -S pam
cat /etc/pam.d/sudo
sudo pacman -S libpwquality     # اعمال سیاست پیچیدگی رمز عبور
```
مثال محدودسازی تلاش‌های ناموفق لاگین (pam_tally2/pam_faillock):
```
auth required pam_faillock.so preauth silent deny=5 unlock_time=900
```

### 210.3 — LDAP کلاینت

```bash
sudo pacman -S openldap
ldapsearch -x -H ldap://ldap.example.com -b "dc=example,dc=com"
ldapsearch -x -D "cn=admin,dc=example,dc=com" -W -b "dc=example,dc=com" "(uid=john)"
```
- `-x` — احراز هویت ساده (simple bind) به‌جای SASL
- `-D` — DN کاربری که با آن bind می‌کنی (Distinguished Name)
- `-W` — درخواست رمز عبور به‌صورت تعاملی
- `-b` — base DN جایی که جستجو از آن شروع می‌شود

مفاهیم پایه LDAP: **DN** (Distinguished Name — مسیر یکتای کامل یک entry، مثل `uid=john,ou=people,dc=example,dc=com`)، **DIT** (Directory Information Tree — ساختار درختی کل دایرکتوری)، **objectClass** (تعیین می‌کند یک entry چه صفت‌هایی می‌تواند داشته باشد).

فایل تنظیمات کلاینت LDAP: `/etc/openldap/ldap.conf` یا `/etc/ldap/ldap.conf`
```
BASE   dc=example,dc=com
URI    ldap://ldap.example.com
```

### 210.4 — احراز هویت متمرکز با LDAP (NSS + PAM)

```bash
sudo pacman -S nss-pam-ldapd
```
فایل تنظیمات NSS: `/etc/nsswitch.conf`
```
passwd: files ldap
group:  files ldap
shadow: files ldap
```
این خط به سیستم می‌گوید: اول در فایل‌های محلی (`/etc/passwd`) بگرد، اگر پیدا نشد به LDAP مراجعه کن — این ترتیب برای عملکرد و پایداری بسیار مهم است (کاربران local همیشه باید قابل‌دسترسی بمانند، حتی اگر LDAP از دسترس خارج شود).

فایل تنظیمات nslcd (سرویس واسط بین NSS و LDAP):
```
# /etc/nslcd.conf
uri ldap://ldap.example.com
base dc=example,dc=com
```
```bash
sudo systemctl enable --now nslcd
```
پیکربندی PAM برای استفاده از LDAP (`/etc/pam.d/system-auth` یا مشابه):
```
auth     sufficient   pam_ldap.so
auth     required     pam_unix.so try_first_pass
account  sufficient   pam_ldap.so
account  required     pam_unix.so
```
> ⚠️ **هشدار مهم:** ترتیب و کلمه `control` (`sufficient` در مقابل `required`) در این خطوط تعیین می‌کند که آیا سیستم در صورت خرابی LDAP همچنان به کاربران local اجازه ورود می‌دهد یا کاملاً قفل می‌شود. همیشه یک fallback به `pam_unix.so` نگه دار تا در صورت قطعی LDAP، حداقل root/کاربران local بتوانند وارد شوند.

## ۵) نکات مهم آزمون LPIC

- ✅ ترتیب دقیق فیلدهای یک خط PAM (`type control module arguments`) و معنای هر `control` را کامل حفظ کن — سنگین‌ترین بخش این Topic در آزمون.
- ✅ در `nsswitch.conf`، ترتیب `files` قبل از `ldap` برای پایداری سیستم حیاتی است.
- ✅ گزینه‌های `ldapsearch` (`-x`, `-D`, `-W`, `-b`, `-H`) را از حفظ باش — سؤال عملی رایج.
- ✅ فرق DN و DIT را با مثال درخت دایرکتوری تمرین کن.
- ✅ همیشه یک fallback احراز هویت local نگه‌داشتن در PAM را به‌عنوان best practice امنیتی بشناس.

## ۶) تمرین عملی امن

> راه‌اندازی کامل LDAP سازمانی خارج از محدوده یک تمرین ساده است؛ این تمرین روی مفاهیم قابل‌مشاهده محلی تمرکز دارد.

1. ساختار فایل‌های PAM را بررسی کن (فقط خواندن، بدون تغییر):
```bash
cat /etc/pam.d/sudo
cat /etc pam.d/login 2>/dev/null || cat /etc/pam.d/system-login
```
2. `nsswitch.conf` فعلی‌ات را ببین:
```bash
cat /etc/nsswitch.conf
```
3. اگر می‌خواهی LDAP را عملاً تجربه کنی (توصیه‌شده در یک container ایزوله، نه سیستم اصلی):
```bash
# روی یک ماشین/کانتینر جدا:
docker run -d -p 389:389 --name openldap-test osixia/openldap:latest
ldapsearch -x -H ldap://localhost -b "dc=example,dc=org"
```
> ⚠️ این مرحله را فقط در یک container جدا امتحان کن، نه با تغییر تنظیمات احراز هویت سیستم اصلی — تغییر نادرست PAM/NSS سیستم می‌تواند تو را از سیستم خودت قفل کند.

## ۷) خلاصه جدولی

| مفهوم | فایل/دستور |
|---|---|
| قوانین PAM | `/etc/pam.d/<service>` |
| انواع type | auth, account, password, session |
| انواع control | required, requisite, sufficient, optional |
| جستجوی LDAP | `ldapsearch -x -D ... -W -b ...` |
| ترتیب منابع کاربر/گروه | `/etc/nsswitch.conf` (files قبل از ldap) |
| واسط NSS↔LDAP | `nslcd` + `/etc/nslcd.conf` |
| مسیر یکتا LDAP | DN (Distinguished Name) |
| ساختار درختی LDAP | DIT |

## ۸) مایندمپ متنی

```
Topic 210: Network Client Management
├── 210.1 DHCP (recap از Topic 205)
├── 210.2 PAM
│   ├── /etc/pam.d/<service>
│   ├── type: auth/account/password/session
│   └── control: required/requisite/sufficient/optional
├── 210.3 LDAP Client
│   ├── ldapsearch -x -D -W -b
│   ├── DN, DIT, objectClass
│   └── /etc/openldap/ldap.conf
└── 210.4 Centralized Auth (NSS+PAM+LDAP)
    ├── /etc/nsswitch.conf (files → ldap)
    ├── nslcd + /etc/nslcd.conf
    └── pam_ldap.so + pam_unix.so (fallback!)
```

---
title: "Topic 207 — Domain Name Server"
exam: LPIC-2 / 202-450
weights: "207.1 (3) + 207.2 (3) + 207.3 (2)"
os_target: "Arch Linux / Omarchy"
tags: [lpic2, dns, bind, chroot]
---

# Topic 207: Domain Name Server

## ۱) مفهوم کلی

سه بخش:
- **207.1** پایه‌های DNS و پیکربندی سرور اصلی (BIND — نرم‌افزار غالب دنیای لینوکس برای DNS)
- **207.2** ایجاد و نگهداری zone فایل‌های DNS
- **207.3** امن‌سازی سرور DNS (chroot، محدودسازی transfer، کنترل دسترسی)

## ۲) چرا این مبحث مهم است؟

DNS «دفترچه تلفن اینترنت» است — بدون آن، هیچ‌کس نمی‌تواند `google.com` را به IP آن ترجمه کند. اگر سرور DNS شرکتت پایین بیاید، عملاً کل اینترنت داخلی سازمان از کار می‌افتد، حتی اگر همه سرویس‌های دیگر سالم باشند. از منظر امنیتی، DNS یکی از پرحمله‌ترین سرویس‌ها است: **DNS cache poisoning**، **zone transfer** غیرمجاز (افشای کل نقشه شبکه داخلی سازمان به مهاجم)، و **DNS amplification attacks** (استفاده از DNS برای تقویت حملات DDoS) همگی از همین جا شروع می‌شوند.

## ۳) مثال‌های واقعی + روی سیستم خودم

**واقعی:** یک مهاجم با یک درخواست AXFR (zone transfer) ساده، کل لیست ساب‌دامین‌ها و IPهای داخلی یک شرکت را دریافت می‌کند چون ادمین فراموش کرده transfer را به IPهای مجاز محدود کند — این دقیقاً بخش 207.3 است.

**روی Omarchy:** BIND (پکیج `bind` در Arch) دقیقاً همان نرم‌افزاری‌ست که آزمون می‌پرسد؛ نصب و تست آن روی لپ‌تاپ شخصی کاملاً امن است چون پیش‌فرض فقط روی localhost گوش می‌دهد.

## ۴) دستورات کامل

### 207.1 — پایه‌ها و نصب BIND

```bash
sudo pacman -S bind
sudo systemctl enable --now named
named -v
named-checkconf /etc/named.conf     # اعتبارسنجی فایل تنظیمات قبل از reload
sudo rndc reload                     # بارگذاری مجدد بدون قطع سرویس
sudo rndc status
```
فایل تنظیمات اصلی: `/etc/named.conf` (روی برخی توزیع‌ها `/etc/bind/named.conf`). بخش‌های کلیدی این فایل:
```
options {
    directory "/var/named";
    allow-query { any; };
    recursion yes;
};

zone "example.com" IN {
    type master;
    file "example.com.zone";
};

zone "1.168.192.in-addr.arpa" IN {
    type master;
    file "1.168.192.rev";
};
```
انواع نقش سرور DNS: **master** (منبع اصلی zone)، **slave** (کپی از master، sync خودکار)، **caching-only** (فقط جواب‌ها را cache می‌کند، خودش authoritative نیست، رایج‌ترین نقش برای DNS داخلی سازمان‌ها).

### 207.2 — ساخت و نگهداری Zone فایل‌ها

```
; مثال فایل زون /var/named/example.com.zone
$TTL 86400
@   IN  SOA   ns1.example.com. admin.example.com. (
        2026090401  ; Serial (باید هر بار تغییر زون افزایش یابد)
        3600        ; Refresh
        1800        ; Retry
        604800      ; Expire
        86400 )     ; Minimum TTL

    IN  NS    ns1.example.com.
ns1 IN  A     192.168.1.10
www IN  A     192.168.1.20
mail IN A     192.168.1.30
    IN  MX 10 mail.example.com.
ftp IN  CNAME www
```
رکوردهای کلیدی: **A** (نام → IPv4)، **AAAA** (نام → IPv6)، **CNAME** (نام مستعار → نام دیگر)، **MX** (سرور ایمیل، همراه با عدد اولویت)، **NS** (سرور نام معتبر برای این zone)، **SOA** (اطلاعات مدیریتی zone)، **PTR** (IP → نام، برای reverse lookup).

```bash
named-checkzone example.com /var/named/example.com.zone     # اعتبارسنجی زون
dig @localhost example.com                                    # تست مستقیم از سرور خودت
dig axfr example.com @ns1.example.com                          # تست zone transfer
```
> ⚠️ سریال (Serial) فایل زون باید هر بار که زون را تغییر می‌دهی افزایش یابد؛ در غیر این صورت سرورهای slave تغییرات را sync نمی‌کنند — یکی از رایج‌ترین اشتباهات ادمین‌های تازه‌کار.

### 207.3 — امن‌سازی DNS

```
// در named.conf، محدود کردن zone transfer فقط به IPهای مجاز
zone "example.com" {
    type master;
    file "example.com.zone";
    allow-transfer { 192.168.1.20; };
};
```
محدود کردن recursion (برای جلوگیری از سوءاستفاده در DDoS amplification):
```
options {
    recursion yes;
    allow-recursion { 192.168.1.0/24; };
};
```
اجرای BIND در chroot (محدود کردن دسترسی سرویس به یک شاخه ایزوله از فایل‌سیستم، به‌طوری‌که حتی در صورت نفوذ به BIND، مهاجم به بقیه سیستم دسترسی نداشته باشد):
```bash
sudo pacman -S bind-chroot   # (در برخی توزیع‌ها به این نام یا مشابه)
```
> ⚠️ **هشدار امنیتی مهم:** هرگز سرور DNS را با `allow-transfer { any; };` روی اینترنت باز نگذار — این یعنی هرکسی می‌تواند کل نقشه دامنه‌ات را با یک درخواست AXFR دانلود کند.

فایل کلیدهای TSIG برای احراز هویت امن بین master و slave (جلوگیری از جعل zone transfer):
```bash
tsig-keygen -a hmac-sha256 example-key
```

## ۵) نکات مهم آزمون LPIC

- ✅ ترتیب فیلدهای SOA (Serial, Refresh, Retry, Expire, Minimum TTL) را دقیق و به همین ترتیب حفظ کن.
- ✅ فرق master/slave/caching-only را با مثال سناریو تمرین کن.
- ✅ رکورد PTR فقط در zone فایل reverse (`in-addr.arpa`) معنا دارد.
- ✅ `allow-transfer` و `allow-recursion` دو تنظیم امنیتی متفاوتند — آزمون این دو را قاطی می‌کند تا ببیند دقیق بلدی یا نه.
- ✅ ابزار `dig` را برای هر نوع تست (query معمولی، AXFR، trace) کامل بلد باش؛ رایج‌ترین ابزار عملی این Topic در آزمون است.
- ✅ فراموش نکردن افزایش Serial را حتماً حفظ کن — سؤال سناریومحور کلاسیک.

## ۶) تمرین عملی امن

> این تمرین کاملاً ایزوله و امن است چون فقط روی `localhost` تست می‌شود، هیچ ترافیک عمومی درگیر نیست.

1. نصب و راه‌اندازی BIND:
```bash
sudo pacman -S bind
sudo systemctl enable --now named
sudo rndc status
```
2. یک zone تستی بساز (مثلاً `test.local`):
```bash
sudo mkdir -p /var/named
sudo tee /var/named/test.local.zone <<'EOF'
$TTL 86400
@   IN  SOA   ns1.test.local. admin.test.local. (
        2026090401 3600 1800 604800 86400 )
    IN  NS    ns1.test.local.
ns1 IN  A     127.0.0.1
www IN  A     127.0.0.1
EOF
```
3. آن را در `named.conf` اضافه کن (نیاز به دسترسی sudo برای ویرایش):
```
zone "test.local" IN {
    type master;
    file "/var/named/test.local.zone";
};
```
4. اعتبارسنجی و reload:
```bash
named-checkconf /etc/named.conf
named-checkzone test.local /var/named/test.local.zone
sudo rndc reload
```
5. تست از خود سیستم:
```bash
dig @localhost www.test.local
```

## ۷) خلاصه جدولی

| مفهوم | جزئیات |
|---|---|
| نرم‌افزار اصلی | BIND (`named`) |
| فایل تنظیمات | `/etc/named.conf` |
| نقش سرور | master / slave / caching-only |
| رکوردهای زون | A, AAAA, CNAME, MX, NS, SOA, PTR |
| اعتبارسنجی | `named-checkconf`, `named-checkzone` |
| بارگذاری مجدد | `rndc reload` |
| امنیت transfer | `allow-transfer` |
| امنیت recursion | `allow-recursion` |
| احراز هویت master↔slave | TSIG keys |
| ایزوله‌سازی سرویس | chroot |

## ۸) مایندمپ متنی

```
Topic 207: Domain Name Server
├── 207.1 Basic DNS Server Config
│   ├── named.conf (options, zone blocks)
│   └── master / slave / caching-only
├── 207.2 Zone Files
│   ├── SOA (Serial, Refresh, Retry, Expire, TTL)
│   ├── A/AAAA/CNAME/MX/NS/PTR
│   └── named-checkzone, dig
└── 207.3 Securing DNS
    ├── allow-transfer / allow-recursion
    ├── TSIG keys
    └── chroot BIND
```

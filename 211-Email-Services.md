---
title: "Topic 211 — E-Mail Services"
exam: LPIC-2 / 202-450
weights: "211.1 (4) + 211.2 (2) + 211.3 (2)"
os_target: "Arch Linux / Omarchy"
tags: [lpic2, email, postfix, dovecot, spam]
---

# Topic 211: E-Mail Services

## ۱) مفهوم کلی

سه بخش:
- **211.1** استفاده از سرور ایمیل (MTA — Mail Transfer Agent، معمولاً Postfix)
- **211.2** مدیریت تحویل محلی ایمیل (Local Mail Delivery)
- **211.3** مدیریت ایمیل از راه دور (POP/IMAP، معمولاً Dovecot)

## ۲) چرا این مبحث مهم است؟

ایمیل یکی از قدیمی‌ترین و در عین حال حساس‌ترین سرویس‌های اینترنت است — هم از نظر تحویل صحیح پیام و هم از نظر امنیت (فیشینگ، اسپم، جعل هویت فرستنده). فهم عمیق SMTP یعنی فهم اینکه چطور یک ایمیل از سرور فرستنده به سرور گیرنده می‌رسد، کجاها ممکن است رد شود (bounce)، و چطور می‌توان جلوی جعل دامنه (spoofing) را با SPF/DKIM/DMARC گرفت — دانشی که مستقیم به تحلیل حملات فیشینگ در Cyber Security وصل می‌شود.

## ۳) مثال‌های واقعی + روی سیستم خودم

**واقعی:** یک شرکت متوجه می‌شود ایمیل‌های ارسالی‌اش به Gmail در پوشه اسپم می‌افتد؛ بررسی می‌کنند که رکورد SPF دامنه‌شان اشتباه تنظیم شده و سرور SMTP‌شان مجاز به ارسال از طرف آن دامنه شناخته نمی‌شود.

**روی Omarchy:** راه‌اندازی یک MTA کامل مثل Postfix روی یک لپ‌تاپ شخصی معمولاً برای دریافت واقعی ایمیل کاربردی ندارد (چون IP دینامیک خانگی توسط اکثر سرویس‌ها بلاک می‌شود)، اما برای **یادگیری** و شبیه‌سازی محلی (ارسال بین کاربران همان سیستم، یا لاگ کردن هشدارهای cron/سیستم) کاملاً قابل نصب و تست است.

## ۴) دستورات کامل

### 211.1 — استفاده از سرور ایمیل (Postfix)

```bash
sudo pacman -S postfix
sudo systemctl enable --now postfix
```
فایل تنظیمات اصلی: `/etc/postfix/main.cf`

```conf
myhostname = mail.example.com
mydomain = example.com
myorigin = $mydomain
inet_interfaces = all
mydestination = $myhostname, localhost.$mydomain, localhost, $mydomain
relayhost =
mynetworks = 127.0.0.0/8
```
دستورات مدیریتی:
```bash
sudo postfix check                # بررسی صحت تنظیمات
sudo postfix reload
mailq                              # نمایش صف ایمیل‌های در انتظار ارسال
sudo postsuper -d ALL              # حذف تمام ایمیل‌های صف (احتیاط!)
sudo postconf -n                   # نمایش تمام تنظیمات غیرپیش‌فرض فعلی
echo "Test body" | mail -s "Test Subject" user@localhost
```
فایل نگاشت (map) برای مسیردهی و آدرس‌های مجازی:
```bash
sudo postmap /etc/postfix/virtual        # کامپایل فایل متنی به hash دیتابیس (db)
```
مکانیزم‌های ضد جعل دامنه (که هر ادمین ایمیل باید بشناسد، هرچند پیکربندی‌شان معمولاً در DNS انجام می‌شود نه Postfix مستقیماً):
- **SPF** (Sender Policy Framework) — مشخص می‌کند چه سرورهایی مجازند از طرف یک دامنه ایمیل بفرستند (رکورد TXT در DNS)
- **DKIM** (DomainKeys Identified Mail) — امضای دیجیتال هر ایمیل با کلید خصوصی دامنه
- **DMARC** — سیاست می‌گوید با ایمیل‌هایی که SPF/DKIM را رد می‌کنند چه کار کنیم (quarantine/reject)

### 211.2 — تحویل محلی ایمیل (Local Mail Delivery)

MDA (Mail Delivery Agent) رایج: **Procmail** یا **Maildrop** — که ایمیل ورودی را قبل از نشستن در inbox فیلتر/دسته‌بندی می‌کنند.

```bash
sudo pacman -S procmail
```
فایل قوانین کاربر: `~/.procmailrc`
```conf
:0:
* ^Subject:.*invoice
$HOME/Mail/invoices
```
این قانون یعنی: هر ایمیلی که موضوعش شامل «invoice» باشد، به پوشه‌ی `Mail/invoices` منتقل شود، نه inbox اصلی.

فیلتر اسپم رایج: **SpamAssassin**
```bash
sudo pacman -S spamassassin
sudo systemctl enable --now spamassassin
spamassassin -t < email.txt      # تست یک ایمیل خاص برای امتیاز اسپم
sa-learn --spam ~/Mail/spam/     # آموزش دستی الگوریتم بیزی روی نمونه‌های اسپم
sa-learn --ham ~/Mail/inbox/     # آموزش روی نمونه‌های معتبر (ham)
```
فرمت ذخیره‌سازی صندوق پستی که باید بشناسی:
- **mbox** — همه ایمیل‌ها در یک فایل متنی بزرگ
- **Maildir** — هر ایمیل یک فایل جدا در یک ساختار پوشه‌ای (`new/`, `cur/`, `tmp/`) — امروز رایج‌تر و امن‌تر در برابر خرابی همزمان (corruption)

### 211.3 — مدیریت ایمیل از راه دور (Dovecot)

```bash
sudo pacman -S dovecot
sudo systemctl enable --now dovecot
```
فایل تنظیمات اصلی: `/etc/dovecot/dovecot.conf` (که خودش فایل‌های `conf.d/*.conf` را include می‌کند)

```conf
protocols = imap pop3
mail_location = maildir:~/Maildir

ssl = yes
ssl_cert = </etc/letsencrypt/live/example.com/fullchain.pem
ssl_key = </etc/letsencrypt/live/example.com/privkey.pem
```
دستورات تست:
```bash
sudo doveconf -n              # نمایش تنظیمات فعال (غیر پیش‌فرض)
telnet localhost 143          # تست دستی اتصال IMAP (بدون رمزنگاری)
openssl s_client -connect localhost:993   # تست اتصال IMAPS (رمزنگاری‌شده)
```

## ۵) نکات مهم آزمون LPIC

- ✅ فرق POP3 (پورت 110 / 995 با SSL) و IMAP (پورت 143 / 993 با SSL) را دقیق بدان: POP3 ایمیل را دانلود و معمولاً حذف می‌کند، IMAP ایمیل را روی سرور نگه می‌دارد و همگام‌سازی می‌کند.
- ✅ محل فایل اصلی Postfix: `/etc/postfix/main.cf`؛ پارامتر `mydestination` تعیین می‌کند سرور برای کدام دامنه‌ها ایمیل محلی تحویل می‌دهد.
- ✅ ترتیب صحیح: بعد از تغییر فایل‌های نگاشت (map)، همیشه `postmap` بزن، بعد `postfix reload`.
- ✅ فرق mbox و Maildir را حتماً بدان — سؤال کلاسیک آزمون.
- ✅ SpamAssassin از الگوریتم بیزی برای امتیازدهی استفاده می‌کند؛ `sa-learn` نحوه‌ی آموزش آن است.
- ✅ SPF/DKIM/DMARC را در حد مفهوم و محل رکورد (DNS TXT) بدان، نه پیکربندی عمیق DNS.

## ۶) تمرین عملی امن

> ⚠️ Postfix را با `inet_interfaces = all` روی یک سیستم متصل مستقیم به اینترنت باز نگه نداشتن — بدون فایروال درست، سرور می‌تواند open relay شود و برای ارسال اسپم توسط مهاجمان سوءاستفاده گردد. برای تمرین این بخش `inet_interfaces = loopback-only` امن‌تر است.

```bash
sudo pacman -S postfix mailutils
sudo systemctl enable --now postfix
sudo postconf -e "inet_interfaces = loopback-only"
sudo systemctl restart postfix

# ارسال یک ایمیل تست به کاربر محلی خودت
echo "این یک تست است" | mail -s "تست LPIC-2" $USER

# بررسی صف و رسیدن ایمیل
mailq
mail   # مشاهده inbox محلی (ابزار متنی ساده mail)
```

## ۷) خلاصه جدولی

| مفهوم | ابزار/فایل |
|---|---|
| MTA اصلی | Postfix — `/etc/postfix/main.cf` |
| صف ایمیل | `mailq`, `postsuper` |
| کامپایل نگاشت | `postmap` |
| MDA فیلتر ورودی | Procmail — `~/.procmailrc` |
| فیلتر اسپم | SpamAssassin — `sa-learn` |
| فرمت ذخیره | mbox (تک‌فایل) vs Maildir (پوشه‌ای) |
| سرور IMAP/POP3 | Dovecot — `/etc/dovecot/dovecot.conf` |
| ضد جعل دامنه | SPF, DKIM, DMARC (رکورد DNS) |

## ۸) مایندمپ متنی

```
Topic 211: E-Mail Services
├── 211.1 Using a Mail Server (MTA)
│   ├── Postfix: main.cf, mynetworks, mydestination
│   ├── mailq, postsuper, postmap
│   └── SPF / DKIM / DMARC (awareness)
├── 211.2 Local Mail Delivery
│   ├── Procmail / Maildrop: ~/.procmailrc
│   ├── SpamAssassin: sa-learn (Bayesian)
│   └── mbox vs Maildir
└── 211.3 Remote Mail (Dovecot)
    ├── protocols: imap, pop3
    ├── ports: 110/995(SSL), 143/993(SSL)
    └── dovecot.conf, mail_location
```

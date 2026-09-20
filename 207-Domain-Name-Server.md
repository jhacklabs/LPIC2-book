---
title: "Topic 207 — Domain Name Server (نسخه عمیق و کامل)"
exam: LPIC-2 / 202-450
weights: "207.1 (3) + 207.2 (3) + 207.3 (2)"
os_target: "Arch Linux / Omarchy"
tags: [lpic2, dns, bind, deep-dive]
---

# Topic 207: Domain Name Server — راهنمای کامل و عمیق

---

# بخش اول — 207.1: اصول پایه‌ی DNS و اجرای BIND

## ۱.۱) چرا DNS اصلاً وجود دارد؟

کامپیوترها با عدد کار می‌کنند (آدرس IP)، انسان‌ها با اسم راحت‌ترند (google.com). DNS یک سیستم **توزیع‌شده و سلسله‌مراتبی** برای ترجمه‌ی نام به آدرس است — و کلمه‌ی «توزیع‌شده» اینجا کلیدی است: هیچ سرور واحدی همه‌ی دنیای اینترنت را نمی‌شناسد؛ به‌جایش یک سلسله‌مراتب از مسئولیت‌ها وجود دارد.

## ۱.۲) سلسله‌مراتب DNS — از روت تا زیردامنه

```
.  (Root)
│
├── com.
│   ├── google.com.
│   ├── example.com.
│   └── ...
├── org.
├── net.
└── ir.
    └── example.ir.
```

- **Root servers** — بالاترین سطح، فقط ۱۳ مجموعه‌ی سرور روت در دنیا وجود دارد (نام‌گذاری‌شده از a تا m)؛ این‌ها نمی‌دانند IP گوگل چیست، اما می‌دانند سرورهای مسئول `.com.` کجا هستند.
- **TLD servers** (Top-Level Domain) — مسئول یک پسوند مثل `.com`, `.org`, `.ir`؛ می‌دانند سرورهای مسئول `google.com.` کجا هستند.
- **Authoritative servers** — سرور واقعی که رکورد نهایی (IP گوگل) را نگه می‌دارد.

نکته‌ی ظریف مهم: هر نام دامنه از نظر فنی با یک نقطه در انتها تمام می‌شود (`example.com.`) — این نقطه‌ی انتهایی نشان‌دهنده‌ی «روت» است، هرچند در استفاده‌ی روزمره معمولاً حذف می‌شود چون مرورگرها و ابزارها خودکار آن را فرض می‌گیرند.

## ۱.۳) انواع سرور DNS از نظر نقش

- **Authoritative (Primary/Master)** — سرور اصلی که رکوردهای یک دامنه را **می‌سازد و ویرایش می‌کند**؛ منبع حقیقت (source of truth) است.
- **Authoritative (Secondary/Slave)** — یک کپی خودکار از Primary که از طریق فرآیندی به نام **Zone Transfer** رکوردها را دریافت می‌کند؛ برای redundancy (اگر Primary از دسترس خارج شود) و توزیع بار.
- **Caching/Recursive Resolver** — سروری که خودش هیچ رکورد اصیلی ندارد، فقط از طرف کلاینت‌ها **پرس‌وجو می‌کند** (recursion انجام می‌دهد) و نتیجه را برای مدت مشخصی (بر اساس TTL) کش می‌کند تا سرعت پاسخ‌دهی به سؤالات بعدی بالا برود. این چیزی است که معمولاً روی روتر خانه‌ات یا سرورهای عمومی مثل `8.8.8.8` (گوگل) و `1.1.1.1` (کلادفلر) اجرا می‌شود.
- **Forwarding server** — یک سرور که سؤالاتی که خودش نمی‌داند را به یک سرور دیگر (معمولاً یک recursive resolver بزرگ‌تر) **فوروارد** می‌کند، به‌جای این‌که خودش مستقیم از روت شروع به پرس‌وجو کند.

## ۱.۴) نصب BIND و ساختار فایل‌ها

**BIND** (Berkeley Internet Name Domain) رایج‌ترین و قدیمی‌ترین نرم‌افزار سرور DNS در دنیای متن‌باز است.

```bash
sudo pacman -S bind
```
فایل تنظیمات اصلی: `/etc/named.conf` (روی برخی توزیع‌ها `/etc/bind/named.conf`)

```conf
options {
    directory "/var/named";
    listen-on port 53 { any; };
    allow-query { any; };
    recursion yes;
    forwarders { 1.1.1.1; 8.8.8.8; };
};

zone "example.local" IN {
    type master;
    file "example.local.zone";
};

zone "1.168.192.in-addr.arpa" IN {
    type master;
    file "192.168.1.rev";
};
```

بیایید مفاهیم کلیدی این فایل را باز کنیم:
- **`recursion yes/no`** — آیا این سرور اجازه دارد از طرف کلاینت‌های خارجی جست‌وجوی recursive انجام دهد؟ این یک تنظیم امنیتی حیاتی است (توضیح بیشتر در ادامه).
- **`forwarders`** — اگر این سرور نتواند خودش جواب بدهد، این سؤال را به کدام سرور(ها) بفرستد.
- **`zone "1.168.192.in-addr.arpa"`** — این یک **Reverse Zone** است (توضیح در بخش بعد).

```bash
sudo systemctl enable --now named
named-checkconf                  # بررسی صحت syntax فایل تنظیمات اصلی، قبل از reload
named-checkzone example.local /var/named/example.local.zone     # بررسی صحت یک فایل zone خاص
sudo rndc reload                  # بارگذاری مجدد بدون قطع سرویس
sudo rndc reload example.local     # بارگذاری مجدد فقط یک zone خاص
```
`rndc` (Remote Name Daemon Control) ابزار مدیریتی رسمی BIND است — بهتر از `systemctl restart` است چون سرویس را قطع نمی‌کند، فقط تنظیمات را دوباره می‌خواند.

> ⚠️ **هشدار امنیتی جدی درباره‌ی `recursion` و «Open Resolver»:** اگر یک سرور DNS را طوری پیکربندی کنی که `recursion yes` باشد **و** `allow-recursion` را محدود نکنی (یعنی هر کسی از هر جای اینترنت بتواند از آن بخواهد recursive query انجام دهد)، سرور تو یک **Open Resolver** می‌شود. مهاجمان از Open Resolver ها برای حملات **DNS Amplification DDoS** سوءاستفاده می‌کنند: یک درخواست کوچک با IP مبدأ جعلی (spoofed، آدرس قربانی) می‌فرستند، سرور تو یک پاسخ بسیار بزرگ‌تر به قربانی می‌فرستد — و چون هزاران Open Resolver در دنیا وجود دارد، این حجم عظیمی از ترافیک ناخواسته روی قربانی متمرکز می‌شود. **همیشه** روی سرورهای عمومی، recursion را یا کاملاً غیرفعال کن یا فقط برای شبکه‌ی داخلی خودت مجاز کن:
> ```conf
> allow-recursion { 192.168.1.0/24; localhost; };
> ```

## ۱.۵) نکات مهم آزمون برای 207.1

- ✅ سلسله‌مراتب کامل را حفظ کن: Root → TLD → Authoritative.
- ✅ چهار نقش سرور را از هم تفکیک کن: Primary/Master، Secondary/Slave، Caching/Recursive، Forwarding.
- ✅ خطر امنیتی Open Resolver و راه‌حل (`allow-recursion` محدود).
- ✅ `named-checkconf`/`named-checkzone` همیشه قبل از reload واقعی.
- ✅ `rndc reload` به‌جای `restart` برای جلوگیری از قطع سرویس.

---

# بخش دوم — 207.2: ایجاد و نگهداری Zone های DNS

## ۲.۱) آناتومی یک فایل Zone

```
$TTL 86400
@       IN      SOA     ns1.example.local. admin.example.local. (
                        2026091001 ; Serial
                        3600       ; Refresh
                        1800       ; Retry
                        604800     ; Expire
                        86400 )    ; Negative Cache TTL

@       IN      NS      ns1.example.local.
@       IN      NS      ns2.example.local.

@       IN      A       192.168.1.10
www     IN      A       192.168.1.10
mail    IN      A       192.168.1.20
        IN      MX  10  mail.example.local.
ftp     IN      CNAME   www.example.local.
```

### رکورد SOA — عمیق روی هر عدد

**SOA** (Start of Authority) اولین و مهم‌ترین رکورد هر zone است — مشخص می‌کند «چه کسی مسئول این zone است» و پارامترهای همگام‌سازی بین Master/Slave را تعریف می‌کند:

- **Serial** — یک عدد که هر بار zone تغییر می‌کند **باید افزایش پیدا کند** (رایج‌ترین قرارداد: فرمت `YYYYMMDDNN` مثل `2026091001`، یعنی اولین تغییر در ۱۰ سپتامبر ۲۰۲۶). این عدد تنها راهی است که یک سرور Secondary می‌فهمد آیا zone تغییر کرده و نیاز به دریافت نسخه‌ی جدید دارد یا نه.
  > ⚠️ **اشتباه بسیار رایج:** اگر zone را ویرایش کنی ولی Serial را فراموش کنی افزایش دهی، سرورهای Secondary **هرگز متوجه تغییر نمی‌شوند** و همچنان اطلاعات قدیمی را serve می‌کنند — یک باگ کلاسیک که هر ادمین DNS حداقل یک‌بار در زندگی‌اش تجربه کرده.
- **Refresh** — هر چند ثانیه سرور Secondary باید بررسی کند آیا Serial تغییر کرده (با پرسیدن مقدار Serial فعلی از Master)
- **Retry** — اگر تلاش Refresh شکست خورد (مثلاً Master موقتاً در دسترس نبود)، بعد از چند ثانیه دوباره تلاش کند
- **Expire** — اگر Secondary مدت طولانی نتوانست به Master وصل شود، بعد از این مدت باید دیگر داده‌ی خودش را «معتبر» نداند و از serve کردن آن دست بکشد (چون احتمالاً خیلی قدیمی شده)
- **Negative Cache TTL** (پارامتر آخر) — چه مدت یک پاسخ **منفی** (مثلاً «این نام وجود ندارد» — NXDOMAIN) باید کش شود

### انواع رکورد — جدول کامل

| رکورد | معنا |
|---|---|
| **A** | نام → آدرس IPv4 |
| **AAAA** | نام → آدرس IPv6 |
| **CNAME** | نام مستعار → نام دیگر (Canonical Name؛ نکته: یک نام نباید همزمان CNAME و رکورد دیگر داشته باشد) |
| **MX** | سرور ایمیل مسئول این دامنه، همراه با یک **اولویت** (عدد کمتر = اولویت بالاتر) |
| **NS** | کدام سرورها برای این zone مرجع (authoritative) هستند |
| **TXT** | متن دلخواه — امروز پرکاربردترین استفاده: رکوردهای SPF/DKIM/DMARC که در Topic 211 دیدیم |
| **PTR** | برعکس A — آدرس IP → نام (برای Reverse DNS) |
| **SRV** | مکان‌یابی یک سرویس خاص (پروتکل، پورت) — رایج در Active Directory و XMPP |

## ۲.۲) Reverse DNS — چرا `in-addr.arpa` این‌قدر عجیب است

Forward DNS نام را به IP تبدیل می‌کند؛ **Reverse DNS** برعکس آن را انجام می‌دهد — IP را به نام. مسئله این است که سیستم سلسله‌مراتبی DNS برای نام‌هایی طراحی شده که از **جزئی‌تر به کلی‌تر از چپ به راست** می‌روند (`www.example.com` → کلی‌ترش `com` است، در سمت راست). اما آدرس IP برعکس این ساختار را دارد (`192.168.1.10` → کلی‌ترین بخشش `192` سمت چپ است).

راه‌حل هوشمندانه: آدرس IP را **معکوس** می‌کنند و به یک دامنه‌ی خاص به نام `in-addr.arpa` می‌چسبانند:
```
192.168.1.10  →  10.1.168.192.in-addr.arpa
```
حالا این با همان منطق سلسله‌مراتبی معمولی DNS کار می‌کند (`arpa` کلی‌ترین، `192` بعدش، و همین‌طور).

```conf
; در فایل reverse zone برای 192.168.1.0/24
10      IN      PTR     www.example.local.
20      IN      PTR     mail.example.local.
```

**چرا Reverse DNS مهم است؟** بسیاری سرورهای ایمیل، Reverse DNS مبدأ را بررسی می‌کنند و اگر یک IP فرستنده، PTR معتبر (که به یک نام معنادار برمی‌گردد، نه فقط IP خام) نداشته باشد، ایمیل را مستقیم رد یا اسپم علامت می‌زنند — این دقیقاً همان چیزی است که در Topic 211 (E-Mail) درباره‌ی SPF/DKIM دیدیم، مکمل آن.

## ۲.۳) Zone Transfer — چطور Secondary از Master کپی می‌گیرد

```bash
dig axfr example.local @ns1.example.local
```
**AXFR** (Full Zone Transfer) کل محتوای zone را یک‌جا منتقل می‌کند — این معمولاً فقط اولین بار (یا وقتی Serial تغییر بزرگی داشته) اتفاق می‌افتد.

**IXFR** (Incremental Zone Transfer) فقط تفاوت بین Serial قدیمی و جدید را منتقل می‌کند — بهینه‌تر برای zone های بزرگ با تغییرات مکرر کوچک.

> ⚠️ **هشدار امنیتی:** Zone Transfer باید **فقط** به سرورهای Secondary شناخته‌شده مجاز باشد، وگرنه هر کسی می‌تواند کل لیست دامنه‌ها/زیردامنه‌های داخلی سازمان تو را (که می‌تواند اطلاعات ارزشمندی برای یک مهاجم باشد — نقشه‌ی کامل زیرساخت داخلی) دانلود کند:
> ```conf
> zone "example.local" {
>     type master;
>     file "example.local.zone";
>     allow-transfer { 192.168.1.2; };   // فقط IP سرور Secondary مشخص
> };
> ```

## ۲.۴) نکات مهم آزمون برای 207.2

- ✅ هر ۵ پارامتر SOA را حفظ کن: Serial, Refresh, Retry, Expire, Negative Cache TTL — و بدان چرا فراموش کردن افزایش Serial یک باگ رایج است.
- ✅ جدول انواع رکورد (A/AAAA/CNAME/MX/NS/TXT/PTR/SRV) را کامل بشناس.
- ✅ منطق معکوس‌سازی آدرس IP در `in-addr.arpa` را بفهمی، نه فقط حفظ کنی.
- ✅ تفاوت AXFR (کامل) و IXFR (افزایشی).
- ✅ `allow-transfer` همیشه باید محدود شود — سؤال امنیتی رایج.

---

# بخش سوم — 207.3: امن‌سازی سرور DNS

## ۳.۱) DNSSEC — امضای رمزنگاری‌شده برای اعتماد

مشکل بنیادین DNS سنتی: **هیچ تضمینی وجود ندارد** که پاسخی که دریافت می‌کنی واقعاً از سرور مرجع صحیح آمده و در مسیر دستکاری نشده — یک مهاجم می‌تواند با یک حمله‌ی **DNS Spoofing/Cache Poisoning** یک پاسخ جعلی (مثلاً IP یک سایت فیشینگ به‌جای IP واقعی بانک) به قربانی تحویل دهد.

**DNSSEC** (DNS Security Extensions) این را با امضای دیجیتال رکوردها حل می‌کند — هر zone یک جفت کلید (خصوصی/عمومی) دارد، رکوردها با کلید خصوصی امضا می‌شوند، و resolver ها می‌توانند با کلید عمومی (که خودش از طریق زنجیره‌ای از اعتماد تا روت DNS قابل تأیید است) صحت امضا را بررسی کنند.

```bash
dnssec-keygen -a RSASHA256 -b 2048 -n ZONE example.local
```
این دو فایل کلید می‌سازد: یکی خصوصی (`.private`) و یکی عمومی (`.key`). سپس با ابزارهایی مثل `dnssec-signzone`، کل zone امضا می‌شود و رکوردهای جدیدی مثل **RRSIG** (امضای هر رکورد)، **DNSKEY** (کلید عمومی zone)، و **DS** (Delegation Signer — که در zone والد ثبت می‌شود تا زنجیره‌ی اعتماد کامل شود) اضافه می‌شوند.

## ۳.۲) TSIG — امضای امن ارتباط بین سرورها

**TSIG** (Transaction Signature) مکانیزم متفاوتی از DNSSEC است: به‌جای امضای رکوردها برای همه‌ی دنیا، TSIG یک **کلید مشترک متقارن** بین دو سرور خاص (مثلاً Master و Secondary یک zone) برقرار می‌کند تا مطمئن شوند پیام‌های ارتباطی بینشان (مثل Zone Transfer یا بروزرسانی‌های داینامیک) واقعاً از طرف مقابل قانونی آمده و دستکاری نشده — این برای **امنیت ارتباط بین دو سرور مشخص** است، نه برای اعتبارسنجی عمومی جهانی مثل DNSSEC.

```conf
key "transfer-key" {
    algorithm hmac-sha256;
    secret "base64-encoded-secret-here";
};

zone "example.local" {
    allow-transfer { key transfer-key; };
};
```

## ۳.۳) چند اقدام امنیتی عملی دیگر

```conf
version none;                    // مخفی کردن نسخه‌ی BIND از پاسخ به query های "version.bind" — جلوگیری از شناسایی آسان نسخه‌ی آسیب‌پذیر توسط مهاجم
```
```bash
sudo systemd-run --unit=named-chroot --property=RootDirectory=/var/named/chroot /usr/sbin/named
```
اجرای BIND درون یک محیط **chroot** (یا سندباکس مشابه) یعنی حتی اگر یک آسیب‌پذیری در BIND مورد سوءاستفاده قرار گیرد، مهاجم فقط به یک محیط فایل‌سیستم محدود و جدا دسترسی پیدا می‌کند، نه کل سیستم — یک لایه‌ی دفاعی اضافه (defense in depth).

## ۳.۴) نکات مهم آزمون برای 207.3

- ✅ تفاوت بنیادین DNSSEC (اعتبارسنجی عمومی جهانی رکوردها با امضای نامتقارن) و TSIG (اعتماد بین دو سرور خاص با کلید متقارن) را دقیق بدان — سؤال مفهومی رایج.
- ✅ رکوردهای کلیدی DNSSEC: RRSIG, DNSKEY, DS.
- ✅ `version none` به‌عنوان یک اقدام ساده‌ی سخت‌سازی.
- ✅ اجرای BIND در chroot به‌عنوان لایه‌ی دفاعی اضافه.

---

## خلاصه‌ی جدولی نهایی کل Topic 207

| زیرمبحث | مفهوم کلیدی | ابزار/رکورد |
|---|---|---|
| 207.1 | سلسله‌مراتب | Root → TLD → Authoritative |
| 207.1 | نقش‌های سرور | Primary/Secondary/Recursive/Forwarding |
| 207.1 | خطر امنیتی | Open Resolver → DNS Amplification DDoS |
| 207.2 | SOA | Serial(!), Refresh, Retry, Expire, NegCacheTTL |
| 207.2 | انواع رکورد | A/AAAA/CNAME/MX/NS/TXT/PTR/SRV |
| 207.2 | Reverse DNS | `in-addr.arpa` (آدرس معکوس) |
| 207.2 | انتقال zone | AXFR(کامل)/IXFR(افزایشی)، `allow-transfer` |
| 207.3 | DNSSEC | امضای نامتقارن، RRSIG/DNSKEY/DS |
| 207.3 | TSIG | کلید متقارن بین دو سرور خاص |

## مایندمپ متنی کامل

```
Topic 207: Domain Name Server
│
├── 207.1 Basic DNS + BIND
│   ├── سلسله‌مراتب: Root→TLD→Authoritative
│   ├── نقش‌ها: Primary/Secondary/Recursive/Forwarding
│   ├── named.conf: recursion, forwarders, allow-recursion
│   └── ⚠️ Open Resolver → DNS Amplification DDoS
│
├── 207.2 Zone Maintenance
│   ├── SOA: Serial/Refresh/Retry/Expire/NegTTL
│   ├── رکوردها: A/AAAA/CNAME/MX/NS/TXT/PTR/SRV
│   ├── Reverse: in-addr.arpa (آدرس معکوس)
│   └── AXFR(کامل) vs IXFR(افزایشی) + allow-transfer
│
└── 207.3 Securing DNS Server
    ├── DNSSEC: امضای نامتقارن (RRSIG/DNSKEY/DS)
    ├── TSIG: کلید متقارن بین دو سرور
    └── version none + chroot (سخت‌سازی)
```

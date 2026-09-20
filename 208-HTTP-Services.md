---
title: "Topic 208 — HTTP Services (نسخه عمیق و کامل)"
exam: LPIC-2 / 202-450
weights: "208.1 (4) + 208.2 (3) + 208.3 (2) + 208.4 (2)"
os_target: "Arch Linux / Omarchy"
tags: [lpic2, apache, nginx, http, tls, deep-dive]
---

# Topic 208: HTTP Services — راهنمای کامل و عمیق

---

# بخش اول — 208.1: پیاده‌سازی سرور وب پایه (Apache)

## ۱.۱) معماری Apache: Prefork در برابر Event/Worker

Apache HTTP Server (`httpd`) چند **MPM** (Multi-Processing Module) مختلف دارد که تعیین می‌کند چطور درخواست‌های همزمان را مدیریت کند — این یکی از مفاهیمی است که در دوره‌های سطحی معمولاً نادیده گرفته می‌شود اما برای فهم کارایی واقعی سرور حیاتی است:

- **Prefork MPM** — برای هر درخواست، یک **پردازه‌ی کاملاً جدا** ساخته می‌شود. ایمن‌ترین از نظر ایزوله بودن (اگر یک ماژول در یک درخواست کرش کند، فقط همان پردازه از بین می‌رود)، اما سنگین‌ترین از نظر مصرف حافظه (هر پردازه‌ی جدید یعنی کپی کامل حافظه). این تنها MPM سازگار با ماژول‌های قدیمی‌تری مثل `mod_php` است که خودشان thread-safe نیستند.
- **Worker MPM** — از **ترد**ها به‌جای پردازه‌های کامل استفاده می‌کند (چند ترد داخل چند پردازه) — سبک‌تر از نظر حافظه، اما نیاز به این دارد که تمام ماژول‌های بارگذاری‌شده thread-safe باشند.
- **Event MPM** — تکامل‌یافته‌ترین، مبتنی بر یک مدل event-driven برای مدیریت اتصالات keep-alive بسیار کارآمدتر (یک ترد جدا فقط برای نگه‌داشتن اتصالات idle، به‌جای اشغال یک ترد کامل کاری برای هر اتصال باز حتی وقتی بی‌کار است).

```bash
sudo pacman -S apache
httpd -V | grep MPM        # نمایش این‌که کدام MPM در نسخه‌ی نصب‌شده کامپایل شده
```

## ۱.۲) ساختار فایل تنظیمات Apache

فایل اصلی: `/etc/httpd/conf/httpd.conf` (مسیر بسته به توزیع فرق دارد — Debian از `/etc/apache2/apache2.conf` استفاده می‌کند با ساختار ماژولار `sites-available`/`sites-enabled`/`mods-available`/`mods-enabled`).

```apache
ServerRoot "/etc/httpd"
Listen 80
ServerName www.example.com

<Directory "/srv/http">
    Options Indexes FollowSymLinks
    AllowOverride None
    Require all granted
</Directory>

DocumentRoot "/srv/http"
```

### دایرکتیوهای کلیدی که باید عمیق بشناسی

- **`Listen`** — روی کدام پورت(ها) گوش بده (می‌تواند چند بار تکرار شود برای چند پورت)
- **`DocumentRoot`** — ریشه‌ی فایل‌های وب‌سایت
- **`Options`** — قابلیت‌های فعال برای یک دایرکتوری:
  - `Indexes` — اگر فایل index موجود نباشد، لیست فایل‌های پوشه را نشان بده (این یک **خطر امنیتی رایج** است — اگر فراموش کنی غیرفعالش کنی، ساختار داخلی سایتت به هر کسی نشان داده می‌شود)
  - `FollowSymLinks` — اجازه‌ی دنبال کردن symlink ها (خطر امنیتی دیگر اگر با احتیاط استفاده نشود — یک symlink می‌تواند به بیرون از DocumentRoot اشاره کند)
  - `ExecCGI` — اجازه‌ی اجرای اسکریپت‌های CGI در این مسیر
- **`AllowOverride`** — آیا فایل‌های `.htaccess` در این دایرکتوری مجاز به override کردن تنظیمات هستند؟ `None` یعنی خیر (توصیه‌شده برای کارایی بهتر — چک کردن `.htaccess` در هر درخواست هزینه دارد)، `All` یعنی بله.
- **`Require all granted`** — نحوه‌ی مدرن (Apache 2.4+) برای مجاز کردن دسترسی؛ جانشین سینتکس قدیمی `Order allow,deny` / `Allow from all` از Apache 2.2.

### Virtual Hosts — میزبانی چند سایت روی یک سرور

```apache
<VirtualHost *:80>
    ServerName site1.example.com
    DocumentRoot /srv/http/site1
</VirtualHost>

<VirtualHost *:80>
    ServerName site2.example.com
    DocumentRoot /srv/http/site2
</VirtualHost>
```
وقتی چند دامنه روی یک IP هست، Apache از هدر **Host** که مرورگر در درخواست HTTP می‌فرستد استفاده می‌کند تا تشخیص دهد کدام `VirtualHost` باید پاسخ بدهد — این مکانیزم به نام **Name-based Virtual Hosting** شناخته می‌شود (در برابر IP-based، که هر سایت یک IP جدا دارد — روش قدیمی‌تر و امروز کمتر رایج).

```bash
apachectl configtest         # بررسی صحت syntax قبل از reload — دقیقاً مثل named-checkconf که در Topic 207 دیدیم
sudo systemctl reload httpd    # اعمال بدون قطع اتصالات فعلی
sudo systemctl restart httpd    # راه‌اندازی مجدد کامل (قطع موقت)
```

## ۱.۳) لاگ‌ها — دو نوع اصلی و چرا هر دو مهم‌اند

```apache
ErrorLog "/var/log/httpd/error_log"
CustomLog "/var/log/httpd/access_log" combined
```
- **error_log** — هر مشکلی که Apache خودش با آن روبرو شده (خطای پیکربندی، کرش یک ماژول، فایل پیدا نشد)
- **access_log** — هر درخواستی که دریافت شده، با فرمت قابل‌تنظیم (`combined` رایج‌ترین است — شامل IP، زمان، متد، مسیر، کد وضعیت، User-Agent)

فرمت `combined` نمونه‌ای از یک خط:
```
192.168.1.5 - - [10/Sep/2026:14:23:01 +0330] "GET /index.html HTTP/1.1" 200 1234 "-" "Mozilla/5.0..."
```
این لاگ پایه‌ی هر تحلیل امنیتی (تشخیص اسکن پورت، حملات brute-force، الگوهای مشکوک) بعداً است — دقیقاً همان چیزی که `fail2ban` (که در Topic 212 دیدیم) روی آن نظارت می‌کند.

## ۱.۴) نکات مهم آزمون برای 208.1

- ✅ سه MPM را با ویژگی‌هایشان جفت کن: Prefork(پردازه، ایزوله، سازگار با mod_php)، Worker(ترد)، Event(پیشرفته‌ترین، بهینه برای keep-alive).
- ✅ `Options Indexes` و `FollowSymLinks` را به‌عنوان ریسک امنیتی رایج بشناس.
- ✅ سینتکس مدرن `Require all granted` در برابر سینتکس قدیمی Apache 2.2.
- ✅ Name-based در برابر IP-based Virtual Hosting.
- ✅ `apachectl configtest` همیشه قبل از reload واقعی.

---

# بخش دوم — 208.2: پیکربندی پیشرفته‌ی Apache

## ۲.۱) HTTPS و TLS — چطور واقعاً کار می‌کند

قبل از دستورها، باید بفهمی چرا HTTPS اصلاً به گواهی (Certificate) نیاز دارد. **TLS** (که HTTPS از آن استفاده می‌کند) دو کار همزمان انجام می‌دهد:
1. **رمزنگاری** ترافیک — تا کسی که ترافیک را می‌بیند (مثلاً با `tcpdump` که در Topic 205 دیدیم) نتواند محتوا را بخواند.
2. **احراز هویت سرور** — تا مطمئن شوی واقعاً با سروری که فکر می‌کنی صحبت می‌کنی، حرف می‌زنی، نه یک سرور جعلی در حمله‌ی Man-in-the-Middle.

گواهی TLS این دومی را حل می‌کند: یک **CA** (Certificate Authority — مرجع صدور گواهی، مثل Let's Encrypt) گواهی سرور را امضا می‌کند؛ مرورگرت از قبل به فهرستی از CA های معتبر «اعتماد» دارد، پس اگر گواهی سرور توسط یکی از این CA ها امضا شده باشد، مرورگر آن را می‌پذیرد.

```bash
sudo pacman -S certbot certbot-apache
sudo certbot --apache -d www.example.com
```
Certbot (ابزار رسمی Let's Encrypt) این فرآیند را کاملاً خودکار می‌کند: گواهی رایگان درخواست می‌کند، مالکیت دامنه را اثبات می‌کند (با یک چالش HTTP یا DNS)، و خودش تنظیمات Apache را برای استفاده از گواهی جدید ویرایش می‌کند.

پیکربندی دستی معادل:
```apache
<VirtualHost *:443>
    ServerName www.example.com
    SSLEngine on
    SSLCertificateFile /etc/letsencrypt/live/example.com/fullchain.pem
    SSLCertificateKeyFile /etc/letsencrypt/live/example.com/privkey.pem
</VirtualHost>
```
- **fullchain.pem** — گواهی خود سایت + گواهی‌های میانی (Intermediate) لازم برای تکمیل زنجیره‌ی اعتماد تا CA ریشه
- **privkey.pem** — کلید خصوصی سرور؛ این فایل باید بسیار محرمانه نگه داشته شود (permission محدود، هرگز در کنترل نسخه‌ی عمومی commit نشود)

> ⚠️ گواهی‌های Let's Encrypt فقط **۹۰ روز** اعتبار دارند (برخلاف گواهی‌های تجاری سنتی که می‌توانستند سال‌ها معتبر باشند) — این عمداً کوتاه است تا **تمدید خودکار** را الزامی کند (کاهش ریسک گواهی‌های فراموش‌شده و منقضی). Certbot معمولاً یک systemd timer یا cron job خودکار برای تمدید نصب می‌کند:
> ```bash
> sudo certbot renew --dry-run     # تست فرآیند تمدید بدون واقعاً تمدید کردن
> systemctl list-timers | grep certbot
> ```

## ۲.۲) Reverse Proxy — Apache به‌عنوان واسط جلوی سرورهای دیگر

```apache
ProxyPass /app http://localhost:8080/app
ProxyPassReverse /app http://localhost:8080/app
```
**Reverse Proxy** یعنی Apache خودش محتوا را serve نمی‌کند، بلکه درخواست را به یک سرور دیگر (که می‌تواند یک اپلیکیشن Node.js، Python، یا هر چیز دیگری روی یک پورت داخلی باشد) پاس می‌دهد و پاسخ را برمی‌گرداند — از دید کاربر خارجی، همه‌چیز از طریق Apache (روی پورت‌های استاندارد ۸۰/۴۴۳) می‌آید، اما پشت صحنه چندین سرویس مختلف می‌تواند در حال کار باشد.

`ProxyPassReverse` یک نکته‌ی ظریف مهم را حل می‌کند: اگر سرور داخلی در هدرهای پاسخش (مثل `Location` برای ریدایرکت) آدرس داخلی خودش (`localhost:8080`) را بنویسد، این برای کاربر خارجی بی‌معنی است. `ProxyPassReverse` این آدرس‌ها را در پاسخ **بازنویسی** می‌کند تا با آدرس عمومی (`/app`) درست جایگزین شوند.

## ۲.۳) احراز هویت پایه در سطح وب سرور

```bash
sudo htpasswd -c /etc/httpd/.htpasswd admin
```
پرچم `-c` یعنی فایل جدید بساز (فقط بار اول از آن استفاده کن — اگر برای کاربر دوم هم `-c` بزنی، فایل قبلی را کاملاً پاک می‌کند!).

```apache
<Directory "/srv/http/admin">
    AuthType Basic
    AuthName "Restricted Area"
    AuthUserFile /etc/httpd/.htpasswd
    Require valid-user
</Directory>
```
> ⚠️ **هشدار امنیتی:** «Basic Authentication» رمز عبور را فقط با Base64 (که رمزنگاری نیست، فقط یک انکودینگ قابل‌برگشت آسان) منتقل می‌کند — **همیشه** باید همراه با HTTPS استفاده شود، وگرنه هرکسی که ترافیک را ببیند (با `tcpdump`) می‌تواند رمز عبور را به‌راحتی استخراج کند.

## ۲.۴) نکات مهم آزمون برای 208.2

- ✅ فرق مفهومی رمزنگاری و احراز هویت سرور در TLS.
- ✅ چرا گواهی‌های Let's Encrypt کوتاه‌مدت‌اند (۹۰ روز) — تشویق به تمدید خودکار.
- ✅ `ProxyPass` + `ProxyPassReverse` همیشه با هم می‌آیند.
- ✅ `htpasswd -c` فقط بار اول (وگرنه فایل قبلی پاک می‌شود).
- ✅ Basic Auth بدون HTTPS یعنی رمز عبور تقریباً در حالت متن‌باز منتقل می‌شود.

---

# بخش سوم — 208.3: پیاده‌سازی سرور Squid (Proxy Cache)

## ۳.۱) پراکسی فورواردینگ در برابر ریورس پراکسی — تفاوت مهم

این جایی است که اغلب دانشجویان با Reverse Proxy (که در 208.2 دیدیم) اشتباه می‌گیرند:

- **Forward Proxy** (کاری که Squid انجام می‌دهد) — بین **کلاینت‌های داخلی** و **اینترنت خارجی** می‌نشیند. کلاینت‌های یک سازمان به‌جای اتصال مستقیم به اینترنت، از طریق Squid عبور می‌کنند — Squid می‌تواند محتوا را کش کند (صرفه‌جویی پهنای‌باند برای درخواست‌های تکراری)، ترافیک را فیلتر کند (مسدود کردن سایت‌های خاص)، و لاگ کند (چه کسی به کجا رفته).
- **Reverse Proxy** (Apache با `ProxyPass`) — بین **اینترنت خارجی** و **سرورهای داخلی** می‌نشیند؛ جهت دقیقاً برعکس است.

## ۳.۲) پیکربندی پایه‌ی Squid

```bash
sudo pacman -S squid
```
فایل تنظیمات اصلی: `/etc/squid/squid.conf`

```conf
http_port 3128
cache_dir ufs /var/spool/squid 100 16 256

acl localnet src 192.168.1.0/24
http_access allow localnet
http_access deny all
```

### ACL — قلب کنترل دسترسی در Squid

`acl` (Access Control List) تعریف می‌کند «چه چیزی» را می‌خواهی شناسایی کنی (یک زیرشبکه، یک دامنه، یک ساعت خاص از روز)، و `http_access` تصمیم می‌گیرد بر اساس آن ACL، دسترسی مجاز است یا نه — و **ترتیب این قوانین حیاتی است**: Squid قوانین را به‌ترتیب از بالا به پایین بررسی می‌کند و به اولین تطابق عمل می‌کند.

```conf
acl blocked_sites dstdomain .facebook.com .twitter.com
acl work_hours time MTWHF 09:00-18:00

http_access deny blocked_sites
http_access allow localnet work_hours
http_access deny all
```
این مثال: سایت‌های مشخص همیشه بلاک شوند؛ شبکه‌ی داخلی فقط در ساعات کاری تعریف‌شده اجازه‌ی دسترسی دارد؛ هرچیز دیگری رد شود.

> ⚠️ **هشدار جدی:** فراموش کردن `http_access deny all` در انتهای فایل (یا قرار دادنش در جای اشتباه) می‌تواند سرور Squid را به یک **Open Proxy** تبدیل کند — یعنی هر کسی از هر جای اینترنت می‌تواند از پراکسی تو برای مخفی کردن هویت خودش در حملات یا فعالیت‌های غیرقانونی استفاده کند؛ این دقیقاً هم‌خانواده‌ی مفهومی Open Resolver در DNS (Topic 207) است — یک الگوی امنیتی تکرارشونده در این دوره: **همیشه با deny پیش‌فرض کار کن، نه allow پیش‌فرض.**

```bash
sudo squid -k parse           # بررسی صحت syntax فایل تنظیمات
sudo squid -k reconfigure      # بارگذاری مجدد بدون قطع
sudo systemctl enable --now squid
```

## ۳.۳) نکات مهم آزمون برای 208.3

- ✅ Forward Proxy (Squid، بین کلاینت داخلی و اینترنت) در برابر Reverse Proxy (Apache، بین اینترنت و سرور داخلی) — سؤال مفهومی بسیار رایج.
- ✅ ترتیب قوانین `http_access` از بالا به پایین بررسی می‌شود؛ اولین تطابق برنده است.
- ✅ همیشه `http_access deny all` در انتها به‌عنوان قانون پیش‌فرض امن.
- ✅ `squid -k parse` قبل از reconfigure واقعی.

---

# بخش چهارم — 208.4: پیاده‌سازی Nginx به‌عنوان سرور وب و Reverse Proxy

## ۴.۱) چرا Nginx در کنار Apache وجود دارد — تفاوت معماری بنیادین

Apache (با MPM سنتی Prefork/Worker) برای هر اتصال یک پردازه یا ترد اختصاص می‌دهد. Nginx از یک معماری کاملاً متفاوت به نام **event-driven, asynchronous, non-blocking** استفاده می‌کند: تعداد کمی پردازه‌ی worker (معمولاً به تعداد هسته‌های CPU) وجود دارد، و هرکدام می‌تواند **هزاران** اتصال همزمان را بدون نیاز به ترد/پردازه‌ی جداگانه برای هرکدام مدیریت کند — این برای سناریوهایی با تعداد بسیار زیاد اتصالات همزمان (مثل یک سایت پرترافیک، یا streaming) کارایی به‌مراتب بهتری می‌دهد.

## ۴.۲) ساختار پیکربندی — بلوک‌محور، نه دایرکتوری‌محور

```bash
sudo pacman -S nginx
```
فایل اصلی: `/etc/nginx/nginx.conf`، که معمولاً فایل‌های سایت‌های جدا را از `/etc/nginx/sites-enabled/` یا `/etc/nginx/conf.d/*.conf` include می‌کند.

```nginx
server {
    listen 80;
    server_name www.example.com;
    root /srv/http/example;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }

    location /api/ {
        proxy_pass http://localhost:3000/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

### مقایسه‌ی مفهومی با Apache

- بلوک `server { }` در Nginx ≈ `<VirtualHost>` در Apache
- بلوک `location { }` در Nginx ≈ `<Directory>` یا `<Location>` در Apache — اما با یک تفاوت مهم: قوانین تطابق `location` بر اساس **اولویت نوع تطابق** (exact match، prefix match، regex) کار می‌کند، نه صرفاً ترتیب نوشتن — این جایی است که خیلی‌ها گیج می‌شوند.

`proxy_pass` دقیقاً همان مفهوم `ProxyPass` در Apache است، اما نکته‌ی مهم Nginx: باید صریح هدرهایی مثل `Host` و `X-Real-IP` را با `proxy_set_header` تنظیم کنی — Nginx به‌صورت پیش‌فرض این‌ها را برای سرور بک‌اند حفظ نمی‌کند (برخلاف Apache که با `ProxyPreserveHost` رفتار مشابهی دارد اما با دیفالت‌های متفاوت).

```bash
sudo nginx -t                     # تست صحت syntax — معادل apachectl configtest
sudo systemctl reload nginx
```

## ۴.۳) نکات مهم آزمون برای 208.4

- ✅ تفاوت معماری بنیادین: Apache (پردازه/ترد به‌ازای اتصال) در برابر Nginx (event-driven، تعداد کم worker برای هزاران اتصال).
- ✅ نگاشت مفهومی: `server{}`≈VirtualHost، `location{}`≈Directory.
- ✅ در Nginx باید صریح `proxy_set_header` بزنی؛ به‌صورت پیش‌فرض هدرهای اصلی حفظ نمی‌شوند.
- ✅ `nginx -t` قبل از reload.

---

## خلاصه‌ی جدولی نهایی کل Topic 208

| زیرمبحث | مفهوم کلیدی | نکته |
|---|---|---|
| 208.1 | MPM Apache | Prefork(پردازه)/Worker(ترد)/Event(پیشرفته) |
| 208.1 | امنیت Directory | `Options Indexes/FollowSymLinks` ریسک است |
| 208.1 | Virtual Host | Name-based (هدر Host) vs IP-based |
| 208.2 | TLS | رمزنگاری + احراز هویت سرور، Let's Encrypt ۹۰ روزه |
| 208.2 | Reverse Proxy | `ProxyPass`+`ProxyPassReverse` |
| 208.2 | Basic Auth | فقط با HTTPS معنا دارد |
| 208.3 | Forward Proxy | Squid: بین کلاینت داخلی و اینترنت |
| 208.3 | ACL ترتیبی | اولین تطابق برنده، `deny all` پیش‌فرض امن |
| 208.4 | معماری Nginx | event-driven، غیر از Apache |
| 208.4 | بلوک‌ها | `server{}`, `location{}`, `proxy_set_header` |

## مایندمپ متنی کامل

```
Topic 208: HTTP Services
│
├── 208.1 Apache Basic
│   ├── MPM: Prefork/Worker/Event
│   ├── httpd.conf: Listen, DocumentRoot, Options, AllowOverride
│   ├── ⚠️ Indexes/FollowSymLinks = ریسک
│   ├── VirtualHost: Name-based(Host header) vs IP-based
│   └── access_log/error_log
│
├── 208.2 Apache Advanced
│   ├── TLS: رمزنگاری + احراز هویت سرور
│   ├── certbot → fullchain.pem + privkey.pem (۹۰ روز!)
│   ├── ProxyPass + ProxyPassReverse
│   └── htpasswd -c (فقط بار اول) + Basic Auth (فقط با HTTPS)
│
├── 208.3 Squid (Forward Proxy)
│   ├── Forward(کلاینت↔اینترنت) vs Reverse(اینترنت↔سرور)
│   ├── acl + http_access (ترتیبی، اولین تطابق)
│   └── ⚠️ deny all پیش‌فرض ضروری (وگرنه Open Proxy)
│
└── 208.4 Nginx
    ├── معماری event-driven (نه پردازه/ترد به‌ازای اتصال)
    ├── server{} ≈ VirtualHost، location{} ≈ Directory
    └── proxy_pass + proxy_set_header (صریح لازم است)
```

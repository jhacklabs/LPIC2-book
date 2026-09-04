---
title: "Topic 208 — HTTP Services"
exam: LPIC-2 / 202-450
weights: "208.1 (4) + 208.2 (3) + 208.3 (2) + 208.4 (2)"
os_target: "Arch Linux / Omarchy"
tags: [lpic2, apache, nginx, tls, reverseproxy]
---

# Topic 208: HTTP Services

## ۱) مفهوم کلی

چهار بخش:
- **208.1** پایه‌های پیکربندی Apache
- **208.2** Apache پیشرفته (virtual hosts، احراز هویت، ماژول‌ها)
- **208.3** پیاده‌سازی proxy با Squid
- **208.4** پیکربندی Nginx به‌عنوان وب‌سرور ساده و reverse proxy

## ۲) چرا این مبحث مهم است؟

وب‌سرور دروازه‌ی اصلی ورود ترافیک عمومی به سرورهای توست — یعنی از منظر امنیتی، **بیشترین سطح حمله** را دارد. اشتباهات پیکربندی HTTP (مثل directory listing باز، هدرهای امنیتی گمشده، یا TLS پیکربندی‌نشده درست) از رایج‌ترین راه‌های نفوذ به وب‌اپلیکیشن‌ها هستند. reverse proxy (Nginx جلوی یک اپ بک‌اند) هم امروز استاندارد صنعتی است — تقریباً هر معماری مدرن یک لایه Nginx/Squid جلوی سرویس اصلی دارد.

## ۳) مثال‌های واقعی + روی سیستم خودم

**واقعی:** یک شرکت Nginx را جلوی یک اپلیکیشن Node.js می‌گذارد تا هم SSL termination انجام دهد (رمزنگاری/رمزگشایی TLS در یک نقطه متمرکز)، هم load balancing بین چند instance اپ.

**روی Omarchy:** می‌توانی هم Apache هم Nginx را همزمان (روی پورت‌های متفاوت) نصب و تست کنی، بدون هیچ تداخلی با بقیه سیستم — چون هر دو فقط سرویس‌های ایزوله‌ای هستند که خودت روشن/خاموش می‌کنی.

## ۴) دستورات کامل

### 208.1 — پایه Apache

```bash
sudo pacman -S apache
sudo systemctl enable --now httpd
apachectl configtest      # اعتبارسنجی تنظیمات قبل از reload (خیلی مهم!)
sudo systemctl reload httpd
```
فایل تنظیمات اصلی: `/etc/httpd/conf/httpd.conf` (روی Arch). دایرکتیوهای پایه:
```apache
Listen 80
DocumentRoot "/srv/http"
<Directory "/srv/http">
    Options Indexes FollowSymLinks
    AllowOverride None
    Require all granted
</Directory>
ErrorLog "/var/log/httpd/error_log"
CustomLog "/var/log/httpd/access_log" combined
```
> ⚠️ **هشدار امنیتی:** `Options Indexes` یعنی اگر یک پوشه `index.html` نداشته باشد، Apache لیست کامل فایل‌های داخلش را نمایش می‌دهد — این می‌تواند فایل‌های حساس (backup، config) را افشا کند. در production معمولاً باید حذف شود.

### 208.2 — Apache پیشرفته

Virtual Hosts (میزبانی چند دامنه روی یک IP):
```apache
<VirtualHost *:80>
    ServerName site1.example.com
    DocumentRoot "/srv/http/site1"
</VirtualHost>
<VirtualHost *:80>
    ServerName site2.example.com
    DocumentRoot "/srv/http/site2"
</VirtualHost>
```
احراز هویت پایه (Basic Auth):
```bash
sudo htpasswd -c /etc/httpd/.htpasswd admin
```
```apache
<Directory "/srv/http/secure">
    AuthType Basic
    AuthName "Restricted Area"
    AuthUserFile /etc/httpd/.htpasswd
    Require valid-user
</Directory>
```
> ⚠️ Basic Auth بدون HTTPS، رمز عبور را به‌صورت base64 (نه رمزنگاری‌شده، فقط encode) روی شبکه می‌فرستد — همیشه همراه با TLS استفاده شود.

فعال/غیرفعال‌سازی ماژول‌ها:
```bash
sudo a2enmod ssl        # (روش Debian؛ روی Arch ماژول‌ها معمولاً مستقیم در httpd.conf با LoadModule فعال می‌شوند)
```
```apache
LoadModule ssl_module modules/mod_ssl.so
```
پیکربندی TLS/SSL:
```apache
<VirtualHost *:443>
    ServerName secure.example.com
    SSLEngine on
    SSLCertificateFile /etc/ssl/certs/example.crt
    SSLCertificateKeyFile /etc/ssl/private/example.key
</VirtualHost>
```

### 208.3 — Proxy با Squid

```bash
sudo pacman -S squid
sudo systemctl enable --now squid
```
فایل تنظیمات: `/etc/squid/squid.conf`
```
http_port 3128
acl localnet src 192.168.1.0/24
http_access allow localnet
http_access deny all
```
> ⚠️ **هشدار جدی:** ترتیب قوانین ACL در Squid مهم است — قوانین از بالا به پایین بررسی می‌شوند و اولین match برنده است. یک `http_access allow all` در بالای فایل، تمام قوانین محدودکننده بعدی را بی‌اثر می‌کند.

```bash
squid -k parse         # اعتبارسنجی تنظیمات
squid -k reconfigure    # بارگذاری مجدد بدون قطع سرویس
tail -f /var/log/squid/access.log
```

### 208.4 — Nginx (وب‌سرور و Reverse Proxy)

```bash
sudo pacman -S nginx
sudo systemctl enable --now nginx
sudo nginx -t              # اعتبارسنجی تنظیمات (خیلی رایج در آزمون و عمل)
sudo systemctl reload nginx
```
وب‌سرور ساده: `/etc/nginx/nginx.conf` یا فایل‌های `/etc/nginx/sites-available/`
```nginx
server {
    listen 80;
    server_name example.com;
    root /srv/http/example;
    index index.html;
}
```
Reverse Proxy (رایج‌ترین کاربرد امروزی Nginx):
```nginx
server {
    listen 80;
    server_name app.example.com;

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```
`proxy_set_header X-Forwarded-For` بسیار مهم است — بدون آن، اپلیکیشن بک‌اند IP واقعی همه کاربران را به‌اشتباه همان IP سرور Nginx می‌بیند (مهم برای لاگ و امنیت).

## ۵) نکات مهم آزمون LPIC

- ✅ همیشه قبل از reload، تست کن: `apachectl configtest` یا `nginx -t` — سؤال رایج درباره ترتیب صحیح کار.
- ✅ ترتیب پردازش ACL در Squid (بالا به پایین، اولین match) را دقیق بدان.
- ✅ فرق `AllowOverride None` (نادیده‌گرفتن `.htaccess`) و `AllowOverride All` را بدان.
- ✅ نام‌های دقیق دایرکتیوهای TLS در Apache: `SSLCertificateFile`, `SSLCertificateKeyFile`.
- ✅ در Nginx، `proxy_pass` و هدرهای `proxy_set_header` را حفظ کن — پایه هر تنظیم reverse proxy.
- ✅ Virtual Host بر اساس `ServerName` تفکیک می‌شود، نه IP (در اکثر سناریوهای name-based hosting).

## ۶) تمرین عملی امن

> این تمرین صرفاً روی `localhost` انجام می‌شود و هیچ ریسکی برای شبکه بیرونی ندارد.

1. نصب و تست Apache:
```bash
sudo pacman -S apache
sudo systemctl enable --now httpd
apachectl configtest
curl http://localhost
```
2. یک صفحه ساده و یک Virtual Host تستی بساز:
```bash
echo "<h1>Test Site 1</h1>" | sudo tee /srv/http/index.html
curl http://localhost
```
3. نصب و تست Nginx روی پورت متفاوت (برای جلوگیری از تداخل با Apache):
```bash
sudo pacman -S nginx
sudo sed -i 's/listen 80/listen 8080/' /etc/nginx/nginx.conf
sudo systemctl enable --now nginx
sudo nginx -t
curl http://localhost:8080
```
4. یک reverse proxy ساده در Nginx بساز که به یک سرور Python ساده وصل شود:
```bash
cd /tmp && python3 -m http.server 3000 &
```
سپس در تنظیمات nginx یک `location / { proxy_pass http://127.0.0.1:3000; }` اضافه کن، reload بزن و تست کن:
```bash
sudo nginx -s reload
curl http://localhost:8080
kill %1   # بستن سرور پایتون تستی
```

## ۷) خلاصه جدولی

| سرویس | فایل تنظیمات | تست اعتبار | دستور reload |
|---|---|---|---|
| Apache | `/etc/httpd/conf/httpd.conf` | `apachectl configtest` | `systemctl reload httpd` |
| Squid | `/etc/squid/squid.conf` | `squid -k parse` | `squid -k reconfigure` |
| Nginx | `/etc/nginx/nginx.conf` | `nginx -t` | `nginx -s reload` |

| مفهوم کلیدی | جزئیات |
|---|---|
| Virtual Host | تفکیک بر اساس `ServerName` |
| Basic Auth | `htpasswd` + `AuthUserFile` |
| TLS در Apache | `SSLCertificateFile/KeyFile` |
| ACL در Squid | ترتیب بالا-به-پایین، اولین match |
| Reverse Proxy در Nginx | `proxy_pass`, `proxy_set_header` |

## ۸) مایندمپ متنی

```
Topic 208: HTTP Services
├── 208.1 Apache Basics
│   ├── httpd.conf, Listen, DocumentRoot
│   └── apachectl configtest
├── 208.2 Apache Advanced
│   ├── VirtualHost (name-based)
│   ├── Basic Auth (htpasswd)
│   └── TLS (mod_ssl)
├── 208.3 Squid Proxy
│   ├── squid.conf, ACLs (order matters!)
│   └── squid -k parse/reconfigure
└── 208.4 Nginx
    ├── server {} blocks
    ├── nginx -t
    └── reverse proxy (proxy_pass, X-Forwarded-For)
```

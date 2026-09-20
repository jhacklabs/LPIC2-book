---
title: "Topic 210 — Network Client Management (نسخه عمیق و کامل)"
exam: LPIC-2 / 202-450
weights: "210.1 (2) + 210.2 (3) + 210.3 (2) + 210.4 (4)"
os_target: "Arch Linux / Omarchy"
tags: [lpic2, ldap, pam, nsswitch, ntp, deep-dive]
---

# Topic 210: Network Client Management — راهنمای کامل و عمیق

---

# بخش اول — 210.1: DNS Cache/Proxy Server در سطح کلاینت

## ۱.۱) چرا یک DNS Cache محلی روی هر ماشین کلاینت؟

در Topic 207 درباره‌ی سرورهای DNS مرکزی صحبت کردیم. اما یک لایه‌ی دیگر هم وجود دارد: نصب یک **DNS caching resolver سبک** مستقیم روی خود ماشین کلاینت (نه یک سرور مرکزی جدا). چرا؟ چون حتی رفت‌وبرگشت شبکه به نزدیک‌ترین سرور DNS (حتی روی همان LAN) چند میلی‌ثانیه طول می‌کشد؛ اگر این پاسخ‌ها مستقیم روی خود ماشین کش شوند، درخواست‌های تکراری (که در استفاده‌ی روزمره‌ی یک کاربر بسیار رایج است — همان دامنه‌ها بارها بارها پرسیده می‌شوند) فوری و بدون هیچ ترافیک شبکه پاسخ داده می‌شوند.

## ۱.۲) `systemd-resolved` — راه‌حل مدرن پیش‌فرض

روی Arch/Omarchy، `systemd-resolved` دقیقاً همین نقش را بازی می‌کند — که در Topic 205 هم مختصر دیدیمش:

```bash
sudo systemctl enable --now systemd-resolved
resolvectl status                    # وضعیت DNS هر اینترفیس، شامل سرورهای DNS فعال و کش
resolvectl statistics                 # آمار کش (تعداد hit/miss)
resolvectl flush-caches                # پاک‌سازی دستی کش (مفید هنگام عیب‌یابی یک تغییر DNS که هنوز اعمال نشده)
resolvectl query example.com            # یک query تستی مستقیم از طریق resolved
```

## ۱.۳) `dnsmasq` — جایگزین سبک و همه‌کاره

**dnsmasq** یک ابزار بسیار محبوب دیگر است که کارهای بیشتری هم انجام می‌دهد — همزمان DNS caching **و** سرور DHCP سبک (که در Topic 205 با ISC DHCP دیدیم، dnsmasq جایگزین ساده‌تری برای شبکه‌های کوچک/خانگی است) و حتی TFTP:

```bash
sudo pacman -S dnsmasq
```
فایل تنظیمات: `/etc/dnsmasq.conf`
```conf
listen-address=127.0.0.1
cache-size=1000
no-resolv
server=1.1.1.1
server=8.8.8.8
```
`no-resolv` یعنی `/etc/resolv.conf` سیستم را برای یافتن سرورهای بالادستی نادیده بگیر و از `server=` های تعریف‌شده در همین فایل استفاده کن — این کنترل بیشتری به تو می‌دهد و از تداخل احتمالی جلوگیری می‌کند.

## ۱.۴) نکات مهم آزمون برای 210.1

- ✅ تفاوت DNS caching محلی (روی کلاینت، سرعت) با DNS سرور مرکزی (Topic 207، مدیریت zone) را بفهم — این‌ها مکمل هم‌اند، نه رقیب.
- ✅ `resolvectl` ابزار اصلی مدیریت DNS در سیستم‌های systemd-resolved.
- ✅ dnsmasq چندمنظوره است: DNS cache + DHCP سبک + TFTP.

---

# بخش دوم — 210.2: LDAP برای احراز هویت

## ۲.۱) LDAP چیست و چه مسئله‌ای را حل می‌کند

تصور کن یک سازمان با ۲۰۰ کارمند و ۵۰ سرور لینوکسی داری. اگر هر کارمند بخواهد به هر سروری SSH بزند، آیا باید حساب کاربری جداگانه روی هر ۵۰ سرور برایش بسازی؟ این یک کابوس مدیریتی است — وقتی یک کارمند اخراج می‌شود، باید حسابش را از ۵۰ جای مختلف حذف کنی (و اگر یکی را فراموش کنی، یک ریسک امنیتی باقی می‌ماند).

**LDAP** (Lightweight Directory Access Protocol) این را با یک **پایگاه‌داده‌ی متمرکز کاربران** حل می‌کند: یک سرور LDAP، اطلاعات همه‌ی کاربران (نام‌کاربری، رمز عبور هش‌شده، گروه‌ها، اطلاعات تماس) را نگه می‌دارد؛ تمام ۵۰ سرور به‌جای داشتن حساب‌های محلی، برای احراز هویت به همین یک سرور مرکزی مراجعه می‌کنند.

## ۲.۲) ساختار درختی LDAP — DN، DC، OU، CN

LDAP داده‌ها را در یک **ساختار درختی سلسله‌مراتبی** (شبیه فایل‌سیستم یا DNS) نگه می‌دارد. هر ورودی (entry) یک **DN** (Distinguished Name) منحصربه‌فرد دارد که مسیر کامل آن در درخت را نشان می‌دهد:

```
dn: uid=jdoe,ou=People,dc=example,dc=com
```

بیایید هر بخش را باز کنیم:
- **`dc=example,dc=com`** (Domain Component) — معادل دامنه‌ی سازمان (`example.com`)، دقیقاً مثل ساختار DNS معکوس‌نویسی‌شده
- **`ou=People`** (Organizational Unit) — یک واحد سازمانی، برای دسته‌بندی منطقی (مثلاً `ou=People` برای کاربران، `ou=Groups` برای گروه‌ها، `ou=Servers` برای اشیای دیگر)
- **`uid=jdoe`** (User ID، نوعی RDN — Relative Distinguished Name) — شناسه‌ی یکتای خود کاربر در این سطح

می‌توانی این ساختار را دقیقاً مثل یک فایل‌سیستم تصور کنی: `dc=com` ریشه‌ای‌ترین، `dc=example` زیرمجموعه‌اش، `ou=People` زیرمجموعه‌ی آن، و `uid=jdoe` برگ نهایی درخت.

## ۲.۳) نصب و پیکربندی سرور OpenLDAP

```bash
sudo pacman -S openldap
```
تنظیمات مدرن OpenLDAP (از نسخه‌ی ۲.۴ به بعد) به‌جای فایل متنی سنتی `slapd.conf`، در یک پایگاه‌داده‌ی پویا به نام **cn=config** نگه‌داری می‌شود (این معماری «Runtime Configuration» یا **slapd-config** نام دارد) — تغییرات را می‌توان بدون ری‌استارت کامل سرویس، مستقیم از طریق پروتکل LDAP خودش اعمال کرد.

```bash
sudo systemctl enable --now slapd
ldapsearch -x -b "dc=example,dc=com"       # جست‌وجوی ساده در کل درخت (پرچم -x یعنی احراز هویت ساده، نه SASL)
ldapadd -x -D "cn=admin,dc=example,dc=com" -W -f newuser.ldif    # افزودن یک ورودی جدید از یک فایل LDIF
ldapmodify -x -D "cn=admin,dc=example,dc=com" -W -f changes.ldif   # ویرایش یک ورودی موجود
ldapdelete -x -D "cn=admin,dc=example,dc=com" -W "uid=jdoe,ou=People,dc=example,dc=com"
```

### فرمت LDIF — زبان تعریف داده‌ی LDAP

**LDIF** (LDAP Data Interchange Format) فرمت متنی استانداردی است که برای تعریف یا ویرایش ورودی‌های LDAP استفاده می‌شود:
```ldif
dn: uid=jdoe,ou=People,dc=example,dc=com
objectClass: inetOrgPerson
objectClass: posixAccount
uid: jdoe
cn: John Doe
sn: Doe
uidNumber: 10001
gidNumber: 10001
homeDirectory: /home/jdoe
loginShell: /bin/bash
```
نکته‌ی مهم: `objectClass: posixAccount` یعنی این ورودی LDAP شامل فیلدهایی است که مستقیماً معادل فیلدهای `/etc/passwd` سنتی هستند (`uidNumber`, `gidNumber`, `homeDirectory`, `loginShell`) — این دقیقاً پلی است که به کلاینت‌های لینوکسی اجازه می‌دهد از LDAP به‌جای `/etc/passwd` محلی برای اطلاعات کاربر استفاده کنند (که در بخش 210.3 با `nsswitch.conf` می‌بینیم چطور).

## ۲.۴) پیکربندی سمت کلاینت برای استفاده از LDAP

```bash
sudo pacman -S nss-pam-ldapd
```
فایل تنظیمات کلاینت: `/etc/nslcd.conf`
```conf
uri ldap://ldap.example.com/
base dc=example,dc=com
```
سرویس `nslcd` (Name Service LDAP Connection Daemon) پلی بین سیستم لینوکس و سرور LDAP است — این را در بخش بعد با `nsswitch.conf` وصل می‌کنیم.

## ۲.۵) نکات مهم آزمون برای 210.2

- ✅ ساختار DN را دقیق بفهم: `dc` (دامنه) → `ou` (واحد سازمانی) → `uid`/`cn` (خود ورودی).
- ✅ LDIF فرمت استاندارد تعریف/ویرایش ورودی‌هاست؛ `ldapadd`/`ldapmodify`/`ldapdelete`/`ldapsearch` ابزارهای اصلی.
- ✅ `objectClass: posixAccount` پل بین LDAP و مفاهیم سنتی یونیکسی (uid/gid/home/shell).
- ✅ معماری مدرن `cn=config` (پویا) در برابر `slapd.conf` سنتی (فایل استاتیک).

---

# بخش سوم — 210.3: PAM و NSS

## ۳.۱) PAM — چارچوب انعطاف‌پذیر احراز هویت

**PAM** (Pluggable Authentication Modules) یک لایه‌ی انتزاعی است که به برنامه‌ها (مثل `login`, `sshd`, `sudo`) اجازه می‌دهد **بدون دانستن جزئیات** چطور یک کاربر باید احراز هویت شود، این کار را انجام دهند — منطق واقعی احراز هویت در «ماژول‌های قابل‌اتصال» (Pluggable Modules) قرار دارد که می‌توانند بدون تغییر خود برنامه، عوض یا اضافه شوند.

### چهار نوع «مدیریت» در PAM

هر خط در فایل تنظیمات PAM یکی از این چهار نوع کار را انجام می‌دهد:
- **`auth`** — خود احراز هویت (آیا رمز عبور/بیومتریک/توکن درست است؟)
- **`account`** — بررسی‌های مربوط به حساب (آیا حساب منقضی نشده؟ آیا در ساعت مجاز است؟)
- **`password`** — منطق تغییر رمز عبور (مثلاً قوانین پیچیدگی رمز عبور)
- **`session`** — کارهایی که باید هنگام شروع/پایان یک session انجام شود (مثلاً mount کردن home directory، ثبت لاگ ورود)

### Control Flags — چطور نتیجه‌ی چند ماژول ترکیب می‌شود

```
auth    required     pam_unix.so
auth    requisite     pam_succeed_if.so uid >= 1000
auth    sufficient    pam_ldap.so
auth    optional      pam_permit.so
```

- **`required`** — اگر شکست بخورد، کل فرآیند نهایتاً شکست می‌خورد، **اما** بقیه‌ی ماژول‌های همان نوع (auth) هم اجرا می‌شوند (تا کاربر نفهمد دقیقاً کدام ماژول رد کرده — یک اقدام امنیتی برای جلوگیری از information leakage)
- **`requisite`** — مثل `required`، اما اگر شکست بخورد، **فوراً** متوقف می‌شود، بدون اجرای ماژول‌های بعدی
- **`sufficient`** — اگر موفق شود، همین برای موفقیت کل مرحله کافی است (به شرطی که قبلش هیچ `required` شکست‌خورده‌ای نباشد) و بقیه‌ی ماژول‌ها نادیده گرفته می‌شوند
- **`optional`** — نتیجه‌اش معمولاً روی تصمیم نهایی تأثیری ندارد مگر این‌که تنها ماژول تعریف‌شده باشد

```bash
cat /etc/pam.d/sshd
cat /etc/pam.d/login
```
این فایل‌ها هرکدام برای یک سرویس خاص (نامشان دقیقاً اسم سرویس است) تعریف می‌شوند — یعنی می‌توانی سیاست احراز هویت SSH را کاملاً متفاوت از سیاست login محلی تعریف کنی.

> ⚠️ **هشدار جدی:** ویرایش دستی فایل‌های PAM بسیار خطرناک است — یک اشتباه سینتکسی می‌تواند **همه‌ی راه‌های ورود به سیستم را قفل کند**، حتی برای root. همیشه قبل از تغییر یک session ترمینال دیگر باز نگه دار (که هنوز لاگین است) تا اگر مشکلی پیش آمد بتوانی برگردانی، و همیشه یک بکاپ از فایل اصلی بگیر.

## ۳.۲) NSS — از کجا اطلاعات کاربر/گروه/میزبان می‌آید؟

**NSS** (Name Service Switch) مکانیزم دیگری است که به سیستم می‌گوید برای انواع مختلف اطلاعات (کاربران، گروه‌ها، نام میزبان‌ها) **از کجا** باید جست‌وجو کند و **به چه ترتیبی**.

```bash
cat /etc/nsswitch.conf
```
```conf
passwd:     files ldap
group:       files ldap
shadow:      files
hosts:        files dns
```

بیایید این را دقیق بخوانیم: خط `passwd: files ldap` یعنی «وقتی سیستم می‌خواهد اطلاعات یک کاربر را پیدا کند (مثلاً هنگام `ls -l` که باید uid را به نام تبدیل کند، یا هنگام لاگین)، اول در `/etc/passwd` محلی (`files`) بگرد؛ اگر آنجا پیدا نشد، برو سراغ سرور LDAP (`ldap`)». همین منطق برای `group` هم اعمال می‌شود.

خط `hosts: files dns` یعنی: اول `/etc/hosts` محلی چک شود (برای override های دستی سریع)، اگر آنجا نبود، برو سراغ DNS.

**این دقیقاً همان چیزی است که PAM و LDAP (210.2) و DNS (207) را به هم متصل می‌کند:** بدون خط `ldap` در `nsswitch.conf`، حتی اگر PAM را برای استفاده از LDAP تنظیم کرده باشی، سیستم اصلاً نمی‌داند باید برای اطلاعات کاربر به LDAP سر بزند.

## ۳.۳) نکات مهم آزمون برای 210.3

- ✅ چهار نوع مدیریت PAM (`auth`/`account`/`password`/`session`) را از هم تفکیک کن.
- ✅ چهار control flag را دقیق بفهم، خصوصاً تفاوت ظریف `required` (ادامه می‌دهد ولی شکست را ثبت می‌کند) و `requisite` (فوراً متوقف می‌شود).
- ✅ `nsswitch.conf` ترتیب منابع جست‌وجو را تعیین می‌کند — و PAM/LDAP بدون این هماهنگی کار نمی‌کنند.
- ✅ همیشه یک session پشتیبان باز نگه دار قبل از ویرایش فایل‌های PAM.

---

# بخش چهارم — 210.4: NTP (Network Time Protocol)

## ۴.۱) چرا زمان دقیق این‌قدر مهم است؟

خیلی‌ها فکر می‌کنند زمان دقیق فقط برای این است که ساعت گوشه‌ی صفحه درست باشد — این کاملاً اشتباه است. زمان دقیق **پایه‌ی امنیتی و عملیاتی حیاتی** بسیاری سیستم‌ها است:

1. **گواهی‌های TLS/SSL** (که در Topic 208 دیدیم) دارای تاریخ انقضا هستند — اگر ساعت سیستم اشتباه باشد، ممکن است یک گواهی معتبر را «منقضی‌شده» تشخیص دهد (یا برعکس، خطرناک‌تر: یک گواهی واقعاً منقضی‌شده را معتبر بداند).
2. **پروتکل‌های احراز هویت مثل Kerberos** (که در پروژه‌های Active Directory/LDAP رایج است) به‌شدت به هماهنگی زمانی بین کلاینت و سرور وابسته‌اند — اگر اختلاف زمانی بیش از چند دقیقه باشد (معمولاً ۵ دقیقه)، احراز هویت **کاملاً شکست می‌خورد**.
3. **همبستگی لاگ‌ها** (Log Correlation) — وقتی می‌خواهی حادثه‌ی امنیتی را در چند سرور مختلف بررسی کنی، اگر ساعت هرکدام چند دقیقه با هم فرق داشته باشد، تشخیص ترتیب واقعی رویدادها (کدام اتفاق اول افتاده) عملاً غیرممکن می‌شود — یک کابوس واقعی در تحقیقات فارنزیک امنیتی.

## ۴.۲) معماری NTP — سلسله‌مراتب Stratum

NTP هم مثل DNS یک ساختار سلسله‌مراتبی دارد، اما اینجا به آن **Stratum** (سطح) می‌گویند:

- **Stratum 0** — خود دستگاه‌های مرجع زمان دقیق فیزیکی (ساعت اتمی، گیرنده‌ی GPS) — این‌ها مستقیماً در شبکه شرکت نمی‌کنند، فقط منبع اولیه هستند.
- **Stratum 1** — سرورهایی که **مستقیماً** به یک دستگاه Stratum 0 وصل‌اند (مثلاً یک سرور با گیرنده‌ی GPS متصل).
- **Stratum 2** — سرورهایی که زمان را از یک سرور Stratum 1 می‌گیرند.
- و به همین ترتیب، هر لایه یک عدد بیشتر...

هرچه عدد Stratum بالاتر برود، دقت به‌طور نظری کمی کمتر می‌شود (چون هر لایه کمی تأخیر/خطای اضافه دارد)، اما در عمل حتی Stratum ۳-۴ برای اکثر نیازها به‌اندازه‌ی کافی دقیق است.

## ۴.۳) پیاده‌سازی‌های مختلف NTP

### ntpd — پیاده‌سازی کلاسیک

```bash
sudo pacman -S ntp
```
فایل تنظیمات: `/etc/ntp.conf`
```conf
server 0.arch.pool.ntp.org
server 1.arch.pool.ntp.org
server 2.arch.pool.ntp.org
```
```bash
ntpq -p          # نمایش وضعیت اتصال به هر سرور تعریف‌شده (offset، delay، stratum هرکدام)
```

### chrony — پیاده‌سازی مدرن‌تر (امروز ترجیح داده‌شده)

**chrony** جانشین مدرن‌تری برای `ntpd` است که امروز پیش‌فرض بسیاری توزیع‌ها (از جمله معمولاً Arch/Omarchy) شده — دلایل ترجیح آن:
- سازگاری بهتر با سیستم‌هایی که همیشه روشن نیستند (لپ‌تاپ‌ها که مرتب suspend/resume می‌شوند) — `chrony` می‌تواند بسیار سریع‌تر بعد از یک دوره‌ی آفلاین طولانی، ساعت را تصحیح کند
- دقت بهتر روی اتصالات اینترنت متناوب یا کند

```bash
sudo pacman -S chrony
sudo systemctl enable --now chronyd
chronyc tracking          # وضعیت دقیق فعلی: offset، drift rate، آخرین sync
chronyc sources -v          # لیست سرورهای NTP پیکربندی‌شده و وضعیت هرکدام
```

### timedatectl — رابط ساده‌ی systemd

```bash
timedatectl status
sudo timedatectl set-ntp true       # فعال/غیرفعال کردن همگام‌سازی خودکار NTP (چه از طریق chrony یا systemd-timesyncd داخلی)
sudo timedatectl set-timezone Asia/Tehran
```
**systemd-timesyncd** یک پیاده‌سازی بسیار سبک NTP (فقط کلاینت، بدون قابلیت سرور شدن) است که برای اکثر سیستم‌های دسکتاپ/لپ‌تاپ کافی است — اگر نیاز واقعی به دقت بسیار بالا یا کارکرد به‌عنوان سرور NTP برای سیستم‌های دیگر نداری، این ساده‌ترین گزینه است.

## ۴.۴) نکات مهم آزمون برای 210.4

- ✅ چرا زمان دقیق مهم است: TLS، Kerberos (حساسیت شدید)، همبستگی لاگ‌ها/فارنزیک.
- ✅ مفهوم Stratum را از پایین (0=منبع فیزیکی) تا بالا بفهم.
- ✅ `ntpd` (کلاسیک) در برابر `chrony` (مدرن، بهتر برای سیستم‌های متناوب) در برابر `systemd-timesyncd` (سبک، فقط کلاینت).
- ✅ `chronyc tracking`/`sources -v` و `ntpq -p` را از هم تفکیک کن.
- ✅ `timedatectl` رابط ساده‌ی سطح بالا برای هر دو زمان و timezone.

---

## خلاصه‌ی جدولی نهایی کل Topic 210

| زیرمبحث | مفهوم کلیدی | ابزار |
|---|---|---|
| 210.1 | DNS Cache محلی | `systemd-resolved`, `dnsmasq` |
| 210.2 | ساختار LDAP | DN: dc→ou→uid/cn |
| 210.2 | ابزار LDAP | `ldapsearch/add/modify/delete`, LDIF |
| 210.3 | PAM | auth/account/password/session |
| 210.3 | Control Flags | required/requisite/sufficient/optional |
| 210.3 | NSS | `nsswitch.conf`: files → ldap/dns |
| 210.4 | چرا زمان مهم | TLS، Kerberos، فارنزیک لاگ |
| 210.4 | Stratum | 0(فیزیکی)→1→2→... |
| 210.4 | پیاده‌سازی | `ntpd`(کلاسیک) vs `chrony`(مدرن) vs `timesyncd`(سبک) |

## مایندمپ متنی کامل

```
Topic 210: Network Client Management
│
├── 210.1 DNS Cache/Proxy
│   ├── systemd-resolved: resolvectl status/flush-caches
│   └── dnsmasq: DNS+DHCP+TFTP سبک
│
├── 210.2 LDAP
│   ├── ساختار: dc(دامنه)→ou(واحد)→uid/cn(ورودی)
│   ├── LDIF + ldapadd/modify/delete/search
│   ├── objectClass: posixAccount (پل به uid/gid یونیکس)
│   └── cn=config (مدرن) vs slapd.conf (سنتی)
│
├── 210.3 PAM & NSS
│   ├── PAM: auth/account/password/session
│   ├── Flags: required/requisite/sufficient/optional
│   └── nsswitch.conf: files → ldap/dns (ترتیب جست‌وجو)
│
└── 210.4 NTP
    ├── چرا مهم: TLS, Kerberos, لاگ فارنزیک
    ├── Stratum 0(فیزیکی)→1→2...
    ├── ntpd(کلاسیک) vs chrony(مدرن) vs systemd-timesyncd(سبک)
    └── timedatectl (رابط ساده)
```

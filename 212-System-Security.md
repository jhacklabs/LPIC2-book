---
title: "Topic 212 — System Security"
exam: LPIC-2 / 202-450
weights: "212.1 (3) + 212.2 (2) + 212.3 (4) + 212.4 (3) + 212.5 (2)"
os_target: "Arch Linux / Omarchy"
tags: [lpic2, security, firewall, ssh, openvpn, fail2ban]
---

# Topic 212: System Security

> این آخرین و از نظر مسیر تو **مهم‌ترین** Topic کل LPIC-2 است — مستقیم‌ترین پل بین لینوکس ادمینی و Cyber Security.

## ۱) مفهوم کلی

پنج بخش:
- **212.1** پیکربندی یک روتر (host به‌عنوان مسیریاب/فایروال ساده)
- **212.2** مدیریت امنیت لینوکس در سطح‌های شبکه‌ای مختلف (فایروال با nftables/iptables)
- **212.3** پیکربندی FreeS/WAN، OpenVPN یا PPP
- **212.4** امنیت خدمات شبکه‌ای (TCP Wrappers، SSH سخت‌شده، fail2ban)
- **212.5** ایمن‌سازی داده با رمزنگاری (GPG)

## ۲) چرا این مبحث مهم است؟

این Topic دقیقاً همان جایی‌ست که «مدیریت لینوکس» به «امنیت سایبری» تبدیل می‌شود. فایروال، VPN، سخت‌سازی SSH، و رمزنگاری داده — این‌ها ابزارهای روزمره‌ی هر متخصص Blue Team یا حتی Red Team (برای فهم اینکه چطور دور زده می‌شوند) هستند. برای مسیری که گفتی — ورود جدی به Cyber Security بعد از LPIC — این Topic عملاً **پایه‌ی مفهومی تمام کارهای بعدی‌ات** خواهد بود.

## ۳) مثال‌های واقعی + روی سیستم خودم

**واقعی:** یک سرور SSH با پورت پیش‌فرض 22 و ورود با پسورد فعال، ظرف چند ساعت هزاران تلاش brute-force از بات‌های خودکار دریافت می‌کند. با فعال‌سازی `fail2ban` و غیرفعال کردن `PasswordAuthentication`، حمله عملاً بی‌اثر می‌شود.

**روی Omarchy:** چون از nftables (جایگزین مدرن‌تر iptables) در کرنل‌های امروزی پشتیبانی کامل می‌شود، بهترین تمرین همین سیستم خودت است — فایروال شخصی‌ات را می‌توانی همین امروز سخت‌تر کنی.

## ۴) دستورات کامل

### 212.1 — پیکربندی مسیریاب (Router)

```bash
sudo sysctl -w net.ipv4.ip_forward=1
```
برای دائمی‌کردن، در `/etc/sysctl.d/99-forwarding.conf`:
```
net.ipv4.ip_forward = 1
```
NAT ساده (Masquerading) برای اشتراک‌گذاری اینترنت بین اینترفیس‌ها:
```bash
sudo nft add rule ip nat postrouting oif "wlan0" masquerade
```

### 212.2 — فایروال (nftables / iptables)

```bash
sudo pacman -S nftables
sudo systemctl enable --now nftables
```
فایل تنظیمات: `/etc/nftables.conf`
```conf
table inet filter {
    chain input {
        type filter hook input priority 0; policy drop;
        ct state established,related accept
        iif lo accept
        tcp dport 22 accept
        tcp dport 443 accept
        icmp type echo-request accept
        counter log prefix "dropped: " drop
    }
}
```
```bash
sudo nft -f /etc/nftables.conf
sudo nft list ruleset
sudo nft flush ruleset
```
معادل سنتی iptables (که هنوز در بسیاری سرورها و در آزمون رایج است):
```bash
sudo iptables -A INPUT -p tcp --dport 22 -j ACCEPT
sudo iptables -A INPUT -j DROP
sudo iptables -L -v -n
sudo iptables-save > /etc/iptables/iptables.rules
sudo iptables-restore < /etc/iptables/iptables.rules
```
> ⚠️ **هشدار جدی:** هرگز روی یک سرور ریموت (که فقط از طریق SSH بهش دسترسی داری) بدون احتیاط قانون `DROP` روی همه چیز اضافه نکن — ممکن است خودت را قفل کنی (lockout) و دسترسی از راه دور از بین برود. همیشه اول قانون Accept برای پورت SSH را اضافه کن.

### 212.3 — VPN (OpenVPN)

```bash
sudo pacman -S openvpn easy-rsa
```
مراحل کلی ساخت PKI (زیرساخت کلید عمومی) برای گواهی‌ها:
```bash
easyrsa init-pki
easyrsa build-ca
easyrsa gen-req server nopass
easyrsa sign-req server server
easyrsa gen-dh
```
فایل تنظیمات سرور (`/etc/openvpn/server.conf`):
```conf
port 1194
proto udp
dev tun
ca ca.crt
cert server.crt
key server.key
dh dh.pem
server 10.8.0.0 255.255.255.0
```
```bash
sudo systemctl enable --now openvpn-server@server
```
IPsec (چارچوب استاندارد VPN لایه‌ی شبکه، پیاده‌سازی رایج آن **strongSwan** یا **Libreswan**، جانشین FreeS/WAN قدیمی):
```bash
sudo pacman -S strongswan
```

### 212.4 — امنیت سرویس‌های شبکه‌ای

سخت‌سازی SSH (`/etc/ssh/sshd_config`):
```conf
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
Port 2222
AllowUsers myuser
```
```bash
sudo systemctl restart sshd
ssh-keygen -t ed25519 -C "myuser@omarchy"
ssh-copy-id -p 2222 myuser@server
```
TCP Wrappers (روش قدیمی کنترل دسترسی — امروز عمدتاً منسوخ شده اما هنوز در آزمون می‌آید):
```conf
# /etc/hosts.allow
sshd: 192.168.1.0/24

# /etc/hosts.deny
sshd: ALL
```
fail2ban (بلاک خودکار IP بعد از تلاش‌های ناموفق مکرر):
```bash
sudo pacman -S fail2ban
sudo systemctl enable --now fail2ban
```
تنظیمات محلی در `/etc/fail2ban/jail.local` (هرگز مستقیم `jail.conf` را ویرایش نکن — بروزرسانی پکیج آن را overwrite می‌کند):
```ini
[sshd]
enabled = true
port = 2222
maxretry = 3
bantime = 3600
```
```bash
sudo fail2ban-client status sshd
sudo fail2ban-client set sshd unbanip 1.2.3.4
```
اسکن پورت برای ممیزی امنیتی خودت:
```bash
nmap -sV localhost
```
> ⚠️ nmap را فقط روی سیستم/شبکه خودت یا با اجازه صریح مالک اجرا کن؛ اسکن پورت شبکه‌های دیگران بدون اجازه در بسیاری قوانین جرم محسوب می‌شود.

### 212.5 — رمزنگاری داده (GPG)

```bash
gpg --full-generate-key
gpg --list-keys
gpg --armor --export user@example.com > public.key
gpg --encrypt --recipient user@example.com file.txt
gpg --decrypt file.txt.gpg > file.txt
gpg --sign file.txt
gpg --verify file.txt.sig file.txt
```
مفهوم کلیدی: رمزنگاری **نامتقارن** (کلید عمومی برای رمزنگاری، کلید خصوصی برای رمزگشایی) در مقابل رمزنگاری **متقارن** که در LUKS (Topic 203) دیدی (یک کلید مشترک برای هر دو کار).

## ۵) نکات مهم آزمون LPIC

- ✅ فرق `iptables` (مبتنی بر chain/table قدیمی) و `nftables` (جانشین رسمی، سینتکس متفاوت) را بدان؛ آزمون هر دو را می‌پرسد.
- ✅ ترتیب صحیح Chain در فایروال: `policy drop` به‌عنوان پیش‌فرض امن‌تر از `policy accept` است (deny-by-default).
- ✅ فرق `/etc/hosts.allow` و `/etc/hosts.deny` در TCP Wrappers، و اینکه اول کدام خوانده می‌شود (allow اول بررسی می‌شود).
- ✅ `fail2ban` تنظیمات محلی را همیشه در فایل `*.local`، نه `*.conf` بنویس.
- ✅ رمزنگاری نامتقارن (GPG، SSH keys) در مقابل متقارن (LUKS) — سؤال مفهومی رایج.
- ✅ SSH hardening کلاسیک: غیرفعال کردن root login و password auth، فعال کردن فقط pubkey.

## ۶) تمرین عملی امن

> ⚠️ تمرین فایروال را با احتیاط انجام بده — اگر روی سیستمی کار می‌کنی که فقط از راه دور بهش دسترسی داری، همیشه قانون Accept برای SSH را قبل از policy drop اضافه کن.

1. یک فایروال ساده و امن با nftables بساز (روی سیستم لوکال خودت):
```bash
sudo pacman -S nftables
sudo nft add table inet filter
sudo nft add chain inet filter input { type filter hook input priority 0 \; policy accept \; }
sudo nft add rule inet filter input tcp dport 22 accept
sudo nft list ruleset
```
2. یک جفت کلید SSH بساز و SSH را برای استفاده فقط از کلید سخت کن (روی یک VM تستی، نه سیستمی که تنها راه دسترسی‌اش SSH است):
```bash
ssh-keygen -t ed25519 -C "test@omarchy"
```
3. fail2ban را نصب و روی SSH فعال کن:
```bash
sudo pacman -S fail2ban
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
sudo systemctl enable --now fail2ban
sudo fail2ban-client status
```
4. یک جفت کلید GPG بساز و یک فایل تست را رمزنگاری/رمزگشایی کن:
```bash
gpg --full-generate-key
echo "متن محرمانه تست" > secret.txt
gpg --encrypt --recipient <your-email> secret.txt
gpg --decrypt secret.txt.gpg
```

## ۷) خلاصه جدولی

| مفهوم | ابزار/فایل |
|---|---|
| فورواردینگ IP | `net.ipv4.ip_forward`, `/etc/sysctl.d/` |
| فایروال مدرن | nftables — `/etc/nftables.conf` |
| فایروال کلاسیک | iptables — `iptables-save/restore` |
| VPN لایه‌ی کاربرد | OpenVPN — `easy-rsa`, `server.conf` |
| VPN لایه‌ی شبکه | IPsec — strongSwan/Libreswan |
| کنترل دسترسی قدیمی | `/etc/hosts.allow` / `hosts.deny` |
| سخت‌سازی SSH | `/etc/ssh/sshd_config` |
| بلاک خودکار brute-force | fail2ban — `jail.local` |
| رمزنگاری فایل | GPG — نامتقارن |

## ۸) مایندمپ متنی

```
Topic 212: System Security
├── 212.1 Router Configuration
│   └── ip_forward, NAT/masquerade
├── 212.2 Firewall
│   ├── nftables (modern): /etc/nftables.conf
│   └── iptables (legacy): chains, policy drop
├── 212.3 VPN
│   ├── OpenVPN: easy-rsa PKI, server.conf
│   └── IPsec: strongSwan / Libreswan
├── 212.4 Network Service Security
│   ├── SSH hardening: sshd_config
│   ├── TCP Wrappers: hosts.allow/deny
│   ├── fail2ban: jail.local
│   └── nmap (self-audit only)
└── 212.5 Data Encryption
    └── GPG: asymmetric (vs LUKS = symmetric)
```

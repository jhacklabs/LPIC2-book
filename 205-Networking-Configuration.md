---
title: "Topic 205 — Networking Configuration"
exam: LPIC-2 / 201-450
weights: "205.1 (3) + 205.2 (4) + 205.3 (4)"
os_target: "Arch Linux / Omarchy"
tags: [lpic2, networking, systemd-networkd, netplan]
---

# Topic 205: Networking Configuration

## ۱) مفهوم کلی

سه بخش:
- **205.1** پایه‌های پیکربندی شبکه پایه (رابط‌ها، مسیریابی)
- **205.2** عیب‌یابی مشکلات شبکه
- **205.3** پیکربندی خودکار آدرس شبکه (DHCP client/server)

## ۲) چرا این مبحث مهم است؟

هر سرویسی که می‌سازی — از DNS تا ایمیل تا فایل‌شیرینگ (که همه در Topicهای بعدی می‌آیند) — روی یک شبکه‌ی درست‌پیکربندی‌شده سوار است. اگر مسیریابی اشتباه باشد یا DNS جواب ندهد، هیچ سرویس دیگری هم کار نمی‌کند. برای امنیت، فهم عمیق ابزارهای عیب‌یابی شبکه (`tcpdump`, `ss`) دقیقاً همان مهارت‌هایی هستند که در تحلیل ترافیک مشکوک یا شناسایی نفوذ استفاده می‌شوند.

## ۳) مثال‌های واقعی + روی سیستم خودم

**واقعی:** یک اپلیکیشن ناگهان کند می‌شود؛ با `tcpdump` مشخص می‌شود سرور دارد بسته‌های DNS را چند بار retransmit می‌کند چون DNS server اصلی جواب نمی‌دهد — مشکل نه از کد، از شبکه بوده.

**روی Omarchy:** Arch معمولاً از **systemd-networkd** یا **NetworkManager** استفاده می‌کند (نه فایل‌های سنتی `/etc/network/interfaces` دبیان‌محور). چون از Omarchy استفاده می‌کنی که پیش‌فرض NetworkManager دارد، تمرکز عملی روی `nmcli` و `ip` است، اما آزمون فایل‌های سنتی‌تر را هم می‌پرسد.

## ۴) دستورات کامل

### 205.1 — پیکربندی پایه

```bash
ip addr show                     # آدرس‌های IP رابط‌ها
ip link show                     # وضعیت رابط‌ها (up/down)
sudo ip addr add 192.168.1.50/24 dev eth0
sudo ip link set eth0 up
sudo ip route add default via 192.168.1.1
ip route show                    # جدول مسیریابی
ip -6 addr show                  # آدرس‌های IPv6
```
پیکربندی دائمی روی Arch/Omarchy با NetworkManager:
```bash
nmcli device status
nmcli connection show
sudo nmcli connection add type ethernet ifname eth0 ip4 192.168.1.50/24 gw4 192.168.1.1
sudo nmcli connection up eth0
```
با systemd-networkd (روش دیگر رایج روی Arch):
```
# فایل /etc/systemd/network/20-wired.network
[Match]
Name=eth0
[Network]
Address=192.168.1.50/24
Gateway=192.168.1.1
```
```bash
sudo systemctl enable --now systemd-networkd
```
فایل سنتی‌تر (LPIC اغلب می‌پرسد، هرچند روی Arch وجود ندارد): `/etc/network/interfaces` (Debian) یا `/etc/sysconfig/network-scripts/ifcfg-eth0` (Red Hat).

### 205.2 — عیب‌یابی شبکه

```bash
ping -c 4 8.8.8.8
traceroute 8.8.8.8
mtr 8.8.8.8               # ترکیب ping + traceroute به‌صورت زنده
ip neigh show              # جدول ARP
sudo tcpdump -i eth0 -n port 53      # ضبط ترافیک DNS
sudo tcpdump -i eth0 -w capture.pcap  # ذخیره برای تحلیل بعدی (مثلاً در Wireshark)
dig example.com
dig +trace example.com     # مسیر کامل resolve از روت‌سرورها
host example.com
nslookup example.com
ss -tulnp                  # پورت‌های باز و پردازه‌ی صاحب هر پورت
```
> ⚠️ **نکته امنیتی:** `tcpdump` می‌تواند ترافیک رمزنگاری‌نشده (مثل HTTP، Telnet، FTP) را کامل بخواند، شامل پسورد؛ فقط روی شبکه‌ای استفاده کن که مجاز به مانیتور آن هستی.

### 205.3 — پیکربندی خودکار آدرس (DHCP)

سمت کلاینت:
```bash
sudo dhclient eth0          # درخواست دستی IP از DHCP
sudo dhclient -r eth0       # آزادسازی IP فعلی
```
سمت سرور (dhcpd — نادر روی لپ‌تاپ شخصی، رایج روی روترها/سرورها):
```bash
sudo pacman -S dhcp
```
فایل تنظیمات: `/etc/dhcpd.conf`
```
subnet 192.168.1.0 netmask 255.255.255.0 {
  range 192.168.1.100 192.168.1.200;
  option routers 192.168.1.1;
  option domain-name-servers 8.8.8.8;
}
```
```bash
sudo systemctl enable --now dhcpd4
journalctl -u dhcpd4 -f       # مانیتور زنده لاگ سرویس DHCP
```
مفاهیم کلیدی DHCP: **DORA** (Discover → Offer → Request → Acknowledge) — چهار مرحله‌ای که یک کلاینت برای گرفتن IP طی می‌کند. **Lease time** — مدت اعتبار یک IP اختصاص‌داده‌شده.

## ۵) نکات مهم آزمون LPIC

- ✅ فرآیند DORA را دقیق حفظ کن — سؤال کلاسیک است.
- ✅ تفاوت `dig` (جزئیات کامل DNS، ابزار مدرن) و `nslookup`/`host` (خروجی ساده‌تر، قدیمی‌تر ولی هنوز رایج) را بدان.
- ✅ `ip` جایگزین مدرن `ifconfig`/`route` است؛ آزمون هر دو نسل ابزار را می‌پرسد.
- ✅ فایل‌های پیکربندی سنتی توزیع‌محور را حتی روی Arch باید بشناسی (چون آزمون توزیع‌محور نیست): `/etc/network/interfaces`, `/etc/sysconfig/network`.
- ✅ `tcpdump` فیلترهای BPF را بشناس: `port`, `host`, `net`, `and`/`or`.

## ۶) تمرین عملی امن

1. وضعیت فعلی شبکه‌ات را ببین:
```bash
ip addr show
ip route show
nmcli device status
```
2. یک رابط شبکه مجازی بی‌خطر بساز (dummy interface، تأثیری روی شبکه واقعی ندارد):
```bash
sudo ip link add dummy0 type dummy
sudo ip addr add 10.10.10.1/24 dev dummy0
sudo ip link set dummy0 up
ip addr show dummy0
sudo ip link delete dummy0
```
3. ترافیک DNS خودت را ۱۰ ثانیه ضبط کن (فقط برای دیدن ساختار، بدون ذخیره‌سازی دائم):
```bash
sudo timeout 10 tcpdump -i any -n port 53
```
4. مسیر resolve یک دامنه را کامل ببین:
```bash
dig +trace example.com
```
5. جدول ARP و مسیریابی را بررسی کن:
```bash
ip neigh show
ip route show
```

## ۷) خلاصه جدولی

| مفهوم | ابزار مدرن | ابزار سنتی |
|---|---|---|
| آدرس IP | `ip addr` | `ifconfig` |
| مسیریابی | `ip route` | `route` |
| DNS lookup | `dig` | `nslookup`, `host` |
| اتصال شبکه | `nmcli`, systemd-networkd | `/etc/network/interfaces` |
| ضبط ترافیک | `tcpdump` | - |
| DHCP کلاینت | `dhclient` | - |
| DHCP سرور | `dhcpd` (`/etc/dhcpd.conf`) | - |

## ۸) مایندمپ متنی

```
Topic 205: Networking Configuration
├── 205.1 Basic Configuration
│   ├── ip addr / ip link / ip route
│   ├── nmcli, systemd-networkd
│   └── legacy: /etc/network/interfaces
├── 205.2 Troubleshooting
│   ├── ping / traceroute / mtr
│   ├── tcpdump (BPF filters)
│   └── dig / host / nslookup
└── 205.3 DHCP
    ├── DORA process
    ├── dhclient (client)
    └── dhcpd + /etc/dhcpd.conf (server)
```

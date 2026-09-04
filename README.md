# 📘 کتاب فارسی LPIC-2 — مرجع کامل آماده‌سازی آزمون‌های 201-450 و 202-450

این مخزن شامل یک دوره‌ی کامل، عمیق و ساختاریافته‌ی فارسی برای آماده‌سازی گواهی‌نامه‌ی **LPIC-2** است — دنباله‌ی مستقیم پروژه‌ی [کتاب فارسی LPIC-1](#) که قبلاً تکمیل شده.

هدف این پروژه فقط قبولی در آزمون نیست؛ فهم عمیق و کاربردی مدیریت پیشرفته‌ی لینوکس (سرورها، شبکه، ذخیره‌سازی، امنیت) به‌عنوان پایه‌ای برای ورود جدی به حوزه‌ی **Cyber Security**.

---

## 📖 درباره‌ی این کتاب

- زبان: **فارسی**
- سیستم‌عامل مرجع تمرین‌های عملی: **Arch Linux / Omarchy**
- منبع الهام اولیه: دوره‌ی فارسی جادی (برای LPIC-1) — با عمق و جزئیات به‌مراتب بیشتر
- نسخه‌ی مبنای سرفصل: **LPIC-2 v4.5** (آخرین نسخه‌ی فعال رسمی)
- هر درس به‌صورت یک فایل Markdown مستقل، قابل وارد کردن به Obsidian

---

## 🗂️ ساختار هر درس

هر فایل درس دقیقاً از این ۸ بخش تشکیل شده:

1. مفهوم کلی
2. چرا این مبحث مهم است؟
3. مثال‌های واقعی + مثال روی سیستم خودم
4. دستورات کامل همراه با توضیح دقیق هر گزینه/فلگ
5. نکات مهم آزمون LPIC (Exam Tips)
6. تمرین عملی امن (قابل‌اجرا بدون آسیب به سیستم)
7. خلاصه مخصوص دفتر (جدول مرور سریع)
8. مایندمپ متنی

پیش از هر دستور خطرناک (`dd`, `rm -rf`, تغییرات فایروال، عملیات روی پارتیشن/بوت)، هشدار صریح در متن درس آمده است.

---

## 📚 فهرست کامل دروس

### آزمون 201-450

| # | Topic | عنوان | فایل |
|---|---|---|---|
| 1 | 200 | Capacity Planning | [`200-Capacity-Planning.md`](./200-Capacity-Planning.md) |
| 2 | 201 | Linux Kernel | [`201-Linux-Kernel.md`](./201-Linux-Kernel.md) |
| 3 | 202 | System Startup | [`202-System-Startup.md`](./202-System-Startup.md) |
| 4 | 203 | Filesystem and Devices | [`203-Filesystem-and-Devices.md`](./203-Filesystem-and-Devices.md) |
| 5 | 204 | Advanced Storage Device Administration | [`204-Advanced-Storage.md`](./204-Advanced-Storage.md) |
| 6 | 205 | Networking Configuration | [`205-Networking-Configuration.md`](./205-Networking-Configuration.md) |
| 7 | 206 | System Maintenance | [`206-System-Maintenance.md`](./206-System-Maintenance.md) |

### آزمون 202-450

| # | Topic | عنوان | فایل |
|---|---|---|---|
| 8 | 207 | Domain Name Server | [`207-Domain-Name-Server.md`](./207-Domain-Name-Server.md) |
| 9 | 208 | HTTP Services | [`208-HTTP-Services.md`](./208-HTTP-Services.md) |
| 10 | 209 | File Sharing | [`209-File-Sharing.md`](./209-File-Sharing.md) |
| 11 | 210 | Network Client Management | [`210-Network-Client-Management.md`](./210-Network-Client-Management.md) |
| 12 | 211 | E-Mail Services | [`211-Email-Services.md`](./211-Email-Services.md) |
| 13 | 212 | System Security | [`212-System-Security.md`](./212-System-Security.md) |

✅ هر ۱۳ Topic رسمی تکمیل شده است.

---

## 🛠️ ابزارها و زیرساخت پروژه

- **نوشتن و آرشیو:** Obsidian (Vault مجزا برای LPIC-2)
- **ورژن‌کنترل و بک‌آپ:** Git + GitHub، از طریق پلاگین Obsidian Git (commit + push خودکار هر ۱۰ دقیقه)
- **خروجی نهایی:** ترکیب همه‌ی درس‌ها در یک EPUB/PDF واحد با pandoc + XeLaTeX
  - فونت متن فارسی: **Amiri**
  - فونت بلوک‌های کد: **FreeMono** (پشتیبانی از کامنت‌های فارسی داخل کد)
  - فلگ لازم: `--no-highlight` (جلوگیری از مشکل رندر فونت Oblique در LaTeX)
- **پلتفرم‌های تمرین عملی امنیت (مراحل بعدی):** TryHackMe، HackTheBox، OverTheWire Bandit

---

## 🖥️ نکته درباره‌ی سیستم‌عامل تمرین

تمام دستورات نصب بسته‌ها (`pacman -S ...`) و مسیرهای فایل تنظیمات، بر اساس **Arch Linux / Omarchy** نوشته شده‌اند. اگر از توزیع دیگری (Debian-family یا Red Hat-family) استفاده می‌کنی، معادل دستورات نصب (`apt`, `dnf`) و مسیرهای احتمالاً متفاوت را در بخش «دستورات کامل» هر درس جست‌وجو کن — مفاهیم و رفتار سرویس‌ها مستقل از توزیع یکسان است.

---

## 🎯 مرحله بعدی این پروژه

- [ ] ساخت فایل Roadmap برای پیگیری پیشرفت مطالعه
- [ ] ترکیب همه‌ی ۱۳ فایل در یک EPUB/PDF نهایی
- [ ] شروع تمرین عملی روی TryHackMe / HackTheBox / OverTheWire Bandit پس از اتمام هر دو گواهی‌نامه

---

## 📄 لایسنس و استفاده

این محتوا برای استفاده‌ی شخصی و آموزشی تهیه شده است. در صورت استفاده یا بازنشر، ذکر منبع محترم شمرده می‌شود.

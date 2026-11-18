# اجرای PasarGuard Node روی Zerops

این فایل برای اجرای Node رسمی PasarGuard روی سرویس **Docker** در Zerops آماده شده است. فایل اصلی تنظیمات استقرار [`zerops.yaml`](./zerops.yaml) است.

## نکته مهم درباره قابلیت‌ها

این تنظیمات برای Nodeی است که API مدیریتی را روی TCP/ gRPC و تونل WireGuard را روی UDP اجرا می‌کند. Zerops باید بتواند برای سرویس شما **Direct Port Access** بسازد؛ دسترسی عمومی HTTP به‌تنهایی کافی نیست.

همچنین سرویس را با یک instance اجرا کنید. WireGuard، routing و state تولیدشده برای چند instance بدون طراحی جداگانه برای shared storage و load balancing مناسب نیست.

## مراحل اجرا

### ۱. ساخت Project و Service در Zerops

1. وارد [app.zerops.io](https://app.zerops.io) شوید و یک Project جدید بسازید.
2. از **Add service**، سرویس **Docker** را اضافه کنید.
3. نام سرویس را دقیقاً `node` بگذارید؛ این نام با `setup: node` در `zerops.yaml` یکسان است.
4. برای شروع حداقل ۲ CPU، ۲ GB RAM و حداقل ۱۰ GB دیسک در نظر بگیرید.
5. تعداد instanceها را روی ۱ قرار دهید.

### ۲. اتصال مخزن GitHub

1. در سرویس `node` بخش **Git integration / Repository** را باز کنید.
2. مخزن خودتان را انتخاب کنید. اگر از fork استفاده می‌کنید، همان fork را انتخاب کنید.
3. شاخه‌ای را که فایل `zerops.yaml` در ریشه آن قرار دارد انتخاب کنید.
4. Pipeline را اجرا کنید. Zerops ابتدا image نسخه `v0.5.7` را دریافت و سپس سرویس را اجرا می‌کند.

### ۳. افزودن Environment Variables

در بخش Environment Variables سرویس، مقدار زیر را بسازید:

```text
API_KEY=<یک UUID جدید و محرمانه>
PG_NODE_CERT_CN=<دامنه یا IP عمومی شما>
```

برای ساخت UUID در Linux/macOS:

```bash
python3 -c 'import uuid; print(uuid.uuid4())'
```

`API_KEY` را در Git commit نکنید و برای آن از مقدار نمونه استفاده نکنید. سایر متغیرهای لازم در `zerops.yaml` قرار گرفته‌اند. اگر خروجی WireGuard از رابط خاصی باید انجام شود، می‌توانید این متغیرها را نیز در Zerops اضافه کنید:

```text
PG_NODE_WG_NAT_OUTPUT_INTERFACE=eth0
# در صورت نیاز:
# PG_NODE_WG_NAT_EGRESS_ONLY=0
# PG_NODE_WG_NAT_DISABLE=0
```

### ۴. فعال‌کردن پورت‌های عمومی

در Zerops به سرویس `node` و بخش **Subdomain & domain & IP access** بروید و **Direct Port Access** را فعال کنید:

- پورت داخلی و عمومی `62050` با پروتکل **TCP** برای gRPC/API مدیریت Node.
- پورت داخلی و عمومی `51820` با پروتکل **UDP** برای WireGuard.

اگر پنل برای هر کاربر یا هر کانفیگ WireGuard پورت دیگری تولید می‌کند، همان پورت UDP را هم در Direct Port Access اضافه کنید. پورت‌های ۸۰ و ۴۴۳ رزرو هستند؛ از بازه ۱۰ تا ۶۵۴۳۵ استفاده کنید.

در صورت امکان Firewall را روی whitelist بگذارید و پورت TCP مدیریت را فقط از IP پنل PasarGuard یا شبکه مدیریتی خود باز کنید. پورت UDP مورد نیاز کلاینت‌ها باید از اینترنت قابل دسترسی باشد.

### ۵. اتصال Node به پنل

بعد از deploy، IP عمومی Zerops و پورت‌های تنظیم‌شده را بردارید. در پنل PasarGuard یک Node جدید بسازید و موارد زیر را وارد کنید:

- آدرس: IP عمومی یا hostname مربوط به Direct Port Access
- پورت API: `62050`
- پروتکل: `gRPC`
- کلید API: همان مقدار `API_KEY`
- گواهی: چون تنظیم فعلی self-signed است، certificate تولیدشده در مسیر `/var/lib/pg-node/certs/ssl_cert.pem` را برای اعتبارسنجی TLS در اختیار پنل قرار دهید، یا در محیط production گواهی مورد اعتماد خودتان را جایگزین کنید.

برای دیدن لاگ‌ها از بخش **Logs** سرویس استفاده کنید. در لاگ موفق باید پیامی شبیه `Starting Node` ببینید.

## تست سریع

از یک سیستم بیرونی بررسی کنید:

```bash
# بررسی TCP API
nc -vz <PUBLIC_IP> 62050

# بررسی UDP فقط نشان می‌دهد پورت در مسیر شبکه تعریف شده است؛
# تست واقعی با یک peer WireGuard انجام می‌شود.
```

اگر Node اجرا نشد، این موارد را بررسی کنید:

1. مقدار `API_KEY` یک UUID معتبر باشد.
2. سرویس Docker با یک instance اجرا شده باشد.
3. پورت UDP WireGuard در Direct Port Access ساخته شده باشد.
4. در لاگ خطای `permission denied` برای WireGuard وجود نداشته باشد. Node برای ساخت interface و اعمال routing به `CAP_NET_ADMIN` و پشتیبانی kernel از WireGuard نیاز دارد؛ `zerops.yaml` مقدار `--cap-add=NET_ADMIN` و `--network=host` را تنظیم می‌کند، اما پشتیبانی kernel را باید از خود سرویس Docker Zerops تأیید کنید.
5. مقدار `PG_NODE_CERT_CN` با hostname/IP مورد استفاده کلاینت هماهنگ باشد.

## به‌روزرسانی نسخه

نسخه image عمداً روی `v0.5.7` pin شده است تا cache Zerops باعث اجرای ناخواسته image جدید نشود. برای به‌روزرسانی، مقدار image را در هر دو قسمت `prepareCommands` و `docker run` به نسخه مشخص جدید تغییر دهید، سپس دوباره deploy کنید.

## محدودیت مهم

فایل‌سیستم runtime در Zerops با هر deploy جایگزین می‌شود. این پیکربندی گواهی و داده‌های runtime را زیر `/var/lib/pg-node` نگه می‌دارد؛ برای production باید در Zerops یک **Local Storage** پایدار بسازید و آن را روی همین مسیر mount کنید. اگر Local Storage mount نشود، با deploy بعدی گواهی self-signed دوباره ساخته می‌شود و کلاینت‌هایی که certificate قبلی را trust کرده‌اند باید دوباره تنظیم شوند.

## منابع رسمی

- [مستندات PasarGuard Node](https://docs.pasarguard.org/en/node/)
- [تنظیمات Node](https://docs.pasarguard.org/en/node/configuration/)
- [Docker Service در Zerops](https://docs.zerops.io/docker/overview)
- [مشخصات zerops.yaml](https://docs.zerops.io/zerops-yaml/specification)
- [دسترسی عمومی TCP/UDP در Zerops](https://docs.zerops.io/references/networking/public-access)

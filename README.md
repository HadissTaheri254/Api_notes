<div dir="rtl">

# راهنمای جامع API Latency (تأخیر در API)

> این سند به صورت صفر تا صد مفهوم Latency در API‌ها، دلایل بروز آن، روش‌های اندازه‌گیری و راهکارهای کاهش آن را توضیح می‌دهد.

---

## فهرست مطالب

- [Latency چیست؟](#1-latency-چیست)
- [تفاوت Latency با Throughput و Response Time](#2-تفاوت-latency-با-throughput-و-response-time)
- [چرا Latency مهم است؟](#3-چرا-latency-مهم-است)
- [مراحل یک درخواست API و منابع تأخیر](#4-مراحل-یک-درخواست-api-و-منابع-تأخیر)
- [معیارهای اندازه‌گیری (Percentileها)](#5-معیارهای-اندازه‌گیری-percentileها)
- [دلایل اصلی بالا رفتن Latency](#6-دلایل-اصلی-بالا-رفتن-latency)
- [ابزارهای اندازه‌گیری و مانیتورینگ](#7-ابزارهای-اندازه‌گیری-و-مانیتورینگ)
- [راهکارهای کاهش Latency](#8-راهکارهای-کاهش-latency)
- [نمونه کد اندازه‌گیری Latency](#9-نمونه-کد-اندازه‌گیری-latency)
- [سناریوهای تست و ثبت نتایج Ping/Latency](#10-سناریوهای-تست-و-ثبت-نتایج-pinglatency)
- [منابع بیشتر](#11-منابع-بیشتر)

---

## 1. Latency چیست؟

**Latency** یا **تأخیر**، مدت زمانی است که از لحظه ارسال یک درخواست (Request) توسط کلاینت تا لحظه دریافت اولین بایت پاسخ (یا کل پاسخ) از سرور طول می‌کشد. این زمان معمولاً بر حسب میلی‌ثانیه (ms) اندازه‌گیری می‌شود.

به زبان ساده:

<div dir="ltr">

```
Latency = Response Time - Request Sent Time
```

</div>

Latency پایین یعنی کاربر سریع‌تر جواب می‌گیرد و تجربه کاربری (UX) بهتری دارد.

---

## 2. تفاوت Latency با Throughput و Response Time

| مفهوم | تعریف |
|---|---|
| **Latency** | زمان لازم برای رفت‌وبرگشت یک درخواست (تأخیر) |
| **Response Time** | کل زمانی که کاربر منتظر پاسخ کامل می‌ماند (شامل Latency + زمان پردازش) |
| **Throughput** | تعداد درخواست‌هایی که سیستم در واحد زمان می‌تواند پردازش کند (مثلاً Requests/sec) |

نکته مهم: یک سیستم می‌تواند Throughput بالایی داشته باشد ولی Latency هر درخواست هم زیاد باشد (مثلاً پردازش موازی زیاد). این دو معیار مستقل از هم هستند.

---

## 3. چرا Latency مهم است؟

- **تجربه کاربری**: کاربران انتظار پاسخ سریع دارند؛ تأخیر بالا باعث ترک اپلیکیشن می‌شود.
- **SEO و رتبه‌بندی**: موتورهای جستجو سرعت را در رتبه‌بندی سایت‌ها لحاظ می‌کنند.
- **درآمد**: مطالعات نشان داده هر 100ms تأخیر اضافه می‌تواند نرخ تبدیل (Conversion Rate) را کاهش دهد.
- **مقیاس‌پذیری**: در سیستم‌های میکروسرویس، تأخیر هر سرویس به‌صورت تجمعی روی کل زنجیره اثر می‌گذارد.

---

## 4. مراحل یک درخواست API و منابع تأخیر

هر درخواست HTTP از چند مرحله عبور می‌کند و در هر مرحله ممکن است تأخیر ایجاد شود:

<div dir="ltr">

```
Client
  │
  ├─ 1. DNS Lookup          (Domain to IP)
  ├─ 2. TCP Handshake       (Connection setup)
  ├─ 3. TLS/SSL Handshake   (Encryption - for HTTPS)
  ├─ 4. Request Sent
  ├─ 5. Server Processing   (App logic, DB queries, etc.)
  ├─ 6. Response Sent
  └─ 7. Client Receives     (Parsing / rendering)
```

</div>

| مرحله | توضیح |
|---|---|
| DNS Lookup | زمان ترجمه دامنه به آدرس IP |
| TCP Handshake | سه‌مرحله‌ای SYN, SYN-ACK, ACK برای برقراری اتصال |
| TLS Handshake | مذاکره گواهی و کلید رمزنگاری (در HTTPS) |
| TTFB (Time To First Byte) | زمان از ارسال درخواست تا دریافت اولین بایت پاسخ |
| Server Processing Time | زمان اجرای منطق برنامه، کوئری دیتابیس، فراخوانی سرویس‌های دیگر |
| Network Transfer | زمان انتقال داده در شبکه (وابسته به حجم پاسخ و پهنای باند) |

---

## 5. معیارهای اندازه‌گیری (Percentileها)

میانگین (Average) به‌تنهایی گمراه‌کننده است، چون مقادیر دورافتاده (Outlier) را پنهان می‌کند. به همین دلیل از **Percentile** استفاده می‌شود:

- **p50 (Median)**: نیمی از درخواست‌ها سریع‌تر از این مقدار پاسخ می‌گیرند.
- **p90 / p95**: 90 یا 95 درصد درخواست‌ها سریع‌تر از این مقدار هستند.
- **p99**: فقط 1 درصد درخواست‌ها کندتر از این مقدار هستند (نشان‌دهنده بدترین حالت‌ها).

> در پروژه‌های واقعی، معمولاً **p95** و **p99** برای SLA (Service Level Agreement) استفاده می‌شوند، نه میانگین.

---

## 6. دلایل اصلی بالا رفتن Latency

### سمت شبکه
- فاصله جغرافیایی زیاد بین کلاینت و سرور
- نبود CDN برای محتوای استاتیک
- عدم استفاده از HTTP/2 یا HTTP/3
- DNS کند یا عدم استفاده از Cache DNS

### سمت سرور
- کوئری‌های سنگین یا بدون ایندکس در دیتابیس
- عدم استفاده از Cache (Redis, Memcached)
- پردازش سنگین و همزمان (Blocking I/O)
- N+1 Query Problem
- عدم Connection Pooling برای دیتابیس

### سمت معماری
- زنجیره طولانی فراخوانی بین میکروسرویس‌ها (Chained Calls)
- عدم استفاده از Async/Message Queue برای عملیات سنگین
- Payload بزرگ و بدون فشرده‌سازی (Compression)
- نبود Rate Limiting مناسب که باعث Overload سرور می‌شود

### سمت کلاینت
- درخواست‌های موازی زیاد بدون batching
- عدم استفاده از Keep-Alive Connection
- Serialization/Deserialization ناکارآمد (مثلاً JSON سنگین)

---

## 7. ابزارهای اندازه‌گیری و مانیتورینگ

| ابزار | کاربرد |
|---|---|
| `curl -w` | اندازه‌گیری سریع Latency از خط فرمان |
| **Postman / Insomnia** | تست دستی API و مشاهده زمان پاسخ |
| **Apache Bench (ab)** / **wrk** | تست بار (Load Testing) ساده |
| **k6** / **JMeter** / **Locust** | تست بار پیشرفته و سناریو‌محور |
| **Prometheus + Grafana** | مانیتورینگ متریک‌ها به‌صورت Real-time |
| **Datadog / New Relic / Dynatrace** | APM (Application Performance Monitoring) در سطح Enterprise |
| **Chrome DevTools (Network tab)** | بررسی Latency در سطح مرورگر |
| **OpenTelemetry** | Distributed Tracing برای ردیابی Latency در میکروسرویس‌ها |

### مثال اندازه‌گیری با curl

<div dir="ltr">

```bash
curl -o /dev/null -s -w "\
DNS Lookup:    %{time_namelookup}s\n\
TCP Connect:   %{time_connect}s\n\
TLS Handshake: %{time_appconnect}s\n\
TTFB:          %{time_starttransfer}s\n\
Total Time:    %{time_total}s\n" \
https://api.example.com/endpoint
```

</div>

---

## 8. راهکارهای کاهش Latency

### 1. Caching (کش کردن)
- کش سمت سرور: Redis, Memcached
- کش HTTP: هدرهای `Cache-Control`, `ETag`, `Last-Modified`
- کش سمت کلاینت (Browser/App Cache)

### 2. CDN (Content Delivery Network)
توزیع محتوای استاتیک و حتی برخی API‌ها در نزدیک‌ترین نقطه به کاربر (مثل Cloudflare, Fastly, Akamai).

### 3. بهینه‌سازی دیتابیس
- ایندکس‌گذاری صحیح
- استفاده از Connection Pooling
- کاهش N+1 Query با Eager Loading / JOIN
- استفاده از Read Replica برای کوئری‌های سنگین

### 4. فشرده‌سازی (Compression)
فعال‌سازی Gzip یا Brotli برای کاهش حجم پاسخ.

### 5. پروتکل‌های جدیدتر
- استفاده از **HTTP/2** یا **HTTP/3 (QUIC)** به‌جای HTTP/1.1
- استفاده از **gRPC** به‌جای REST در ارتباطات داخلی بین سرویس‌ها (سریع‌تر به دلیل Protocol Buffers)

### 6. پردازش ناهمزمان (Async Processing)
عملیات سنگین (مثل ارسال ایمیل، پردازش تصویر) را به Message Queue (RabbitMQ, Kafka, SQS) بسپارید و بلافاصله پاسخ 202 Accepted برگردانید.

### 7. Connection Reuse
استفاده از `Keep-Alive` برای جلوگیری از باز کردن مجدد TCP/TLS Handshake در هر درخواست.

### 8. Load Balancing و Auto Scaling
توزیع بار بین چند سرور و افزایش خودکار منابع در زمان ترافیک بالا.

### 9. Pagination و محدود کردن حجم داده
به‌جای بازگرداندن کل داده، از `limit`/`offset` یا `cursor-based pagination` استفاده کنید.

### 10. Geographic Distribution
قرار دادن سرورها (یا Edge Functions) در چند منطقه جغرافیایی نزدیک به کاربران.

---

## 9. نمونه کد اندازه‌گیری Latency

### در Node.js (Express Middleware)

<div dir="ltr">

```javascript
function latencyLogger(req, res, next) {
  const start = process.hrtime.bigint();

  res.on('finish', () => {
    const end = process.hrtime.bigint();
    const latencyMs = Number(end - start) / 1_000_000;
    console.log(`${req.method} ${req.originalUrl} - ${latencyMs.toFixed(2)}ms`);
  });

  next();
}

app.use(latencyLogger);
```

</div>

### در Python (FastAPI Middleware)

<div dir="ltr">

```python
import time
from fastapi import FastAPI, Request

app = FastAPI()

@app.middleware("http")
async def add_latency_header(request: Request, call_next):
    start_time = time.time()
    response = await call_next(request)
    latency_ms = (time.time() - start_time) * 1000
    response.headers["X-Response-Time-ms"] = f"{latency_ms:.2f}"
    return response
```

</div>

---
<div dir="rtl">

## 10. سناریوهای تست و ثبت نتایج Ping/Latency

برای اینکه بتونی وضعیت واقعی یک API رو بسنجی، بهتره چند سناریوی مشخص رو تست کنی و نتایج رو مستند کنی. در ادامه چند سناریوی رایج و روش محاسبه Ping/Latency آورده شده.

### سناریوهای پیشنهادی تست

| سناریو | هدف تست |
|---|---|
| درخواست ساده GET بدون پارامتر | اندازه‌گیری Latency پایه (Baseline) |
| درخواست GET با فیلتر/جستجوی سنگین | بررسی تأثیر کوئری دیتابیس |
| درخواست POST با Payload بزرگ | بررسی تأثیر حجم داده بر Latency |
| درخواست‌های همزمان (Concurrent) | بررسی رفتار سیستم زیر بار (Load Test) |
| تست از موقعیت جغرافیایی متفاوت | بررسی تأثیر فاصله/CDN |
| تست در ساعات پیک ترافیک | بررسی پایداری Latency در بار واقعی |

### محاسبه Ping (زمان رفت‌وبرگشت ساده)

ساده‌ترین روش برای گرفتن یک تخمین اولیه از تأخیر شبکه، استفاده از دستور `ping` است:

<div dir="ltr">

```bash
ping -c 10 api.example.com
```

</div>

خروجی این دستور میانگین، کمینه، بیشینه و انحراف‌معیار زمان رفت‌وبرگشت (RTT) رو نشون می‌ده. توجه کن که `ping` از پروتکل ICMP استفاده می‌کنه، نه HTTP؛ پس فقط تأخیر شبکه رو نشون می‌ده، نه زمان پردازش واقعی API.

### محاسبه Latency واقعی API (با curl)

برای اندازه‌گیری دقیق‌تر و در سطح HTTP، از همون دستور curl که در بخش ۷ گفته شد استفاده کن و چند بار (مثلاً ۱۰ بار) تکرارش کن تا میانگین بگیری:

<div dir="ltr">

```bash
for i in {1..10}; do
  curl -o /dev/null -s -w "%{time_total}\n" https://api.example.com/endpoint
done
```

</div>

###  ثبت نتایج

| تاریخ تست | سناریو | تعداد درخواست | میانگین (ms) | p95 (ms) | p99 (ms) | بیشینه (ms) | توضیحات |
|---|---|---|---|---|---|---|---|
| ۱۴۰۴/۰۶/۰۱ | GET ساده | 100 | 45 | 80 | 120 | 150 | Baseline بدون کش |
| ۱۴۰۴/۰۶/۰۱ | GET با فیلتر سنگین | 100 | 210 | 350 | 480 | 600 | نیاز به ایندکس دیتابیس |
| ۱۴۰۴/۰۶/۰۱ | POST با Payload بزرگ | 50 | 180 | 300 | 400 | 520 | - |
| ۱۴۰۴/۰۶/۰۱ | همزمان (Concurrent 50) | 50 | 260 | 500 | 700 | 900 | نیاز به بررسی Connection Pool |

> **نکته**: برای گرفتن معیارهای p95/p99 به‌صورت خودکار، بهتره از ابزارهایی مثل **k6** یا **Apache Bench** استفاده کنی (معرفی‌شده در بخش ۷) به‌جای محاسبه دستی.

### نمونه اسکریپت تست بار با k6

<div dir="ltr">

```javascript
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  vus: 20,        // تعداد کاربران مجازی همزمان
  duration: '30s',
};

export default function () {
  const res = http.get('https://api.example.com/endpoint');
  check(res, { 'status is 200': (r) => r.status === 200 });
  sleep(1);
}
```

</div>

اجرای این اسکریپت:

<div dir="ltr">

```bash
k6 run script.js
```

</div>

k6 به‌صورت خودکار میانگین، p90، p95 و p99 رو در خروجی گزارش می‌ده؛ کافیه این اعداد رو در جدول بالا کپی کنی.

</div>



## 11. منابع بیشتر

- [Google Web.dev - Performance](https://web.dev/performance)
- [MDN - HTTP Performance](https://developer.mozilla.org/en-US/docs/Web/Performance)
- [Cloudflare Learning Center - Latency](https://www.cloudflare.com/learning/performance/glossary/what-is-latency/)
- [Grafana k6 Documentation](https://k6.io/docs/)







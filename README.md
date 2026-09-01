# گزارش پیاده‌سازی Nextcloud روی Kubernetes با Traefik

## هدف پروژه

پیاده‌سازی معماری زیر روی یک Kubernetes تک‌نودی مبتنی بر K3s:

```text
                         Internet
                             │
                             │ HTTP / HTTPS
                             ▼
                    ┌──────────────────┐
                    │     Traefik      │
                    │ Ingress Controller│
                    └─────────┬────────┘
                              │
                  ┌───────────┴───────────┐
                  │                       │
                  ▼                       ▼
      Traefik Dashboard              Nextcloud
          api@internal                    │
                                           ▼
                                      PostgreSQL
```

## مشخصات فنی

| مورد | مقدار |
|---|---|
| دامنه Nextcloud | `cloud.rezaabolghasemi.ir` |
| دامنه Traefik Dashboard | `traefik.rezaabolghasemi.ir` |
| IP سرور | `81.12.50.30` |
| Kubernetes | K3s v1.36.3+k3s1 |
| سیستم‌عامل | Ubuntu 24.04.4 LTS |
| Container Runtime | containerd |
| Traefik | نسخه 3.7.12 |
| StorageClass | local-path (WaitForFirstConsumer) |

---

## معماری نهایی

```text
                                Internet
                                    │
                     ┌──────────────▼──────────────┐
                     │        81.12.50.30           │
                     └──────────────┬──────────────┘
                                    │
                         HTTP 80 / HTTPS 443
                                    ▼
                         ┌──────────────────┐
                         │     Traefik      │
                         │ Ingress Controller│
                         │  SSL Termination  │
                         └─────────┬────────┘
                                   │
              ┌────────────────────┴────────────────────┐
              ▼                                         ▼
 traefik.rezaabolghasemi.ir                cloud.rezaabolghasemi.ir
   Path: /dashboard/                        Nextcloud IngressRoute
              │                                        │
              ▼                                        ▼
        api@internal                         Nextcloud Service
              │                                        │
              ▼                                        ▼
      Traefik Dashboard                         Nextcloud Pod
                                                        │
                                                        │ PostgreSQL
                                                        ▼
                                                 PostgreSQL Service
                                                        │
                                                        ▼
                                                 PostgreSQL Pod
```

---

## مشکلات پیش‌آمده و راه‌حل‌ها

### ۱. عدم آمادگی Node (`NotReady`)

در شروع کار، وضعیت Node به‌صورت زیر بود:

```text
NAME     STATUS     ROLES
ubuntu   NotReady   control-plane
```

با بررسی `kubectl describe node` و اطمینان از سالم بودن Memory/Disk/PID Pressure، Node به وضعیت `Ready` بازگشت.

---

### ۲. Traefik پیش‌فرض K3s فعال نبود

- Pod مربوط به Traefik در `kube-system` وجود نداشت.
- Service وجود داشت اما `Endpoints` آن خالی بود (`<none>`).
- بررسی `helmcharts` نشان داد HelmChart و Manifest وجود دارند، اما Deployment واقعی ساخته نشده بود.

**نتیجه:** وجود Service به معنای سالم بودن سرویس نیست؛ باید Pod و Endpoint هم بررسی شوند.

**راه‌حل:** نصب مستقل Traefik از طریق Helm، به‌جای تکیه بر Traefik نیمه‌فعال پیش‌فرض K3s.

---

### ۳. خطای Helm در نصب مجدد

```text
cannot reuse a name that is still in use
```

**علت:** Release با نام `traefik` از قبل وجود داشت.

**راه‌حل:** استفاده از `helm upgrade` به‌جای `helm install` برای نصب‌های بعدی.

---

### ۴. هشدار Deprecation در Gateway API

پس از نصب Traefik، هشدار داده شد که CRDهای Gateway API در آینده همراه Chart ارائه نخواهند شد.

**راه‌حل:** نصب مستقیم Gateway API از مخزن رسمی (نسخه v1.5.1).

---

### ۵. Pending ماندن PVC

PVCهای PostgreSQL و Nextcloud پس از ساخت در وضعیت `Pending` قرار داشتند.

**علت:** StorageClass از حالت `WaitForFirstConsumer` استفاده می‌کند؛ یعنی PV تا زمان Schedule شدن Pod مصرف‌کننده ساخته نمی‌شود.

**نتیجه:** `Pending` بودن PVC در این حالت طبیعی است و لزوماً خطا نیست.

---

### ۶. تست اتصال داخلی (DNS و شبکه)

با اجرای یک Pod آزمایشی BusyBox، اتصال به سرویس‌های داخلی تست شد:

- `nslookup postgres` → `postgres.nextcloud.svc.cluster.local`
- `nc -zv postgres 5432` → `open`
- `nslookup nextcloud` → `nextcloud.nextcloud.svc.cluster.local`
- `wget -qO- http://nextcloud` → دریافت موفق صفحه HTML

این تست‌ها سلامت مسیر CoreDNS → Service → Pod را تأیید کردند.

---

### ۷. تست Routing با IngressRoute

با تعریف یک `IngressRoute` آزمایشی روی `nextcloud.local` و تست با `curl` (با هدر `Host` دستی)، کد پاسخ `200` دریافت شد. این موضوع صحت مسیر Traefik → IngressRoute → Service → Pod را تأیید کرد.

---

### ۸. خطای `405` هنگام تست Dashboard

```bash
curl -I -H "Host: dashboard.localhost" http://81.12.50.30
```

**علت:** درخواست `HEAD` (ناشی از فلگ `-I`) توسط Dashboard پشتیبانی نمی‌شد.

**راه‌حل:** استفاده از درخواست `GET` که پاسخ `302 Redirect` صحیح را برگرداند.

---

### ۹. Certificate منقضی‌شده

بررسی SSL نشان داد Certificate قبلی منقضی شده بود (`notAfter=Aug 25 2026`). Certificate جدید از نوع **Wildcard** (`*.rezaabolghasemi.ir`) از Let's Encrypt دریافت و جایگزین شد.

---

### ۱۰. اشتباه در مسیر فایل Private Key

مسیر اشتباه:

```text
/data/cert/private.key
```

مسیر صحیح:

```text
/data/cert/privkey.pem
```

با بررسی محتویات پوشه (`ls -lah`) مشکل شناسایی و اصلاح شد.

---

### ۱۱. عدم بروزرسانی خودکار Secret در Kubernetes

**نکته کلیدی:** جایگزینی فایل Certificate روی دیسک سرور (`/data/cert/`) به‌تنهایی کافی نیست؛ Traefik گواهی را از **Kubernetes Secret** می‌خواند، نه مستقیماً از فایل روی سرور. بنابراین پس از هر تغییر Certificate، Secret مربوطه باید مجدداً با `kubectl create secret tls ... --dry-run=client -o yaml | kubectl apply -f -` بروزرسانی شود.

---

### ۱۲. خطای `404 page not found`

پس از تنظیم SSL، خطای ۴۰۴ رخ داد که نیاز به بررسی زنجیره‌ای از اجزا داشت:

```text
DNS → Traefik Service → EntryPoint → IngressRoute → Service → Pod
```

با بررسی EntryPointهای `web` (HTTP) و `websecure` (HTTPS) و اطمینان از Map شدن صحیح پورت‌های ۸۰/۴۴۳ مشکل برطرف شد.

---

### ۱۳. مسیر نادرست برای دسترسی به Dashboard

آدرس ریشه (`/`) برای Dashboard مسیر معتبری نبود؛ آدرس صحیح دسترسی:

```text
https://traefik.rezaabolghasemi.ir/dashboard/
```

---

### ۱۴. Redirect نادرست Nextcloud به HTTP

از آنجا که ارتباط بین Traefik و Nextcloud داخل Cluster به‌صورت HTTP برقرار می‌شد، Nextcloud تصور می‌کرد کل اتصال HTTP است و کاربر را به آدرس `http://` ریدایرکت می‌کرد.

**راه‌حل:** تنظیم آگاهی Nextcloud از Reverse Proxy:

```bash
php occ config:system:set overwriteprotocol --value=https
php occ config:system:set overwritehost --value=cloud.rezaabolghasemi.ir
php occ config:system:set overwrite.cli.url --value=https://cloud.rezaabolghasemi.ir
```

---

## درس‌های کلیدی این پروژه

1. **وجود Service به معنی سالم بودن برنامه نیست** — همیشه باید `pods`، `svc` و `endpoints` هر سه بررسی شوند.
2. **Pending بودن PVC همیشه خطا نیست** — با StorageClass از نوع `WaitForFirstConsumer` طبیعی است.
3. **DNS داخلی Kubernetes** بر پایه الگوی `service.namespace.svc.cluster.local` کار می‌کند.
4. **Traefik صرفاً یک Reverse Proxy ساده نیست** — مسئول Routing، TLS Termination، Service Discovery، CRD Integration و Dashboard است.
5. **Certificate روی سرور با Certificate داخل Kubernetes متفاوت است** — Traefik گواهی را از Secret می‌خواند، نه از فایل سرور.
6. **برنامه پشت Reverse Proxy باید از وجود Proxy آگاه باشد** — تنظیماتی مانند `overwriteprotocol` و `trusted_proxies` برای این منظور ضروری‌اند.

---

## وضعیت فعلی پروژه

| بخش | وضعیت |
|---|---|
| Kubernetes Node | ✅ Ready |
| K3s | ✅ در حال اجرا |
| CoreDNS | ✅ |
| StorageClass | ✅ local-path |
| PostgreSQL PVC | ✅ Bound |
| PostgreSQL Pod | ✅ Running |
| PostgreSQL Service | ✅ |
| DNS داخلی PostgreSQL | ✅ |
| Nextcloud PVC | ✅ Bound |
| Nextcloud Pod | ✅ Running |
| Nextcloud Service | ✅ |
| DNS داخلی Nextcloud | ✅ |
| Traefik | ✅ Running |
| Traefik LoadBalancer | ✅ |
| Traefik CRD | ✅ |
| IngressRoute Nextcloud | ✅ |
| Routing داخلی Traefik | ✅ |
| DNS دامنه‌ها | ✅ |
| TLS Secret | ✅ |
| Wildcard Certificate | ✅ |
| SSL Traefik | ✅ |
| Traefik Dashboard | ✅ |
| Traefik Dashboard HTTPS | ✅ |
| Nextcloud HTTPS | ✅ |
| HTTPS Redirect Awareness | ✅ |
| trusted_proxies | ⏳ مرحله بعد |

---

## نقشه راه (مراحل بعدی)

1. **تکمیل `trusted_proxies`** — تعریف شبکه Pod (`10.42.0.0/24`) یا آدرس دقیق Traefik به‌عنوان Proxy مورد اعتماد Nextcloud.
2. **HTTP → HTTPS Redirect** — ریدایرکت خودکار تمام درخواست‌های HTTP به HTTPS در سطح Traefik.
3. **Security Headers** — افزودن هدرهایی مانند `Strict-Transport-Security`، `X-Content-Type-Options`، `X-Frame-Options` و `Referrer-Policy` از طریق Traefik Middleware.
4. **Authentication برای Dashboard** — افزودن BasicAuth یا ForwardAuth برای جلوگیری از دسترسی عمومی به Dashboard.
5. **Health Checks** — تنظیم بررسی سلامت برای سرویس‌ها.
6. **Resource Limits** — تعیین `requests`/`limits` برای CPU و Memory سرویس‌های Nextcloud، PostgreSQL و Traefik.
7. **Backup PostgreSQL** — پیاده‌سازی Backup زمان‌بندی‌شده با Kubernetes CronJob.
8. **Backup Nextcloud Data** — پشتیبان‌گیری جداگانه از PVC داده‌های Nextcloud و پایگاه‌داده PostgreSQL.
9. **Monitoring** — افزودن پایش وضعیت سرویس‌ها و منابع Cluster.

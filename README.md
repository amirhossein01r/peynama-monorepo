# پی‌نما (Peynama)

پروژه از دو بخش تشکیل شده:

| بخش       | مسیر          | تکنولوژی                                                                  |
| --------- | ------------- | ------------------------------------------------------------------------- |
| بک‌اند    | `peynama/`    | Django 6 + Django REST Framework + Celery + PostgreSQL (pgvector) + Redis |
| فرانت‌اند | `peynama-ui/` | React 19 + Vite + TanStack Router/Query + Tailwind CSS                    |

---

## ۱. پیش‌نیازها

قبل از شروع، این ابزارها باید روی سیستم نصب باشند:

- **Python 3.13+**
- **[Poetry](https://python-poetry.org/)** برای مدیریت پکیج‌های پایتون
- **Node.js 20+** و **[pnpm](https://pnpm.io/)** برای فرانت‌اند
- **Docker** و **Docker Compose** برای بالا آوردن دیتابیس و Redis
- **[Ollama](https://ollama.com/)** به همراه مدل `translategemma:12b` (برای فعال بودن قابلیت ترجمهٔ خودکار محتوا)

> اگر فقط می‌خواهید فرانت‌اند و API اصلی را ببینید (بدون فرآیند ایمپورت/ترجمه)، نیازی به نصب Ollama نیست.

---

## ۲. راه‌اندازی سرویس‌های زیرساخت با Docker

فایل `docker-compose.yml` در مسیر `peynama/` دو سرویس زیر را بالا می‌آورد:

- **PostgreSQL** (با اکستنشن pgvector) روی پورت `5433`
- **Redis** (برای Celery) روی پورت `6379`

برای اجرا:

```bash
cd peynama
docker compose up -d
```

بعد از اجرا می‌توانید با دستور زیر وضعیت کانتینرها را بررسی کنید:

```bash
docker compose ps
```

> توجه: مقدار پسورد دیتابیس در `docker-compose.yml` و `peynama/settings.py` به‌صورت پیش‌فرض `PASSWORD` تنظیم شده است. برای محیط توسعه مشکلی ندارد، ولی حتماً پیش از استفاده در محیط واقعی (production) آن را تغییر دهید.

---

## ۳. راه‌اندازی بک‌اند (Django)

### ۳.۱. نصب وابستگی‌ها

```bash
cd peynama
poetry install
```

### ۳.۲. اجرای Migration ها

```bash
poetry run python manage.py migrate
```

### ۳.۳. ساخت کاربر ادمین (اختیاری، برای دسترسی به پنل ادمین)

```bash
poetry run python manage.py createsuperuser
```

### ۳.۴. اجرای سرور توسعه

```bash
poetry run python manage.py runserver
```

سرور روی آدرس `http://localhost:8000` بالا می‌آید.

### ۳.۵. اجرای Celery Worker

فرآیند ایمپورت عنوان‌ها (فیلم/سریال) به‌صورت async و از طریق Celery انجام می‌شود، پس باید یک worker جدا هم اجرا شود:

```bash
cd peynama
poetry run celery -A peynama worker --loglevel=info
```

### ۳.۶. نکات مهم پیکربندی

- **کلید API تی‌ام‌دی‌بی (TMDB):** در حال حاضر در فایل `core/tasks.py` به‌صورت مقدار placeholder یعنی `"API Key"` قرار دارد. برای فعال‌شدن ایمپورت واقعی از TMDB، باید این مقدار را با کلید API واقعی خودتان (از [themoviedb.org](https://www.themoviedb.org/settings/api)) جایگزین کنید.

- **ترجمه با Ollama:** سرویس `core/services/translator.py` از مدل `translategemma:12b` روی Ollama محلی استفاده می‌کند. پیش از اجرای فرآیند ایمپورت، مطمئن شوید Ollama در حال اجراست و مدل موردنظر pull شده باشد:
  ```bash
  ollama pull translategemma:12b
  ```
- **محاسبهٔ embedding عنوان‌ها** (برای سیستم پیشنهاددهی) با دستور مدیریتی زیر انجام می‌شود:
  ```bash
  poetry run python manage.py fill_title_embeddings
  ```

### ۳.۷. اجرای تست‌ها

```bash
cd peynama
poetry run pytest
```

---

## ۴. راه‌اندازی فرانت‌اند (React)

```bash
cd peynama-ui
pnpm install
pnpm dev
```

اپلیکیشن روی آدرس `http://localhost:5173` در دسترس خواهد بود.

فرانت‌اند به‌صورت پیش‌فرض به آدرس `http://localhost:8000` (بک‌اند) وصل می‌شود (تعریف‌شده در `src/lib/api.ts`) و برای احراز هویت از کوکی refresh token + کوکی‌های httpOnly استفاده می‌کند، پس حتماً بک‌اند باید هم‌زمان اجرا باشد.

سایر اسکریپت‌های مفید:

```bash
pnpm build       # ساخت نسخهٔ production
pnpm preview     # پیش‌نمایش نسخهٔ production
pnpm lint        # اجرای ESLint
pnpm storybook   # اجرای Storybook روی پورت 6006
```

---

## ۵. مستندات خودکار API (Swagger / Redoc)

بک‌اند از `drf-spectacular` استفاده می‌کند و مستندات تعاملی به‌صورت خودکار تولید می‌شود:

- Swagger UI: `http://localhost:8000/api/docs/`
- Redoc: `http://localhost:8000/api/redoc/`
- Schema خام (OpenAPI JSON/YAML): `http://localhost:8000/api/schema/`

---

## ۶. فهرست Endpoint های API

### ۶.۱. احراز هویت (`/auth/`)

| متد  | مسیر             | توضیح                                                                              | نیاز به احراز هویت |
| ---- | ---------------- | ---------------------------------------------------------------------------------- | ------------------ |
| POST | `/auth/login/`   | ورود کاربر؛ access token در پاسخ و refresh token در کوکی httpOnly برگردانده می‌شود | خیر                |
| POST | `/auth/refresh/` | صدور access token جدید با استفاده از کوکی refresh                                  | خیر (نیاز به کوکی) |
| POST | `/auth/logout/`  | خروج و بی‌اعتبارسازی refresh token (blacklist)                                     | خیر                |
| GET  | `/auth/me/`      | دریافت اطلاعات کاربر لاگین‌شده (username)                                          | بله                |

### ۶.۲. عنوان‌ها، ایمپورت و کتابخانه (`/api/v1/`)

| متد                 | مسیر                                                 | توضیح                                                                                                                      | نیاز به احراز هویت |
| ------------------- | ---------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------- | ------------------ |
| POST                | `/api/v1/import/<title_type>/`                       | شروع فرآیند ایمپورت یک عنوان جدید از TMDB (`title_type`: `movie` یا `tv_show`)، صف‌شدن job در Celery                       | خیر                |
| GET                 | `/api/v1/import/<title_type>/<external_id>/`         | مشاهدهٔ جزئیات و وضعیت یک job ایمپورت                                                                                      | خیر                |
| PATCH               | `/api/v1/import/<title_type>/<external_id>/`         | ویرایش دستی بخش ترجمه‌شدهٔ payload قبل از تایید نهایی                                                                      | خیر                |
| POST                | `/api/v1/import/<title_type>/<external_id>/retry/`   | تلاش مجدد برای job هایی که با شکست مواجه شده‌اند (`failed`)                                                                | خیر                |
| POST                | `/api/v1/import/<title_type>/<external_id>/approve/` | تایید نهایی job در وضعیت `review` و ذخیرهٔ عنوان در دیتابیس اصلی                                                           | خیر                |
| GET                 | `/api/v1/movies/<id>-<slug>`                         | دریافت جزئیات یک فیلم؛ اگر slug اشتباه باشد، مسیر صحیح (`canonical_route`) برگردانده می‌شود                                | خیر                |
| GET                 | `/api/v1/tv-shows/<id>-<slug>`                       | دریافت جزئیات یک سریال (مشابه فیلم)                                                                                        | خیر                |
| GET                 | `/api/v1/<metadata_type>/<slug>`                     | لیست عنوان‌های مرتبط با یک متادیتا؛ `metadata_type` یکی از: `languages`، `genres`، `countries`، `keywords` (صفحه‌بندی‌شده) | خیر                |
| GET / POST / DELETE | `/api/v1/titles/<title_id>/tracking/`                | مشاهده / ثبت / حذف (نرم) وضعیت تماشای یک عنوان توسط کاربر                                                                  | بله                |
| GET                 | `/api/v1/library/<slug>`                             | لیست عنوان‌های کتابخانهٔ شخصی کاربر؛ `slug` یکی از: `watched`، `plan-to-watch`، `dropped` (صفحه‌بندی‌شده)                  | بله                |
| GET                 | `/api/v1/home-snippets/`                             | داده‌های صفحهٔ اصلی: پیشنهادها + خلاصهٔ کتابخانهٔ شخصی (هرکدام حداکثر ۵ آیتم)                                              | بله                |
| GET                 | `/api/v1/recommendations/`                           | لیست کامل پیشنهادها بر اساس تاریخچهٔ تماشا (صفحه‌بندی‌شده)                                                                 | بله                |
| GET                 | `/api/v1/search?q=...`                               | جست‌وجوی عنوان بر اساس عنوان (حداکثر ۱۰ نتیجه)                                                                             | خیر                |

**وضعیت‌های ایمپورت (`ImportJob.Status`):** `pending`, `processing`, `review`, `completed`, `failed`

**وضعیت‌های ردیابی (`TitleTracking.Status`):** `plan_to_watch`, `completed`, `dropped`

> برای مسیرهایی که نیاز به احراز هویت دارند، توکن دسترسی باید در هدر `Authorization: Bearer <access_token>` ارسال شود.

---

## ۷. ساختار کلی پروژه

```
project/
├── peynama/                # بک‌اند Django
│   ├── peynama/             # تنظیمات پروژه (settings, urls, celery, wsgi/asgi)
│   ├── authentication/      # اپ احراز هویت (JWT)
│   ├── core/                # اپ اصلی: مدل‌ها، سریالایزرها، سرویس‌ها، API
│   │   ├── api/              # views و urls مربوط به API
│   │   ├── services/         # سرویس‌های TMDB، ترجمه، تصاویر، پرسیستنس
│   │   ├── schemas/           # اسکیمای Pydantic
│   │   └── recommender.py    # منطق پیشنهاددهی مبتنی بر pgvector
│   └── docker-compose.yml   # سرویس‌های PostgreSQL و Redis
└── peynama-ui/              # فرانت‌اند React (Vite + TanStack Router)
    └── src/
        ├── pages/ , routes/  # صفحات و مسیرها
        ├── components/       # کامپوننت‌های UI
        ├── lib/              # api client، auth، query client و ...
        └── stories/          # استوری‌های Storybook
```

---

## ۸. جمع‌بندی سریع اجرای کامل پروژه (Quick Start)

```bash
# ۱. بالا آوردن دیتابیس و ردیس
cd peynama
docker compose up -d

# ۲. نصب و اجرای بک‌اند
poetry install
poetry run python manage.py migrate
poetry run python manage.py runserver

# ۳. در یک ترمینال جدا: اجرای Celery worker
poetry run celery -A peynama worker --loglevel=info

# ۴. در یک ترمینال جدا: اجرای فرانت‌اند
cd ../peynama-ui
pnpm install
pnpm dev
```

سپس فرانت‌اند روی `http://localhost:5173` و API روی `http://localhost:8000` در دسترس است.

---
## ۹. اجرای سریع با بکاپ و Images آماده (Quick Test Setup)

اگر فایل بکاپ دیتابیس (`db-backup.dump`) و پوشهٔ `images/` از قبل به‌صورت دستی داخل مونوریپو قرار داده شده باشند:

```
peynama-monorepo/
├── db-backup.dump  # فایل dump دیتابیس
├── images/          # نسخهٔ آمادهٔ media
├── peynama/          # بک‌اند
└── peynama-ui/       # فرانت‌اند
```

برای اجرای سریع، کافیست دیتابیس را از بکاپ restore کنید و پوشهٔ images را داخل `peynama` بگذارید:

```bash
cd peynama
docker compose up -d

# restore بکاپ
PGPASSWORD=PASSWORD pg_restore -h localhost -p 5433 -U postgres -d peynama_db \
  --clean --if-exists ../db-backup.dump

# قرار دادن پوشهٔ images
cp -r ../images images
```

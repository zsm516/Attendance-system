# سامانه حضور و غیاب (Django + React + Android)

سامانه‌ی چندشرکتی (Multi-tenant) برای ثبت ورود و خروج، مدیریت شیفت، مرخصی و گزارش کارکرد.
الهام‌گرفته از مفاهیم سامانه‌های Ding Ontime و Finto: مشاهده‌ی زنده‌ی وضعیت حاضرین، ثبت با موقعیت مکانی (جیوفنس)،
درخواست مرخصی/ماموریت، گزارش ماهانه و کارکرد آفلاین موبایل.

## ساختار

```
backend/    Django 5/6 + DRF (API + پنل ادمین)
  apps/core          احراز هویت JWT، دسترسی‌ها، توابع تقویم جلالی
  apps/accounts      کاربر سفارشی (مدیر/کارمند)، ورود، ثبت شرکت، پروفایل
  apps/companies     شرکت، تنظیمات (تعطیلی هفتگی، جیوفنس، مهلت تأخیر)، تعطیلات
  apps/employees     کارمند + شیفت هفتگی (تمام‌وقت/پاره‌وقت)
  apps/attendance    ثبت ورود/خروج، همگام‌سازی آفلاین، تابلوی مدیر، تقویم ماهانه
  apps/leaves        درخواست مرخصی (روزانه/ساعتی/استعلاجی/ماموریت) و تأیید/رد
  apps/notifications اعلان‌ها
  apps/reports       گزارش ماهانه + خروجی CSV
frontend/   React + Vite (راست‌چین، تم تیره، PWA) + پروژه‌ی Capacitor Android
.github/workflows/android.yml   ساخت خودکار APK
```

هر اپ مستقل است (مدل، سریالایزر، ویو، URL و ادمین جدا) و فقط از طریق API/ForeignKey با بقیه ارتباط دارد.
یک API واحد (`/api/v1/`) هم وب و هم اپ اندروید را تغذیه می‌کند؛ بنابراین همه‌چیز همیشه همگام است.

## اجرای محلی

```bash
cd backend
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
python manage.py migrate
python manage.py createsuperuser        # برای پنل ادمین /admin/
python manage.py seed_demo              # (اختیاری) داده‌ی نمونه
python manage.py runserver
```

فرانت (حالت توسعه):

```bash
cd frontend && npm install && npm run dev      # http://localhost:5173  (پروکسی به :8000)
```

فرانت (حالت یکپارچه، Django خودش سرو می‌کند):

```bash
cd frontend && npm run build
rm -rf ../backend/frontend_dist && cp -r dist ../backend/frontend_dist
cd ../backend && python manage.py runserver     # http://localhost:8000
```

حساب‌های نمونه (بعد از `seed_demo`): مدیر `09120000000` و کارمندان `09121000001` تا `09121000005`، رمز همه `demo1234`.

## پنل‌ها

| پنل | آدرس | کاربر |
|---|---|---|
| ادمین جنگو (سوپرادمین سامانه: شرکت‌ها، کاربران، رکوردها) | `/admin/` | `createsuperuser` |
| پنل مدیر شرکت (داشبورد، کارمندان، مرخصی‌ها، گزارش، تنظیمات) | `/` | مدیری که شرکت ثبت کرده |
| پنل کارمند (ثبت ورود/خروج، کارکرد من، مرخصی) | `/` | کارمند |

## اپ اندروید

اپ همان فرانت React است که با Capacitor در یک WebView بسته‌بندی می‌شود (GPS با افزونه‌ی بومی).

**روش ۱ – Android Studio:**
```bash
cd frontend
npm install
VITE_API_URL=https://your-domain.com/api/v1 npm run cap:sync   # VITE_API_URL اختیاری است
npm run android:open       # در Android Studio: Build > Build APK(s)
```
**روش ۲ – خط فرمان** (نیازمند JDK 17 و Android SDK):
```bash
cd frontend/android && ./gradlew assembleDebug
# خروجی: app/build/outputs/apk/debug/app-debug.apk
```
**روش ۳ – GitHub Actions:** پروژه را روی GitHub بگذارید و workflow به نام «Build Android APK» را اجرا کنید؛ APK به‌صورت artifact دانلود می‌شود.

اگر `VITE_API_URL` تعیین نشود، در اولین اجرا در صفحه‌ی ورود «آدرس سرور» پرسیده می‌شود.
برای انتشار در گوگل‌پلی باید APK/AAB را با keystore خودتان امضا کنید.

## همگام‌سازی آفلاین

اگر اینترنت قطع باشد، ورود/خروج با زمان واقعی دستگاه و یک `client_uuid` در صف محلی ذخیره می‌شود و با وصل شدن
اینترنت به `POST /api/v1/attendance/sync/` ارسال می‌شود. سرور درخواست‌های تکراری را نادیده می‌گیرد (idempotent)،
زمان آینده یا قدیمی‌تر از ۳ روز را رد می‌کند و ترتیب رویدادها را رعایت می‌کند.

## استقرار (Production)

1. `cp backend/.env.example backend/.env` و مقدارها (SECRET_KEY، `DJANGO_DEBUG=0`، دامنه، دیتابیس Postgres) را تنظیم کنید.
2. `pip install -r requirements.txt && python manage.py migrate && python manage.py collectstatic --noinput`
3. اجرا با `gunicorn config.wsgi:application --bind 0.0.0.0:8000` پشت Nginx با HTTPS (برای GPS مرورگر HTTPS لازم است).
4. پوشه‌ی `media/` را روی دیسک پایدار نگه دارید.

## API (خلاصه)

`POST auth/register-company/ · auth/login/ · auth/refresh/ · GET/PATCH auth/me/ · POST auth/change-password/`
`GET/PATCH company/ · company/holidays/` · `employees/` (CRUD) ·
`attendance/today/ · check-in/ · check-out/ · sync/ · my/ · board/ · employee/<id>/ · manual/ · records/<id>/`
`leaves/ (+approve/reject) · notifications/ · reports/monthly/ · reports/monthly.csv`

## تست

```bash
cd backend && python manage.py test     # ۱۴ تست: احراز هویت، جداسازی شرکت‌ها، جیوفنس، همگام‌سازی آفلاین، مرخصی، گزارش…
```

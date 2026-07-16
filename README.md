# معمار (The Architect) — فاز ۱

این فاز شامل: احراز هویت (ثبت‌نام/ورود با فایربیس)، اتصال به Firestore، و داشبورد اصلی
با کارت‌های آماری (مطالعه، ورزش، اسکرین‌تایم، XP امروز) + نمودار روند ۷ روز اخیر + فرم ثبت
دستی. فازهای بعدی (اهداف، Skill Tree، عادت‌ها، ژورنال هوشمند، Cloud Functions تحلیل شبانه و...)
روی همین پایه اضافه می‌شوند.

## ⚠️ نکته مهم درباره‌ی پوشه android/

چون این پروژه بدون نصب Flutter روی سیستم من ساخته شده، پوشه‌ی استاندارد `android/`
(که معمولاً با دستور `flutter create` تولید می‌شود) در این پکیج **وجود ندارد**.
به‌جایش `codemagic.yaml` طوری تنظیم شده که در همان سرور بیلد Codemagic (که Flutter
از قبل روی آن نصب است) این پوشه را خودش بسازد و تنظیمات فایربیس را داخلش قرار دهد.
پس نگران نباش — نیازی به نصب Flutter روی لپ‌تاپت نیست.

## مراحل راه‌اندازی (قبل از آپلود به Codemagic)

### ۱. ساخت پروژه Firebase

۱. برو به [console.firebase.google.com](https://console.firebase.google.com) و یک پروژه جدید بساز (مثلاً `architect-app`).
۲. داخل پروژه، یک اپ Android اضافه کن با **Package name دقیقاً برابر**:
   ```
   com.farzan.architect
   ```
   (اگر می‌خواهی اسم دیگری بگذاری، هم اینجا و هم در `ci/android_app_build.gradle`
   مقدار `applicationId` و `namespace` را همزمان عوض کن.)
۳. فایل `google-services.json` را دانلود کن.
۴. در Firebase Console:
   - **Authentication → Sign-in method → Email/Password** را فعال کن.
   - **Firestore Database** را بساز (Start in production mode کافی است، چون فایل
     `firestore/firestore.rules` قوانین امن را برایت نوشته).
   - محتوای `firestore/firestore.rules` را در تب **Rules** پیست و Publish کن.

### ۲. قرار دادن google-services.json

دو راه داری (یکی را انتخاب کن):

**راه ساده (برای اپ شخصی مشکلی ندارد):**
فایل دانلودشده را مستقیماً در مسیر زیر داخل ریپو/زیپ قرار بده:
```
android/app/google-services.json
```
چون پوشه `android/` هنوز وجود ندارد، بعد از اولین بیلد Codemagic (که این پوشه را
می‌سازد) باید این فایل را دوباره در پروژه commit کنی، یا از روش دوم استفاده کنی.

**راه امن‌تر (توصیه‌شده):** در Codemagic یک Environment Variable Group به اسم
`firebase_config` بساز و متغیر `GOOGLE_SERVICES_JSON` را برابر با محتوای
base64-شده‌ی فایل قرار بده:
```bash
base64 -i google-services.json | pbcopy   # مک
base64 google-services.json               # لینوکس/ویندوز (WSL)
```
مقدار خروجی را در Codemagic به عنوان مقدار متغیر (secure) ذخیره کن.
`codemagic.yaml` خودش این را در زمان بیلد به فایل تبدیل می‌کند.

### ۳. آپلود در Codemagic

۱. پروژه را به‌صورت زیپ یا از طریق یک ریپوی Git (GitHub/GitLab/Bitbucket) به Codemagic بده.
۲. Codemagic باید فایل `codemagic.yaml` را خودش شناسایی کند (workflow با نام
   `architect-android`).
۳. بیلد را اجرا کن. خروجی، فایل APK نصبی زیر مسیر
   `build/app/outputs/flutter-apk/app-release.apk` خواهد بود که Codemagic
   به‌صورت artifact قابل دانلود نمایش می‌دهد.

## ساختار داده Firestore (فاز ۱)

```
users/{uid}
  email, displayName, createdAt, xpRates

users/{uid}/daily_logs/{yyyy-MM-dd}
  studyMinutes, workoutMinutes, totalScreenTimeMinutes,
  instagramMinutes, youtubeMinutes, sleepHours, moodScore,
  xpEarned, journalWritten

users/{uid}/screen_time/{yyyy-MM-dd}
  totalScreenTimeMinutes, instagramMinutes, youtubeMinutes, lastUpdated
```

این ساختار عیناً همان چیزی است که در بخش ۱ پرامپت اصلی توضیح داده شده، تا Cloud
Function تحلیل شبانه در فاز بعد بدون تغییر ساختار داده روی همین بنا شود.

## فاز ۲ — سیستم هدف + DailyBriefing واقعی

### ⚠️ چرا بدون Cloud Functions؟
طبق بررسی دقیق، **Cloud Functions حتی در حجم مصرف رایگان هم نیاز به فعال‌سازی پلن
Blaze (پرداخت به‌ازای مصرف + کارت بانکی) دارد** — این محدودیتی از طرف گوگل است و
هیچ راه دوری ندارد. چون خواسته بودی کاملاً روی Spark Plan بمانیم، منطق
`analyzeDayAndPlanTomorrow` را **داخل خود اپ فلاتر** پیاده کردم: هر بار اپ باز
می‌شود، اگر بریفینگ امروز هنوز ساخته نشده باشد، داده‌ی دیروز را با میانگین ۷ روز
قبل‌تر مقایسه می‌کند، علت ریشه‌ای را حدس می‌زند، و متن انگیزشی می‌سازد — دقیقا
همان منطق Cloud Function توضیح‌داده‌شده در پرامپت، فقط اجراکننده‌اش کلاینت است
نه سرور. یک دکمه‌ی 🔄 «تحلیل مجدد» هم روی کارت گذاشته‌ام تا هر وقت خواستی
دستی دوباره تولیدش کنی.

### فعال‌سازی هوش مصنوعی واقعی (Gemini)
۱. به [aistudio.google.com/apikey](https://aistudio.google.com/apikey) برو و یک
   کلید رایگان Gemini بساز (کارت بانکی لازم نیست، این سرویس جدا از Firebase است).
۲. داخل اپ روی آیکون ⚙️ در بالای داشبورد بزن و کلید را وارد کن.
۳. اگر کلید را وارد نکنی، اپ همچنان کار می‌کند اما به‌جای متن تولیدشده توسط
   Gemini، یک تحلیل قالبی (بدون AI ولی مبتنی بر همان اعداد واقعی‌ات) نشان می‌دهد.

### سیستم هدف (Goal Hierarchy)
تب «اهداف» در نوار پایین اضافه شده. ساختار درختی دقیقا طبق بخش ۲ پرامپت:
هدف اصلی ← پروژه ← نقطه عطف ← هفته ← روز، و تسک در هر سطحی قابل افزودن است.
از منوی سه‌نقطه‌ی کنار هر گره می‌توانی زیرمجموعه/تسک اضافه کنی، ویرایش کنی، یا
با تایید، آن گره و تمام زیرمجموعه‌ها/تسک‌هایش را حذف کنی (حذف آبشاری).
تیک‌زدن یک تسک، XP آن را خودکار به `xpEarned` لاگ امروز اضافه می‌کند.

## نقشه راه فازهای بعدی

- **فاز ۳:** XP و Skill Tree (۴ دسته: دانش، بدن، ذهن، روابط) + منطق Level-up
  (سمت کلاینت، هم‌راستا با تصمیم Spark-only فاز ۲).
- **فاز ۴:** ردیاب عادت با Heat Map (`flutter_heatmap_calendar`) + هشدار شکست عادت.
- **فاز ۵:** ژورنال هوشمند با تحلیل Gemini 1.5 Flash.
- **فاز ۶:** سیستم ضد اهمال‌کاری، تحلیل خواب/ورزش، Reward & Punishment.
- **فاز ۷:** یکپارچه‌سازی نهایی، نوتیفیکیشن‌های هوشمند، گزارش ماهانه.

هر فاز را در یک پیام جدا از من بخواه تا کامل و تست‌شده تحویل بدهم؛ در یک پیام
واحد بیلد کردن کل ۱۵ بخش عملاً قابل کنترل کیفی نیست.

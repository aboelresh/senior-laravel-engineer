# الدرس الثامن: نظام الملفات في Linux

## مصطلحات مهم تعرفها قبل ما نبدأ

| المصطلح | المعنى |
|---------|--------|
| Root Directory | الجذر — نقطة البداية لكل شيء في Linux وتُرمز بـ / |
| Absolute Path | مسار يبدأ بـ / ويصف الموقع الكامل من الجذر |
| Relative Path | مسار يبدأ من الموقع الحالي ولا يبدأ بـ / |
| Symbolic Link | اختصار يشير إلى ملف أو مجلد في موقع آخر |
| File Descriptor | رقم يُعرّف ملفاً أو اتصالاً مفتوحاً داخل Process |
| Standard Output | مخرجات البرنامج العادية — File Descriptor رقم 1 |
| Standard Error | مخرجات الأخطاء — File Descriptor رقم 2 |
| Virtual Filesystem | نظام ملفات وهمي يُولّده الـ Kernel في الذاكرة |
| Mount | ربط جهاز تخزين بمجلد في هيكل Linux |
| Web Root | المجلد الذي يشير إليه Nginx أو Apache لخدمة الموقع |

---

## شرح المفاهيم

### أولاً: Everything is a File

في Linux كل شيء يُعامَل كملف. الملفات النصية والمجلدات والأجهزة والاتصالات الشبكية ومعلومات النظام — كلها تُقرأ وتُكتب عبر نفس الـ System Calls.

```
open() → read() → write() → close()

تقرأ ملفاً نصياً؟          نفس الأوامر.
تتحدث مع كارت الشبكة؟     نفس الأوامر.
تقرأ معلومات Process؟      نفس الأوامر.
```

هذا يوفر واجهة موحدة وبسيطة للتفاعل مع كل مكونات النظام.

### ثانياً: هيكل المجلدات

```
/
├── bin/      ← أوامر أساسية: ls, cp, mv, bash
├── etc/      ← إعدادات النظام والبرامج
├── var/      ← بيانات متغيرة: Logs, Cache
├── home/     ← ملفات المستخدمين
├── root/     ← Home Directory لمستخدم root
├── tmp/      ← ملفات مؤقتة تُمسح عند Restart
├── proc/     ← Virtual Filesystem لمعلومات النظام
├── dev/      ← الأجهزة
├── usr/      ← برامج المستخدمين
└── srv/      ← بيانات الخدمات
```

**`/etc` — إعدادات النظام:**

```
/etc/nginx/          ← إعدادات Nginx
/etc/php/            ← إعدادات PHP
/etc/mysql/          ← إعدادات MySQL
/etc/hosts           ← جدول DNS المحلي
/etc/crontab         ← جدولة المهام
```

**`/var` — البيانات المتغيرة:**

```
/var/log/nginx/      ← Logs الخاصة بـ Nginx
/var/log/php/        ← Logs الخاصة بـ PHP
/var/www/            ← ملفات المواقع
```

**`/proc` — نافذة على النظام:**

مجلد وهمي لا يوجد على الـ Disk. يُولّده الـ Kernel في الذاكرة ويعكس حالة النظام في الوقت الفعلي.

```
/proc/cpuinfo        ← معلومات المعالج
/proc/meminfo        ← معلومات الذاكرة
/proc/1234/          ← معلومات Process رقم 1234
/proc/1234/status    ← حالة الـ Process
/proc/1234/mem       ← استهلاك الذاكرة
/proc/1234/fd/       ← الملفات والاتصالات المفتوحة
/proc/1234/cmdline   ← الأمر الذي أنشأ الـ Process
```

**`/dev` — الأجهزة:**

```
/dev/sda             ← القرص الصلب الأول
/dev/sda1            ← القسم الأول منه
/dev/null            ← يبتلع أي بيانات تُرسل إليه
/dev/random          ← مولّد أرقام عشوائية
```

### ثالثاً: Absolute Path مقابل Relative Path

```
Absolute Path:
يبدأ دائماً بـ /
يصف الموقع الكامل بغض النظر عن موقعك الحالي.
مثال: /var/www/myapp/public/index.php

Relative Path:
يبدأ من موقعك الحالي.
لا يبدأ بـ /
مثال: إذا كنت في /var/www/myapp
       فـ public/index.php يعني
       /var/www/myapp/public/index.php
```

```
رموز مهمة:

/   ← Root Directory
~   ← Home Directory للمستخدم الحالي
.   ← المجلد الحالي
..  ← المجلد الأعلى

أمثلة:
cd ..        ← اذهب للمجلد الأعلى
cd ../..     ← اذهب لمجلدين للأعلى
cd ~/projects ← اذهب لمجلد projects في Home
```

### رابعاً: أنواع الملفات

```
يظهر نوع الملف في أول حرف عند كتابة ls -la:

-  ← Regular File    (ملف عادي)
d  ← Directory       (مجلد)
l  ← Symbolic Link   (اختصار)
b  ← Block Device    (جهاز مثل القرص الصلب)
c  ← Character Device (جهاز مثل الطرفية)
s  ← Socket          (اتصال شبكي)

مثال:
drwxr-xr-x  ← d = Directory
-rw-r--r--  ← - = Regular File
lrwxrwxrwx  ← l = Symbolic Link
```

### خامساً: Symbolic Links

```
إنشاء Symbolic Link:
ln -s /var/www/myapp/public /home/ubuntu/public

الآن /home/ubuntu/public يشير إلى
/var/www/myapp/public

في Laravel:
php artisan storage:link

ينشئ Symbolic Link من:
public/storage
إلى:
storage/app/public

يتيح الوصول للملفات المرفوعة عبر URL عام.
```

### سادساً: `/dev/null` وإخفاء المخرجات

```
/dev/null يبتلع أي بيانات تُرسل إليه.

command > /dev/null 2>&1

>        ← أعد توجيه Standard Output
/dev/null ← إلى /dev/null (تجاهله)
2        ← Standard Error
>&1      ← أرسله لنفس مكان Standard Output

النتيجة: كل المخرجات والأخطاء تختفي.

استخدام شائع في Cron Jobs:
0 * * * * php /var/www/myapp/artisan schedule:run > /dev/null 2>&1
```

### سابعاً: Laravel على السيرفر

```
ملفات التطبيق:
/var/www/myapp/

Nginx Root (يشير للـ public فقط):
/var/www/myapp/public/

إعدادات Nginx:
/etc/nginx/sites-available/myapp.conf

إعدادات PHP:
/etc/php/8.2/fpm/php.ini

Logs التطبيق:
/var/www/myapp/storage/logs/laravel.log

Logs النظام:
/var/log/nginx/error.log
/var/log/php8.2-fpm.log

ملفات مؤقتة للرفع:
/tmp/
```

Nginx Root يجب أن يشير لـ `/var/www/myapp/public` وليس للمجلد الرئيسي. لو أشار للمجلد الرئيسي أصبح ملف `.env` متاحاً للعالم — ثغرة أمنية خطيرة.

---

## الأسئلة والإجابات

**السؤال الأول:** ما وظيفة كل من `/etc` و `/var/log` و `/tmp` و `/proc`؟

`/etc` يحتوي إعدادات كل البرامج والنظام. `/var/log` يحتوي ملفات الـ Logs لكل الخدمات. `/tmp` يحتوي ملفات مؤقتة تُمسح تلقائياً عند Restart. `/proc` مجلد وهمي يُولّده الـ Kernel في الذاكرة يعكس معلومات النظام والـ Processes في الوقت الفعلي.

---

**السؤال الثاني:** ما الفرق بين Absolute Path و Relative Path مع أمثلة؟

Absolute Path يبدأ بـ `/` ويصف الموقع الكامل من الجذر بغض النظر عن موقعك الحالي، مثل `/var/www/myapp/public/index.php`. Relative Path يبدأ من موقعك الحالي ولا يبدأ بـ `/`، فإذا كنت في `/var/www/myapp` فإن `public/index.php` يعني نفس الملف السابق.

---

**السؤال الثالث:** أين تضع ملفات تطبيق Laravel وأين يشير Nginx Root؟

ملفات Laravel توضع في `/var/www/myapp/`. Nginx Root يجب أن يشير إلى `/var/www/myapp/public/` تحديداً وليس للمجلد الرئيسي، لأن الإشارة للمجلد الرئيسي تُعرّض ملف `.env` وملفات حساسة أخرى للوصول العام.

---

**السؤال الرابع:** ما هو Symbolic Link وما الذي يفعله `php artisan storage:link`؟

Symbolic Link اختصار يشير إلى ملف أو مجلد في موقع آخر، مثل Shortcut في Windows لكن أقوى. `php artisan storage:link` ينشئ Symbolic Link من `public/storage` إلى `storage/app/public` مما يُتيح الوصول للملفات المرفوعة عبر URL عام بينما تبقى مخزنة في مجلد محمي.

---

**السؤال الخامس:** ما هو `/dev/null` ولماذا يستخدم المهندسون `command > /dev/null 2>&1`؟

`/dev/null` جهاز وهمي يبتلع أي بيانات تُرسل إليه. `command > /dev/null 2>&1` تعني إعادة توجيه Standard Output لـ `/dev/null` ثم إرسال Standard Error لنفس المكان. النتيجة إخفاء كل مخرجات الأمر وأخطائه. يُستخدم كثيراً في Cron Jobs لمنع إرسال رسائل بريد إلكتروني في كل تشغيل.

---

**السؤال السادس:** تشخيص فشل رفع الصور في Laravel.

```
الخطوات بالترتيب:

1. Laravel Logs:
   /var/www/myapp/storage/logs/laravel.log

2. Nginx Logs:
   /var/log/nginx/error.log

3. فحص /tmp:
   df -h /tmp
   du -sh /tmp/*

4. فحص صلاحيات Storage:
   ls -la /var/www/myapp/storage/

5. تصحيح الصلاحيات إذا لزم:
   chown -R www-data:www-data /var/www/myapp/storage/
   chmod -R 775 /var/www/myapp/storage/
```

أكثر سبب شائع: المجلد مملوك لـ root لكن PHP-FPM يعمل بمستخدم www-data فيفشل في الكتابة ويظهر في Log كـ "Permission denied" أو "Unable to write to disk".

---

## ملاحظات مهمة

- Nginx Root يشير دائماً لـ `public/` في Laravel وليس للمجلد الرئيسي.
- صلاحيات مجلد `storage/` و `bootstrap/cache/` في Laravel يجب أن تكون قابلة للكتابة لمستخدم PHP-FPM.
- `/tmp` الممتلئ يوقف عمليات رفع الملفات وبعض عمليات قاعدة البيانات.
- مراقبة `/var/log/nginx/error.log` أول خطوة في تشخيص أي مشكلة في Production.

---

## ملخص

Linux يُعامل كل شيء كملف مما يوفر واجهة موحدة بسيطة. هيكل المجلدات منظم بوضوح: `/etc` للإعدادات و `/var/log` للسجلات و `/tmp` للمؤقت و `/proc` لمعلومات النظام الديناميكية. التمييز بين Absolute و Relative Path ضرورة يومية. معرفة أين توضع ملفات Laravel وأين يشير Nginx Root وأين تجد الـ Logs تُمكّن مهندس الـ Backend من الـ Deployment والتشخيص بكفاءة.
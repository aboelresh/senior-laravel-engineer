# الدرس العاشر: أوامر الملفات والمجلدات

## مصطلحات مهم تعرفها قبل ما نبدأ

| المصطلح | المعنى |
|---------|--------|
| mkdir — Make Directory | إنشاء مجلد جديد |
| touch | إنشاء ملف فارغ أو تحديث وقت التعديل |
| cp — Copy | نسخ ملف أو مجلد مع إبقاء الأصل |
| mv — Move | نقل أو إعادة تسمية ملف — الأصل يختفي من موقعه |
| rm — Remove | حذف نهائي بدون Recycle Bin |
| Pipe \| | تمرير مخرجات أمر كمدخلات للأمر التالي |
| Redirection > | كتابة مخرجات أمر لملف — يمسح المحتوى القديم |
| Redirection >> | إضافة مخرجات أمر لملف — لا يمسح المحتوى القديم |
| grep | البحث عن نص داخل ملف أو مخرجات أمر |
| Hard Link | مؤشر مباشر لنفس الـ inode في الـ Disk |
| Symbolic Link — Symlink | اختصار يشير لمسار ملف آخر |
| inode | وحدة بيانات في نظام الملفات تحتوي معلومات الملف |
| wc — Word Count | عد الأسطر أو الكلمات أو الأحرف |

---

## شرح المفاهيم

### أولاً: إنشاء الملفات والمجلدات

```bash
# إنشاء مجلد
mkdir myproject

# إنشاء مجلدات متداخلة دفعة واحدة
mkdir -p /var/www/myapp/storage/logs

# إنشاء ملف فارغ
touch file.txt
touch .env

# تحديث وقت التعديل لملف موجود
touch existing-file.php
```

**مثال عملي في Laravel:**

```bash
mkdir -p /var/www/myapp/storage/{logs,app/public,framework/cache}
touch /var/www/myapp/storage/logs/laravel.log
touch /var/www/myapp/.env
```

---

### ثانياً: النسخ والنقل

**cp — النسخ:**

```bash
# نسخ ملف
cp file.txt backup.txt

# نسخ مجلد كامل (-r = recursive)
cp -r folder/ folder_backup/

# نسخ مع الحفاظ على الصلاحيات والتواريخ
cp -rp /var/www/myapp /var/www/myapp_backup

# نسخ مع عرض ما يحدث
cp -rv /var/www/myapp /backup/
```

**mv — النقل وإعادة التسمية:**

```bash
# إعادة تسمية
mv old-name.php new-name.php

# نقل لمجلد آخر
mv file.php /var/www/myapp/app/

# نقل مع تسمية جديدة
mv /tmp/upload_123.jpg /var/www/myapp/storage/app/public/avatar.jpg
```

`mv` لا يُنشئ نسخة احتياطية — الأصل يختفي من موقعه.

---

### ثالثاً: الحذف بحذر — rm

```bash
# حذف ملف
rm file.txt

# حذف مجلد وكل محتوياته
rm -r folder/

# حذف قسري بدون أسئلة
rm -rf folder/
```

```
أخطر الأوامر في Linux:

rm -rf /         ← يحذف كل شيء على الجهاز
rm -rf /*        ← نفس الكارثة
rm -rf .         ← يحذف كل شيء في مجلدك الحالي
```

القاعدة الذهبية: راجع الأمر مرتين قبل الضغط Enter خاصة مع `-rf`.

البديل الأكثر أماناً في Production:
```bash
mv folder/ folder_backup_$(date +%Y%m%d)/
```

انقل للـ backup بدلاً من الحذف المباشر.

---

### رابعاً: Pipes والـ Redirection

**Pipe | — قوة Linux الحقيقية:**

مخرجات الأمر الأول تصبح مدخلات الثاني.

```bash
# اقرأ الـ log، صفّ الأخطاء فقط، اعرض آخر 20
cat laravel.log | grep "ERROR" | tail -20

# عدد PHP processes الشاغلة
ps aux | grep php | grep -v grep | wc -l

# أكثر 10 IPs طلباً
awk '{print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head -10
```

**Redirection:**

```bash
# كتابة — يستبدل المحتوى القديم
echo "APP_ENV=production" > .env

# إضافة — لا يستبدل
echo "APP_DEBUG=false" >> .env

# حفظ أخطاء migrate في ملف log
php artisan migrate 2>> /var/log/migrate.log

# إخفاء كل المخرجات — شائع في Cron Jobs
php artisan schedule:run > /dev/null 2>&1
```

---

### خامساً: البحث داخل الملفات — grep

```bash
# بحث أساسي
grep "ERROR" /var/www/myapp/storage/logs/laravel.log

# بدون حساسية للحروف
grep -i "error" laravel.log

# مع أرقام الأسطر
grep -n "Exception" laravel.log

# عرض 3 أسطر بعد كل نتيجة
grep -A 3 "ERROR" laravel.log

# عرض 3 أسطر قبل كل نتيجة
grep -B 3 "ERROR" laravel.log

# عدد الأخطاء اليوم
grep "$(date +%Y-%m-%d)" laravel.log | grep "ERROR" | wc -l

# البحث في كل الملفات بشكل recursive
grep -r "DB_PASSWORD" /var/www/myapp/
```

---

## التشبيه الحياتي

```
تخيل مكتبك:

cp  = مكنة تصوير — تصنع نسخة وتُبقي الأصل
mv  = تنقل الورقة — الأصل يختفي من مكانه
rm  = سلة مهملات بدون قاع — لا عودة أبداً
|   = ناقل بضائع بين الأقسام
>   = الكتابة على ورقة جديدة (يمسح القديم)
>>  = الإضافة في أسفل الورقة الموجودة
```

---

## أسئلة المقابلات

**س: ما الفرق بين > و >>؟**

`>` يكتب فوق المحتوى الموجود ويمسحه. `>>` يُضيف في نهاية الملف دون المساس بالمحتوى الموجود. في الـ Logs دائماً استخدم `>>` لتجنب مسح السجلات القديمة.

**س: لماذا rm -rf خطير ولا يوجد Recycle Bin في Linux؟**

Linux مصمم للسيرفرات حيث الأداء أولوية. الـ Recycle Bin تعني نسخ الملف أولاً قبل الحذف مما يستهلك موارد. `rm` يحذف مباشرة بدون وسيط. لهذا يستخدم المهندسون في Production `mv` للـ backup بدلاً من `rm` مباشرة.

---

## الأسئلة والإجابات

### المستوى الأول

**السؤال الأول:** ما الفرق بين cp و mv؟

`cp` تصنع نسخة وتُبقي الأصل في مكانه. `mv` تنقل الملف والأصل يختفي من موقعه.

```bash
cp .env .env.backup           # الأصل موجود + نسخة جديدة
mv old-name.php new-name.php  # الأصل اختفى، فقط الاسم الجديد موجود
```

---

**السؤال الثاني:** ما الفرق بين > و >>؟

`>` يكتب فوق المحتوى الموجود ويمسحه. `>>` يضيف في نهاية الملف بدون مسح.

```bash
echo "APP_ENV=production" > .env   # مسح كل اللي كان وكتب من جديد
echo "APP_DEBUG=false" >> .env     # أضاف سطر في الآخر
```

في الـ Logs دائماً `>>` عشان ما تمسحش السجلات القديمة.

---

**السؤال الثالث:** لماذا rm -rf خطير وما البديل في Production؟

لأن Linux مفيش فيه Recycle Bin — الحذف مباشر ولا رجعة. غلطة واحدة في المسار تمسح Production كامل.

البديل الأكثر أماناً:
```bash
mv /var/www/myapp/public/ /var/www/myapp/public_backup_$(date +%Y%m%d)/
```

تنقل للـ backup بدل الحذف المباشر.

---

### المستوى الثاني

**السؤال الرابع:** ابحث عن Exception في ملفات .log مع سطرين بعد كل نتيجة.

```bash
grep -r "Exception" /var/www/myapp/storage/logs/*.log -A 2
```

---

**السؤال الخامس:** PHP processes الشاغلة مع عددها.

```bash
# لعرضهم
ps aux | grep php | grep -v grep

# لعد عددهم
ps aux | grep php | grep -v grep | wc -l
```

`grep -v grep` يستبعد الـ grep process نفسها من النتائج.

---

### المستوى الثالث

**السؤال السادس:** ضغط المجلد بـ tar.gz مع تاريخ اليوم في الاسم.

```bash
tar -czf /backup/myapp_$(date +%Y%m%d).tar.gz /var/www/myapp
```

`-c` = create، `-z` = gzip compression، `-f` = اسم الملف الناتج.

---

**السؤال السابع:** الفرق بين Hard Link و Symbolic Link.

**Hard Link:** مؤشر مباشر لنفس الـ inode في الـ Disk. لو حذفت الأصل، الـ Hard Link لسه شغال لأن البيانات موجودة.

```bash
ln /var/www/myapp/storage/app/file.jpg /var/www/public/file.jpg
```

**Symbolic Link:** اختصار يشير لمسار. لو حذفت الأصل، الـ Symlink يتكسر.

```bash
ln -s /var/www/myapp/storage/app/public /var/www/myapp/public/storage
```

في Laravel بنستخدم Symlink في `php artisan storage:link` بالظبط.

---

### المستوى الرابع

**السؤال الثامن:** Backup Script يحتفظ بآخر 7 نسخ.

```bash
#!/bin/bash

BACKUP_DIR="/backup/myapp"
SOURCE="/var/www/myapp"
LOG="/var/log/backup.log"
DATE=$(date +%Y%m%d_%H%M%S)

mkdir -p $BACKUP_DIR

tar -czf $BACKUP_DIR/myapp_$DATE.tar.gz $SOURCE

echo "[$DATE] Backup created: myapp_$DATE.tar.gz" >> $LOG

ls -t $BACKUP_DIR/myapp_*.tar.gz | tail -n +8 | xargs rm -f

echo "[$DATE] Old backups cleaned. Kept last 7." >> $LOG
```

`ls -t` يرتب من الأحدث. `tail -n +8` يتجاهل أول 7 ويجيب الباقي. `xargs rm -f` يحذفهم.

---

## الاختبار السريع

**السؤال 1:** ما الذي يفعله هذا الأمر؟

```bash
cat /var/log/nginx/access.log | grep "500" | wc -l
```

**الإجابة:** ب — يعدّ عدد الطلبات التي أعطت خطأ 500.

اقرأ الـ log كاملاً → صفّ الأسطر التي تحتوي "500" → عدّ عددها.

---

**السؤال 2:** `>>` يمسح محتوى الملف ويكتب من البداية. صح أم خطأ؟

**الإجابة:** خطأ. `>>` يضيف في نهاية الملف. اللي يمسح هو `>`.

---

**السؤال 3:** مطور كتب بالخطأ `rm -rf /var/www/myapp/public/` بدلاً من `rm -rf /var/www/myapp/public/temp/`. ما الذي حدث؟

حذف مجلد `public` كامل — CSS وJS والصور والـ index.php كلها راحت نهائياً بدون أي إمكانية استرجاع.

للتجنب مستقبلاً:
```bash
# 1. تأكد من المسار قبل الحذف
ls /var/www/myapp/public/temp/

# 2. استخدم mv للـ backup أولاً
mv /var/www/myapp/public/temp/ /tmp/temp_backup/

# 3. في Production اعمل backup قبل أي حذف
```

---

**السؤال 4:** كيف تنسخ مجلد Laravel كاملاً مع الحفاظ على الصلاحيات الأصلية؟

```bash
cp -rp /var/www/myapp /var/www/myapp_backup
```

`-r` = recursive لنسخ المجلدات. `-p` = preserve للحفاظ على الصلاحيات والـ timestamps والـ ownership.

---

**السؤال 5:** ما ناتج هذا الأمر؟

```bash
echo "Hello" > file.txt
echo "World" > file.txt
cat file.txt
```

**الإجابة:**
```
World
```

لأن `>` الثانية مسحت "Hello" وكتبت "World" فوقيها.

---

## ملخص

```
إنشاء:
mkdir -p /path        ← إنشاء مجلدات متداخلة
touch file.txt        ← إنشاء ملف فارغ

نسخ ونقل:
cp -rp src/ dst/      ← نسخ مع الصلاحيات
mv old new            ← نقل أو إعادة تسمية

حذف:
rm -rf folder/        ← حذف نهائي — تأكد مرتين
mv folder/ backup/    ← البديل الآمن في Production

Pipes:
cmd1 | cmd2 | cmd3    ← سلسلة أوامر

Redirection:
>   ← كتابة (يمسح القديم)
>>  ← إضافة (يبقي القديم)

grep:
grep "ERROR" file         ← بحث أساسي
grep -r "text" /path/     ← بحث recursive
grep -A 3 "ERROR" file    ← 3 أسطر بعد النتيجة
grep -n "text" file       ← مع أرقام الأسطر
```

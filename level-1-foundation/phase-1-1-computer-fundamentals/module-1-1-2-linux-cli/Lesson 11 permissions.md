# الدرس الحادي عشر: الصلاحيات في Linux

## مصطلحات مهم تعرفها قبل ما نبدأ

| المصطلح | المعنى |
|---------|--------|
| Permission | صلاحية الوصول للملف أو المجلد |
| Owner | المالك — المستخدم الذي يملك الملف |
| Group | مجموعة المستخدمين الذين لهم نفس الصلاحية |
| Others | أي مستخدم آخر على النظام |
| chmod — Change Mode | تغيير صلاحيات الملف |
| chown — Change Owner | تغيير مالك الملف |
| r — Read | صلاحية القراءة = 4 |
| w — Write | صلاحية الكتابة = 2 |
| x — Execute | صلاحية التنفيذ = 1 |
| www-data | المستخدم الذي يعمل به Nginx وPHP-FPM |
| root | المدير الأعلى — له كل الصلاحيات |
| sudo | تنفيذ أمر بصلاحيات root |
| usermod | تعديل إعدادات مستخدم |
| inode | وحدة بيانات تحتوي معلومات الملف بما فيها الصلاحيات |

---

## لماذا الصلاحيات مهمة؟

أكثر سبب لفشل رفع الصور في Laravel:
```
failed to open stream: Permission denied
```

أكثر سبب لعدم عمل storage:link:
```
Permission denied
```

أكثر سبب لعدم كتابة الـ Logs:
```
Permission denied
```

كل هذه المشاكل لها سبب واحد: الملف مملوك لـ `root` لكن PHP-FPM يعمل بمستخدم `www-data` فيفشل في الكتابة. فهم الصلاحيات يحل هذه المشاكل في ثوانٍ.

---

## شرح المفاهيم

### أولاً: قراءة الصلاحيات

```bash
ls -la /var/www/myapp/storage/
```

المخرجات:
```
drwxr-xr-x  5  www-data  www-data  4096  Jan 15  logs/
-rw-r--r--  1  www-data  www-data  2048  Jan 15  laravel.log
lrwxrwxrwx  1  root      root        20  Jan 15  public -> storage/app/public
```

```
drwxr-xr-x   www-data  www-data
│││││││││    │         │
│││││││││    │         └── المجموعة (Group)
│││││││││    └──────────── المالك (Owner)
│││└──────── صلاحيات الآخرين (Others): r-x
│└────────── صلاحيات المجموعة (Group): r-x
└──────────── صلاحيات المالك (Owner): rwx
│
└─── النوع: d=Directory, -=File, l=Link
```

---

### ثانياً: معنى الحروف والأرقام

```
r = Read    (قراءة)   = 4
w = Write   (كتابة)  = 2
x = Execute (تنفيذ)  = 1
- = لا صلاحية        = 0

الحساب:
rwx = 4+2+1 = 7    ← كل الصلاحيات
rw- = 4+2+0 = 6    ← قراءة وكتابة
r-x = 4+0+1 = 5    ← قراءة وتنفيذ
r-- = 4+0+0 = 4    ← قراءة فقط
--- = 0+0+0 = 0    ← لا شيء
```

| الرقم | الصلاحيات | الاستخدام |
|-------|-----------|-----------|
| 777 | rwxrwxrwx | الكل يقرأ ويكتب — خطير في Production |
| 755 | rwxr-xr-x | المجلدات في Web Server |
| 644 | rw-r--r-- | الملفات العادية |
| 600 | rw------- | .env وملفات الأسرار |
| 775 | rwxrwxr-x | storage في Laravel |
| 664 | rw-rw-r-- | ملفات يعدلها أكثر من مستخدم |

---

### ثالثاً: chmod — تغيير الصلاحيات

```bash
# بالأرقام (الأوضح والأشيع)
chmod 755 myfolder/
chmod 644 config.php
chmod 600 .env

# مع المجلدات بشكل recursive
chmod -R 755 /var/www/myapp/
chmod -R 775 /var/www/myapp/storage/
chmod -R 644 /var/www/myapp/config/

# بالحروف
chmod u+x script.sh      # أضف Execute للمالك
chmod g-w file.txt       # أزِل Write من المجموعة
chmod o-r secret.txt     # أزِل Read من الآخرين
chmod a+r public.txt     # أضف Read للجميع
```

**الإعداد الصحيح لـ Laravel:**

```bash
sudo chmod -R 755 /var/www/myapp
sudo chmod -R 775 /var/www/myapp/storage
sudo chmod -R 775 /var/www/myapp/bootstrap/cache
sudo chmod 600 /var/www/myapp/.env
```

---

### رابعاً: chown — تغيير المالك

```bash
# تغيير المالك
chown ahmed file.txt

# تغيير المالك والمجموعة
chown ahmed:www-data file.txt

# تغيير recursive
chown -R www-data:www-data /var/www/myapp/storage/

# فحص المستخدم الحالي
whoami     # من أنا؟
id         # UID وGID وكل المجموعات
groups     # المجموعات التي أنتمي إليها
```

**المشكلة الأكثر شيوعاً في Laravel:**

```bash
# المشكلة: ملفات storage مملوكة لـ root
# PHP-FPM يعمل بـ www-data → "Permission denied"

# فحص المشكلة
ls -la /var/www/myapp/storage/

# الحل الكامل
sudo chown -R www-data:www-data /var/www/myapp/storage
sudo chown -R www-data:www-data /var/www/myapp/bootstrap/cache
sudo chmod -R 775 /var/www/myapp/storage
sudo chmod -R 775 /var/www/myapp/bootstrap/cache
```

---

### خامساً: Users وGroups

```bash
# عرض كل المستخدمين
cat /etc/passwd

# عرض كل المجموعات
cat /etc/group

# إضافة مستخدم لمجموعة
sudo usermod -aG www-data ubuntu
# يتيح لـ ubuntu التعامل مع ملفات www-data

# التحقق من التغيير
groups ubuntu
```

المستخدمون الشائعون في سيرفر Laravel:
```
root      ← المدير الأعلى (تجنب استخدامه)
ubuntu    ← مستخدم SSH العادي
www-data  ← مستخدم Nginx وPHP-FPM
```

---

### سادساً: قواعد أمان الصلاحيات

```bash
# لا تفعل:
chmod 777 /var/www/myapp/storage/    # أي مستخدم يكتب!
chmod 777 /var/www/myapp/.env        # أسرارك للعالم!
chown root:root /var/www/myapp/      # PHP-FPM لن يكتب

# افعل:
chmod 600 .env                       # المالك فقط
chmod 755 للمجلدات
chmod 644 للملفات العادية
chmod 775 لـ storage وbootstrap/cache
chown -R www-data:www-data storage/
```

اختبار أمني سريع:
```bash
curl https://yourapp.com/.env
# يجب أن يُعطي 403 Forbidden
# لو أعطى 200 OK → مشكلة أمنية خطيرة
```

---

## التشبيه الحياتي

```
تخيل شقة سكنية:

المالك (Owner):   صاحب الشقة — له كل الصلاحيات
المجموعة (Group): سكان البناية — صلاحيات محدودة
الآخرون (Others): الزوار — صلاحيات أقل

chmod 755 = صاحب الشقة يفعل كل شيء
            سكان البناية يدخلون ويتفرجون
            الزوار يدخلون ويتفرجون فقط

chmod 600 = صاحب الشقة فقط يفتح
            لا أحد غيره يدخل
            مثالي للـ .env

www-data  = عامل الصيانة
            يحتاج صلاحية الكتابة في storage
            لكن لا يحتاج أكثر من ذلك
```

---

## أسئلة المقابلات

**س: ما هي الصلاحيات المناسبة لملف .env في Production؟**

600 — المالك فقط يقرأ ويكتب، لا أحد غيره يصل إليه. مع `chown` للمستخدم المناسب.

**س: مطور يشتكي "Permission denied" عند رفع الصور في Laravel. ما خطوات التشخيص؟**

أولاً `ls -la /var/www/myapp/storage/app/public/` لمعرفة المالك الحالي. ثم `ps aux | grep php-fpm | head -1` لمعرفة مستخدم PHP-FPM. إذا اختلفا تُصلح بـ `chown -R www-data:www-data storage/` و`chmod -R 775 storage/`.

**س: ما الفرق بين chmod 775 و 777 ولماذا 777 خطير؟**

في 775 الآخرون يملكون `r-x` فقط — يقرأون وينفذون لكن لا يكتبون. في 777 الجميع بما فيهم أي مستخدم على السيرفر يستطيع الكتابة، مما يتيح رفع ملفات خبيثة.

---

## الأسئلة والإجابات

### المستوى الأول

**السؤال الأول:** ما الفرق بين chmod و chown؟

`chmod` يغير **نوع الصلاحية** — من يقرأ، من يكتب، من ينفذ. `chown` يغير **مالك الملف** — مين صاحبه.

```bash
chmod 644 file.php                    # غيّر الصلاحيات
chown www-data:www-data file.php      # غيّر المالك
```

---

**السؤال الثاني:** ماذا تعني 755؟

```
7 = rwx = 4+2+1 → المالك: قراءة وكتابة وتنفيذ
5 = r-x = 4+0+1 → المجموعة: قراءة وتنفيذ
5 = r-x = 4+0+1 → الآخرون: قراءة وتنفيذ
```

المالك يعمل كل حاجة، الباقيين يقرأون وينفذون فقط.

---

**السؤال الثالث:** لماذا 777 خطير في Production؟

لأن الـ Others — وده بيشمل أي مستخدم على السيرفر — يقدر يكتب في الملف. لو في ثغرة في التطبيق، المهاجم يقدر يرفع ملف خبيث في الـ storage. الصح إن الكتابة تبقى محصورة في `www-data` بس.

---

### المستوى الثاني

**السؤال الرابع:** الإعداد الكامل لـ Laravel على سيرفر.

```bash
sudo chown -R www-data:www-data /var/www/myapp
sudo chmod -R 755 /var/www/myapp
sudo chmod -R 775 /var/www/myapp/storage
sudo chmod -R 775 /var/www/myapp/bootstrap/cache
sudo chmod 600 /var/www/myapp/.env
```

---

**السؤال الخامس:** "Permission denied" عند كتابة الـ Logs. خطوات التشخيص والحل.

```bash
# خطوة 1: مين مالك storage؟
ls -la /var/www/myapp/storage/

# خطوة 2: بأي مستخدم PHP-FPM شغال؟
ps aux | grep php-fpm | head -1

# خطوة 3: الحل
sudo chown -R www-data:www-data /var/www/myapp/storage
sudo chmod -R 775 /var/www/myapp/storage
```

---

### المستوى الثالث

**السؤال السادس:** Script يضبط صلاحيات Laravel تلقائياً.

```bash
#!/bin/bash

APP_PATH="/var/www/myapp"
WEB_USER="www-data"

echo "Setting ownership..."
sudo chown -R $WEB_USER:$WEB_USER $APP_PATH

echo "Setting base permissions..."
sudo find $APP_PATH -type f -exec chmod 644 {} \;
sudo find $APP_PATH -type d -exec chmod 755 {} \;

echo "Setting storage permissions..."
sudo chmod -R 775 $APP_PATH/storage
sudo chmod -R 775 $APP_PATH/bootstrap/cache

echo "Securing .env..."
sudo chmod 600 $APP_PATH/.env

echo "Done."
```

`find -type f` للملفات فقط و`-type d` للمجلدات فقط — عشان ما نديش مجلدات نفس صلاحيات الملفات.

---

**السؤال السابع:** مستخدم ubuntu يحتاج يعدل في storage.

```bash
sudo usermod -aG www-data ubuntu
```

يضيف `ubuntu` لمجموعة `www-data` من غير ما يشيله من مجموعاته الأصلية. الـ `-aG` = append to Group.

محتاج logout وlogin عشان التغيير ياخد أثره:
```bash
groups ubuntu
```

---

### المستوى الرابع

**السؤال الثامن:** Audit Script للصلاحيات الخطرة.

```bash
#!/bin/bash

APP_PATH="/var/www/myapp"
LOG="/var/log/permissions_audit.log"
DATE=$(date +%Y-%m-%d_%H:%M:%S)
ISSUES=0

echo "[$DATE] Starting permissions audit..." >> $LOG

# فحص 777
FOUND_777=$(find $APP_PATH -perm 777)
if [ -n "$FOUND_777" ]; then
    echo "[WARNING] Found 777 permissions:" >> $LOG
    echo "$FOUND_777" >> $LOG
    ISSUES=$((ISSUES + 1))
fi

# فحص .env
ENV_PERM=$(stat -c "%a" $APP_PATH/.env 2>/dev/null)
if [ "$ENV_PERM" != "600" ]; then
    echo "[WARNING] .env permissions are $ENV_PERM — should be 600" >> $LOG
    ISSUES=$((ISSUES + 1))
fi

# فحص storage owner
STORAGE_OWNER=$(stat -c "%U" $APP_PATH/storage)
if [ "$STORAGE_OWNER" != "www-data" ]; then
    echo "[WARNING] storage owner is $STORAGE_OWNER — should be www-data" >> $LOG
    ISSUES=$((ISSUES + 1))
fi

echo "[$DATE] Audit complete. Issues found: $ISSUES" >> $LOG
echo "---" >> $LOG
```

---

## الاختبار السريع

**السؤال 1:** اقرأ الصلاحيات: `-rwxr-xr-x`

```
المالك:   rwx = 7
المجموعة: r-x = 5
الآخرون:  r-x = 5
→ 755
```

---

**السؤال 2:** `chown` يغير صلاحيات الملف. صح أم خطأ؟

خطأ. `chown` يغير المالك. اللي يغير الصلاحيات هو `chmod`.

---

**السؤال 3:** Laravel لا يكتب في storage/logs. خطوات التشخيص.

```bash
# معرفة المالك الحالي
ls -la /var/www/myapp/storage/logs/
```

لو المالك `root` والـ PHP-FPM شغال بـ `www-data` هذا هو سبب المشكلة.

```bash
sudo chown -R www-data:www-data /var/www/myapp/storage
sudo chmod -R 775 /var/www/myapp/storage
```

---

**السؤال 4:** الصلاحيات الصحيحة لـ .env

```bash
chmod 600 /var/www/myapp/.env
chown www-data:www-data /var/www/myapp/.env
```

---

**السؤال 5:** لماذا نستخدم `find` مع `chmod` بدلاً من `chmod -R` مباشرة؟

لأن `find` يتيح تطبيق صلاحيات مختلفة على الملفات والمجلدات بشكل منفصل. لو استخدمنا `chmod -R 644` على كل شيء المجلدات ستفقد صلاحية Execute ولن نتمكن من الدخول إليها. الملفات تحتاج `644` والمجلدات تحتاج `755`.

---

## ملخص

```
قراءة الصلاحيات:
ls -la /path/         ← عرض كل الصلاحيات والملاك

chmod:
chmod 755 folder/     ← المجلدات
chmod 644 file.php    ← الملفات العادية
chmod 600 .env        ← الملفات السرية
chmod -R 775 storage/ ← recursive

chown:
chown user:group file       ← ملف واحد
chown -R www-data:www-data  ← recursive

إعداد Laravel كامل:
chown -R www-data:www-data /var/www/myapp
find /var/www/myapp -type f -exec chmod 644 {} \;
find /var/www/myapp -type d -exec chmod 755 {} \;
chmod -R 775 /var/www/myapp/storage
chmod -R 775 /var/www/myapp/bootstrap/cache
chmod 600 /var/www/myapp/.env

قواعد أمنية:
777 → ممنوع في Production
600 → للـ .env دايماً
www-data → مالك storage دايماً
```
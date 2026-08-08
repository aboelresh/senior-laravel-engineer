# المشروع التطبيقي الأول: وثيقة معمارية تقنية

## هدف المشروع

إثبات الفهم العميق لكل مفاهيم الوحدة من خلال كتابة وثيقة تقنية متكاملة تشرح رحلة تنفيذ تطبيق PHP/Laravel من لحظة كتابة الكود حتى وصول الـ Response للمستخدم، مع توضيح دور كل طبقة من طبقات النظام.

---

## الوثيقة التقنية

### المقدمة

عندما يكتب المطور كود PHP فإنه لا يُنفَّذ مباشرة بواسطة المعالج، بل يمر بعدة طبقات تبدأ بتحليل الكود وتحويله إلى تعليمات وسيطة، ثم تنفيذها بواسطة محرك PHP، مع الاعتماد على نظام التشغيل لإدارة الموارد، وأخيراً إرسال النتيجة للمستخدم عبر الشبكة. تهدف هذه الوثيقة إلى شرح هذه الرحلة كاملة مع توضيح دور كل من الـ Hardware ونظام التشغيل والشبكة، وأماكن حدوث مشاكل الأداء.

---

### أولاً: رحلة الكود

```
Source Code
      │
      ▼
┌─────────────────┐
│ Lexical Analysis│  ← تقطيع الكود إلى Tokens
└────────┬────────┘
         │
         ▼
      Tokens
         │
         ▼
┌─────────────────┐
│     Parser      │  ← بناء الشجرة البنيوية
└────────┬────────┘
         │
         ▼
        AST
         │
         ▼
┌─────────────────┐
│  PHP Compiler   │  ← توليد التعليمات الوسيطة
└────────┬────────┘
         │
         ▼
      Opcodes
         │
    ┌────┴────┐
    │         │
OPcache    بدون OPcache
(من RAM)  (من الصفر)
    │         │
    └────┬────┘
         │
         ▼
   Zend Engine
         │
         ▼
   JIT (PHP 8)
  للكود الساخن
         │
         ▼
        CPU
```

**Source Code:** الكود الذي يكتبه المطور. المعالج لا يفهمه مباشرة.

**Lexical Analysis:** يقوم الـ Lexer بتقطيع الكود إلى وحدات صغيرة تُسمى Tokens. السطر `echo "Hello";` يتحول إلى T_ECHO و T_CONSTANT_ENCAPSED_STRING و T_SEMICOLON.

**AST:** يحوّل الـ Parser الـ Tokens إلى شجرة هرمية تعبّر عن الهيكل المنطقي للكود وتداخله. هذه الشجرة ضرورية لأن الـ Tokens قائمة مسطحة لا تعبّر عن من يحتوي من.

**Opcodes:** يُولّد الـ PHP Compiler تعليمات وسيطة يفهمها Zend Engine. ليست Machine Code ولا Source Code بل لغة داخلية وسيطة.

**OPcache:** يحفظ الـ Opcodes في RAM بعد توليدها أول مرة. في كل Request تالٍ تُقرأ الـ Opcodes من RAM مباشرة متجاوزةً مراحل Lexing و Parsing و Compilation. يُحسّن أداء Laravel من ~100ms إلى ~15ms لكل Request.

**JIT:** في PHP 8 يراقب الـ JIT Compiler الـ Opcodes أثناء التنفيذ ويترجم الأجزاء الأكثر استخداماً إلى Machine Code للتنفيذ المباشر.

---

### ثانياً: دور الـ Hardware

```
               Hardware

       CPU
        │
 RAM ───┼─── Cache
        │
      Storage
```

**المعالج:** ينفذ التعليمات القادمة من Zend Engine بدورة Fetch ثم Decode ثم Execute. كل Opcode تحتاج عدداً من CPU Cycles. كثرة الـ Cores تُتيح معالجة عدة Requests في وقت واحد فعلياً.

**RAM:** عند تشغيل PHP يُحمَّل الكود والمتغيرات والـ Stack والـ Heap في RAM. كل Process لها مساحة ذاكرة مستقلة. OPcache يحتجز جزءاً من RAM لحفظ الـ Opcodes.

**Storage:** ملفات Laravel — routes وconfig وvendor والـ Controllers والـ Views — موجودة على الـ Disk. قراءتها أبطأ بكثير من RAM وهنا تُبرز OPcache قيمتها بتخزين الـ Opcodes مباشرة في RAM.

**Cache:** المعالج يحتفظ بالبيانات الأكثر استخداماً في L1 و L2 و L3 Cache. وصول بـ 1 nanosecond مقارنة بـ 100 nanosecond للـ RAM.

---

### ثالثاً: دور نظام التشغيل

```
Application (PHP, Laravel, Nginx)
              │
         System Calls
              │
            Kernel
              │
           Hardware
```

**الـ Kernel:** الجزء الوحيد الذي يملك صلاحية الوصول المباشر للـ Hardware. يُدير Process Manager و Memory Manager و File System و Network Stack.

**User Space و Kernel Space:** PHP و Laravel يعملان في User Space بصلاحيات محدودة. للوصول للـ Hardware يجب المرور عبر System Calls. هذا الفصل يحمي النظام من تعطل برنامج واحد يُوقف الجهاز كله.

**إدارة الـ Processes:** كل Request في PHP-FPM يُعالَج بواسطة Worker Process مستقلة لها PID وذاكرة خاصة. عند انتهاء الـ Request تُمسح الذاكرة. هذا هو سبب أن PHP Stateless بطبيعتها.

**System Calls في العمل:**

```php
file_get_contents("app.log");

// ما يحدث:
// open()  → Kernel يفتح الملف
// read()  → Kernel يقرأ البيانات
// close() → Kernel يُغلق الملف
```

---

### رابعاً: دور الشبكة

```
المستخدم يضغط زر
         │
         ▼
   HTTP Request
         │
         ▼
Network Card تستقبل الـ Packets
         │
         ▼
Kernel يُجمّع الـ Packets
         │
         ▼
Nginx يستقبل الـ Request
         │
         ▼
Nginx يُحيل لـ PHP-FPM
         │
         ▼
Laravel ينفذ الكود
         │
         ▼
MySQL يُجيب على الاستعلام
         │
         ▼
Laravel يُنشئ الـ Response
         │
         ▼
Nginx يُعيد الـ Response
         │
         ▼
Kernel يُرسلها عبر الشبكة
         │
         ▼
المستخدم يرى النتيجة
```

---

### خامساً: نقاط الأداء

**OPcache:**

```
بدون OPcache — يتكرر في كل Request:
Read File → Lexer → Parser → Compiler → Opcodes → Execute

مع OPcache:
Opcodes من RAM → Execute
```

التحسن: 5-7 أضعاف في سرعة تحميل الـ Framework.

**RAM:**

عند امتلاء الـ RAM يبدأ نظام التشغيل باستخدام Swap على الـ SSD الأبطأ بألف مرة. المراقبة المستمرة لاستهلاك RAM و Swap ضرورة في Production.

**CPU Cores:**

عدد الـ PHP-FPM Workers يجب أن يتناسب مع عدد الـ CPU Cores. زيادته بشكل مفرط تُسبب Context Switching مرتفعاً يرفع استهلاك المعالج دون زيادة حقيقية في الأداء.

**Disk I/O:**

قراءة الملفات من الـ Disk أبطأ بكثير من RAM. OPcache يُقلل هذه القراءات بشكل جوهري.

**Database:**

أكبر سبب للبطء في تطبيقات Laravel هو الاستعلامات البطيئة و N+1 Problem وكثرة الاتصالات بقاعدة البيانات. تحسين هذه الجوانب يُحقق أثراً أكبر بكثير من أي تحسين في طبقة PHP نفسها.

---

### الصورة الكاملة

```
Developer يكتب الكود
         │
         ▼
Source Code → Lexer → Tokens → Parser → AST
         │
         ▼
PHP Compiler → Opcodes
         │
         ▼
OPcache (إن وجد) ← يحفظ في RAM
         │
         ▼
Zend Engine → JIT (PHP 8) للكود الساخن
         │
         ▼
System Calls → Kernel → Hardware
(CPU / RAM / Disk / Network)
         │
         ▼
Response → Nginx → Kernel → Network → Browser
```

---

### الخاتمة

رحلة تنفيذ كود PHP سلسلة مترابطة تبدأ بتحليل الكود وتحويله إلى Opcodes، ثم تنفيذها بواسطة Zend Engine، مع اعتماد كامل على نظام التشغيل لإدارة العمليات والذاكرة والملفات، وعلى الـ Hardware لتنفيذ التعليمات فعلياً، وعلى الشبكة لنقل الطلبات والاستجابات. فهم هذه الطبقات يُمكّن مهندس الـ Backend من تشخيص مشاكل الأداء واتخاذ قرارات صحيحة كتفعيل OPcache وضبط PHP-FPM وتحسين استخدام الموارد وكتابة تطبيقات أكثر كفاءة وقابلية للتوسع.
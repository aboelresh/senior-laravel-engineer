# الدرس السادس: نظام التشغيل — ما الذي يفعله فعلاً؟

## مصطلحات مهم تعرفها قبل ما نبدأ

| المصطلح | المعنى |
|---------|--------|
| Operating System | نظام التشغيل — الطبقة الوسيطة بين البرامج والـ Hardware |
| Kernel | النواة — الجزء الأساسي من نظام التشغيل بصلاحيات كاملة |
| User Space | المنطقة التي تعمل فيها التطبيقات بصلاحيات محدودة |
| Kernel Space | المنطقة التي يعمل فيها الـ Kernel بصلاحيات كاملة |
| System Call | الطريقة الوحيدة لطلب خدمة من الـ Kernel |
| Process | برنامج قيد التشغيل له ذاكرة مستقلة ومعزولة |
| Thread | وحدة تنفيذ داخل Process تتشارك ذاكرتها |
| PID — Process ID | رقم فريد يُعرّف كل Process في النظام |
| Scheduler | المُجدول — يُقرر أي Process تستخدم المعالج ومتى |
| Time Quantum | الشريحة الزمنية الممنوحة لكل Process |
| Context Switching | تبديل المعالج بين الـ Processes مع حفظ واستعادة الحالة |
| Zombie Process | Process انتهت لكن الـ Parent لم يجمع حالتها بعد |
| IPC — Inter-Process Communication | آليات التواصل بين الـ Processes المنفصلة |
| File Descriptor | رقم يُعرّف ملفاً أو اتصالاً مفتوحاً داخل Process |
| Concurrency | التعامل مع عدة مهام في نفس الفترة الزمنية |
| Parallelism | تنفيذ عدة مهام في نفس اللحظة على أكثر من Core |
| OOM Killer | آلية Linux تُغلق Processes قسراً عند نفاد الذاكرة |

---

## شرح المفاهيم

### أولاً: لماذا يوجد نظام التشغيل؟

بدون نظام تشغيل يجب على كل مطور كتابة كود للتعامل مع كل جهاز Hardware بشكل مباشر — إشارات الشاشة وأوامر القرص الصلب وبروتوكولات كارت الشبكة. هذا مستحيل عملياً. نظام التشغيل يحل هذه المشكلة بتوفير واجهة موحدة تُخفي تعقيد الـ Hardware وتُدير الموارد المشتركة بين جميع البرامج.

```
بدون OS:
برنامجك ──────────────────────▶ Hardware مباشرة

مع OS:
برنامجك ──▶ System Calls ──▶ Kernel ──▶ Hardware
```

### ثانياً: الـ Kernel

الـ Kernel الجزء الأساسي من نظام التشغيل الذي يعمل بصلاحيات كاملة ويتحدث مع الـ Hardware مباشرة. يحتوي على:

```
Process Manager  ← إدارة البرامج قيد التشغيل
Memory Manager   ← توزيع الذاكرة بين البرامج
File System      ← تنظيم الملفات على Storage
Network Stack    ← إدارة الاتصالات الشبكية
Device Drivers   ← التواصل مع أجهزة Hardware المختلفة
Security Module  ← التحكم في الصلاحيات
```

### ثالثاً: User Space و Kernel Space

```
┌─────────────────────────────────────┐
│           User Space                │
│  PHP, Laravel, Nginx, Chrome        │
│  صلاحيات محدودة                    │
│  لا وصول مباشر للـ Hardware        │
└──────────────┬──────────────────────┘
               │ System Calls
┌──────────────▼──────────────────────┐
│           Kernel Space              │
│  صلاحيات كاملة                     │
│  وصول مباشر للـ Hardware           │
└──────────────┬──────────────────────┘
               │
┌──────────────▼──────────────────────┐
│            Hardware                 │
│       CPU   RAM   Disk   Network    │
└─────────────────────────────────────┘
```

هذا الفصل يحمي النظام. برنامج واحد معطوب لا يستطيع تعطيل الجهاز كله. برنامج خبيث لا يستطيع قراءة ذاكرة برامج أخرى.

### رابعاً: System Calls

الـ System Call الطريقة الوحيدة التي يستطيع بها برنامج في User Space طلب خدمة من الـ Kernel.

```
أمثلة على System Calls:

open()   ← فتح ملف
read()   ← قراءة بيانات
write()  ← كتابة بيانات
close()  ← إغلاق ملف
fork()   ← إنشاء Process جديدة
socket() ← إنشاء اتصال شبكي
exit()   ← إنهاء Process
```

كل System Call تتطلب تبديل Context من User Space إلى Kernel Space مع حفظ حالة المعالج الحالية وتغيير مستوى الصلاحيات. هذا يستغرق عشرات الـ nanoseconds وهو سبب أن تقليل عدد الـ System Calls في التطبيقات الحرجة يُحسّن الأداء.

```php
// هذا السطر الواحد يُولّد عدة System Calls:
$content = file_get_contents('/var/log/app.log');

// ما يحدث فعلاً:
// open()  → Kernel يفتح الملف
// read()  → Kernel يقرأ البيانات
// close() → Kernel يُغلق الملف
```

### خامساً: Process

الـ Process برنامج قيد التشغيل. الفرق بين البرنامج والـ Process أن البرنامج ملف ساكن على الـ Disk، أما الـ Process فهو البرنامج بعد تحميله في RAM وبدء تنفيذه.

```
كل Process تملك:

PID            ← رقم فريد
Memory Space:
  ├── Code     ← الـ Opcodes أو Machine Code
  ├── Stack    ← متغيرات محلية وFunction calls
  └── Heap     ← بيانات مخصصة ديناميكياً
File Descriptors ← الملفات والاتصالات المفتوحة
State          ← Running / Waiting / Sleeping
```

في PHP التقليدي: كل Request = Process منفصلة تبدأ من الصفر وتنتهي بانتهاء الـ Response. هذا هو سبب أن PHP Stateless بطبيعتها.

### سادساً: Thread

الـ Thread وحدة تنفيذ أصغر داخل الـ Process. تتشارك Threads داخل نفس الـ Process نفس الـ Heap و Code و File Descriptors، لكن لكل Thread Stack مستقل.

```
Process واحدة
├── Thread 1 (Stack خاص)
├── Thread 2 (Stack خاص)
└── Thread 3 (Stack خاص)
    (الـ Heap مشترك بين الثلاثة)
```

| المعيار | Process | Thread |
|---------|---------|--------|
| الذاكرة | معزولة تماماً | مشتركة |
| إنشاء جديد | بطيء | سريع |
| التواصل | صعب عبر IPC | سهل |
| الأمان | عالٍ | منخفض |
| تأثير التعطل | لا يؤثر على الأخريات | قد يُعطل الـ Process كلها |

### سابعاً: Process Scheduling

```
Time Slicing:

الـ Scheduler يُعطي كل Process شريحة زمنية
────────────────────────────────────────────
[P1][P2][P3][P1][P2][P3][P1]...

كل Process تعتقد أنها تعمل باستمرار.
في الواقع تتناوب بسرعة هائلة.
```

حالات الـ Process:

```
New ──▶ Ready ──▶ Running
                    │
            ┌───────┴────────┐
         انتهت            تنتظر I/O
         دورتها               │
            │             Waiting
            ▼                 │
          Ready ◀─────────────┘
```

### ثامناً: Zombie Process

Zombie Process هي Process انتهت من التنفيذ لكن الـ Parent Process لم تستدعِ `wait()` لجمع حالتها. يحتفظ الـ Kernel بسجلها في جدول الـ Processes حتى يجمعها الـ Parent.

تراكم الـ Zombie Processes يستهلك أرقام الـ PID. في Linux الحد الأقصى للـ PIDs هو 32768 افتراضياً. امتلاؤها يمنع إنشاء Processes جديدة ويُوقف استقبال الـ Requests.

```bash
# الكشف عن Zombie Processes:
ps aux | grep defunct
```

### تاسعاً: Concurrency مقابل Parallelism

**Concurrency:** هيكل — الكود مصمم للتعامل مع عدة مهام في نفس الفترة الزمنية حتى لو نُفّذت واحدة في كل لحظة.

**Parallelism:** تنفيذ — عدة مهام تعمل في نفس اللحظة على أكثر من Core.

PHP-FPM مع Core واحد يحقق Concurrency فقط. مع عدة Cores يحقق Concurrency وParallelism معاً.

### عاشراً: رحلة Request في Laravel

```
HTTP Request من المستخدم
         ↓
Network Card تستقبل الـ Packets
         ↓
Kernel يُجمّع الـ Packets → HTTP Request
         ↓
Nginx يستقبل الـ Request (User Space)
         ↓
Nginx يُحيل لـ PHP-FPM عبر Unix Socket
(System Call)
         ↓
PHP-FPM Worker Process ينفذ Laravel
يقرأ ملفات:  read()   ← System Call
يتصل بـ DB:  socket() ← System Call
يكتب Log:   write()  ← System Call
         ↓
Response يعود عبر Nginx للمستخدم
```

---

## الأسئلة والإجابات

**السؤال الأول:** ما الفرق بين User Space و Kernel Space؟

User Space المنطقة التي تعمل فيها التطبيقات بصلاحيات محدودة لا تسمح بالوصول المباشر للـ Hardware. Kernel Space المنطقة التي يعمل فيها الـ Kernel بصلاحيات كاملة للتحدث مع Hardware مباشرة. التواصل بينهما يتم حصراً عبر System Calls. هذا الفصل يحمي النظام من تعطل برنامج واحد يؤثر على كل شيء.

---

**السؤال الثاني:** لماذا لكل Process PID فريد؟

حتى يستطيع نظام التشغيل تمييز الـ Processes وإدارتها. يستخدمه في إرسال Signals للـ Process مثل SIGTERM للإيقاف النظيف و SIGKILL للإيقاف الفوري، وفي تتبع استهلاك كل Process للموارد، وفي إنهاء Process معينة دون التأثير على الأخريات.

---

**السؤال الثالث:** ما الفرق بين Program و Process مع مثال؟

البرنامج ملف ساكن على الـ Disk كـ `/usr/bin/php`. الـ Process هو البرنامج بعد تحميله في RAM وبدء تنفيذه. عند تشغيل `php artisan serve` يتحول ملف php إلى Process لها PID وذاكرة مستقلة وحالة تنفيذ.

---

**السؤال الرابع:** لماذا تعطل Thread واحدة قد يُوقف الـ Process كلها بينما تعطل Process لا يؤثر على الأخريات؟

لأن جميع الـ Threads داخل نفس الـ Process تتشارك نفس الـ Heap. إذا أفسدت Thread الذاكرة المشتركة قد تتعطل الـ Process كلها. أما الـ Processes فلكل منها مساحة ذاكرة مستقلة معزولة تماماً يُديرها الـ Kernel، لذلك تعطل واحدة لا يصل أثره للأخريات.

---

**السؤال الخامس:** ما هو الـ System Call ولماذا له تكلفة أداء؟

الـ System Call الطريقة الوحيدة لطلب خدمة من الـ Kernel. له تكلفة لأنه يتطلب تبديل Context من User Space إلى Kernel Space مع حفظ حالة المعالج الحالية وتغيير مستوى الصلاحيات والتحقق من أمان الطلب. هذا يستغرق عشرات الـ nanoseconds. في تطبيق يُجري آلاف العمليات في الثانية يُصبح تقليل عدد الـ System Calls تحسيناً ملحوظاً.

---

**السؤال السادس:** تطبيق Laravel يخدم 500 Request في الثانية وكل Request تفتح اتصالاً جديداً بـ MySQL. ما المشكلة وما الحل؟

كل Request تُجري System Calls متعددة: `socket()` ثم `connect()` ثم `read()` ثم `write()` ثم `close()`. فتح وإغلاق 500 اتصال في الثانية يُضيف ضغطاً هائلاً على الـ OS ويستهلك موارد MySQL. الحل هو Database Connection Pooling بأدوات كـ ProxySQL للـ MySQL، إذ تُحفظ الاتصالات مفتوحة ومُصادقاً عليها وتُعاد استخدامها بدلاً من إنشاء اتصال جديد في كل Request، مما يختزل System Calls متعددة لـ `read()` و `write()` فقط.

---

**السؤال السابع:** ما هو Zombie Process وكيف يُسبب مشاكل على السيرفر؟

Zombie Process هي Process انتهت لكن الـ Parent لم تستدعِ `wait()` لجمع حالتها. يبقي الـ Kernel على سجلها في جدول الـ Processes. تراكمها يستهلك أرقام الـ PID حتى نفادها عند الحد الأقصى 32768 مما يمنع إنشاء أي Process جديدة ويُوقف استقبال الـ Requests تماماً.

---

**السؤال الثامن:** ما الفرق بين Concurrency و Parallelism وأيهما يُحقق PHP-FPM؟

Concurrency هيكل تصميمي يتيح التعامل مع عدة مهام في نفس الفترة الزمنية حتى لو كانت تتناوب على معالج واحد. Parallelism تنفيذ حقيقي لعدة مهام في نفس اللحظة على أكثر من Core. PHP-FPM يُحقق Concurrency دائماً عبر عدة Workers، ويُحقق Parallelism أيضاً إذا كان السيرفر يملك أكثر من Core إذ تعمل عدة Workers فعلياً في وقت واحد.

---

## ملاحظات الأداء

- ضبط `pm.max_children` في PHP-FPM بما يتناسب مع عدد الـ CPU Cores والـ RAM يمنع الـ Context Switching المفرط.
- مراقبة Zombie Processes بانتظام على السيرفرات طويلة التشغيل ضرورة وقائية.
- تقليل عدد الـ System Calls في المسارات الحرجة يُحسن الأداء في التطبيقات ذات الحمل العالي.
- Connection Pooling للـ Database يُقلل System Calls ويُحسن الأداء بمقدار 3-10 أضعاف في حالات الحمل العالي.

---

## ملخص

نظام التشغيل طبقة وسيطة أساسية تُخفي تعقيد الـ Hardware وتُدير الموارد المشتركة. الـ Kernel يعمل بصلاحيات كاملة في Kernel Space بينما تعمل التطبيقات في User Space بصلاحيات محدودة. التواصل بينهما يتم عبر System Calls ذات تكلفة زمنية حقيقية. الـ Processes معزولة لحماية النظام بينما الـ Threads تتشارك الذاكرة لأداء أعلى مع مخاطر أكبر. الـ Scheduler يوزع وقت المعالج بين الـ Processes بعدل. فهم هذه الطبقة يمكّن مهندس الـ Backend من تشخيص مشاكل الأداء المتعلقة بإدارة الـ Processes وضبط السيرفر واتخاذ قرارات صحيحة في البنية التحتية للتطبيق.
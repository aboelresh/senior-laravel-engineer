# الدرس الخامس: كيف تُنفَّذ البرامج؟

## مصطلحات مهم تعرفها قبل ما نبدأ

| المصطلح | المعنى |
|---------|--------|
| Compiler | برنامج يترجم الكود كاملاً إلى Machine Code قبل التشغيل |
| Interpreter | برنامج يفسر الكود سطراً بسطر أثناء التشغيل |
| JIT — Just-In-Time | ترجمة أجزاء الكود الأكثر استخداماً أثناء التشغيل |
| AOT — Ahead-of-Time | الترجمة الكاملة قبل التشغيل |
| Lexer | يقطّع الكود إلى وحدات صغيرة تُسمى Tokens |
| Token | وحدة بناء أولية في الكود كـ T_ECHO أو T_VARIABLE |
| Parser | يحوّل الـ Tokens إلى شجرة بنيوية |
| AST — Abstract Syntax Tree | شجرة تمثل الهيكل المنطقي للكود |
| Opcode | تعليمة وسيطة يفهمها Zend Engine |
| Zend Engine | المحرك الداخلي لـ PHP الذي ينفذ الـ Opcodes |
| OPcache | آلية تحفظ الـ Opcodes في RAM لتجنب إعادة تحليل الكود |
| Hot Code | أجزاء الكود التي تُنفذ بتكرار عالٍ |
| Bytecode | لغة وسيطة بين Source Code و Machine Code |
| Executable | ملف تنفيذي ناتج عن الـ Compilation |

---

## شرح المفاهيم

### أولاً: المشكلة الأساسية

المعالج لا يفهم PHP أو Python أو أي لغة برمجة. يفهم فقط Machine Code. توجد ثلاث طرق لسد هذه الفجوة.

```
ما يكتبه المطور:
if ($age > 18) { echo "مسموح"; }

ما يفهمه المعالج:
10110000 01001000 11000011 00000000 ...
```

### ثانياً: Compilation

```
Source Code
     ↓
  [Compiler]
  1. Lexical Analysis  → Tokens
  2. Parsing           → AST
  3. Semantic Analysis → التحقق من المنطق
  4. Optimization      → تحسين الكود
  5. Code Generation   → Machine Code
     ↓
Executable File
     ↓
CPU ينفذه مباشرة
```

**المزايا:**
- سرعة تنفيذ قصوى
- اكتشاف الأخطاء قبل التشغيل
- توزيع الملف التنفيذي بدون الكود المصدري

**العيوب:**
- يجب إعادة الـ Compilation عند كل تعديل
- الملف التنفيذي مرتبط بنظام تشغيل محدد

أمثلة: C, C++, Go, Rust

### ثالثاً: Interpretation

```
Source Code
     ↓
[Interpreter] ← يعمل في كل تشغيل
يقرأ السطر → يفسره → ينفذه → السطر التالي
     ↓
النتيجة مباشرة
```

**المزايا:**
- لا حاجة لخطوة Compilation
- مرونة عالية في التطوير
- يعمل على أي نظام به الـ Interpreter

**العيوب:**
- أبطأ من الـ Compiled Languages
- الأخطاء تظهر فقط عند تنفيذ السطر المعيب

أمثلة: Python, Ruby

### رابعاً: رحلة كود PHP

PHP لا يُترجم مباشرة للـ Machine Code ولا يُفسَّر بشكل بدائي، بل يمر بمراحل:

```
Source Code (.php)
        ↓
[Lexer] → Tokens
T_ECHO, T_CONSTANT_ENCAPSED_STRING, T_SEMICOLON...
        ↓
[Parser] → AST
      Echo
        │
     "Hello"
        ↓
[PHP Compiler] → Opcodes
ECHO "Hello"
        ↓
[OPcache] ← يحفظ الـ Opcodes في RAM
        ↓
[Zend Engine] ← ينفذ الـ Opcodes
        ↓
[JIT في PHP 8] ← يترجم Hot Code إلى Machine Code
        ↓
CPU
```

### خامساً: لماذا AST وليس مباشرة من Tokens للـ Opcodes؟

الـ Tokens قائمة مسطحة لا تعبّر عن التداخل في الكود. الـ AST شجرة هرمية تعبّر عن من يحتوي من.

```php
if ($a > $b) {
    for ($i = 0; $i < 10; $i++) {
        echo $i;
    }
}
```

هذا التداخل يحتاج شجرة لتمثيله بدقة قبل توليد Opcodes صحيحة.

### سادساً: OPcache

```
بدون OPcache — يتكرر في كل Request:
PHP File → Lexer → Parser → Compiler → Opcodes → Execute

مع OPcache — مرة واحدة فقط:
PHP File → Lexer → Parser → Compiler → Opcodes → RAM
                                                    ↓
كل Request تالٍ:                          Opcodes من RAM → Execute
```

التأثير على الأداء:

```
بدون OPcache: تحميل Laravel ~100ms
مع OPcache:   تحميل Laravel ~15ms
تحسن: 5-7 أضعاف
```

في Development يمكن تشغيل OPcache مع إعداد `opcache.validate_timestamps=1` للتحقق من تغيير الملفات في كل Request. في Production يُضبط على `opcache.validate_timestamps=0` لأقصى أداء.

### سابعاً: JIT في PHP 8

```
Profiler يراقب الـ Opcodes أثناء التنفيذ
         ↓
اكتشاف Hot Code (كود يُنفذ بتكرار عالٍ)
         ↓
JIT Compiler يترجمه → Machine Code
         ↓
يُخزن في RAM
         ↓
التنفيذ المباشر بدون Zend Engine في المرات التالية
```

JIT يُفيد في العمليات الحسابية المكثفة. في تطبيقات Web الاعتيادية يقضي التطبيق معظم وقته في انتظار Database و Network وليس في عمليات حسابية، لذلك التحسن في CRUD APIs قد لا يكون ملحوظاً.

### ثامناً: مقارنة الطرق الثلاث

| المعيار | Compilation | Interpretation | JIT |
|---------|-------------|----------------|-----|
| توقيت الترجمة | قبل التشغيل | أثناء التشغيل | أثناء التشغيل |
| السرعة | عالية جداً | متوسطة | جيدة |
| المرونة | منخفضة | عالية | عالية |
| اكتشاف الأخطاء | قبل التشغيل | وقت التشغيل | وقت التشغيل |
| أمثلة | C, Go, Rust | Python, Ruby | PHP 8, Java |

### تاسعاً: AOT مقابل JIT

**AOT:** الترجمة الكاملة قبل التشغيل. ينتج Machine Code أو ملفاً تنفيذياً. مثال: C, Go.

**JIT:** الترجمة أثناء التشغيل للأجزاء الأكثر استخداماً فقط. مثال: PHP 8 JIT, Java HotSpot.

### عاشراً: Laravel Octane وتأثيره

في PHP التقليدي كل Request:

```
تحميل Laravel → تحميل Classes → تنفيذ → مسح الذاكرة
```

في Laravel Octane مع Swoole:

```
تحميل Laravel مرة واحدة فقط في RAM
كل Request → تنفيذ → الذاكرة تبقى
```

هذا يُقلل زمن الاستجابة بشكل كبير لكنه يستلزم الانتباه لـ State Pollution، إذ إن البيانات المخزنة في Static Variables تبقى بين الـ Requests وقد تتسرب لمستخدمين آخرين.

---

## الأسئلة والإجابات

**السؤال الأول:** ما الفرق بين Compiler و Interpreter مع مثال من الحياة؟

الـ Compiler يشبه تنزيل فيلم مترجم كاملاً مسبقاً — الترجمة تحدث مرة واحدة وبعدها تشاهد الفيلم مباشرة في كل مرة. الـ Interpreter يشبه مترجماً فورياً يجلس بجانبك أثناء المشاهدة — في كل مرة تشاهد الفيلم يجب أن يُعيد الترجمة من البداية.

---

**السؤال الثاني:** ما هو الـ Opcode ولماذا هو مرحلة وسيطة في PHP؟

الـ Opcode تعليمة تنفيذية وسيطة يُولّدها PHP Compiler بعد تحليل الكود. ليس Machine Code يفهمه المعالج مباشرة، وليس Source Code يكتبه المطور، بل لغة داخلية يفهمها Zend Engine وينفذها. وجود هذه المرحلة الوسيطة يُتيح OPcache تخزين الـ Opcodes في RAM لتجنب إعادة التحليل في كل Request.

---

**السؤال الثالث:** لماذا يجب تفعيل OPcache في Production؟

في Production الكود ثابت لا يتغير، لذلك يُعيد PHP تحليله وتوليد Opcodes منه في كل Request دون حاجة فعلية. OPcache يحفظ الـ Opcodes في RAM ويتجاوز مراحل Lexing و Parsing و Compilation في الـ Requests التالية مما يُحسن الأداء بمقدار 5-7 أضعاف.

---

**السؤال الرابع:** ما الفرق بين AOT و JIT؟

AOT يترجم الكود كاملاً قبل التشغيل وينتج ملفاً تنفيذياً. الترجمة تحدث مرة واحدة والتنفيذ يكون بأقصى سرعة. JIT يترجم أثناء التشغيل لكنه ذكي يترجم فقط الأجزاء الأكثر استخداماً، مما يجمع بين مرونة الـ Interpretation وسرعة الـ Compilation للكود الحيوي.

---

**السؤال الخامس:** هل JIT في PHP 8 يُحسّن أداء تطبيقات Laravel بشكل ملحوظ؟

ليس بالضرورة. JIT يُفيد في العمليات الحسابية المكثفة. تطبيقات Web الاعتيادية تقضي معظم وقتها في انتظار Database و Network I/O وليس في عمليات حسابية تُنفذ بتكرار كافٍ ليُترجمها JIT. التحسن الملحوظ يكون في Scientific Computing و Image Processing وليس في CRUD APIs البسيطة.

---

**السؤال السادس:** لماذا ملف .exe لا يعمل على Linux؟

لأن الـ .exe يحتوي Machine Code مُولَّداً خصيصاً لمعمارية Windows ويستخدم تنسيق PE للملفات التنفيذية. Linux يستخدم تنسيقاً مختلفاً هو ELF ويستدعي System Calls مختلفة. الكود المصدري نفسه يمكن إعادة ترجمته على Linux لأن الـ Compiler يُولّد Machine Code مناسباً للنظام المستهدف.

---

**السؤال السابع:** ما الخطوة الأولى عند اكتشاف أن تطبيق Laravel بطيء بينما CPU 20% و RAM 40% و OPcache غير مفعّل؟

تفعيل OPcache فوراً. بما أن CPU و RAM غير مستهلكَين بشكل كبير فالمشكلة ليست في قدرة الموارد بل في إعادة تحليل وتوليد Opcodes في كل Request دون داعٍ. تفعيل OPcache قد يُحل المشكلة فوراً بدون أي تعديل في الكود أو البنية التحتية.

---

## ملاحظات الأداء

- OPcache ضروري في كل بيئة Production بدون استثناء.
- في Development يُضبط `opcache.revalidate_freq=0` مع `opcache.validate_timestamps=1` لرؤية التغييرات فوراً.
- Laravel Octane يُحسن الأداء بشكل كبير لكنه يستلزم كوداً واعياً بطبيعة الـ Long-running Processes.
- اختناق تطبيقات Web في الغالب في Database و Network وليس في تنفيذ كود PHP نفسه.

---

## ملخص

رحلة كود PHP من النص إلى التنفيذ تمر بمراحل: Source Code ثم Tokens ثم AST ثم Opcodes ثم Zend Engine ثم المعالج. OPcache يقطع هذه السلسلة من مرحلة Opcodes بحفظها في RAM وتوفير الخطوات السابقة في كل Request. JIT في PHP 8 يُضيف طبقة ترجمة إضافية للكود الحيوي المتكرر. فهم هذه المراحل يمكّن المهندس من تشخيص مشاكل الأداء واتخاذ قرارات صحيحة كتفعيل OPcache أو اعتماد Octane أو فهم متى يُفيد JIT ومتى لا يُفيد.
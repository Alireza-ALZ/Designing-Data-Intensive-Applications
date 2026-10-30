# Chapter 10 — Batch Processing

## Summary

در این chapter با batch processing آشنا شدیم. از ابزارهای Unix مانند `awk`، `grep` و `sort` شروع کردیم و دیدیم فلسفهٔ طراحی آن‌ها چگونه در MapReduce و dataflow engineهای جدیدتر ادامه پیدا می‌کند. از اصول مشترک این ابزارها می‌توان به immutable بودن input، طراحی output با این فرض که ممکن است ورودی برنامه‌ای دیگر ــ که هنوز معلوم نیست ــ شود، و حل مسئله‌های پیچیده با ترکیب ابزارهای کوچک اشاره کرد؛ ابزارهایی که هرکدام یک کار را خوب انجام می‌دهند.

در دنیای Unix، interface یکنواخت برای ترکیب برنامه‌ها file و pipe است؛ در MapReduce این interface را distributed filesystem فراهم می‌کند. dataflow engineها سازوکارهای انتقال data شبیه pipe اضافه می‌کنند تا از materialize کردن intermediate state در distributed filesystem پرهیز شود، اما input اولیه و output نهایی job معمولاً همچنان در HDFS قرار دارند.

دو مسئلهٔ اصلی که distributed batch processing frameworkها باید حل کنند عبارت‌اند از:

**Partitioning**

در MapReduce، mapperها بر اساس blockهای file ورودی partition می‌شوند. سپس output mapperها دوباره partition، sort و merge می‌شود تا تعداد قابل‌تنظیمی از reducer partitionها ساخته شود. هدف این است که dataهای مرتبط ــ مثلاً همهٔ recordهای دارای key یکسان ــ در یک محل کنار هم قرار بگیرند. dataflow engineهای پس از MapReduce می‌کوشند sorting را، مگر در موارد ضروری، حذف کنند؛ اما در بقیهٔ موارد رویکرد partitioning آن‌ها در کلیات مشابه است.

**Fault tolerance**

MapReduce مرتب روی disk می‌نویسد. این کار بازیابی از failure یک task را بدون restart کردن کل job آسان می‌کند، اما در اجرای بدون failure سرعت را کاهش می‌دهد. dataflow engineها intermediate state کمتری materialize می‌کنند و بخش بیشتری را در memory نگه می‌دارند؛ در نتیجه، هنگام از کار افتادن node ناچارند data بیشتری را دوباره محاسبه کنند. deterministic بودن operatorها حجم dataای را که باید دوباره محاسبه شود کاهش می‌دهد.

چند الگوریتم join برای MapReduce را بررسی کردیم که بیشترشان در MPP databaseها و dataflow engineها نیز به‌صورت داخلی استفاده می‌شوند. این الگوریتم‌ها نمونهٔ خوبی از نحوهٔ کار الگوریتم‌های partitionشده‌اند:

- **Sort-merge join:** هر input با mapperای پردازش می‌شود که join key را استخراج می‌کند. با partition، sort و merge، همهٔ recordهای دارای key یکسان به یک فراخوانی از reducer می‌رسند؛ reducer می‌تواند recordهای joinشده را تولید کند.
- **Broadcast hash join:** یکی از دو input کوچک است و partition نمی‌شود؛ می‌توان کل آن را در hash table بارگذاری کرد. برای هر partition از input بزرگ، mapperای اجرا می‌شود، hash table مربوط به input کوچک را بارگذاری می‌کند و سپس input بزرگ را recordبه‌record می‌خواند و برای هر record در hash table جست‌وجو می‌کند.
- **Partitioned hash join:** اگر هر دو input با روش یکسان partition شده باشند ــ یعنی از key، hash function و تعداد partitionهای یکسان استفاده کنند ــ می‌توان روش hash table را مستقلاً روی هر partition اجرا کرد.

distributed batch processing engineها عمداً programming model محدودی دارند: فرض می‌شود callback functionها، مانند mapper و reducer، stateless هستند و به‌جز output مشخص‌شده‌شان side effect قابل‌مشاهده‌ای ندارند. این محدودیت به framework اجازه می‌دهد بعضی از مسئله‌های دشوار distributed system را پشت abstraction خود پنهان کند: در صورت crash یا مشکل network، taskها را می‌توان با اطمینان retry کرد و output هر task شکست‌خورده را کنار گذاشت. اگر چند task برای یک partition موفق شوند، فقط output یکی از آن‌ها قابل‌مشاهده می‌شود.

به کمک framework، code یک batch processing job لازم نیست سازوکارهای fault tolerance را خودش پیاده‌سازی کند. framework می‌تواند تضمین کند output نهایی job همان باشد که در نبود fault به دست می‌آمد، هرچند در عمل شاید لازم شده باشد بعضی taskها چند بار اجرا شوند. این semantics قابل‌اتکا بسیار قوی‌تر از چیزی است که معمولاً در online serviceهایی داریم که request کاربر را پردازش می‌کنند و در جریان آن به‌عنوان side effect در database می‌نویسند.

ویژگی متمایز batch processing job این است که مقداری input data می‌خواند و output data تولید می‌کند، بی‌آنکه input را تغییر دهد؛ به بیان دیگر، output از input مشتق می‌شود. نکتهٔ مهم این است که input محدود و اندازه‌اش معلوم و ثابت است؛ برای مثال، مجموعه‌ای از log fileها در یک مقطع زمانی یا snapshotی از محتوای database. چون input محدود است، job می‌داند چه زمانی همهٔ آن را خوانده و سرانجام پس از پایان پردازش کامل می‌شود.

در chapter بعد به stream processing می‌پردازیم که input آن نامحدود است: job همچنان وجود دارد، اما streamهای data ورودی پایانی ندارند. در این حالت job هرگز کامل نمی‌شود، چون هر لحظه ممکن است کار بیشتری از راه برسد. stream و batch processing از بعضی جهت‌ها به هم شبیه‌اند، اما فرض نامحدود بودن streamها تغییرهای زیادی در شیوهٔ ساخت systemها ایجاد می‌کند.

## Key Terms

- `Bounded Input` — ورودی‌ای با اندازه و مرز مشخص که job می‌تواند پایان خواندن آن را تشخیص دهد.
- `Unbounded Input` — جریان ورودیِ ادامه‌داری که پایان ازپیش‌مشخصی ندارد و پردازش آن به jobای دائماً فعال نیاز دارد.
- `Stateless Operator` — operatorای که state پایدار میان فراخوانی‌ها نگه نمی‌دارد و side effect قابل‌مشاهده‌ای بیرون از output تعریف‌شده ایجاد نمی‌کند؛ این ویژگی retry امن را ممکن می‌سازد.
- `Partitioning` — تقسیم data میان taskها یا nodeها به‌گونه‌ای که recordهای مرتبط برای پردازش در یک محل جمع شوند.
- `Fault Tolerance` — توانایی framework برای بازیابی از failure task یا node و تولید output نهاییِ معادل اجرای بدون failure.

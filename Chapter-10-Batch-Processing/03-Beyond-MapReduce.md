# Chapter 10 — Batch Processing

## Beyond MapReduce

با اینکه MapReduce در اواخر دههٔ ۲۰۰۰ بسیار محبوب شد و توجه زیادی به خود جلب کرد، فقط یکی از چندین programming model ممکن برای distributed systemهاست. بسته به حجم و ساختار data و نوع پردازشی که روی آن انجام می‌شود، ابزارهای دیگری ممکن است برای بیان computation مناسب‌تر باشند.

بااین‌حال، زمان زیادی را در این chapter به MapReduce اختصاص دادیم، چون ابزار آموزشی مفیدی است: abstraction نسبتاً روشن و ساده‌ای روی یک distributed filesystem ارائه می‌کند. منظور از «ساده» این است که می‌توان فهمید سیستم چه‌کاری انجام می‌دهد، نه اینکه استفاده از آن آسان باشد. برعکس، پیاده‌سازی یک processing job پیچیده با raw APIهای MapReduce دشوار و پرزحمت است؛ مثلاً باید الگوریتم‌های join را از ابتدا پیاده‌سازی کنید.

برای پاسخ به دشواری استفادهٔ مستقیم از MapReduce، programming modelهای سطح‌بالاتری مانند Pig، Hive، Cascading و Crunch به‌عنوان abstractionهایی روی آن ساخته شدند. اگر سازوکار MapReduce را بدانید، یادگیری این ابزارها نسبتاً آسان است و constructهای سطح‌بالاترشان پیاده‌سازی بسیاری از taskهای رایج batch processing را به‌مراتب ساده‌تر می‌کند.

اما خود execution modelِ MapReduce هم محدودیت‌هایی دارد که با افزودن یک لایهٔ abstraction دیگر برطرف نمی‌شوند و در بعضی نوع‌های processing به performance ضعیف می‌انجامند. از یک سو، MapReduce بسیار robust است: می‌توان با آن حجم تقریباً دلخواهی از data را روی systemی multi-tenant و غیرقابل‌اتکا پردازش کرد، با وجود اینکه taskها مرتب terminate می‌شوند، و در نهایت job را به پایان رساند؛ هرچند آهسته. از سوی دیگر، برای بعضی نوع‌های processing ابزارهای دیگر گاهی چندین مرتبهٔ بزرگی سریع‌ترند.

در ادامهٔ این chapter، چند جایگزین برای batch processing را بررسی می‌کنیم. در Chapter 11 به stream processing می‌پردازیم که می‌توان آن را راه دیگری برای سریع‌ترکردن batch processing دانست.

### Materialization of Intermediate State

همان‌طور که پیش‌تر دیدیم، هر MapReduce job از jobهای دیگر مستقل است. نقطه‌های اصلی ارتباط یک job با بقیهٔ system، directoryهای input و output آن در distributed filesystem هستند. اگر بخواهید output یک job، input job دوم شود، باید directory ورودی job دوم را با directory خروجی job اول یکی تنظیم کنید و یک workflow scheduler بیرونی باید job دوم را فقط پس از اتمام job اول اجرا کند.

این سازوکار وقتی منطقی است که output job اول datasetی باشد که می‌خواهید به‌طور گسترده در سازمان منتشر کنید. در این صورت باید بتوان به آن با یک نام مشخص ارجاع داد و آن را به‌عنوان input چند job مختلف—از جمله jobهایی که teamهای دیگر توسعه داده‌اند—به کار برد. انتشار data در مکانی شناخته‌شده در distributed filesystem باعث loose coupling می‌شود؛ jobها لازم نیست بدانند چه کسی inputشان را تولید می‌کند یا چه کسی outputشان را مصرف خواهد کرد (رجوع کنید به «Separation of logic and wiring»).

اما در بسیاری از موارد می‌دانید که output یک job فقط input یک job دیگر خواهد شد که همان team نگهداری‌اش می‌کند. در این حالت، fileهای distributed filesystem صرفاً **intermediate state** هستند: وسیله‌ای برای انتقال data از یک job به job بعدی. در workflowهای پیچیده‌ای که برای ساخت recommendation systemها به کار می‌روند و ممکن است از ۵۰ یا ۱۰۰ MapReduce job تشکیل شده باشند، مقدار زیادی از این intermediate state وجود دارد.

نوشتن این intermediate state در fileها را **materialization** می‌نامند. قبلاً این اصطلاح را در بحث materialized viewها دیدیم («Aggregation: Data Cubes and Materialized Views»). materialization یعنی نتیجهٔ یک operation را از پیش محاسبه و ذخیره کنیم، به‌جای آنکه هنگام درخواست، همان موقع محاسبه‌اش کنیم.

در مقابل، مثال تحلیل log در ابتدای chapter از Unix pipeها استفاده می‌کرد تا output یک command را به input command بعدی وصل کند. pipeها intermediate state را به‌طور کامل materialize نمی‌کنند؛ output را تدریجی به input منتقل می‌کنند و فقط به buffer کوچکی در memory نیاز دارند.

روش MapReduce که intermediate state را به‌طور کامل materialize می‌کند، در مقایسه با Unix pipeها چند عیب دارد:

- اجرای MapReduce job فقط وقتی می‌تواند شروع شود که همهٔ taskهای jobهای قبلیِ تولیدکنندهٔ input آن تمام شده باشند. اما processهایی که با Unix pipe به هم وصل شده‌اند هم‌زمان شروع می‌شوند و هر output به‌محض تولیدشدن مصرف می‌شود. به‌دلیل skew یا تفاوت load روی machineها، معمولاً چند task باقی‌مانده (straggler) بسیار دیرتر از بقیه تمام می‌شوند. انتظار برای اتمام همهٔ taskهای job قبلی، کل workflow را کند می‌کند.
- Mapperها اغلب کار تکراری انجام می‌دهند: همان fileای را می‌خوانند که reducer لحظاتی پیش نوشته و آن را برای مرحلهٔ بعدی partitioning و sorting آماده می‌کنند. در بسیاری از موارد، می‌توان code مربوط به mapper را در reducer قبلی ادغام کرد. اگر output reducer به همان شکل output mapper partition و sort شده باشد، می‌توان reducerها را مستقیم به هم زنجیر کرد و دیگر لازم نیست stageهای mapper میانشان قرار بگیرد.
- نگهداری intermediate state در distributed filesystem یعنی این fileها روی چند node replicate می‌شوند؛ برای data موقتی، این کار اغلب بیش از نیاز است.

#### Dataflow engineها

برای رفع این مشکل‌های MapReduce، چند execution engine تازه برای distributed batch computation ساخته شدند. از شناخته‌شده‌ترین آن‌ها Spark، Tez و Flink هستند. طراحی‌هایشان با هم تفاوت‌هایی دارد، اما در یک ویژگی مشترک‌اند: کل workflow را یک job در نظر می‌گیرند، نه مجموعه‌ای از subjobهای مستقل.

این سیستم‌ها چون جریان data را در چندین processing stage به‌صراحت مدل می‌کنند، **dataflow engine** نامیده می‌شوند. مانند MapReduce، یک user-defined function را بارها فراخوانی می‌کنند تا هر بار یک record را روی یک thread پردازش کند. parallelism با partition کردن inputها حاصل می‌شود و output یک function از طریق network به input function بعدی منتقل می‌شود.

اما برخلاف MapReduce، functionها مجبور نیستند نقش‌های ثابت و یکی‌درمیانِ map و reduce داشته باشند؛ می‌توان آن‌ها را به شکل‌های انعطاف‌پذیرتری کنار هم گذاشت. به این functionها **operator** می‌گوییم. dataflow engine چند روش برای اتصال output یک operator به input operator دیگر فراهم می‌کند:

- می‌توان recordها را بر اساس key دوباره partition و sort کرد؛ مانند مرحلهٔ shuffle در MapReduce. این کار امکان اجرای sort-merge join و grouping را، مشابه MapReduce، فراهم می‌کند.
- می‌توان چند input را به یک شیوه partition کرد، اما sorting را کنار گذاشت. این کار برای partitioned hash join هزینه را کم می‌کند: partitioning رکوردها مهم است، اما ترتیبشان اهمیتی ندارد، چون ساخت hash table به‌هرحال ترتیب را تغییر می‌دهد.
- در broadcast hash join می‌توان output یک operator را به همهٔ partitionهای join operator فرستاد.

این سبک execution engine بر پایهٔ سیستم‌های پژوهشی‌ای مانند Dryad و Nephele است و نسبت به MapReduce چند مزیت دارد:

- کار پرهزینه‌ای مثل sorting فقط در جاهایی انجام می‌شود که واقعاً لازم است؛ لازم نیست به‌طور پیش‌فرض بین هر مرحلهٔ map و reduce تکرار شود.
- taskهای mapper غیرضروری حذف می‌شوند. چون mapper partitioning یک dataset را تغییر نمی‌دهد، اغلب می‌توان کارش را در reduce operator قبلی ادغام کرد.
- از آنجا که همهٔ joinها و dependencyهای data در workflow به‌صراحت تعریف شده‌اند، scheduler می‌داند هر data کجا لازم است و می‌تواند locality را بهینه کند. مثلاً می‌تواند task مصرف‌کنندهٔ data را روی همان machineای قرار دهد که task تولیدکنندهٔ آن اجرا می‌شود؛ در این صورت data از طریق buffer مشترک در memory منتقل می‌شود و لازم نیست روی network کپی شود.
- معمولاً کافی است intermediate state میان operatorها در memory یا disk محلی نگهداری شود. این کار I/O کمتری از نوشتن در HDFS لازم دارد؛ HDFS باید data را روی چند machine replicate کند و روی disk هر replica بنویسد. MapReduce همین بهینه‌سازی را برای output mapperها به کار می‌برد، اما dataflow engineها آن را به همهٔ intermediate stateها تعمیم می‌دهند.
- operatorها به‌محض آماده‌شدن inputشان می‌توانند اجرا را شروع کنند؛ لازم نیست منتظر بمانند کل stage قبلی تمام شود.
- می‌توان processهای موجود Java Virtual Machine (JVM) را برای اجرای operatorهای جدید دوباره به کار گرفت. این کار در مقایسه با MapReduce که برای هر task یک JVM تازه راه‌اندازی می‌کند، هزینهٔ startup را کاهش می‌دهد.

با dataflow engineها می‌توان همان computationهایی را پیاده‌سازی کرد که در MapReduce workflowها انجام می‌شوند؛ اما به‌دلیل بهینه‌سازی‌های بالا، معمولاً اجرای آن‌ها به‌مراتب سریع‌تر است. چون operatorها تعمیمی از map و reduce هستند، یک code پردازشی می‌تواند روی هر دو execution engine اجرا شود. مثلاً workflowهایی که با Pig، Hive یا Cascading نوشته شده‌اند را می‌توان تنها با تغییر configuration از MapReduce به Tez یا Spark منتقل کرد، بی‌آنکه code را تغییر داد.

Tez library نسبتاً سبکی است و برای انتقال واقعی data میان nodeها به YARN shuffle service تکیه می‌کند؛ در مقابل، Spark و Flink frameworkهای بزرگی‌اند که لایهٔ ارتباط network، scheduler و APIهای قابل‌استفاده برای کاربر را خودشان دارند. کمی بعد به این APIهای سطح‌بالا می‌پردازیم.

#### Fault tolerance

یکی از مزیت‌های materialize کردن کامل intermediate state در distributed filesystem این است که data دوام‌پذیر می‌شود. در نتیجه، پیاده‌سازی fault tolerance در MapReduce ساده است: اگر taskی شکست بخورد، می‌توان آن را روی machine دیگری از نو اجرا کرد و همان input را دوباره از filesystem خواند.

Spark، Flink و Tez از نوشتن intermediate state در HDFS پرهیز می‌کنند و برای تحمل faultها رویکرد دیگری دارند: اگر machineای از کار بیفتد و intermediate state موجود در memory آن از دست برود، state را از dataهای دیگری که هنوز در دسترس‌اند دوباره محاسبه می‌کنند؛ در صورت امکان از یک stage میانی قبلی، وگرنه از input اصلی که معمولاً در HDFS قرار دارد.

برای امکان‌پذیرکردن این محاسبهٔ مجدد، framework باید ثبت کند هر قطعهٔ data چگونه تولید شده است: از کدام input partitionها استفاده شده و چه operatorهایی روی آن‌ها اجرا شده‌اند. Spark برای ثبت ancestry یا زنجیرهٔ تولید data از abstractionی به نام resilient distributed dataset (RDD) استفاده می‌کند. Flink از operator state checkpoint می‌گیرد تا بتواند operatorای را که هنگام اجرا با fault روبه‌رو شده، از سر بگیرد.

هنگام محاسبهٔ مجدد data باید دانست computation deterministic است یا نه: آیا operatorها با دریافت همان input همیشه همان output را تولید می‌کنند؟ اگر بخشی از data ازدست‌رفته قبلاً به operatorهای downstream فرستاده شده باشد، این موضوع اهمیت پیدا می‌کند. اگر operator دوباره اجرا شود و data محاسبه‌شده با data ازدست‌رفتهٔ قبلی فرق داشته باشد، operatorهای پایین‌دست به‌سختی می‌توانند تناقض میان data قدیمی و جدید را حل کنند. برای operatorهای nondeterministic معمولاً راه‌حل این است که operatorهای downstream را هم متوقف کنیم و آن‌ها را با data جدید دوباره اجرا کنیم.

برای جلوگیری از چنین زنجیره‌ای از faultها، بهتر است operatorها deterministic باشند. البته واردشدن رفتار nondeterministic به برنامه ناخواسته آسان است: بسیاری از زبان‌های برنامه‌نویسی ترتیب مشخصی برای پیمایش elementهای hash table تضمین نمی‌کنند؛ بسیاری از الگوریتم‌های probabilistic و statistical صریحاً از عدد تصادفی استفاده می‌کنند؛ و هرگونه استفاده از system clock یا منبع data بیرونی نیز nondeterministic است. برای recovery قابل‌اتکا، باید علت‌های چنین nondeterminismای را حذف کرد؛ مثلاً با تولید عدد شبه‌تصادفی بر اساس seed ثابت.

محاسبهٔ مجدد data همیشه بهترین راه recovery نیست. اگر intermediate data در مقایسه با source data بسیار کوچک‌تر باشد، یا computation به CPU زیادی نیاز داشته باشد، احتمالاً materialize کردن intermediate data در file ارزان‌تر از محاسبهٔ دوبارهٔ آن است.

#### دربارهٔ materialization

با بازگشت به قیاس Unix، MapReduce شبیه این است که output هر command را در یک file موقت بنویسیم؛ dataflow engineها بیشتر به Unix pipe شباهت دارند. Flink به‌ویژه بر pipelined execution تکیه می‌کند: output یک operator به‌صورت تدریجی به operatorهای دیگر منتقل می‌شود و لازم نیست پیش از شروع پردازش، input کامل شده باشد.

عملیات sorting ناچار است کل input را مصرف کند و بعد output بدهد؛ چون ممکن است آخرین record ورودی کمترین key را داشته باشد و در نتیجه باید اولین record خروجی باشد. بنابراین هر operatorای که به sorting نیاز دارد باید دست‌کم موقتاً state جمع کند. اما بخش‌های زیادی از workflow را می‌توان به‌صورت pipelined اجرا کرد.

وقتی job تمام می‌شود، output باید در جایی durable قرار بگیرد تا userها بتوانند آن را پیدا و استفاده کنند؛ به‌احتمال زیاد، دوباره در distributed filesystem نوشته می‌شود. پس در dataflow engineها هم datasetهای materializeشده در HDFS معمولاً input و output نهایی job هستند. مانند MapReduce، inputها immutable می‌مانند و output به‌طور کامل جایگزین می‌شود. تفاوت و بهبود این است که دیگر لازم نیست همهٔ intermediate stateها را هم در filesystem بنویسید.

### Graphs and Iterative Processing

در «Graph-Like Data Models» دربارهٔ استفاده از graph برای مدل‌سازی data و به‌کارگیری graph query languageها برای پیمایش edgeها و vertexهای آن صحبت کردیم. بحث Chapter 2 بیشتر بر کاربردهای سبک OLTP متمرکز بود: اجرای سریع queryهایی که تعداد کمی vertex منطبق با معیارهای مشخص پیدا می‌کنند.

بررسی graphها در بستر batch processing هم جالب است؛ در اینجا هدف، اجرای نوعی پردازش یا تحلیل offline روی کل graph است. این نیاز اغلب در کاربردهای machine learning، مانند recommendation engineها، یا در systemهای ranking پیش می‌آید. برای نمونه، PageRank یکی از مشهورترین الگوریتم‌های تحلیل graph است. این الگوریتم می‌کوشد محبوبیت یک web page را بر اساس linkهایی که از pageهای دیگر به آن داده شده تخمین بزند و بخشی از فرمولی است که ترتیب نمایش نتیجه‌ها در search engineهای وب را تعیین می‌کند.

dataflow engineهایی مانند Spark، Flink و Tez معمولاً operatorهای یک job را به شکل directed acyclic graph (DAG) سازمان‌دهی می‌کنند. این با graph processing یکی نیست: در dataflow engineها، جریان data از یک operator به operator دیگر ساختاری graphمانند دارد، اما خود data معمولاً از tupleهایی شبیه داده‌های relational تشکیل شده است. در graph processing، خود data شکل graph دارد. این شباهت نام‌ها متأسفانه باعث ابهام می‌شود.

بسیاری از الگوریتم‌های graph با پیمایش edgeها یکی‌یکی بیان می‌شوند: یک vertex با vertex مجاورش join می‌شود تا اطلاعاتی را منتقل کند و این کار تا برآورده‌شدن شرطی ادامه پیدا می‌کند؛ مثلاً تا وقتی edge دیگری برای دنبال‌کردن باقی نمانده یا metric به مقدار همگرا نرسیده است. نمونه‌ای که در Figure 2-6 دیدیم، با دنبال‌کردن مکرر edgeهایی که نشان می‌دادند هر location درون کدام location دیگر قرار دارد، فهرست همهٔ مکان‌های آمریکای شمالی موجود در database را می‌ساخت. این نوع الگوریتم **transitive closure** نام دارد.

می‌توان graph را در distributed filesystem و در fileهایی شامل فهرست vertexها و edgeها ذخیره کرد، اما عبارت «آن‌قدر تکرار کن تا کار تمام شود» را نمی‌توان در MapReduce ساده بیان کرد، چون MapReduce فقط یک pass روی data انجام می‌دهد. بنابراین چنین الگوریتم‌هایی اغلب به‌صورت iterative پیاده‌سازی می‌شوند:

1. یک scheduler بیرونی batch processای اجرا می‌کند تا یک گام از الگوریتم را محاسبه کند.
2. پس از پایان process، scheduler بررسی می‌کند که آیا الگوریتم تمام شده است یا نه؛ این بررسی بر اساس شرط پایان انجام می‌شود، مثلاً اینکه edge دیگری برای پیمایش باقی نمانده یا تغییر نسبت به iteration قبلی از threshold مشخصی کمتر شده باشد.
3. اگر الگوریتم هنوز تمام نشده باشد، scheduler به گام اول برمی‌گردد و دور دیگری از batch process را اجرا می‌کند.

این روش کار می‌کند، اما پیاده‌سازی آن با MapReduce اغلب بسیار ناکارآمد است، چون MapReduce ماهیت iterative الگوریتم را در نظر نمی‌گیرد: در هر دور کل input dataset را می‌خواند و یک output dataset کاملاً جدید می‌سازد، حتی اگر نسبت به دور قبلی فقط بخش کوچکی از graph تغییر کرده باشد.

#### مدل پردازش Pregel

برای بهینه‌سازی batch processing روی graphها، مدل محاسباتی **bulk synchronous parallel (BSP)** رواج پیدا کرده است. از جمله پیاده‌سازی‌های آن Apache Giraph، API مربوط به GraphX در Spark و API مربوط به Gelly در Flink هستند. این مدل را Pregel model هم می‌نامند، چون مقالهٔ Pregel از Google این رویکرد را برای پردازش graphها رواج داد.

در MapReduce، mapperها به‌طور مفهومی به یک فراخوانی مشخص از reducer «پیام می‌فرستند»؛ چون framework همهٔ outputهای mapperها را که key یکسانی دارند کنار هم جمع می‌کند. ایدهٔ مشابهی در Pregel وجود دارد: یک vertex می‌تواند به vertex دیگری «پیام بفرستد» و این پیام‌ها معمولاً در امتداد edgeهای graph منتقل می‌شوند.

در هر iteration، functionای برای هر vertex فراخوانی می‌شود و همهٔ پیام‌هایی که برای آن فرستاده شده‌اند در اختیارش قرار می‌گیرند؛ این کار تا حدی شبیه فراخوانی reducer است. تفاوت MapReduce با Pregel این است که در Pregel هر vertex state خود را از یک iteration تا iteration بعد در memory نگه می‌دارد؛ بنابراین function فقط باید پیام‌های تازه‌رسیده را پردازش کند. اگر در بخشی از graph پیامی ارسال نشود، آن بخش به هیچ کاری نیاز ندارد.

این مدل کمی به actor model شباهت دارد (رجوع کنید به «Distributed actor frameworks»)، اگر هر vertex را یک actor در نظر بگیریم؛ با این تفاوت که state vertexها و پیام‌های میانشان fault-tolerant و durable هستند و ارتباط در roundهای ثابتی پیش می‌رود: در هر iteration، framework همهٔ پیام‌های iteration قبلی را تحویل می‌دهد. در actorها معمولاً چنین تضمینی دربارهٔ زمان‌بندی وجود ندارد.

#### Fault tolerance

اینکه vertexها فقط از راه message passing با هم ارتباط دارند، نه با query مستقیم یکدیگر، به بهبود performance jobهای Pregel کمک می‌کند؛ پیام‌ها را می‌توان batch کرد و انتظار برای ارتباط کاهش می‌یابد. تنها انتظار میان iterationهاست: چون Pregel تضمین می‌کند همهٔ پیام‌های ارسال‌شده در یک iteration در iteration بعد تحویل داده شوند، iteration قبلی باید کاملاً تمام شود و همهٔ پیام‌هایش روی network منتقل شوند تا iteration بعدی شروع شود.

با اینکه network زیربنایی ممکن است پیام‌ها را حذف کند، تکراری بفرستد یا با delay دلخواه تحویل دهد (رجوع کنید به «Unreliable Networks»)، پیاده‌سازی‌های Pregel تضمین می‌کنند هر پیام در iteration بعد، دقیقاً یک بار در vertex مقصد پردازش شود. مانند MapReduce، framework به‌طور شفاف از faultها recovery می‌کند تا مدل برنامه‌نویسی الگوریتم‌های مبتنی بر Pregel ساده بماند.

این fault tolerance با checkpoint گرفتن دوره‌ای از state همهٔ vertexها در پایان هر iteration حاصل می‌شود؛ یعنی state کامل آن‌ها در storage پایدار نوشته می‌شود. اگر nodeای از کار بیفتد و state موجود در memory آن از دست برود، ساده‌ترین راه این است که کل محاسبهٔ graph را به آخرین checkpoint برگردانیم و از آنجا دوباره شروع کنیم. اگر الگوریتم deterministic باشد و پیام‌ها log شده باشند، می‌توان فقط partition ازدست‌رفته را بازیابی کرد؛ مشابه روشی که برای dataflow engineها دیدیم.

#### اجرای موازی

یک vertex لازم نیست بداند روی کدام machine فیزیکی اجرا می‌شود؛ هنگام فرستادن پیام به vertexهای دیگر، فقط ID آن vertex را مشخص می‌کند. تقسیم graph میان machineها—یعنی تعیین اینکه هر vertex روی کدام machine اجرا شود—و مسیریابی پیام‌ها روی network تا رسیدن به مقصد، بر عهدهٔ framework است.

چون programming model در هر لحظه فقط با یک vertex سروکار دارد (رویکردی که گاهی «thinking like a vertex» نامیده می‌شود)، framework می‌تواند graph را به شکل‌های گوناگون partition کند. در حالت ایدئال، vertexهایی که زیاد با هم ارتباط دارند روی یک machine قرار می‌گیرند. اما پیدا کردن چنین partition بهینه‌ای دشوار است؛ در عمل، graph را اغلب بر اساس ID دلخواه vertexها partition می‌کنند، بدون آنکه vertexهای مرتبط را کنار هم قرار دهند.

در نتیجه، الگوریتم‌های graph معمولاً هزینهٔ زیادی بابت ارتباط میان machineها می‌پردازند و intermediate state—یعنی پیام‌های مبادله‌شده میان nodeها—اغلب از خود graph بزرگ‌تر است. سربار ارسال پیام از طریق network می‌تواند الگوریتم‌های distributed graph را به‌شدت کند کند.

به همین دلیل، اگر graph در memory یک computer جا شود، الگوریتمی که روی همان یک machine اجرا می‌شود—حتی شاید با یک thread—به‌احتمال زیاد از distributed batch process سریع‌تر خواهد بود. اگر graph از memory بزرگ‌تر باشد اما روی disk یک computer جا شود، پردازش تک‌ماشینی با frameworkی مانند GraphChi همچنان گزینهٔ مناسبی است. اگر graph در یک machine جا نشود، رویکرد distributed مانند Pregel اجتناب‌ناپذیر می‌شود؛ parallel کردن کارآمد الگوریتم‌های graph همچنان موضوع پژوهش است.

### High-Level APIs and Languages

از زمانی که MapReduce رواج پیدا کرد، execution engineهای distributed batch processing پخته‌تر شده‌اند. زیرساخت اکنون آن‌قدر robust شده که می‌تواند petabyteها data را روی clusterهایی با بیش از ۱۰٬۰۰۰ machine ذخیره و پردازش کند. حالا که به نظر می‌رسد مسئلهٔ operation فیزیکی batch processها در این مقیاس تا حد زیادی حل شده، توجه به حوزه‌های دیگری معطوف شده است: بهبود programming model، افزایش efficiency پردازش و گسترش دامنهٔ مسئله‌هایی که این فناوری‌ها می‌توانند حل کنند.

همان‌طور که پیش‌تر گفتیم، زبان‌ها و APIهای سطح‌بالایی مانند Hive، Pig، Cascading و Crunch رواج یافتند، چون نوشتن دستی MapReduce jobها پرزحمت است. با ظهور Tez، این زبان‌های سطح‌بالا مزیت دیگری هم پیدا کردند: می‌توانستند بدون بازنویسی code job به execution engine تازهٔ dataflow منتقل شوند. Spark و Flink نیز dataflow APIهای سطح‌بالای خود را دارند که اغلب از FlumeJava الهام گرفته‌اند.

این dataflow APIها معمولاً computation را با building blockهای relational بیان می‌کنند: join کردن datasetها بر اساس مقدار یک field، group کردن tupleها بر اساس key، filter کردن بر مبنای یک شرط و aggregate کردن tupleها با count، sum یا functionهای دیگر. این operationها درون سیستم با الگوریتم‌های مختلف join و grouping که پیش‌تر در این chapter دیدیم، پیاده‌سازی می‌شوند.

مزیت آشکار این interfaceها نیاز به code کمتر است؛ اما استفادهٔ تعاملی را هم ممکن می‌کنند: analysis code را کم‌کم در shell می‌نویسید و مرتب اجرا می‌کنید تا نتیجهٔ کارش را ببینید. این شیوه برای کاوش dataset و آزمودن روش‌های مختلف پردازش بسیار مفید است و به Unix philosophy هم شباهت دارد (رجوع کنید به «The Unix Philosophy»).

این interfaceهای سطح‌بالا نه‌فقط بهره‌وری انسان‌هایی را که با system کار می‌کنند بالا می‌برند، بلکه efficiency اجرای job را در سطح machine هم بهتر می‌کنند.

#### حرکت به‌سوی query languageهای declarative

مشخص‌کردن join با operatorهای relational، در مقایسه با نوشتن code اجرای join، این مزیت را دارد که framework می‌تواند ویژگی‌های inputهای join را تحلیل کند و خودکار تصمیم بگیرد کدام الگوریتم join برای آن task مناسب‌تر است. Hive، Spark و Flink query optimizerهای cost-based دارند که این کار را انجام می‌دهند و حتی می‌توانند ترتیب joinها را عوض کنند تا حجم intermediate state به حداقل برسد.

انتخاب الگوریتم join می‌تواند تأثیر زیادی بر performance یک batch job داشته باشد. خوب است مجبور نباشیم همهٔ الگوریتم‌های join این chapter را بشناسیم و به خاطر بسپاریم. وقتی joinها به‌شکل declarative بیان شوند، این کار ممکن می‌شود: application فقط می‌گوید چه joinهایی لازم دارد و query optimizer تصمیم می‌گیرد بهترین روش اجرای آن‌ها چیست. پیش‌تر این ایده را در «Query Languages for Data» دیدیم.

بااین‌حال، MapReduce و dataflow engineهای پس از آن از جهات دیگری با مدل query کاملاً declarative در SQL تفاوت دارند. MapReduce بر پایهٔ callback functionها ساخته شده است: برای هر record یا گروهی از recordها، یک user-defined function (mapper یا reducer) فراخوانی می‌شود و آن function برای تصمیم‌گیری دربارهٔ output آزاد است هر code دلخواهی را اجرا کند. مزیت این رویکرد آن است که می‌توان از ecosystem گستردهٔ libraryهای موجود برای کارهایی مانند parsing، تحلیل زبان طبیعی، تحلیل تصویر و اجرای الگوریتم‌های عددی یا آماری استفاده کرد.

این آزادی برای اجرای آسان code دلخواه، مدت‌هاست سیستم‌های batch processing با ریشهٔ MapReduce را از MPP databaseها متمایز کرده است (رجوع کنید به «Comparing Hadoop to Distributed Databases»). databaseها هم امکان نوشتن user-defined function دارند، اما این قابلیت اغلب استفادهٔ دشواری دارد و با package managerها و systemهای مدیریت dependency رایج در زبان‌های برنامه‌نویسی—مانند Maven برای Java، npm برای JavaScript و Rubygems برای Ruby—خوب یکپارچه نیست.

بااین‌حال، dataflow engineها دریافته‌اند که افزودن ویژگی‌های declarative در حوزه‌هایی فراتر از join هم مزیت دارد. مثلاً اگر callback function فقط یک شرط سادهٔ filtering داشته باشد یا صرفاً چند field را از record انتخاب کند، فراخوانی function برای تک‌تک recordها سربار CPU قابل‌توجهی دارد. اگر چنین operationهای سادهٔ filter و map به‌شکل declarative بیان شوند، query optimizer می‌تواند از layoutهای column-oriented storage استفاده کند و فقط columnهای لازم را از disk بخواند.

Hive، Spark DataFrames و Impala از vectorized execution هم استفاده می‌کنند: data را در loop داخلی فشرده‌ای پیمایش می‌کنند که با CPU cache سازگار است و از فراخوانی مکرر function پرهیز می‌کند. Spark، JVM bytecode تولید می‌کند و Impala برای این loopهای داخلی با LLVM کد native می‌سازد.

با افزودن جنبه‌های declarative به high-level APIها و استفاده از query optimizerهایی که هنگام اجرا از این ویژگی‌ها بهره می‌برند، batch processing frameworkها بیشتر به MPP databaseها شبیه می‌شوند و می‌توانند به performance قابل‌مقایسه برسند. درعین‌حال، چون قابلیت اجرای code دلخواه و خواندن data در formatهای متنوع را حفظ می‌کنند، مزیت انعطاف‌پذیری خود را هم از دست نمی‌دهند.

#### تخصص‌یافتن برای domainهای مختلف

قابلیت اجرای code دلخواه مفید است، اما بسیاری از الگوهای پردازش بارها تکرار می‌شوند؛ بنابراین ارزش دارد implementationهای قابل‌بازاستفاده‌ای از building blockهای رایج داشته باشیم. MPP databaseها به‌طور سنتی نیازهای تحلیل‌گران business intelligence و گزارش‌گیری کسب‌وکار را برآورده کرده‌اند، اما batch processing در domainهای بسیار دیگری هم استفاده می‌شود.

یکی از domainهای مهمی که اهمیتش رو به افزایش است، الگوریتم‌های آماری و عددی موردنیاز machine learning است؛ برای مثال، classification و recommendation systemها. implementationهای قابل‌بازاستفاده‌ای در حال شکل‌گیری‌اند: Mahout الگوریتم‌های مختلف machine learning را روی MapReduce، Spark و Flink پیاده‌سازی می‌کند؛ MADlib نیز قابلیت‌های مشابهی را درون یک relational MPP database یعنی Apache HAWQ فراهم می‌کند.

الگوریتم‌های مکانی، مانند k-nearest neighbors، هم مفیدند. این الگوریتم موارد نزدیک به یک مورد مشخص را در فضای چندبعدی جست‌وجو می‌کند؛ نوعی similarity search. جست‌وجوی تقریبی برای الگوریتم‌های تحلیل genome نیز مهم است، چون باید رشته‌هایی را پیدا کنند که مشابه‌اند اما دقیقاً یکسان نیستند.

از batch processing engineها برای اجرای distributed الگوریتم‌های حوزه‌های هرچه متنوع‌تری استفاده می‌شود. با اضافه‌شدن قابلیت‌های داخلی و operatorهای declarative سطح‌بالا به batch processing systemها، و درعین‌حال برنامه‌پذیرتر و انعطاف‌پذیرترشدن MPP databaseها، این دو دسته بیش‌ازپیش به هم شبیه می‌شوند: در نهایت، هر دو systemهایی برای ذخیره و پردازش data هستند.

## Key Terms

- `Materialization` — محاسبه و ذخیرهٔ پیشاپیش نتیجهٔ یک operation، به‌جای محاسبهٔ آن هنگام درخواست.
- `Dataflow Engine` — execution engineای که کل workflow را به شکل جریان data میان operatorها مدل و اجرا می‌کند.
- `Pipelined Execution` — انتقال تدریجی output یک operator به operator بعدی، بدون انتظار برای کامل‌شدن کل input یا stage قبلی.
- `Resilient Distributed Dataset (RDD)` — abstraction در Spark برای نمایش data توزیع‌شده و ثبت زنجیرهٔ operationهایی که آن را ساخته‌اند تا پس از failure قابل‌بازسازی باشد.
- `Bulk Synchronous Parallel (BSP)` — مدل محاسبات موازیِ مرحله‌ای که در آن computation در roundهای همگام اجرا می‌شود و ارتباط هر round پیش از شروع round بعدی تکمیل می‌شود.
- `Pregel Model` — مدل graph processing مبتنی بر BSP که در هر iteration برای هر vertex function اجرا می‌کند و پیام‌های vertexها را در roundهای متوالی تحویل می‌دهد.
- `Iterative Graph Processing` — اجرای تکراری computation روی vertexها و edgeهای graph تا رسیدن به شرط پایان یا همگرایی.
- `Deterministic Computation` — computationای که با input یکسان، در اجرای مجدد همان output را تولید می‌کند و recovery را قابل‌پیش‌بینی‌تر می‌سازد.
- `Vectorized Execution` — اجرای operationها روی batchهایی از data در loopهای فشرده، برای کاهش function callها و استفادهٔ بهتر از CPU cache.
- `Directed Acyclic Graph (DAG)` — graph جهت‌داری که cycle ندارد و می‌تواند وابستگی و ترتیب اجرای stageهای یک workflow را نمایش دهد.

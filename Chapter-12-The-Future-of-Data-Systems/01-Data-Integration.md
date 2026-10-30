# Chapter 12 — The Future of Data Systems

## Data Integration

یکی از themeهای تکرارشوندهٔ این کتاب این بود که برای هر مسئله چند راه‌حل وجود دارد و هرکدام مزایا، معایب و trade-offهای خود را دارند. برای مثال، در بحث storage engineها در Chapter 3 با log-structured storage، B-tree و column-oriented storage آشنا شدیم؛ و در بحث replication در Chapter 5 رویکردهای single-leader، multi-leader و leaderless را دیدیم.

برای مسئله‌ای مثل «می‌خواهم dataای را ذخیره کنم و بعداً دوباره پیدایش کنم»، یک راه‌حل واحد و درست وجود ندارد؛ رویکردهای مختلف در شرایط مختلف مناسب‌اند. پیاده‌سازی نرم‌افزاری معمولاً باید یکی از آن‌ها را انتخاب کند. robust و پربازده‌کردن حتی یک code path هم به‌اندازهٔ کافی دشوار است؛ تلاش برای انجام همه‌چیز در یک نرم‌افزار، تقریباً حتماً به پیاده‌سازی ضعیفی می‌انجامد.

بنابراین انتخاب مناسب‌ترین ابزار نرم‌افزاری به شرایط بستگی دارد. هر نرم‌افزاری، حتی databaseای که «general-purpose» نامیده می‌شود، برای الگوی استفادهٔ مشخصی طراحی شده است.

در برابر این همه گزینه، نخستین چالش پیدا کردن ارتباط میان محصول‌های نرم‌افزاری و شرایطی است که در آن‌ها مناسب‌اند. طبیعی است vendorها چندان مشتاق نباشند بگویند نرم‌افزارشان برای چه workloadهایی انتخاب خوبی نیست؛ اما امیدواریم chapterهای پیشین پرسش‌هایی در اختیارتان گذاشته باشند که با کمکشان بتوانید منظور را از لابه‌لای گفته‌ها پیدا و trade-offها را بهتر درک کنید.

حتی اگر ارتباط ابزارها و شرایط استفاده‌شان را کاملاً درک کنید، چالش دیگری باقی می‌ماند: در applicationهای پیچیده، data اغلب به چند شکل مختلف استفاده می‌شود. بعید است یک نرم‌افزار واحد برای همهٔ موقعیت‌هایی که data در آن‌ها به کار می‌رود مناسب باشد؛ پس برای فراهم‌کردن قابلیت‌های application ناچار می‌شوید چند نرم‌افزار متفاوت را کنار هم قرار دهید.

### Combining Specialized Tools by Deriving Data

برای مثال، معمولاً برای پاسخ به queryهایی با keywordهای دلخواه لازم است OLTP database را با full-text search index یکپارچه کنیم. بعضی databaseها، مانند PostgreSQL، قابلیت full-text indexing دارند که برای applicationهای ساده کافی است؛ اما search پیشرفته‌تر به ابزارهای تخصصی information retrieval نیاز دارد. از سوی دیگر، search indexها معمولاً برای system of record پایدار مناسب نیستند؛ بنابراین بسیاری از applicationها باید دو ابزار متفاوت را ترکیب کنند تا همهٔ نیازهایشان برآورده شود.

در «Keeping Systems in Sync» به مسئلهٔ یکپارچه‌کردن data systemها اشاره کردیم. هرچه تعداد representationهای مختلف data بیشتر شود، مسئلهٔ integration دشوارتر می‌شود. علاوه بر database و search index، شاید لازم باشد copyهایی از data را در analytics systemها ــ مانند data warehouse یا batch و stream processing systemها ــ نگه دارید؛ cache یا نسخه‌های denormalized از objectهای مشتق‌شده از data اصلی بسازید؛ data را از machine learning، classification، ranking یا recommendation system بگذرانید؛ یا بر اساس تغییرهای data notification بفرستید.

بارها از مهندس‌های نرم‌افزار جمله‌هایی شبیه این شنیده‌ام: «طبق تجربهٔ من، ۹۹٪ افراد به X نیاز ندارند» یا «فقط ۱٪ به X نیاز دارند». به‌نظرم چنین حرف‌هایی بیشتر دربارهٔ تجربهٔ گوینده می‌گویند تا سودمندی واقعی یک فناوری. دامنهٔ کارهایی که ممکن است بخواهید با data انجام دهید به‌طرز سرگیجه‌آوری گسترده است. قابلیتی که از نظر یک نفر نامتعارف و بی‌فایده به نظر می‌رسد، شاید برای شخص دیگری یک نیاز اساسی باشد. نیاز به data integration اغلب فقط وقتی آشکار می‌شود که از بالا به dataflowهای کل سازمان نگاه کنیم.

#### استدلال دربارهٔ dataflowها

وقتی برای پاسخ‌گویی به access patternهای مختلف باید copyهایی از data یکسان را در چند storage system نگه دارید، لازم است input و output را دقیق مشخص کنید: data ابتدا کجا نوشته می‌شود و هر representation از کدام source مشتق می‌شود؟ چگونه data را با format درست به همهٔ محل‌های لازم می‌رسانید؟

برای مثال، می‌توان ابتدا data را در system of record database نوشت، تغییرهای database را با Change Data Capture (CDC) ثبت کرد و بعد همان تغییرها را به همان ترتیب در search index اعمال کرد. اگر CDC تنها راه update کردن index باشد، می‌توان مطمئن بود index کاملاً از system of record مشتق شده و با آن consistent است؛ مگر اینکه در نرم‌افزار bugای وجود داشته باشد. در این system تنها راه واردکردن input تازه، نوشتن در database است.

اگر به application اجازه دهید مستقیماً هم در search index و هم در database بنویسد، مسئله‌ای مانند Figure 11-4 به وجود می‌آید: دو client هم‌زمان writeهای متعارضی می‌فرستند و دو storage system آن‌ها را با ترتیب متفاوت پردازش می‌کنند. در این وضعیت نه database و نه search index مسئول تعیین ترتیب writeها نیست؛ هرکدام ممکن است تصمیم متناقضی بگیرد و دو system برای همیشه ناسازگار شوند.

اگر بتوانید همهٔ inputهای user را از یک system عبور دهید که ترتیب همهٔ writeها را تعیین می‌کند، مشتق‌کردن representationهای دیگر data با پردازش writeها به همان ترتیب بسیار آسان‌تر می‌شود. این کاربردی از رویکرد state machine replication است که در «Total Order Broadcast» دیدیم. اینکه از CDC استفاده کنید یا event sourcing log اهمیت کمتری از خود اصل دارد: باید یک total order برای eventها تعیین کنید.

update کردن derived data system بر پایهٔ event log را اغلب می‌توان deterministic و idempotent کرد؛ در نتیجه recovery از faultها ساده‌تر می‌شود.

#### Derived data در برابر distributed transaction

رویکرد کلاسیک برای consistent نگه‌داشتن data systemهای مختلف، استفاده از distributed transaction است. روش derived data system در مقایسه با distributed transaction چگونه عمل می‌کند؟

در سطح انتزاعی، هر دو هدفی مشابه را با روش‌های متفاوت دنبال می‌کنند. distributed transaction با استفاده از lock و mutual exclusion ترتیب writeها را تعیین می‌کند، درحالی‌که CDC و event sourcing برای تعیین ترتیب از log استفاده می‌کنند. distributed transaction با atomic commit تضمین می‌کند تغییرها دقیقاً یک‌بار اعمال شوند؛ log-based systemها اغلب بر retryهای deterministic و idempotence تکیه دارند.

بزرگ‌ترین تفاوت این است که transaction systemها معمولاً linearizability فراهم می‌کنند؛ این ویژگی guaranteeهای مفیدی مانند read-your-writes را به همراه دارد. از سوی دیگر، derived data systemها اغلب asynchronous update می‌شوند و به‌طور پیش‌فرض همان guaranteeهای زمانی را ارائه نمی‌کنند.

distributed transactionها در محیط‌های محدودی که حاضرند هزینه‌شان را بپردازند، با موفقیت استفاده شده‌اند. بااین‌حال، به‌نظر من XA در برابر faultها و از نظر performance ویژگی‌های ضعیفی دارد که کاربردش را به‌شدت محدود می‌کند. شاید بتوان protocol بهتری برای distributed transaction ساخت؛ اما فراگیرشدن و یکپارچه‌کردن چنین protocolی با ابزارهای موجود دشوار است و بعید است به‌زودی اتفاق بیفتد.

تا زمانی که پشتیبانی گسترده‌ای از protocol مناسب distributed transaction وجود نداشته باشد، به‌نظر من derived data مبتنی بر log امیدبخش‌ترین رویکرد برای یکپارچه‌کردن data systemهای مختلف است. البته guaranteeهایی مانند read-your-writes مفیدند. گفتنِ «eventual consistency اجتناب‌ناپذیر است؛ خودتان را وفق دهید» هم رویکرد سازنده‌ای نیست، دست‌کم نه بدون راهنمایی روشن دربارهٔ اینکه چطور باید با آن کنار آمد.

در «Aiming for Correctness» چند رویکرد برای پیاده‌سازی guaranteeهای قوی‌تر روی systemهایی را بررسی می‌کنیم که derived data آن‌ها به‌شکل asynchronous به‌روز می‌شود؛ هدف، پیدا کردن راه میانه‌ای میان distributed transaction و systemهای asynchronous مبتنی بر log است.

#### محدودیت‌های total ordering

در systemهایی که اندازه‌شان به‌اندازهٔ کافی کوچک است، ساخت event logای با ترتیب کلی کاملاً عملی است؛ محبوبیت databaseهای single-leader replication که دقیقاً چنین logای می‌سازند، این موضوع را نشان می‌دهد. اما با بزرگ‌شدن system و پیچیده‌ترشدن workloadها، محدودیت‌هایی ظاهر می‌شوند:

- در بیشتر موارد برای ساخت log با ترتیب کلی، همهٔ eventها باید از یک leader node عبور کنند تا leader ترتیبشان را تعیین کند. اگر throughput eventها از توان یک machine بیشتر باشد، باید کار را میان چند machine partition کنید. در این صورت ترتیب eventها در دو partition مختلف مبهم می‌شود.
- اگر serverها در چند datacenter جغرافیایی پخش شده باشند ــ مثلاً برای تحمل از دسترس خارج‌شدن یک datacenter کامل ــ معمولاً در هر datacenter leader جداگانه‌ای وجود دارد، چون delay شبکه هماهنگی synchronous میان datacenterها را ناکارآمد می‌کند. در نتیجه ترتیب eventهایی که از دو datacenter متفاوت می‌آیند مشخص نیست.
- وقتی application به‌شکل microservice deploy می‌شود، انتخاب رایج این است که هر service و state پایدارش واحد مستقلی باشد و state پایدار میان serviceها share نشود. بنابراین برای دو event که در serviceهای جداگانه رخ داده‌اند ترتیب تعریف‌شده‌ای وجود ندارد.
- بعضی applicationها state سمت client را بلافاصله پس از input کاربر update می‌کنند، بی‌آنکه منتظر تأیید server بمانند؛ حتی ممکن است offline هم کار کنند. در این applicationها به‌احتمال زیاد client و server eventها را با ترتیب متفاوتی می‌بینند.

در اصطلاح رسمی، تعیین total order برای eventها **total order broadcast** نام دارد و با consensus معادل است. بیشتر consensus algorithmها برای شرایطی طراحی شده‌اند که throughput یک node برای پردازش کل event stream کافی باشد؛ این algorithmها سازوکاری برای تقسیم کار ترتیب‌دهی eventها میان چند node ندارند. طراحی consensus algorithmهایی که بتوانند از throughput یک node فراتر بروند و در محیط جغرافیایی توزیع‌شده هم خوب کار کنند، همچنان مسئله‌ای باز در پژوهش است.

#### مرتب‌کردن eventها برای ثبت causality

وقتی میان eventها رابطهٔ علّی وجود ندارد، نداشتن total order مشکل بزرگی نیست؛ eventهای concurrent را می‌توان با هر ترتیبی قرار داد. بعضی موارد دیگر هم ساده‌اند: مثلاً updateهای یک object مشخص را می‌توان با مسیریابی همهٔ updateهای همان object ID به یک log partition، به‌طور کامل مرتب کرد. اما dependencyهای علّی گاهی ظریف‌ترند.

یک social networking service و دو user را در نظر بگیرید که رابطه‌شان تازه تمام شده است. یکی از آن‌ها دیگری را از فهرست دوست‌ها حذف می‌کند و بعد برای دوست‌های باقی‌مانده پیامی می‌فرستد و از شریک قبلی‌اش شکایت می‌کند. منظورش این است که شریک سابق پیام توهین‌آمیز را نبیند، چون پیام پس از لغو رابطهٔ دوستی فرستاده شده است.

اما اگر وضعیت دوستی در یک جا و پیام‌ها در جای دیگری ذخیره شوند، dependency ترتیبی میان event حذف‌کردن دوست و event فرستادن پیام ممکن است از بین برود. اگر این رابطهٔ علّی ثبت نشده باشد، service ارسال notification ممکن است event ارسال پیام را پیش از event حذف دوست پردازش کند و در نتیجه اشتباهاً برای شریک سابق notification بفرستد.

در این مثال، notificationها عملاً joinای میان messageها و friend list هستند و به مسئلهٔ زمانی joinها که پیش‌تر بررسی کردیم مربوط می‌شوند. متأسفانه پاسخ ساده‌ای برای این مشکل وجود ندارد. چند نقطهٔ شروع ممکن است:

- logical timestampها می‌توانند بدون coordination ترتیب کلی ایجاد کنند و در مواردی که total order broadcast عملی نیست کمک‌کننده باشند. بااین‌حال، recipientها همچنان باید eventهای خارج از ترتیب را مدیریت کنند و metadata بیشتری باید همراه eventها منتقل شود.
- اگر بتوانید eventای ثبت کنید که stateای را که user پیش از تصمیم‌گیری دیده توصیف می‌کند و شناسهٔ یکتایی به آن بدهید، eventهای بعدی می‌توانند برای ثبت dependency علّی به شناسهٔ آن event ارجاع دهند. در «Reads are events too» دوباره به این ایده برمی‌گردیم.
- conflict resolution algorithmها در پردازش eventهایی که با ترتیب غیرمنتظره تحویل داده شده‌اند مفیدند. این algorithmها به نگهداری state کمک می‌کنند، اما برای actionهایی با side effect بیرونی ــ مانند فرستادن notification به user ــ کافی نیستند.

شاید در گذر زمان الگوهایی برای توسعهٔ application شکل بگیرند که dependencyهای علّی را کارآمد ثبت و derived state را درست نگهداری کنند، بی‌آنکه همهٔ eventها مجبور باشند از bottleneck مربوط به total order broadcast عبور کنند.

### Batch and Stream Processing

به‌نظر من هدف data integration این است که data در format درست و در همهٔ محل‌های لازم قرار بگیرد. برای این کار باید input را مصرف، transform، join، filter و aggregate کرد؛ model آموزش داد و ارزیابی کرد؛ و در نهایت data را به output مناسب نوشت. batch processor و stream processor ابزارهای رسیدن به این هدف‌اند.

outputهای batch process و stream process، datasetهای مشتق‌شده‌ای مانند search index، materialized view، recommendationهایی برای نمایش به user و aggregate metric هستند.

همان‌طور که در Chapter 10 و Chapter 11 دیدیم، batch processing و stream processing اصول مشترک زیادی دارند. تفاوت بنیادی اصلی این است که stream processor روی dataset نامحدود کار می‌کند، درحالی‌که اندازهٔ input batch process معلوم و محدود است. تفاوت‌های اجرایی دیگری هم میان engineهایشان وجود دارد، اما این مرزها کم‌کم محوتر می‌شوند.

Spark با شکستن stream به microbatch، stream processing را روی batch processing engine اجرا می‌کند؛ در مقابل، Apache Flink batch processing را روی stream processing engine انجام می‌دهد. در اصل، می‌توان یک نوع processing را روی نوع دیگر شبیه‌سازی کرد، هرچند ویژگی‌های performance متفاوت است؛ برای مثال، microbatching ممکن است برای hopping window یا sliding window عملکرد ضعیفی داشته باشد.

#### نگهداری derived state

batch processing، حتی اگر code با زبان functional نوشته نشده باشد، حال‌وهوای functional نیرومندی دارد: functionهای pure و deterministic را تشویق می‌کند که outputشان فقط به input وابسته است و جز output صریح خود side effect دیگری ندارند؛ input را immutable و output را append-only در نظر می‌گیرد. stream processing هم مشابه است، اما operatorها را گسترش می‌دهد تا state مدیریت‌شده و fault-tolerant داشته باشند.

اصل functionهای deterministic با input و output مشخص فقط برای fault tolerance مفید نیست؛ استدلال دربارهٔ dataflowهای داخل سازمان را هم ساده می‌کند. فرقی نمی‌کند derived data یک search index باشد، یک مدل آماری یا cache؛ مفید است آن را pipelineای در نظر بگیریم که یک داده را از داده‌ای دیگر مشتق می‌کند: تغییرهای state در یک system از code کاربردی functional عبور می‌کنند و اثرشان در systemهای مشتق‌شده اعمال می‌شود.

در اصل می‌توان derived data systemها را synchronous نگه داشت؛ درست مانند relational database که secondary indexها را در همان transaction و هم‌زمان با writeهای table update می‌کند. اما asynchronous بودن است که event-log-based systemها را robust می‌سازد: fault در یک بخش system به همان بخش محدود می‌ماند. در مقابل، distributed transactionها اگر هر participantای شکست بخورد abort می‌شوند و در نتیجه با گسترش failure به بقیهٔ system، آن را تشدید می‌کنند.

دیدیم secondary indexها اغلب از مرز partitionها عبور می‌کنند. system partitionشده‌ای که secondary index دارد یا باید writeها را به چند partition بفرستد ــ اگر index بر اساس term partition شده باشد ــ یا readها را به همهٔ partitionها ارسال کند ــ اگر index بر اساس document partition شده باشد. چنین ارتباطی میان partitionها نیز وقتی index به‌صورت asynchronous نگهداری شود، قابل‌اتکاتر و مقیاس‌پذیرتر است.

#### بازپردازش data برای تکامل application

برای نگهداری derived data، هم batch processing و هم stream processing مفیدند. stream processing باعث می‌شود تغییرهای input با delay کم در derived viewها منعکس شوند؛ batch processing امکان می‌دهد حجم زیادی از data تاریخیِ جمع‌شده را دوباره پردازش کنیم و viewهای تازه‌ای از dataset موجود بسازیم.

به‌ویژه، بازپردازش data موجود سازوکار خوبی برای نگهداری و تکامل system و پشتیبانی از featureها و نیازهای تازه است. بدون reprocessing، schema evolution به تغییرهای ساده‌ای مثل افزودن field اختیاری به record یا اضافه‌کردن نوع تازه‌ای از record محدود می‌شود؛ چه در schema-on-write و چه در schema-on-read. اما با reprocessing می‌توان dataset را به model کاملاً متفاوتی بازآرایی کرد تا نیازهای تازه را بهتر برآورده کند.

##### Schema migration در راه‌آهن

«schema migration»های بزرگ‌مقیاس در systemهای غیردیجیتال هم رخ می‌دهند. در سال‌های آغازین ساخت راه‌آهن در انگلستان قرن نوزدهم، چند استاندارد رقیب برای gauge ــ فاصلهٔ میان دو ریل ــ وجود داشت. قطاری که برای یک gauge ساخته شده بود نمی‌توانست روی ریل‌هایی با gauge دیگر حرکت کند و این مسئله امکان اتصال مسیرهای شبکهٔ راه‌آهن را محدود می‌کرد.

پس از آنکه در سال 1846 یک gauge استاندارد انتخاب شد، باید مسیرهایی با gaugeهای دیگر تبدیل می‌شدند. اما چگونه می‌توان این کار را بدون تعطیل‌کردن خط راه‌آهن برای ماه‌ها یا سال‌ها انجام داد؟ راه‌حل این بود که ابتدا با افزودن ریل سوم، مسیر را به gauge دوگانه یا ترکیبی تبدیل کنند. این کار را می‌شد تدریجی انجام داد و پس از پایان، قطارهای هر دو gauge می‌توانستند با استفاده از دو ریل از سه ریل در مسیر حرکت کنند. سرانجام، پس از تبدیل همهٔ قطارها به gauge استاندارد، ریل مربوط به gauge غیراستاندارد برداشته می‌شد.

«بازپردازش» مسیرهای موجود به این شکل و اجازه‌دادن به نسخه‌های قدیمی و جدید برای هم‌زیستی، امکان تغییر تدریجی gauge طی چند سال را فراهم کرد. البته این کار پرهزینه بود و به همین دلیل هنوز gaugeهای غیراستاندارد وجود دارند؛ مثلاً systemِ BART در منطقهٔ خلیج سان‌فرانسیسکو از gaugeای متفاوت با بیشتر نقاط آمریکا استفاده می‌کند.

derived viewها تکامل تدریجی را ممکن می‌کنند. برای بازآرایی dataset لازم نیست migration را در یک لحظه و با تعویض ناگهانی انجام دهید. می‌توانید schema قدیمی و جدید را به‌شکل دو view مستقل و مشتق‌شده از data زیربنایی یکسان، کنار هم نگه دارید. سپس تعداد کمی از userها را به view جدید منتقل کنید تا performance آن را بسنجید و bugها را پیدا کنید، درحالی‌که بیشتر userها همچنان از view قدیمی استفاده می‌کنند. به‌تدریج می‌توانید سهم userهای متصل به view جدید را افزایش دهید و در نهایت view قدیمی را حذف کنید.

مزیت این migration تدریجی آن است که اگر مشکلی پیش بیاید، بازگشت از هر مرحله آسان است: همیشه systemی عملیاتی برای برگشت دارید. با کاهش خطر آسیب برگشت‌ناپذیر، اطمینان بیشتری برای پیش‌بردن تغییر پیدا می‌کنید و می‌توانید سریع‌تر system را بهبود دهید.

#### Lambda Architecture

اگر batch processing برای بازپردازش data تاریخی و stream processing برای پردازش updateهای تازه استفاده شود، این دو را چگونه با هم ترکیب می‌کنیم؟ **Lambda Architecture** یکی از پیشنهادهایی است که در این زمینه توجه زیادی جلب کرده است.

ایدهٔ اصلی Lambda Architecture این است که data ورودی با append کردن eventهای immutable به datasetی که پیوسته بزرگ می‌شود ثبت شود؛ شبیه event sourcing. سپس از این eventها viewهای read-optimized مشتق می‌شوند. این معماری پیشنهاد می‌کند دو system جداگانه را به‌صورت موازی اجرا کنیم: batch processing systemای مانند Hadoop MapReduce و stream processing systemای جداگانه، مثلاً Storm.

در این رویکرد، stream processor eventها را مصرف می‌کند و سریع updateای تقریبی برای view می‌سازد؛ سپس batch processor همان مجموعهٔ eventها را مصرف می‌کند و نسخهٔ اصلاح‌شده‌ای از derived view می‌سازد. استدلال این طراحی آن است که batch processing ساده‌تر و در نتیجه کم‌احتمال‌تر است که bug داشته باشد، درحالی‌که stream processorها کم‌قابلیت‌اتکاتر تلقی می‌شوند و ساخت fault tolerance در آن‌ها دشوارتر است. همچنین stream process می‌تواند از algorithmهای سریع و تقریبی استفاده کند، درحالی‌که batch process algorithmهای کندتر اما دقیق را به کار می‌گیرد.

Lambda Architecture ایده‌ای اثرگذار بود و طراحی data systemها را بهبود داد؛ به‌خصوص با رواج‌دادن اصل مشتق‌کردن view از stream eventهای immutable و بازپردازش eventها هنگام نیاز. بااین‌حال، از نظر عملی چند مشکل هم دارد:

- نگهداری منطق یکسان برای اجرا در framework مربوط به batch و framework مربوط به stream، کار اضافهٔ قابل‌توجهی است. کتابخانه‌هایی مانند Summingbird برای computationهایی که در هر دو context قابل‌اجرا هستند abstraction فراهم می‌کنند، اما پیچیدگی عملیاتی debugging، tuning و نگهداری دو system متفاوت همچنان باقی می‌ماند.
- چون stream pipeline و batch pipeline outputهای جداگانه تولید می‌کنند، برای پاسخ به requestهای user باید آن‌ها را merge کرد. اگر computation aggregation ساده‌ای روی tumbling window باشد این کار نسبتاً آسان است؛ اما با operationهای پیچیده‌تری مانند join و sessionization یا outputی که time series نیست، بسیار دشوارتر می‌شود.
- هرچند بازپردازش کل dataset تاریخی قابلیت خوبی است، انجام مکرر آن برای dataset بزرگ پرهزینه است. بنابراین batch pipeline اغلب باید incremental batchها را پردازش کند؛ مثلاً data یک ساعت را در پایان هر ساعت. این کار مسئله‌هایی را که در «Reasoning About Time» بررسی کردیم، مانند مدیریت eventهای عقب‌مانده و windowهایی که از مرز batchها عبور می‌کنند، دوباره مطرح می‌کند. incremental کردن batch computation پیچیدگی می‌افزاید و آن را به لایهٔ stream processing شبیه‌تر می‌کند؛ برخلاف هدف ساده نگه‌داشتن batch layer.

#### یکپارچه‌کردن batch و stream processing

کارهای جدیدتر امکان بهره‌بردن از مزایای Lambda Architecture بدون معایبش را فراهم کرده‌اند: batch computationها ــ بازپردازش data تاریخی ــ و stream computationها ــ پردازش eventها هنگام رسیدن ــ را می‌توان در یک system پیاده‌سازی کرد.

یکپارچه‌کردن batch و stream processing در یک system به قابلیت‌های زیر نیاز دارد که دسترسی به آن‌ها روزبه‌روز رایج‌تر می‌شود:

- امکان replay کردن eventهای تاریخی با همان processing engineای که stream eventهای تازه را مدیریت می‌کند. برای مثال، message brokerهای مبتنی بر log قابلیت replay پیام دارند و بعضی stream processorها می‌توانند input را از distributed filesystemای مانند HDFS بخوانند.
- exactly-once semantics برای stream processorها؛ یعنی تضمین اینکه output همان باشد که انگار faultی رخ نداده، حتی اگر در واقع fault رخ داده باشد. مانند batch processing، این کار مستلزم کنارگذاشتن output ناقص taskهای شکست‌خورده است.
- ابزارهایی برای windowing بر اساس event time، نه processing time؛ چون هنگام بازپردازش eventهای تاریخی، processing time معنایی ندارد. برای مثال، Apache Beam APIای برای بیان چنین computationهایی فراهم می‌کند که بعداً می‌توان آن‌ها را با Apache Flink یا Google Cloud Dataflow اجرا کرد.

## Key Terms

- `Data Integration` — رساندن data به representation مناسب در همهٔ systemهای موردنیاز و هماهنگ‌کردن جریان میان آن‌ها.
- `Derived Data` — dataای که با transformation، aggregation یا ترکیب از source data تولید می‌شود؛ مانند search index، cache یا materialized view.
- `Lambda Architecture` — معماری‌ای که batch processing تاریخی و stream processing بلادرنگ را موازی اجرا و outputهایشان را ترکیب می‌کند.

# Chapter 11 — Stream Processing

## Databases and Streams

پیش‌تر message brokerها را با databaseها مقایسه کردیم. این دو دسته ابزار سنتاً جدا از هم در نظر گرفته می‌شدند، اما دیدیم message brokerهای مبتنی بر log ایده‌هایی از database گرفته‌اند و در messaging به کار برده‌اند. مسیر برعکس هم ممکن است: ایده‌های messaging و stream را به databaseها ببریم.

گفتیم event رکوردی از چیزی است که در زمانی رخ داده است. آن اتفاق می‌تواند فعالیت user، مانند واردکردن search query، یا خواندن sensor باشد؛ اما ممکن است یک write به database هم باشد. اینکه چیزی در database نوشته شده خود یک event است که می‌توان آن را capture، ذخیره و پردازش کرد. بنابراین رابطهٔ database و stream عمیق‌تر از ذخیره‌سازی فیزیکی log روی disk است و جنبه‌ای بنیادی دارد.

در واقع، replication log یک stream از eventهای write در database است که leader هنگام پردازش transactionها تولید می‌کند. followerها همین stream از writeها را روی copy خود از database apply می‌کنند و در نتیجه به copy دقیقی از همان data می‌رسند. eventهای replication log تغییرهایی را توصیف می‌کنند که در data رخ داده‌اند.

در «Total Order Broadcast» نیز با اصل state machine replication آشنا شدیم: اگر هر event نمایندهٔ یک write به database باشد و همهٔ replicaها eventهای یکسان را به ترتیب یکسان پردازش کنند، سرانجام همه به state نهایی یکسانی می‌رسند؛ با این فرض که پردازش هر event deterministic باشد. این هم نمونهٔ دیگری از event stream است.

در این بخش ابتدا مسئله‌ای را در systemهای ناهمگون بررسی می‌کنیم و سپس می‌بینیم چگونه می‌توان با آوردن ایده‌های event stream به databaseها آن را حل کرد.

### Keeping Systems in Sync

همان‌طور که در سراسر کتاب دیدیم، هیچ system واحدی نمی‌تواند همهٔ نیازهای ذخیره‌سازی، query و پردازش data را برآورده کند. در عمل، بیشتر applicationهای جدی برای پاسخ‌گویی به نیازهایشان چند فناوری مختلف را کنار هم به کار می‌گیرند: مثلاً OLTP database برای پاسخ‌دادن به requestهای user، cache برای سریع‌کردن requestهای پرتکرار، full-text index برای search queryها و data warehouse برای analytics. هرکدام copy خودش از data را با representationای نگه می‌دارد که برای هدف خودش بهینه شده است.

وقتی data یکسان یا مرتبط در چند جا وجود دارد، باید این نسخه‌ها را با هم sync نگه داشت: اگر موردی در database تغییر کند، cache، search index و data warehouse هم باید update شوند. در data warehouseها این synchronization معمولاً با ETL process انجام می‌شود؛ اغلب یک copy کامل از database گرفته می‌شود، transformation روی آن صورت می‌گیرد و به‌شکل bulk در data warehouse بارگذاری می‌شود؛ یعنی با یک batch process. در «The Output of Batch Workflows» هم دیدیم search index، recommendation system و systemهای دیگر برای derived data را می‌توان با batch process ساخت.

اگر dump کامل دوره‌ای از database بیش‌ازحد کند باشد، گاهی از **dual writes** استفاده می‌شود: code خود application هنگام تغییر data صریحاً در هر system می‌نویسد؛ مثلاً اول database را update می‌کند، بعد search index را و سپس entryهای cache را invalidate می‌کند. حتی ممکن است این writeها هم‌زمان انجام شوند.

اما dual writeها مشکل‌های جدی دارند؛ یکی از آن‌ها race conditionای است که در Figure 11-4 نشان داده شده است. فرض کنید دو client هم‌زمان می‌خواهند itemِ `X` را تغییر دهند: client 1 مقدار `A` و client 2 مقدار `B` را می‌نویسد. هر دو ابتدا مقدار جدید را در database و بعد در search index می‌نویسند. به‌دلیل زمان‌بندی نامناسب، requestها در هم می‌آمیزند: database اول write مربوط به client 1 را می‌بیند و مقدار را `A` می‌کند، سپس write مربوط به client 2 را می‌بیند و مقدار نهایی‌اش `B` می‌شود. اما search index ابتدا write client 2 و بعد write client 1 را می‌بیند؛ پس مقدار نهایی آن `A` است. در نتیجه database و search index برای همیشه ناسازگار می‌شوند، با اینکه هیچ خطایی رخ نداده است.

مگر اینکه سازوکار دیگری برای تشخیص concurrency داشته باشید، مثل version vectorهایی که در «Detecting Concurrent Writes» بررسی کردیم، حتی متوجه هم‌زمانی writeها نمی‌شوید؛ یک مقدار بی‌سروصدا مقدار دیگر را overwrite می‌کند.

مشکل دیگر dual write این است که یکی از writeها ممکن است موفق شود و دیگری شکست بخورد. این بار مسئله fault tolerance است، نه concurrency، اما نتیجه باز هم ناسازگارشدن دو system است. تضمین اینکه هر دو write با هم موفق شوند یا هر دو شکست بخورند، مسئلهٔ atomic commit است و حل آن پرهزینه است.

اگر فقط یک databaseِ replicated با یک leader داشته باشید، leader ترتیب writeها را تعیین می‌کند و state machine replication میان replicaهای آن database کار می‌کند. اما در Figure 11-4 یک leader واحد وجود ندارد: ممکن است database یک leader و search index هم leader خودش را داشته باشد، ولی هیچ‌کدام از دیگری پیروی نمی‌کند و در نتیجه conflict ممکن است رخ دهد.

وضع بهتر می‌شد اگر واقعاً فقط یک leader وجود داشت ــ مثلاً خود database ــ و می‌توانستیم search index را follower آن کنیم. آیا این کار در عمل امکان‌پذیر است؟

### Change Data Capture

مشکل replication log بیشتر databaseها این است که مدت‌ها جزئیات داخلی implementation database تلقی می‌شدند، نه public API. انتظار می‌رفت clientها از طریق data model و query language با database کار کنند، نه اینکه replication log را parse کنند و data را از آن بیرون بکشند.

برای دهه‌ها، بسیاری از databaseها راه مستند و رسمی‌ای برای دریافت log تغییرهایشان نداشتند. به همین دلیل، برداشتن همهٔ تغییرهای یک database و replicate کردن آن‌ها به فناوری ذخیره‌سازی دیگری مانند search index، cache یا data warehouse دشوار بود.

در سال‌های اخیر علاقه به **change data capture (CDC)** بیشتر شده است. CDC فرایند مشاهدهٔ همهٔ تغییرهایی است که در database نوشته می‌شوند و استخراج آن‌ها به شکلی است که بتوان به systemهای دیگر replicateشان کرد. CDC به‌ویژه وقتی جذاب است که تغییرها بلافاصله پس از write، به‌صورت stream در دسترس قرار بگیرند.

برای مثال، می‌توان تغییرهای database را capture کرد و به‌طور پیوسته همان تغییرها را در search index apply کرد. اگر log تغییرها به همان ترتیب اعمال شود، می‌توان انتظار داشت data در search index با database منطبق بماند. search index و هر system دیگری برای derived data در این معماری consumerهای change stream هستند؛ همان‌طور که در Figure 11-5 نشان داده شده است.

#### پیاده‌سازی change data capture

می‌توان مصرف‌کنندگان log را systemهای derived data نامید: data ذخیره‌شده در search index و data warehouse فقط view دیگری از data در **system of record** است. CDC سازوکاری است برای اطمینان از اینکه همهٔ تغییرهای system of record در systemهای derived data هم منعکس شوند و آن systemها copy دقیقی از data داشته باشند.

در اصل، CDC یک database را leader می‌کند ــ databaseای که تغییرهایش capture می‌شوند ــ و بقیه را به follower تبدیل می‌کند. message broker مبتنی بر log برای انتقال change eventها از database مبدأ مناسب است، چون ترتیب پیام‌ها را حفظ می‌کند و از جابه‌جایی ترتیبی مانند Figure 11-2 جلوگیری می‌کند.

برای پیاده‌سازی CDC می‌توان از database trigger استفاده کرد: triggerهایی ثبت می‌شوند که همهٔ تغییرهای tableهای data را می‌بینند و entry متناظری در changelog table می‌نویسند. اما triggerها معمولاً شکننده‌اند و سربار performance قابل‌توجهی دارند. parse کردن replication log می‌تواند رویکردی robustتر باشد، هرچند خودش چالش‌هایی دارد؛ از جمله رسیدگی به تغییر schema.

LinkedIn Databus، Facebook Wormhole و Yahoo! Sherpa این ایده را در مقیاس بزرگ به کار می‌گیرند. Bottled Water برای PostgreSQL با APIای که write-ahead log را decode می‌کند CDC فراهم می‌کند. Maxwell و Debezium کاری مشابه را برای MySQL با parse کردن binlog انجام می‌دهند؛ Mongoriver، MongoDB oplog را می‌خواند و GoldenGate امکاناتی مشابه برای Oracle ارائه می‌کند.

CDC، مانند message brokerها، معمولاً asynchronous است: databaseِ system of record پیش از commit کردن منتظر نمی‌ماند تا تغییر به consumerها اعمال شود. مزیت عملیاتی این طراحی آن است که اضافه‌کردن consumer کند تأثیر زیادی بر system of record نمی‌گذارد؛ عیبش این است که همهٔ مسئله‌های replication lag همچنان وجود دارند.

#### Snapshot اولیه

اگر log همهٔ تغییرهایی را که تاکنون در database رخ داده نگه دارد، می‌توان با replay کردن log کل state database را بازسازی کرد. اما نگهداری همیشگی همهٔ changeها ممکن است به disk زیادی نیاز داشته باشد و replay کردنشان زمان‌بر باشد؛ بنابراین log باید گاهی truncate شود.

برای ساخت full-text index تازه، مثلاً، به copy کاملی از کل database نیاز است. اعمال log تغییرهای جدید کافی نیست، چون itemهایی که اخیراً update نشده‌اند در آن log وجود ندارند. بنابراین اگر تاریخچهٔ کامل log را ندارید، باید با snapshotای consistent شروع کنید.

snapshot database باید با position یا offset مشخصی در change log متناظر باشد تا معلوم شود پس از پردازش snapshot، اعمال تغییرها را از کجا ادامه دهیم. بعضی CDC toolها این قابلیت snapshot را یکپارچه فراهم می‌کنند و بعضی دیگر آن را به‌صورت عملیاتی دستی بر عهدهٔ کاربر می‌گذارند.

#### Log compaction

اگر فقط بتوان مقدار محدودی از تاریخچهٔ log را نگه داشت، هر بار که system جدیدی برای derived data اضافه می‌کنید باید فرایند snapshot را انجام دهید. **Log compaction** جایگزین مناسبی فراهم می‌کند.

اصل log compaction ساده است و پیش‌تر در «Hash Indexes»، در زمینهٔ log-structured storage engineها، با آن آشنا شدیم. storage engine به‌طور دوره‌ای recordهای log دارای key یکسان را پیدا می‌کند، نسخه‌های تکراری را کنار می‌گذارد و فقط جدیدترین update هر key را نگه می‌دارد. این compaction و merge در background انجام می‌شود.

در log-structured storage engine، updateای با value ویژهٔ `null` ــ یعنی tombstone ــ نشان می‌دهد key حذف شده است و باعث می‌شود آن key هنگام log compaction کنار گذاشته شود. اما تا وقتی key overwrite یا حذف نشده باشد، برای همیشه در log باقی می‌ماند. فضای disk لازم برای log compactشده فقط به محتوای فعلی database وابسته است، نه تعداد writeهایی که از ابتدا انجام شده‌اند. اگر یک key بارها overwrite شود، valueهای قبلی سرانجام garbage-collect می‌شوند و فقط جدیدترین value باقی می‌ماند.

همین ایده برای log-based message broker و CDC هم کار می‌کند. اگر CDC طوری تنظیم شده باشد که هر تغییر primary key داشته باشد و هر update مقدار قبلی همان key را جایگزین کند، کافی است جدیدترین write هر key نگه داشته شود.

در این صورت، هر وقت بخواهید derived data systemی مانند search index را از نو بسازید، می‌توانید consumer تازه‌ای را از offset صفرِ topic فشرده‌شده اجرا کنید و پیام‌های log را ترتیبی بخوانید. تضمین می‌شود log جدیدترین value هر key در database را در خود داشته باشد ــ و شاید تعدادی value قدیمی‌تر را هم. بنابراین می‌توان محتوای کامل database را از آن به دست آورد، بدون اینکه دوباره از CDC source database snapshot بگیریم.

Apache Kafka از log compaction پشتیبانی می‌کند. همان‌طور که بعدتر در chapter خواهیم دید، این قابلیت message broker را به storage پایدار تبدیل می‌کند، نه فقط ابزاری برای messaging گذرا.

#### پشتیبانی API از change streamها

databaseها روزبه‌روز بیشتر change stream را به‌عنوان interfaceای درجه‌یک پشتیبانی می‌کنند، نه اینکه CDC صرفاً راه‌حلی وصله‌شده و reverse-engineered باشد. برای مثال، RethinkDB اجازه می‌دهد queryها برای notification هنگام تغییر نتیجه subscribe شوند؛ Firebase و CouchDB data synchronization را بر پایهٔ change feedای فراهم می‌کنند که applicationها هم به آن دسترسی دارند؛ و Meteor از MongoDB oplog برای subscribe شدن به تغییرهای data و update کردن user interface استفاده می‌کند.

VoltDB اجازه می‌دهد transactionها data را به‌طور پیوسته از database به شکل stream export کنند. در relational data model، database یک output stream را به‌شکل tableای نمایش می‌دهد که transactionها می‌توانند tuple در آن درج کنند، اما نمی‌توان آن را query کرد. stream در این صورت log tupleهایی است که transactionهای commitشده در آن table ویژه نوشته‌اند، به ترتیب commit شدنشان. consumerهای بیرونی می‌توانند این log را asynchronous مصرف کنند و از آن برای update کردن derived data systemها بهره ببرند.

Kafka Connect تلاشی برای یکپارچه‌کردن CDC toolهای systemهای database متنوع با Kafka است. وقتی stream مربوط به change eventها وارد Kafka شد، می‌توان از آن برای update کردن derived data systemهایی مثل search index و نیز تغذیهٔ stream processing systemها استفاده کرد.

### Event Sourcing

میان ایده‌هایی که اینجا بررسی کردیم و **event sourcing** شباهت‌هایی وجود دارد؛ تکنیکی که در جامعهٔ domain-driven design (DDD) شکل گرفته است. چون این روش ایده‌های مفیدی برای stream processing دارد، آن را مختصر بررسی می‌کنیم.

event sourcing هم، مانند CDC، همهٔ تغییرهای application state را به‌شکل logای از change eventها ذخیره می‌کند. تفاوت اصلی این است که event sourcing این ایده را در سطح abstraction دیگری به کار می‌برد:

- در CDC، application از database به‌شکل mutable استفاده می‌کند و recordها را هر طور لازم باشد update یا delete می‌کند. log تغییرها در سطح پایین از database استخراج می‌شود، مثلاً با parse کردن replication log؛ بنابراین ترتیب writeهای استخراج‌شده با ترتیب واقعی writeها در database منطبق است و race condition شکل‌گرفته در Figure 11-4 رخ نمی‌دهد. لازم نیست applicationای که در database می‌نویسد بداند CDC در حال انجام است.
- در event sourcing، منطق application صریحاً بر پایهٔ eventهای immutable ساخته می‌شود که در event log نوشته می‌شوند. event store از نوع append-only است و update یا delete در آن منع یا دست‌کم discouraged می‌شود. eventها اتفاق‌های سطح application را بازتاب می‌دهند، نه تغییرهای low-level در state.

event sourcing تکنیک قدرتمندی برای data modeling است: از دید application، ثبت actionهای user به‌شکل eventهای immutable معنادارتر از ثبت اثر آن actionها در databaseای mutable است. event sourcing تکامل application در گذر زمان را آسان‌تر می‌کند، با روشن‌کردن علت رخدادها debugging را بهبود می‌دهد و در برابر bugهای application محافظت ایجاد می‌کند.

برای مثال، event «دانشجو ثبت‌نام خود در درس را لغو کرد» قصد یک action را به‌شکل روشن و خنثی بیان می‌کند؛ اما side effectهایی مانند «یک entry از enrollments table حذف شد و دلیل لغو به student feedback table اضافه شد» فرض‌های زیادی دربارهٔ نحوهٔ استفادهٔ آینده از data در خود دارند. اگر feature تازه‌ای اضافه شود ــ مثلاً «جای خالی به نفر بعدی در فهرست انتظار پیشنهاد شود» ــ در رویکرد event sourcing می‌توان side effect جدید را به‌سادگی به event موجود متصل کرد.

event sourcing به chronicle data model شباهت دارد و میان event log و fact table در star schema هم شباهت‌هایی هست. databaseهای تخصصی مانند Event Store برای پشتیبانی از applicationهای event-sourced ساخته شده‌اند، اما این رویکرد به ابزار خاصی وابسته نیست؛ می‌توان applicationی با این سبک را با database معمولی یا message broker مبتنی بر log هم ساخت.

#### استخراج state فعلی از event log

event log به‌تنهایی چندان کاربردی نیست، چون userها معمولاً انتظار دارند state فعلی system را ببینند، نه تاریخچهٔ تغییرهای آن را. مثلاً در وب‌سایت فروشگاهی، user می‌خواهد محتوای فعلی سبد خریدش را ببیند، نه فهرست append-only همهٔ تغییرهایی که تاکنون در سبد داده است.

بنابراین applicationهای event-sourced باید log eventها ــ که data نوشته‌شده به system را بازنمایی می‌کند ــ به application stateای تبدیل کنند که برای نمایش به user مناسب باشد؛ یعنی شکل read data. این transformation می‌تواند منطق دلخواه داشته باشد، اما باید deterministic باشد تا اجرای دوبارهٔ آن همان application state را از event log بسازد.

مانند CDC، replay کردن event log امکان بازسازی state فعلی system را می‌دهد. اما log compaction در اینجا با CDC تفاوت دارد:

- در CDC، event مربوط به update یک record معمولاً کل نسخهٔ تازهٔ record را در خود دارد. بنابراین جدیدترین event هر primary key به‌تنهایی مقدار فعلی آن key را مشخص می‌کند و log compaction می‌تواند eventهای قبلی همان key را دور بریزد.
- در event sourcing، eventها در سطح بالاتری مدل می‌شوند: event معمولاً قصد action کاربر را بیان می‌کند، نه جزئیات state update حاصل از آن. eventهای بعدی معمولاً eventهای قبلی را جایگزین نمی‌کنند؛ پس برای بازسازی state نهایی به تاریخچهٔ کامل eventها نیاز است و نمی‌توان به همان روش log compaction انجام داد.

applicationهای event-sourced معمولاً سازوکاری برای ذخیرهٔ snapshotهایی از state فعلی دارند که از event log مشتق شده است؛ در نتیجه لازم نیست هر بار کل log را دوباره پردازش کنند. اما این فقط بهینه‌سازی performance برای سریع‌کردن read و recovery از crash است. مقصود این است که system بتواند همهٔ eventهای خام را برای همیشه نگه دارد و هر زمان لازم شد کل event log را دوباره پردازش کند. محدودیت این فرض را در ادامه بررسی می‌کنیم.

#### Commandها و eventها

فلسفهٔ event sourcing میان **command** و **event** تمایز دقیقی می‌گذارد. وقتی requestی از user می‌رسد، در ابتدا command است: هنوز ممکن است شکست بخورد، مثلاً چون یک شرط integrity نقض شده است. application باید اول اعتبار command را بررسی کند. اگر validation موفق بود و command پذیرفته شد، آن command به eventی durable و immutable تبدیل می‌شود.

برای نمونه، اگر user بخواهد username خاصی را ثبت کند یا صندلی‌ای در هواپیما یا سالن تئاتر رزرو کند، application باید بررسی کند که آن username یا صندلی قبلاً گرفته نشده باشد. وقتی این بررسی موفق شد، application می‌تواند eventی تولید کند که نشان دهد user مشخصی با user ID معین ثبت‌نام کرده، یا صندلی مشخصی برای customer مشخصی رزرو شده است.

وقتی event تولید شد، به یک fact تبدیل می‌شود. حتی اگر customer بعداً تصمیم بگیرد رزرو را تغییر دهد یا لغو کند، این fact همچنان درست است که او قبلاً آن صندلی را رزرو کرده بود؛ تغییر یا لغو نیز event جداگانه‌ای است که بعدتر به log افزوده می‌شود.

consumer مربوط به event stream اجازه ندارد event را رد کند: وقتی event به consumer می‌رسد، بخشی immutable از log شده و ممکن است consumerهای دیگر هم آن را دیده باشند. بنابراین validation یک command باید به‌صورت synchronous و پیش از تبدیل‌شدنش به event انجام شود؛ مثلاً با serializable transactionای که command را به‌طور atomic اعتبارسنجی و event را منتشر می‌کند.

راه دیگر این است که request رزرو صندلی را به دو event تقسیم کنیم: ابتدا یک رزرو موقت و سپس، پس از تأیید اعتبار رزرو، event تأیید جداگانه‌ای. این تفکیک اجازه می‌دهد validation در processای asynchronous انجام شود.

### State, Streams, and Immutability

در Chapter 10 دیدیم immutable بودن input fileها برای batch processing سودمند است؛ چون می‌توان jobهای آزمایشی را روی fileهای موجود اجرا کرد، بی‌آنکه نگران خراب‌شدنشان بود. همین اصل immutability است که event sourcing و CDC را هم قدرتمند می‌کند.

معمولاً فکر می‌کنیم databaseها state فعلی application را ذخیره می‌کنند. این representation برای read بهینه و معمولاً برای پاسخ‌گویی به queryها مناسب است. از آنجا که state تغییر می‌کند، databaseها از update و delete در کنار insert پشتیبانی می‌کنند. این با immutability چگونه سازگار است؟

هر state متغیری در طول زمان نتیجهٔ eventهایی است که آن را تغییر داده‌اند. مثلاً فهرست صندلی‌های در دسترس حاصل reservationهایی است که پردازش شده‌اند؛ موجودی فعلی حساب نتیجهٔ credit و debitهاست؛ و نمودار response time وب‌سرور aggregation زمان پاسخ تک‌تک requestهای web است. هر تغییری در state رخ داده باشد، دنباله‌ای از eventها علت آن بوده است. حتی اگر کاری انجام و بعد لغو شود، واقعیت رخ‌دادن eventهای آن از بین نمی‌رود.

ایدهٔ اصلی این است که mutable state و logِ append-only از eventهای immutable با هم تناقض ندارند؛ دو روی یک سکه‌اند. log همهٔ تغییرها ــ یعنی changelog ــ تحول state در طول زمان را نشان می‌دهد.

اگر بخواهیم این رابطه را به زبان ریاضی بیان کنیم، application state حاصل integrate کردن event stream در طول زمان است و change stream را می‌توان با differentiate کردن state نسبت به زمان به دست آورد. این قیاس محدودیت‌هایی دارد؛ مثلاً مشتق دوم state ظاهراً معنای روشنی ندارد، اما نقطهٔ شروع مفیدی برای فکرکردن دربارهٔ data است.

ذخیرهٔ durable changelog باعث می‌شود state بازتولیدپذیر باشد. اگر event log را system of record بدانیم و هر mutable state را مشتق‌شده از آن در نظر بگیریم، دنبال‌کردن جریان data در system آسان‌تر می‌شود. به بیان دیگر، transaction log همهٔ تغییرهای database را ثبت می‌کند و فقط با appendهای سریع تغییر می‌کند؛ محتوای database را می‌توان cacheای از تازه‌ترین valueهای recordهای موجود در log دانست. منبع اصلی حقیقت log است و database cacheای از بخشی از log است که تازه‌ترین value هر record و index را در خود نگه می‌دارد.

log compaction یکی از راه‌های پیوندزدن log با state database است: فقط جدیدترین نسخهٔ هر record را نگه می‌دارد و نسخه‌هایی را که overwrite شده‌اند حذف می‌کند.

#### مزیت‌های eventهای immutable

immutability در databaseها ایدهٔ تازه‌ای نیست. حسابدارها قرن‌هاست در bookkeeping مالی از آن استفاده می‌کنند. وقتی transactionای رخ می‌دهد، در دفتر کل append-only ثبت می‌شود؛ دفتری که log eventهای مربوط به جابه‌جایی پول، کالا یا service است. accountهایی مانند سود و زیان یا balance sheet از جمع‌زدن transactionهای این دفتر به دست می‌آیند.

اگر اشتباهی رخ دهد، حسابدارها transaction نادرست را از دفتر پاک یا اصلاح نمی‌کنند؛ transaction دیگری اضافه می‌کنند که اثر اشتباه را جبران کند، مثلاً مبلغی را که نادرست دریافت شده refund می‌کنند. transaction اشتباه برای همیشه در دفتر می‌ماند، چون ممکن است برای audit مهم باشد. اگر رقم‌های نادرستِ مشتق‌شده از آن دفتر قبلاً منتشر شده باشند، اصلاح در رقم‌های دورهٔ حسابداری بعدی وارد می‌شود. در حسابداری این روال کاملاً عادی است.

قابلیت audit در systemهای مالی به‌ویژه اهمیت دارد، اما برای systemهای دیگری که زیر چنین مقررات سختی نیستند هم سودمند است. همان‌طور که در «Philosophy of batch process outputs» دیدیم، اگر تصادفاً code معیوبی deploy کنید که data نادرستی در database بنویسد، وقتی آن code بتواند data را به‌شکل مخرب overwrite کند recovery دشوارتر خواهد بود. با log append-only از eventهای immutable، تشخیص اتفاق و بازیابی از مشکل آسان‌تر است.

eventهای immutable اطلاعاتی بیش از state فعلی نگه می‌دارند. مثلاً در وب‌سایت فروشگاهی ممکن است customer کالایی را به سبد خرید اضافه و بعد حذف کند. از دید fulfillment سفارش، event دوم اثر event اول را خنثی می‌کند؛ اما برای analytics مفید است بدانیم customer به کالایی فکر کرده و بعد از خریدش منصرف شده است. شاید بعداً آن را بخرد یا شاید جایگزینی پیدا کرده باشد. این اطلاعات در event log ثبت می‌شود، اما در databaseای که با حذف کالا از سبد، record آن را پاک می‌کند از بین می‌رود.

#### ساخت چند view از یک event log

با جداکردن mutable state از event log immutable می‌توان چند representation متفاوت و read-oriented را از همان log ساخت. این درست مانند داشتن چند consumer برای یک stream است. برای مثال، analytic databaseای به نام Druid مستقیماً از Kafka data می‌گیرد؛ Pistachio یک distributed key-value store است که Kafka را به‌عنوان commit log به کار می‌برد؛ و Kafka Connect می‌تواند data را از Kafka به databaseها و indexهای مختلف صادر کند. منطقی است که storage و indexing systemهای دیگری مانند search serverها هم input خود را به همین شکل از distributed log بگیرند.

وجود مرحلهٔ صریحی برای تبدیل event log به database، تکامل application را آسان‌تر می‌کند. اگر بخواهید feature تازه‌ای اضافه کنید که data موجود را به شکل جدیدی نمایش دهد، می‌توانید با event log یک view مستقل و read-optimized برای آن بسازید و کنار systemهای موجود اجرا کنید، بی‌آنکه آن‌ها را تغییر دهید. اجرای system قدیمی و جدید در کنار هم اغلب از schema migration پیچیده در system موجود آسان‌تر است. وقتی دیگر به system قدیمی نیازی نباشد، می‌توان آن را خاموش کرد و resourceهایش را آزاد ساخت.

اگر لازم نباشد از ابتدا نگران queryها و روش دسترسی به data باشید، ذخیره‌کردن data معمولاً ساده است. بسیاری از پیچیدگی‌های schema design، indexing و storage engine از این نیاز ناشی می‌شوند که query و access patternهای مشخصی را پشتیبانی کنیم. به همین دلیل، جداکردن شکلی که data با آن نوشته می‌شود از شکل read آن و اجازه‌دادن به چند read view انعطاف زیادی ایجاد می‌کند. این ایده گاهی **command query responsibility segregation (CQRS)** نامیده می‌شود.

رویکرد سنتی database و schema design بر این تصور نادرست بنا شده که data باید به همان شکلی نوشته شود که قرار است query شود. اگر بتوان data را از event log بهینه‌شده برای write به application state بهینه‌شده برای read تبدیل کرد، بحث‌های normalization و denormalization تا حد زیادی بی‌موضوع می‌شوند: denormalize کردن read view کاملاً معقول است، چون فرایند تبدیل سازوکاری برای consistent نگه‌داشتن آن با event log فراهم می‌کند.

در «Describing Load»، home timelineهای Twitter را بررسی کردیم: cacheای از tweetهای اخیر کسانی که user دنبال می‌کند، شبیه mailbox. این هم نمونه‌ای از state بهینه‌شده برای read است. home timelineها به‌شدت denormalize شده‌اند، چون tweet هر شخص در timeline همهٔ دنبال‌کنندگانش تکرار می‌شود. بااین‌حال، fan-out service این state تکراری را با tweetهای تازه و رابطه‌های تازهٔ follow همگام می‌کند و تکرار data را قابل‌مدیریت نگه می‌دارد.

#### کنترل concurrency

بزرگ‌ترین عیب event sourcing و CDC این است که consumerهای event log معمولاً asynchronous هستند. بنابراین ممکن است userای در log write کند و بلافاصله از view مشتق‌شده از log بخواند، اما ببیند writeاش هنوز در read view منعکس نشده است. پیش‌تر این مسئله و راه‌حل‌های احتمالی آن را در «Reading Your Own Writes» بررسی کردیم.

یک راه‌حل این است که update کردن read view هم‌زمان با append کردن event به log انجام شود. این کار به transactionای نیاز دارد که هر دو write را در یک واحد atomic قرار دهد؛ بنابراین یا باید event log و read view در یک storage system باشند، یا بین systemهای مختلف distributed transaction اجرا شود. راه دیگر استفاده از رویکردی است که در «Implementing linearizable storage using total order broadcast» بررسی کردیم.

از سوی دیگر، مشتق‌کردن state فعلی از event log بعضی جنبه‌های کنترل concurrency را ساده‌تر می‌کند. بخش زیادی از نیاز به multi-object transaction از اینجا می‌آید که یک action کاربر باید data را در چند جای مختلف تغییر دهد. در event sourcing می‌توان event را طوری طراحی کرد که توصیفی مستقل از یک action کاربر باشد. در نتیجه، action کاربر فقط به یک write در یک محل نیاز دارد: append کردن eventها به log؛ atomic کردن این عملیات ساده است.

اگر event log و application state به یک شکل partition شده باشند ــ برای مثال، پردازش event مربوط به customer در partition 3 فقط به update کردن partition 3 از application state نیاز داشته باشد ــ یک log consumer ساده و تک‌threadی برای writeها به concurrency control نیاز ندارد. این consumer بنا به ساختار فقط یک event را در هر لحظه پردازش می‌کند. log با تعیین ترتیب serial eventها در هر partition، nondeterminism ناشی از concurrency را حذف می‌کند. اگر event چند state partition را تحت‌تأثیر قرار دهد، کار بیشتری لازم است؛ این موضوع را در Chapter 12 بررسی می‌کنیم.

#### محدودیت‌های immutability

بسیاری از systemهایی که از event-sourced model استفاده نمی‌کنند هم به immutability متکی‌اند: بعضی databaseها در داخل از data structureهای immutable یا multi-version data برای پشتیبانی از snapshotهای یک نقطهٔ زمانی استفاده می‌کنند. version control systemهایی مانند Git، Mercurial و Fossil نیز برای نگهداری تاریخچهٔ نسخهٔ fileها از data immutable بهره می‌برند.

نگهداری تاریخچهٔ immutable همهٔ تغییرها برای همیشه تا چه حد عملی است؟ پاسخ به میزان churn در dataset بستگی دارد. بعضی workloadها بیشتر data اضافه می‌کنند و به‌ندرت update یا delete دارند؛ immutable کردنشان آسان است. workloadهای دیگری روی dataset نسبتاً کوچکی update و delete فراوانی انجام می‌دهند. در چنین مواردی تاریخچهٔ immutable ممکن است بیش‌ازحد بزرگ شود، fragmentation مشکل‌ساز شود و performance مربوط به compaction و garbage collection برای پایداری عملیاتی اهمیت زیادی پیدا کند.

جدا از ملاحظات performance، گاهی به دلایل مدیریتی لازم است data حذف شود، با وجود اصل immutability. مثلاً مقررات privacy ممکن است پس از بسته‌شدن account، حذف اطلاعات شخصی user را لازم بدانند؛ قوانین حفاظت از data ممکن است حذف اطلاعات نادرست را الزامی کنند؛ یا باید جلوی انتشار بیشتر اطلاعات حساسِ به‌اشتباه افشاشده گرفته شود.

در چنین شرایطی کافی نیست event تازه‌ای append کنیم که بگوید data قبلی باید حذف‌شده تلقی شود؛ باید تاریخچه را بازنویسی کنیم و چنان رفتار کنیم که گویی data از ابتدا نوشته نشده است. Datomic این قابلیت را excision می‌نامد و version control systemِ Fossil مفهومی مشابه به نام shunning دارد.

حذف واقعی data به‌شکل شگفت‌آوری دشوار است، چون copyهای آن ممکن است در جاهای زیادی وجود داشته باشند. برای مثال، storage engineها، filesystemها و SSDها اغلب به‌جای overwrite کردن در همان محل، data را در محل تازه‌ای می‌نویسند؛ backupها هم عمداً immutable هستند تا از حذف یا corruption تصادفی محافظت شوند. در عمل، حذف بیشتر به معنای «دشوارترکردن بازیابی data» است تا «ناممکن‌کردن کامل بازیابی آن». بااین‌حال، گاهی باید برای حذف تلاش کرد؛ موضوعی که بعدتر در «Legislation and self-regulation» به آن می‌پردازیم.

## Key Terms

- `Dual Writes` — نوشتن مستقیم یک تغییر در چند system مستقل از داخل application؛ ممکن است race condition یا ناسازگاری بر اثر failure ایجاد کند.
- `System of Record` — منبع معتبر اصلی data که نسخه‌های دیگر، مانند cache و search index، از آن مشتق یا با آن همگام می‌شوند.
- `Change Data Capture (CDC)` — capture کردن changeهای database و ارائهٔ آن‌ها به‌صورت stream برای replicate کردن به systemهای دیگر.
- `Log Compaction` — حذف نسخه‌های قدیمی‌تر هر key از log و نگه‌داشتن جدیدترین value، با امکان بازسازی state فعلی.
- `Event Sourcing` — مدل‌سازی state application با ثبت eventهای immutable و append-only به‌جای اتکا به overwrite کردن state فعلی.
- `Command-Event Distinction` — تمایز میان درخواست قابل‌رد یا validationنشدهٔ کاربر (command) و fact پذیرفته‌شده و immutable (event).
- `Command Query Responsibility Segregation (CQRS)` — جداسازی model نوشتن از read modelها و ساخت viewهای گوناگون متناسب با queryها.
- `Read Model` — representation مشتق‌شده و بهینه‌شده برای پاسخ به الگوی مشخصی از read یا query.
- `Immutable Event` — record تغییرناپذیری از اتفاقی که رخ داده و برای اصلاح یا لغو آن event تازه‌ای ثبت می‌شود، نه اینکه event قبلی بازنویسی شود.
- `Excision` — حذف انتخابی data از تاریخچهٔ immutable با بازنویسی آن، به‌گونه‌ای که data هدف دیگر در دسترس نباشد.

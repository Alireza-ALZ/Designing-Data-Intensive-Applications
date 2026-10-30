# Glossary — واژه‌نامه

تعریف‌های این واژه‌نامه کوتاه و ساده‌اند و فقط ایدهٔ اصلی هر اصطلاح را می‌رسانند، نه تمام ظرافت‌های آن را. برای جزئیات بیشتر، به ارجاع‌های متن اصلی مراجعه کنید.

## asynchronous

منتظر کامل‌شدن کاری نماندن (برای مثال، ارسال داده از طریق network به node دیگر) و هیچ فرضی هم دربارهٔ مدت‌زمان انجام آن نداشتن. به بخش «Synchronous Versus Asynchronous Replication» در صفحهٔ ۱۵۳، «Synchronous Versus Asynchronous Networks» در صفحهٔ ۲۸۴ و «System Model and Reality» در صفحهٔ ۳۰۶ مراجعه کنید.

## atomic

۱. در زمینهٔ operationهای هم‌زمان: عملیاتی که چنان به نظر می‌رسد گویی در یک لحظهٔ واحد اثر کرده است؛ بنابراین process هم‌زمان دیگری هرگز نمی‌تواند آن را در وضعیتی «نیمه‌تمام» ببیند. همچنین ببینید `isolation`.

۲. در زمینهٔ transactionها: گروه‌بندی مجموعه‌ای از writeها که باید یا همگی commit شوند یا همگی rollback، حتی اگر fault رخ دهد. به بخش «Atomicity» در صفحهٔ ۲۲۳ و «Atomic Commit and Two-Phase Commit (2PC)» در صفحهٔ ۳۵۴ مراجعه کنید.

## backpressure

وادارکردن فرستندهٔ داده به کاهش سرعت، چون گیرنده نمی‌تواند با آن همگام شود. به آن `flow control` هم گفته می‌شود. به بخش «Messaging Systems» در صفحهٔ ۴۴۱ مراجعه کنید.

## batch process

محاسبه‌ای که مجموعه‌ای ثابت (و معمولاً بزرگ) از داده‌ها را به‌عنوان input می‌گیرد و داده‌های دیگری را به‌عنوان output تولید می‌کند، بی‌آنکه input را تغییر دهد. به فصل ۱۰ مراجعه کنید.

## bounded

دارای حد بالا یا اندازهٔ مشخص و معلوم. برای نمونه، در بحث network delay (به بخش «Timeouts and Unbounded Delays» در صفحهٔ ۲۸۱ مراجعه کنید) و datasetها (به مقدمهٔ فصل ۱۱ مراجعه کنید) به کار می‌رود.

## Byzantine fault

وضعیتی که در آن یک node به شکلی دلخواه نادرست رفتار می‌کند؛ برای مثال، پیام‌های متناقض یا مخرب برای nodeهای دیگر می‌فرستد. به بخش «Byzantine Faults» در صفحهٔ ۳۰۴ مراجعه کنید.

## cache

componentی که داده‌های اخیراً استفاده‌شده را نگه می‌دارد تا خواندن‌های بعدی همان داده سریع‌تر انجام شوند. cache معمولاً کامل نیست؛ بنابراین اگر داده‌ای در آن نباشد، باید آن داده را از storage زیربناییِ کندتری دریافت کرد که نسخهٔ کامل داده را در اختیار دارد.

## CAP theorem

نتیجه‌ای نظری که بسیار بدفهمیده شده و در عمل فایده‌ای ندارد. به بخش «The CAP theorem» در صفحهٔ ۳۳۶ مراجعه کنید.

## causality

وابستگی میان eventها که وقتی چیزی در یک system «پیش از» چیز دیگری رخ می‌دهد به وجود می‌آید. برای مثال، event متأخرتری که در پاسخ به event قبلی رخ می‌دهد، بر آن بنا شده است یا باید با درنظرگرفتن آن فهمیده شود. به بخش‌های «The “happens-before” relationship and concurrency» در صفحهٔ ۱۸۶ و «Ordering and Causality» در صفحهٔ ۳۳۹ مراجعه کنید.

## consensus

مسئله‌ای بنیادی در distributed computing: چند node باید دربارهٔ موضوعی به توافق برسند (برای مثال، اینکه کدام node در یک database cluster رهبر باشد). این مسئله بسیار دشوارتر از چیزی است که در نگاه اول به نظر می‌رسد. به بخش «Fault-Tolerant Consensus» در صفحهٔ ۳۶۴ مراجعه کنید.

## data warehouse

databaseای که داده‌های چند OLTP system متفاوت در آن ترکیب و آماده شده‌اند تا برای analytics استفاده شوند. به بخش «Data Warehousing» در صفحهٔ ۹۱ مراجعه کنید.

## declarative

توصیف ویژگی‌هایی که چیزی باید داشته باشد، بدون مشخص‌کردن گام‌های دقیق رسیدن به آن ویژگی‌ها. در زمینهٔ queryها، query optimizer یک query declarative را می‌گیرد و تصمیم می‌گیرد بهترین روش اجرای آن چیست. به بخش «Query Languages for Data» در صفحهٔ ۴۲ مراجعه کنید.

## denormalize

واردکردن مقداری redundancy یا تکرار در dataset نرمال‌شده، معمولاً به شکل cache یا index، برای سریع‌ترکردن readها. مقدار denormalized نوعی نتیجهٔ ازپیش‌محاسبه‌شدهٔ query است و به materialized view شباهت دارد. به بخش‌های «Single-Object and Multi-Object Operations» در صفحهٔ ۲۲۸ و «Deriving several views from the same event log» در صفحهٔ ۴۶۱ مراجعه کنید.

## derived data

datasetای که از داده‌های دیگری و با فرایندی تکرارپذیر ساخته می‌شود؛ در صورت نیاز می‌توان آن فرایند را دوباره اجرا کرد. معمولاً derived data برای سریع‌ترکردن نوع خاصی از دسترسی خواندنی لازم است. Indexها، cacheها و materialized viewها نمونه‌هایی از derived data هستند. به مقدمهٔ بخش سوم کتاب مراجعه کنید.

## deterministic

ویژگی تابعی که اگر input یکسانی به آن بدهید، همیشه output یکسانی تولید می‌کند. یعنی تابع نمی‌تواند به عدد تصادفی، زمان روز، ارتباط شبکه‌ای یا چیزهای پیش‌بینی‌ناپذیر دیگر وابسته باشد.

## distributed

اجراشونده روی چند node که از طریق network به هم متصل‌اند. ویژگی آن partial failure است: ممکن است بخشی از system از کار افتاده باشد، در حالی که بخش‌های دیگر همچنان کار می‌کنند؛ و اغلب software نمی‌تواند دقیقاً تشخیص دهد چه چیزی خراب شده است. به بخش «Faults and Partial Failures» در صفحهٔ ۲۷۴ مراجعه کنید.

## durable

ذخیره‌شده به شکلی که انتظار می‌رود داده حتی با رخ‌دادن faultهای گوناگون از بین نرود. به بخش «Durability» در صفحهٔ ۲۲۶ مراجعه کنید.

## ETL

مخفف Extract–Transform–Load (استخراج، تبدیل، بارگذاری). فرایند استخراج داده از یک source database، تبدیل آن به شکلی مناسب‌تر برای analytic queryها و بارگذاری آن در data warehouse یا batch processing system است. به بخش «Data Warehousing» در صفحهٔ ۹۱ مراجعه کنید.

## failover

در systemهایی که یک leader دارند، failover فرایند انتقال نقش رهبری از یک node به node دیگر است. به بخش «Handling Node Outages» در صفحهٔ ۱۵۶ مراجعه کنید.

## fault-tolerant

توانایی بازیابی خودکار هنگام بروز مشکل (برای مثال، crashکردن machine یا ازکارافتادن network link). به بخش «Reliability» در صفحهٔ ۶ مراجعه کنید.

## flow control

به `backpressure` مراجعه کنید.

## follower

replicaای که مستقیماً writeهای clientها را نمی‌پذیرد و فقط تغییرهای داده‌ای را پردازش می‌کند که از leader دریافت می‌کند. به آن secondary، slave، read replica یا hot standby هم گفته می‌شود. به بخش «Leaders and Followers» در صفحهٔ ۱۵۲ مراجعه کنید.

## full-text search

جست‌وجو در متن با keywordهای دلخواه، که اغلب قابلیت‌های دیگری مانند تطبیق واژه‌هایی با املای مشابه یا synonymها را هم دارد. full-text index نوعی secondary index است که از چنین queryهایی پشتیبانی می‌کند. به بخش «Full-text search and fuzzy indexes» در صفحهٔ ۸۸ مراجعه کنید.

## graph

ساختار داده‌ای متشکل از vertexها (چیزهایی که می‌توان به آن‌ها ارجاع داد و node یا entity هم نامیده می‌شوند) و edgeها (اتصال‌های یک vertex به vertex دیگر که relationship یا arc هم نامیده می‌شوند). به بخش «Graph-Like Data Models» در صفحهٔ ۴۹ مراجعه کنید.

## hash

تابعی که input را به عددی با ظاهری تصادفی تبدیل می‌کند. یک input مشخص همیشه همان عدد را به‌عنوان output برمی‌گرداند. احتمال زیادی دارد که دو input متفاوت، outputهای متفاوتی داشته باشند؛ بااین‌حال ممکن است دو input متفاوت یک output تولید کنند که به آن collision می‌گویند. به بخش «Partitioning by Hash of Key» در صفحهٔ ۲۰۳ مراجعه کنید.

## idempotent

ویژگی operationای که تکرار آن امن است: اگر بیش از یک بار اجرا شود، اثری یکسان با اجرای یک‌باره دارد. به بخش «Idempotence» در صفحهٔ ۴۷۸ مراجعه کنید.

## index

ساختار داده‌ای که جست‌وجوی کارآمد همهٔ recordهایی را ممکن می‌کند که در یک field مشخص، مقدار خاصی دارند. به بخش «Data Structures That Power Your Database» در صفحهٔ ۷۰ مراجعه کنید.

## isolation

در زمینهٔ transactionها، درجه‌ای که مشخص می‌کند transactionهای هم‌زمان چگونه می‌توانند بر یکدیگر اثر بگذارند. Serializable isolation قوی‌ترین guaranteeها را فراهم می‌کند، اما isolation levelهای ضعیف‌تر هم استفاده می‌شوند. به بخش «Isolation» در صفحهٔ ۲۲۵ مراجعه کنید.

## join

ترکیب recordهایی که وجه مشترکی دارند. این اصطلاح معمولاً وقتی به کار می‌رود که یک record به record دیگری ارجاع می‌دهد (از طریق foreign key، document reference یا edge در یک graph) و query باید record مورد اشاره را بازیابی کند. به بخش‌های «Many-to-One and Many-to-Many Relationships» در صفحهٔ ۳۳ و «Reduce-Side Joins and Grouping» در صفحهٔ ۴۰۳ مراجعه کنید.

## leader

وقتی داده یا service روی چند node تکثیر شده باشد، leader همان replica تعیین‌شده‌ای است که اجازه دارد تغییر ایجاد کند. ممکن است leader با یک protocol انتخاب شود یا administrator آن را به‌صورت دستی تعیین کند. به آن primary یا master هم گفته می‌شود. به بخش «Leaders and Followers» در صفحهٔ ۱۵۲ مراجعه کنید.

## linearizable

رفتاری گویی فقط یک نسخه از داده در system وجود دارد و آن نسخه با operationهای atomic به‌روزرسانی می‌شود. به بخش «Linearizability» در صفحهٔ ۳۲۴ مراجعه کنید.

## locality

بهینه‌سازی performance: قرار دادن چند قطعهٔ داده در یک محل، اگر معمولاً هم‌زمان به آن‌ها نیاز باشد. به بخش «Data locality for queries» در صفحهٔ ۴۱ مراجعه کنید.

## lock

سازوکاری برای اطمینان از اینکه فقط یک thread، node یا transaction می‌تواند به چیزی دسترسی داشته باشد؛ هر کس دیگری بخواهد به همان چیز دسترسی پیدا کند باید تا آزادشدن lock منتظر بماند. به بخش‌های «Two-Phase Locking (2PL)» در صفحهٔ ۲۵۷ و «The leader and the lock» در صفحهٔ ۳۰۱ مراجعه کنید.

## log

فایلی append-only برای ذخیره‌سازی داده. از write-ahead log برای مقاوم‌کردن storage engine در برابر crash استفاده می‌شود (به بخش «Making B-trees reliable» در صفحهٔ ۸۲ مراجعه کنید)؛ log-structured storage engine از logها به‌عنوان قالب اصلی ذخیره‌سازی استفاده می‌کند (به بخش «SSTables and LSM-Trees» در صفحهٔ ۷۶ مراجعه کنید)؛ replication log برای کپی‌کردن writeها از leader به followerها به کار می‌رود (به بخش «Leaders and Followers» در صفحهٔ ۱۵۲ مراجعه کنید)؛ و event log می‌تواند یک data stream را نمایش دهد (به بخش «Partitioned Logs» در صفحهٔ ۴۴۶ مراجعه کنید).

## materialize

اجرای eager یک محاسبه و نوشتن نتیجهٔ آن، در برابر محاسبهٔ نتیجه در زمان درخواست. به بخش‌های «Aggregation: Data Cubes and Materialized Views» در صفحهٔ ۱۰۱ و «Materialization of Intermediate State» در صفحهٔ ۴۱۹ مراجعه کنید.

## node

یک instance از software که روی computer اجرا می‌شود و برای انجام کاری از طریق network با nodeهای دیگر ارتباط برقرار می‌کند.

## normalized

ساختاریافته به‌گونه‌ای که redundancy یا تکرار نداشته باشد. در database نرمال‌شده، با تغییر بخشی از داده کافی است آن را فقط در یک محل تغییر دهید، نه در نسخه‌های متعدد و محل‌های گوناگون. به بخش «Many-to-One and Many-to-Many Relationships» در صفحهٔ ۳۳ مراجعه کنید.

## OLAP

مخفف Online analytic processing. الگوی دسترسی‌ای که با aggregateکردن (برای مثال، count، sum یا average) روی تعداد زیادی record مشخص می‌شود. به بخش «Transaction Processing or Analytics?» در صفحهٔ ۹۰ مراجعه کنید.

## OLTP

مخفف Online transaction processing. الگوی دسترسی‌ای که با queryهای سریعی مشخص می‌شود که تعداد کمی record را می‌خوانند یا می‌نویسند؛ این recordها معمولاً با key index شده‌اند. به بخش «Transaction Processing or Analytics?» در صفحهٔ ۹۰ مراجعه کنید.

## partitioning

تقسیم dataset یا computation بزرگی که برای یک machine بیش از حد بزرگ است، به بخش‌های کوچک‌تر و پخش‌کردن آن‌ها میان چند machine. به آن sharding هم گفته می‌شود. به فصل ۶ مراجعه کنید.

## percentile

روشی برای سنجش توزیع مقدارها با شمردن اینکه چند مقدار بالاتر یا پایین‌تر از یک آستانه‌اند. برای مثال، صدک ۹۵ response time در یک بازه، زمان tای است که ۹۵٪ requestها در آن بازه در کمتر از t کامل می‌شوند و ۵٪ بیشتر طول می‌کشند. به بخش «Describing Performance» در صفحهٔ ۱۳ مراجعه کنید.

## primary key

مقداری (معمولاً عدد یا رشته) که یک record را به‌طور یکتا مشخص می‌کند. در بسیاری از applicationها، هنگام ایجاد record، system primary key را تولید می‌کند (برای مثال، به‌شکل ترتیبی یا تصادفی) و معمولاً userها آن را تعیین نمی‌کنند. همچنین ببینید `secondary index`.

## quorum

حداقل تعداد nodeهایی که باید دربارهٔ یک operation رأی بدهند تا آن operation موفق تلقی شود. به بخش «Quorums for reading and writing» در صفحهٔ ۱۷۹ مراجعه کنید.

## rebalance

انتقال داده یا service از یک node به node دیگر برای توزیع منصفانهٔ load. به بخش «Rebalancing Partitions» در صفحهٔ ۲۰۹ مراجعه کنید.

## replication

نگه‌داشتن یک نسخه از دادهٔ یکسان روی چند node (replica)، تا اگر nodeای دسترس‌ناپذیر شد، داده همچنان قابل دسترسی بماند. به فصل ۵ مراجعه کنید.

## schema

توصیفی از ساختار داده، شامل fieldها و datatypeهای آن. انطباق داده با schema را می‌توان در نقاط مختلف چرخهٔ عمر داده بررسی کرد (به بخش «Schema flexibility in the document model» در صفحهٔ ۳۹ مراجعه کنید) و schema هم می‌تواند در گذر زمان تغییر کند (به فصل ۴ مراجعه کنید).

## secondary index

ساختار داده‌ای اضافی که در کنار storage اصلی داده نگهداری می‌شود و جست‌وجوی کارآمد recordهایی را ممکن می‌سازد که شرطی مشخص را برآورده می‌کنند. به بخش‌های «Other Indexing Structures» در صفحهٔ ۸۵ و «Partitioning and Secondary Indexes» در صفحهٔ ۲۰۶ مراجعه کنید.

## serializable

تضمینی که می‌گوید اگر چند transaction هم‌زمان اجرا شوند، رفتاری خواهند داشت که گویی یکی‌یکی و با ترتیبی serial اجرا شده‌اند. به بخش «Serializability» در صفحهٔ ۲۵۱ مراجعه کنید.

## shared-nothing

معماری‌ای که در آن nodeهای مستقل ــ هرکدام با CPU، memory و diskهای خود ــ از طریق network متعارف به هم متصل‌اند؛ در مقابل معماری‌های shared-memory یا shared-disk. به مقدمهٔ بخش دوم کتاب مراجعه کنید.

## skew

۱. نامتوازن‌بودن load میان partitionها، به‌طوری‌که بعضی partitionها request یا دادهٔ بسیار بیشتری از بقیه دارند. به این وضعیت hot spot هم گفته می‌شود. به بخش‌های «Skewed Workloads and Relieving Hot Spots» در صفحهٔ ۲۰۵ و «Handling skew» در صفحهٔ ۴۰۷ مراجعه کنید.

۲. ناهنجاری زمانی‌ای که باعث می‌شود eventها با ترتیبی غیرمنتظره و غیرترتیبی ظاهر شوند. بحث‌های مربوط به read skew در «Snapshot Isolation and Repeatable Read» در صفحهٔ ۲۳۷، write skew در «Write Skew and Phantoms» در صفحهٔ ۲۴۶ و clock skew در «Timestamps for ordering events» در صفحهٔ ۲۹۱ را ببینید.

## split brain

وضعیتی که در آن دو node هم‌زمان خود را leader می‌دانند و این امر ممکن است باعث نقض guaranteeهای system شود. به بخش‌های «Handling Node Outages» در صفحهٔ ۱۵۶ و «The Truth Is Defined by the Majority» در صفحهٔ ۳۰۰ مراجعه کنید.

## stored procedure

روشی برای کدگذاری منطق transaction به‌گونه‌ای که بتوان آن را به‌طور کامل روی database server اجرا کرد، بی‌آنکه در طول transaction لازم باشد با client رفت‌وبرگشت ارتباطی داشته باشد. به بخش «Actual Serial Execution» در صفحهٔ ۲۵۲ مراجعه کنید.

## stream process

محاسبه‌ای که پیوسته اجرا می‌شود، یک stream بی‌پایان از eventها را به‌عنوان input مصرف می‌کند و از آن outputای به دست می‌آورد. به فصل ۱۱ مراجعه کنید.

## synchronous

نقطهٔ مقابل asynchronous.

## system of record

systemی که نسخهٔ اصلی و مرجعِ داده‌ای را نگه می‌دارد؛ به آن source of truth هم گفته می‌شود. تغییرها ابتدا در این system نوشته می‌شوند و ممکن است datasetهای دیگر از system of record مشتق شوند. به مقدمهٔ بخش سوم کتاب مراجعه کنید.

## timeout

یکی از ساده‌ترین راه‌های تشخیص fault: مشاهدهٔ نرسیدن response در مدت‌زمانی مشخص. بااین‌حال، نمی‌توان فهمید timeout ناشی از مشکل در node راه‌دور بوده یا از ایرادی در network. به بخش «Timeouts and Unbounded Delays» در صفحهٔ ۲۸۱ مراجعه کنید.

## total order

روشی برای مقایسهٔ چیزها (برای مثال، timestampها) که همیشه می‌توان گفت کدام‌یک از دو چیز بزرگ‌تر و کدام کوچک‌تر است. ترتیبی که در آن بعضی چیزها قابل مقایسه نیستند (نمی‌توان گفت کدام بزرگ‌تر یا کوچک‌تر است) partial order نام دارد. به بخش «The causal order is not a total order» در صفحهٔ ۳۴۱ مراجعه کنید.

## transaction

گروه‌بندی چند read و write در یک واحد منطقی برای ساده‌کردن مدیریت خطاها و مسائل concurrency. به فصل ۷ مراجعه کنید.

## two-phase commit (2PC)

الگوریتمی برای اطمینان از اینکه چند database node یا همگی transaction را commit می‌کنند یا همگی آن را abort می‌کنند. به بخش «Atomic Commit and Two-Phase Commit (2PC)» در صفحهٔ ۳۵۴ مراجعه کنید.

## two-phase locking (2PL)

الگوریتمی برای دستیابی به serializable isolation. transaction در این الگوریتم روی همهٔ داده‌هایی که می‌خواند یا می‌نویسد lock می‌گیرد و آن lockها را تا پایان transaction نگه می‌دارد. به بخش «Two-Phase Locking (2PL)» در صفحهٔ ۲۵۷ مراجعه کنید.

## unbounded

فاقد هرگونه حد بالا یا اندازهٔ معلوم. نقطهٔ مقابل bounded.

# Chapter 6 — Partitioning

## Rebalancing Partitions

با گذشت زمان، شرایط database تغییر می‌کند:

- query throughput افزایش پیدا می‌کند و برای مدیریت load به CPUهای بیشتری نیاز دارید.
- اندازهٔ dataset افزایش پیدا می‌کند و برای ذخیرهٔ آن به disk و RAM بیشتری نیاز دارید.
- یک machine fail می‌شود و machineهای دیگر باید responsibilityهای آن machine را بر عهده بگیرند.

تمام این تغییرات باعث می‌شوند data و requestها از یک node به node دیگری منتقل شوند. به فرایند انتقال load از یک node در cluster به node دیگر **rebalancing** گفته می‌شود.

صرف‌نظر از اینکه از چه partitioning schemeای استفاده می‌کنید، معمولاً انتظار می‌رود rebalancing حداقل این requirementها را برآورده کند:

- پس از rebalancing، load—شامل data storage و read و write requestها—باید به‌صورت منصفانه میان nodeهای cluster تقسیم شده باشد.
- در زمان انجام rebalancing، database باید به پذیرش read و write ادامه دهد.
- برای سریع بودن rebalancing و کمینه کردن network و disk I/O load، نباید dataای بیشتر از مقدار ضروری میان nodeها جابه‌جا شود.

### Strategies for Rebalancing

چند روش متفاوت برای assign کردن partitionها به nodeها وجود دارد [23]. در ادامه هرکدام را به‌اختصار بررسی می‌کنیم.

#### How Not to Do It: `hash mod N`

وقتی دربارهٔ partitioning بر اساس hash یک key صحبت کردیم، گفتیم (شکل ۶-۳) بهتر است hashهای ممکن را به rangeهایی تقسیم کنیم و هر range را به یک partition اختصاص دهیم؛ برای مثال، key را در partition ۰ قرار دهیم اگر `0 ≤ hash(key) < b0` باشد و در partition ۱ قرار دهیم اگر `b0 ≤ hash(key) < b1` باشد، و به همین ترتیب.

شاید از خود پرسیده باشید چرا از `mod`—عملگر `%` در بسیاری از programming languageها—استفاده نمی‌کنیم. برای مثال، `hash(key) mod 10` عددی بین ۰ و ۹ برمی‌گرداند (اگر hash را به‌صورت یک عدد اعشاری بنویسیم، `hash mod 10` آخرین رقم آن خواهد بود). اگر ۱۰ node داشته باشیم که از ۰ تا ۹ شماره‌گذاری شده‌اند، این روش در نگاه اول راه ساده‌ای برای assign کردن هر key به یک node به نظر می‌رسد.

مشکل approach مربوط به `mod N` این است که اگر تعداد nodeها یعنی `N` تغییر کند، بیشتر keyها باید از یک node به node دیگری منتقل شوند. برای مثال، فرض کنید `hash(key) = 123456` باشد. اگر ابتدا ۱۰ node داشته باشید، این key روی node ۶ قرار می‌گیرد، چون `123456 mod 10 = 6`. وقتی تعداد nodeها به ۱۱ برسد، key باید به node ۳ منتقل شود، چون `123456 mod 11 = 3`؛ و وقتی تعداد nodeها به ۱۲ برسد، باید به node ۰ منتقل شود، چون `123456 mod 12 = 0`. چنین جابه‌جایی‌های مکرری، rebalancing را بیش از حد پرهزینه می‌کند.

ما به approachی نیاز داریم که data را بیش از مقدار لازم جابه‌جا نکند.

#### Fixed Number of Partitions

خوشبختانه یک راه‌حل نسبتاً ساده وجود دارد: تعداد partitionها را بسیار بیشتر از تعداد nodeها در نظر بگیرید و چند partition را به هر node assign کنید. برای مثال، databaseای که روی clusterای با ۱۰ node اجرا می‌شود، می‌تواند از ابتدا به ۱۰۰۰ partition تقسیم شود تا تقریباً ۱۰۰ partition به هر node اختصاص پیدا کند.

حالا اگر node جدیدی به cluster اضافه شود، آن node می‌تواند چند partition را از هرکدام از nodeهای موجود بگیرد تا partitionها دوباره به‌صورت منصفانه توزیع شوند. شکل ۶-۶ این فرایند را نشان می‌دهد. اگر nodeای از cluster حذف شود، همین فرایند در جهت معکوس انجام می‌شود.

فقط partitionهای کامل میان nodeها جابه‌جا می‌شوند. تعداد partitionها تغییر نمی‌کند و assignment مربوط به keyها به partitionها نیز ثابت می‌ماند. تنها چیزی که تغییر می‌کند assignment مربوط به partitionها به nodeهاست. این تغییر assignment فوری نیست—انتقال حجم بزرگی از data روی network زمان می‌برد—بنابراین تا زمانی که transfer در حال انجام است، برای read و writeهایی که رخ می‌دهند از assignment قدیمی partitionها استفاده می‌شود.

**شکل ۶-۶.** اضافه کردن یک node جدید به database clusterای که چند partition برای هر node دارد.

در اصل، حتی می‌توانید تفاوت hardwareهای cluster را نیز در نظر بگیرید: با assign کردن partitionهای بیشتر به nodeهای قدرتمندتر، می‌توانید آن nodeها را وادار کنید سهم بیشتری از load را بر عهده بگیرند.

این approach مربوط به rebalancing در Riak [15]، Elasticsearch [24]، Couchbase [10] و Voldemort [25] استفاده می‌شود.

در این configuration، تعداد partitionها معمولاً هنگام setup اولیهٔ database ثابت می‌شود و بعد از آن تغییر نمی‌کند. هرچند در اصل split و merge کردن partitionها ممکن است (به بخش بعدی مراجعه کنید)، fixed number of partitions از نظر operational ساده‌تر است؛ بنابراین بسیاری از databaseهای fixed-partition تصمیم می‌گیرند partition splitting را implement نکنند. در نتیجه، تعداد partitionهایی که در ابتدا configure می‌کنید، حداکثر تعداد nodeهایی را که می‌توانید داشته باشید تعیین می‌کند و باید آن را آن‌قدر بزرگ انتخاب کنید که رشد آینده را پوشش دهد. بااین‌حال، هر partition نیز management overhead دارد؛ بنابراین انتخاب تعداد بسیار زیاد partitionها نتیجهٔ معکوس خواهد داشت.

اگر اندازهٔ کل dataset بسیار متغیر باشد—برای مثال، ابتدا کوچک باشد اما به‌مرور بسیار بزرگ‌تر شود—انتخاب تعداد مناسب partition دشوار است. از آنجا که هر partition سهم ثابتی از کل data را در خود دارد، اندازهٔ هر partition متناسب با اندازهٔ کل data در cluster رشد می‌کند. اگر partitionها بسیار بزرگ باشند، rebalancing و recovery از node failureها پرهزینه می‌شوند؛ اما اگر partitionها بیش از حد کوچک باشند، overhead زیادی ایجاد می‌کنند. بهترین performance زمانی به دست می‌آید که اندازهٔ partitionها «به‌اندازهٔ مناسب» باشد؛ نه بیش از حد بزرگ و نه بیش از حد کوچک. اگر تعداد partitionها ثابت باشد اما اندازهٔ dataset تغییر کند، دستیابی به این وضعیت دشوار است.

#### Dynamic Partitioning

برای databaseهایی که از key-range partitioning استفاده می‌کنند (به بخش «Partitioning by Key Range» در صفحهٔ ۲۰۲ مراجعه کنید)، تعداد ثابتی partition با boundaryهای ثابت بسیار نامناسب است: اگر boundaryها را اشتباه انتخاب کنید، ممکن است تمام data در یک partition قرار بگیرد و تمام partitionهای دیگر خالی بمانند. تغییر دستی partition boundaryها نیز بسیار tedious خواهد بود.

به همین دلیل، databaseهای key-range-partitioned مانند HBase و RethinkDB partitionها را به‌صورت dynamic ایجاد می‌کنند. وقتی یک partition از اندازهٔ configureشده بزرگ‌تر شود—در HBase مقدار پیش‌فرض ۱۰ GB است—به دو partition split می‌شود تا تقریباً نیمی از data در هر طرف split قرار بگیرد [26]. برعکس، اگر data زیادی delete شود و یک partition از threshold مشخصی کوچک‌تر شود، می‌توان آن را با partition مجاور merge کرد. این فرایند شبیه اتفاقی است که در سطح بالایی یک B-tree رخ می‌دهد (به بخش «B-Trees» در صفحهٔ ۷۹ مراجعه کنید).

هر partition به یک node assign می‌شود و هر node می‌تواند چند partition را مدیریت کند؛ درست مانند حالت fixed number of partitions. پس از split شدن یک partition بزرگ، می‌توان یکی از دو نیمهٔ آن را برای متعادل کردن load به node دیگری transfer کرد. در HBase، transfer کردن fileهای partition از طریق HDFS، یعنی distributed filesystem زیرین، انجام می‌شود [3].

مزیت dynamic partitioning این است که تعداد partitionها خود را با total data volume تطبیق می‌دهد. اگر data کمی وجود داشته باشد، تعداد کمی partition کافی است و overhead پایین می‌ماند؛ اگر data بسیار زیاد باشد، اندازهٔ هر partition با یک maximum قابل‌تنظیم محدود می‌شود [23].

بااین‌حال، یک caveat وجود دارد: database خالی با یک partition شروع می‌شود، چون از قبل informationای دربارهٔ محل مناسب partition boundaryها نداریم. تا زمانی که dataset کوچک است—یعنی تا قبل از زمانی که اولین partition split شود—تمام writeها باید توسط یک node پردازش شوند و nodeهای دیگر idle بمانند. برای mitigate کردن این مشکل، HBase و MongoDB اجازه می‌دهند مجموعهٔ اولیه‌ای از partitionها روی database خالی configure شود؛ به این کار **pre-splitting** گفته می‌شود. در key-range partitioning، pre-splitting نیاز دارد که از قبل بدانید distribution مربوط به keyها چگونه خواهد بود [4, 26].

dynamic partitioning فقط برای dataیی که با key range partition شده مناسب نیست و می‌تواند برای dataیی که با hash partition شده نیز استفاده شود. MongoDB از version 2.4 به بعد هم key-range partitioning و هم hash partitioning را پشتیبانی می‌کند و در هر دو حالت partitionها را به‌صورت dynamic split می‌کند.

#### Partitioning Proportionally to Nodes

در dynamic partitioning، تعداد partitionها متناسب با اندازهٔ dataset است، چون فرایندهای split و merge اندازهٔ هر partition را بین minimum و maximum مشخصی نگه می‌دارند. از سوی دیگر، در fixed number of partitions، اندازهٔ هر partition متناسب با اندازهٔ dataset است. در هر دو حالت، تعداد partitionها مستقل از تعداد nodeهاست.

گزینهٔ سوم که Cassandra و Ketama از آن استفاده می‌کنند، این است که تعداد partitionها را متناسب با تعداد nodeها قرار دهیم؛ به بیان دیگر، تعداد ثابتی partition برای هر node داشته باشیم [23, 27, 28]. در این حالت، تا وقتی تعداد nodeها ثابت بماند، اندازهٔ هر partition متناسب با اندازهٔ dataset رشد می‌کند؛ اما وقتی تعداد nodeها را افزایش می‌دهید، partitionها دوباره کوچک‌تر می‌شوند. از آنجا که برای ذخیرهٔ data volume بزرگ‌تر معمولاً به nodeهای بیشتری نیاز است، این approach اندازهٔ هر partition را نسبتاً ثابت نگه می‌دارد.

وقتی node جدیدی به cluster ملحق می‌شود، به‌صورت random تعداد ثابتی از partitionهای موجود را برای split انتخاب می‌کند، سپس مالکیت یکی از نیمه‌های هر partition splitشده را به دست می‌گیرد و نیمهٔ دیگر را در جای خود باقی می‌گذارد. این randomization ممکن است splitهای ناعادلانه ایجاد کند، اما وقتی تعداد زیادی partition را با هم در نظر بگیریم—در Cassandra به‌صورت پیش‌فرض ۲۵۶ partition برای هر node—node جدید در نهایت سهم منصفانه‌ای از load nodeهای موجود را به دست می‌آورد. Cassandra 3.0 یک algorithm جایگزین برای rebalancing معرفی کرد که از splitهای ناعادلانه جلوگیری می‌کند [29].

انتخاب random partition boundaryها نیاز دارد که از hash-based partitioning استفاده شود؛ بنابراین boundaryها را می‌توان از range اعدادی انتخاب کرد که hash function تولید می‌کند. در واقع این approach بیش از هر چیز به تعریف اولیهٔ consistent hashing نزدیک است [7] (به بخش «Consistent Hashing» در صفحهٔ ۲۰۴ مراجعه کنید). hash functionهای جدیدتر می‌توانند effect مشابهی را با metadata overhead کمتر ایجاد کنند [8].

### Operations: Automatic or Manual Rebalancing

در مورد rebalancing یک سؤال مهم وجود دارد که تا اینجا از آن عبور کرده‌ایم: آیا rebalancing باید به‌صورت automatic انجام شود یا manual؟

میان rebalancing کاملاً automatic—که در آن system بدون interaction administrator و به‌صورت automatic تصمیم می‌گیرد چه زمانی partitionها را از یک node به node دیگر منتقل کند—و rebalancing کاملاً manual—که در آن administrator assignment مربوط به partitionها به nodeها را صراحتاً configure می‌کند و این assignment فقط با reconfigure کردن صریح administrator تغییر می‌کند—طیف گسترده‌ای از حالت‌ها وجود دارد. برای مثال، Couchbase، Riak و Voldemort یک partition assignment پیشنهادی را به‌صورت automatic تولید می‌کنند، اما پیش از effectگذار شدن آن، به administrator نیاز دارند تا assignment را commit کند.

rebalancing کاملاً automatic می‌تواند convenient باشد، چون مقدار operational work لازم برای maintenance معمول را کاهش می‌دهد. بااین‌حال، ممکن است رفتار آن unpredictable باشد. Rebalancing یک operation پرهزینه است، چون به reroute کردن requestها و انتقال حجم زیادی از data از یک node به node دیگر نیاز دارد. اگر این فرایند با دقت انجام نشود، ممکن است network یا nodeها را overloaded کند و هنگام انجام rebalancing performance مربوط به requestهای دیگر را کاهش دهد.

ترکیب چنین automationای با automatic failure detection می‌تواند خطرناک باشد. برای مثال، فرض کنید یک node overloaded شده و موقتاً در پاسخ دادن به requestها کند است. Nodeهای دیگر نتیجه می‌گیرند که node overloaded مرده است و به‌صورت automatic cluster را rebalance می‌کنند تا load را از آن node دور کنند. این کار load بیشتری روی node overloaded، nodeهای دیگر و network قرار می‌دهد؛ در نتیجه وضعیت بدتر می‌شود و حتی ممکن است یک cascading failure رخ دهد.

به همین دلیل، داشتن یک انسان در loop برای rebalancing می‌تواند تصمیم خوبی باشد. این روش از فرایند کاملاً automatic کندتر است، اما می‌تواند از operational surpriseها جلوگیری کند.

## Key Terms

- `Rebalancing` — جابه‌جا کردن load، data و requestها میان nodeها برای توزیع منصفانه‌تر partitionها.
- `Partition Assignment` — mappingای که مشخص می‌کند هر partition روی کدام node قرار دارد.
- `Partition Movement` — انتقال یک partition کامل یا بخشی از آن از یک node به node دیگر.
- `Hash Mod N` — روش تعیین node با محاسبهٔ `hash(key) mod N` که با تغییر تعداد nodeها جابه‌جایی گسترده ایجاد می‌کند.
- `Fixed Number of Partitions` — ایجاد تعداد ثابتی partition و assign کردن چند partition به هر node.
- `Static Partitioning` — partitioning با تعداد یا boundaryهای ثابت که در طول زمان خودکار تغییر نمی‌کنند.
- `Dynamic Partitioning` — split و merge کردن خودکار partitionها بر اساس اندازهٔ data یا thresholdهای مشخص.
- `Partition Splitting` — تقسیم یک partition بزرگ به دو یا چند partition کوچک‌تر.
- `Partition Merging` — ادغام partitionهای کوچک مجاور برای کاهش overhead.
- `Pre-Splitting` — ایجاد partitionهای اولیه روی database خالی، پیش از آنکه data به اندازهٔ لازم برای split شدن برسد.
- `Automatic Rebalancing` — تصمیم‌گیری و اجرای automatic دربارهٔ جابه‌جایی partitionها بدون تأیید مستقیم administrator.
- `Manual Rebalancing` — مدیریت و اعمال assignment مربوط به partitionها با تصمیم یا commit صریح administrator.
- `Operational Complexity` — دشواری مدیریت، پایش و کنترل behavior یک system در محیط production.
- `Cascading Failure` — failure زنجیره‌ای که در آن واکنش system به یک fault، load و failure بیشتری در بخش‌های دیگر ایجاد می‌کند.

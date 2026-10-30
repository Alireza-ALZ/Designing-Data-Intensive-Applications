# Chapter 10 — Batch Processing

## MapReduce and Distributed Filesystems

MapReduce تا حدی شبیه ابزارهای Unix است، با این تفاوت که روی هزاران machine توزیع می‌شود. مثل ابزارهای Unix، ابزاری نسبتاً ساده و brute-force است، اما به‌شکل شگفت‌آوری کارآمد عمل می‌کند. یک MapReduce job از نظر کلی شبیه یک Unix process است: یک یا چند ورودی می‌گیرد و یک یا چند خروجی تولید می‌کند.

معمولاً اجرای MapReduce job ورودی را تغییر نمی‌دهد و جز تولید خروجی side effect دیگری ندارد. فایل‌های خروجی یک‌بار و به‌شکل sequential نوشته می‌شوند؛ پس از نوشتن، بخش‌های قبلی file تغییر نمی‌کنند.

ابزارهای Unix از `stdin` و `stdout` به‌عنوان ورودی و خروجی استفاده می‌کنند، اما MapReduce jobها فایل‌ها را در distributed filesystem می‌خوانند و می‌نویسند. در پیاده‌سازی Hadoop، این filesystem با نام HDFS (Hadoop Distributed File System) شناخته می‌شود؛ پیاده‌سازی open sourceای از Google File System (GFS). Distributed filesystemهای دیگری مانند GlusterFS و Quantcast File System (QFS) نیز وجود دارند. Object storageهایی مانند Amazon S3، Azure Blob Storage و OpenStack Swift هم از جهات زیادی مشابه‌اند. در این chapter عمدتاً HDFS را مثال می‌زنیم، اما اصول برای distributed filesystemهای دیگر نیز کاربرد دارند.

HDFS بر اساس اصل **shared-nothing** ساخته شده و با معماری‌های **shared-disk** مانند Network Attached Storage (NAS) و Storage Area Network (SAN) فرق دارد. در shared-disk، storage appliance مرکزی—اغلب با hardware سفارشی و زیرساخت شبکه‌ای ویژه مثل Fibre Channel—کار را انجام می‌دهد. در shared-nothing به hardware خاصی نیاز نیست؛ computerهای معمولی که با شبکهٔ متعارف datacenter به هم وصل شده‌اند کافی‌اند.

روی هر machine یک daemon در حال اجراست که network serviceای در اختیار nodeهای دیگر می‌گذارد تا به fileهای ذخیره‌شده روی آن machine دسترسی داشته باشند. یک server مرکزی به‌نام **NameNode** نگه می‌دارد هر block از هر file روی کدام machine قرار دارد. از دید مفهومی، HDFS یک filesystem بزرگ می‌سازد که از فضای disk تمام machineهای دارای daemon استفاده می‌کند.

برای تحمل failureهای machine و disk، file blockها روی چند machine replicate می‌شوند. این کار می‌تواند با نگهداری چند copy از data انجام شود یا با **erasure coding** مانند Reed–Solomon که با overhead ذخیره‌سازی کمتر از replication کامل، data ازدست‌رفته را بازیابی می‌کند. این تکنیک‌ها به RAID شباهت دارند که روی چند disk متصل به یک machine redundancy ایجاد می‌کند؛ تفاوت این است که در distributed filesystem، دسترسی به file و replication از طریق شبکهٔ معمول datacenter و بدون hardware اختصاصی انجام می‌شود.

در زمان نگارش کتاب، بزرگ‌ترین deploymentهای HDFS روی ده‌ها هزار machine و با ظرفیت ذخیره‌سازی چندصد petabyte اجرا می‌شدند. این مقیاس به‌دلیل هزینهٔ بسیار کمتر storage و دسترسی به data روی commodity hardware و open source software، در مقایسه با storage appliance اختصاصی هم‌ظرفیت، امکان‌پذیر شده بود.

نکته‌ای دربارهٔ data locality: در HDFS می‌توان computing task را روی machineای زمان‌بندی کرد که copy فایل را نگه می‌دارد؛ object storeها معمولاً storage و computation را جدا نگه می‌دارند. اگر network bandwidth، bottleneck باشد، خواندن از disk محلی سریع‌تر است. بااین‌حال، در erasure coding مزیت locality از بین می‌رود، چون برای بازسازی file اصلی باید data چند machine با هم ترکیب شود.

### MapReduce Job Execution

MapReduce یک programming framework برای نوشتن code پردازش datasetهای بزرگ در distributed filesystemهایی مانند HDFS است. برای درک آن، مثال تحلیل log مربوط به web server را به یاد بیاورید. الگوی پردازش در MapReduce به آن مثال شباهت زیادی دارد:

1. مجموعه‌ای از input fileها را بخوانید و به record تقسیم کنید. در مثال log، هر خط یک record است و `\n` جداکنندهٔ recordهاست.
2. برای هر input record، تابع **mapper** را صدا بزنید تا key و value استخراج کند. در مثال قبلی، `awk '{print $7}'` نقش mapper را داشت: URL (`$7`) را به‌عنوان key استخراج می‌کرد و value را خالی می‌گذاشت.
3. تمام key-value pairها را بر اساس key مرتب کنید. در مثال log این کار را اولین `sort` انجام می‌داد.
4. تابع **reducer** را روی key-value pairهای مرتب‌شده اجرا کنید. اگر keyای چند بار تکرار شده باشد، sort مواردش را کنار هم قرار داده و reducer می‌تواند بدون نگه‌داشتن state زیاد در memory آن‌ها را ترکیب کند. در مثال قبل، `uniq -c` نقش reducer را داشت و تعداد recordهای مجاور با key یکسان را می‌شمرد.

این چهار مرحله را می‌توان با یک MapReduce job انجام داد. مرحله‌های ۲ (map) و ۴ (reduce) شامل code سفارشی پردازش data هستند. مرحلهٔ ۱، یعنی شکستن file به record، را parser قالب ورودی انجام می‌دهد. مرحلهٔ ۳، sort، در MapReduce ضمنی است؛ لازم نیست آن را خودتان بنویسید، چون خروجی mapper همیشه پیش از تحویل به reducer مرتب می‌شود.

برای ساخت MapReduce job باید دو callback function پیاده‌سازی کنید:

- **Mapper:** برای هر input record یک‌بار فراخوانی می‌شود و از آن key و value استخراج می‌کند. برای هر input می‌تواند هر تعداد key-value pair تولید کند، حتی هیچ‌کدام. بین recordهای ورودی state نگه نمی‌دارد؛ بنابراین هر record مستقل پردازش می‌شود.
- **Reducer:** framework تمام valueهای متعلق به یک key را جمع می‌کند و reducer را با iteratorای روی آن‌ها فراخوانی می‌کند. Reducer می‌تواند output record بسازد؛ مثلاً تعداد تکرار هر URL را بنویسد.

در مثال log، یک `sort` دوم URLها را بر اساس تعداد request مرتب می‌کرد. در MapReduce می‌توان مرحلهٔ sort دوم را با MapReduce job دیگری ساخت و خروجی job نخست را ورودی job دوم کرد. به این دید، mapper data را برای sort آماده می‌کند و reducer data مرتب‌شده را پردازش می‌کند.

#### اجرای distributed در MapReduce

تفاوت اصلی MapReduce با pipeline فرمان‌های Unix این است که MapReduce computation را در چندین machine موازی می‌کند، بدون آنکه لازم باشد خودتان parallelism را صریحاً مدیریت کنید. Mapper و reducer هرکدام فقط با یک record در هر لحظه کار می‌کنند و لازم نیست بدانند input از کجا می‌آید یا output به کجا می‌رود؛ framework پیچیدگی جابه‌جایی data میان machineها را برعهده می‌گیرد.

می‌توان در distributed computation از ابزارهای استاندارد Unix به‌عنوان mapper و reducer استفاده کرد، اما معمولاً این‌ها functionهایی در یک زبان برنامه‌نویسی متعارف‌اند. در Hadoop MapReduce، mapper و reducer هرکدام Java classای هستند که interface مشخصی را پیاده‌سازی می‌کنند. در MongoDB و CouchDB، mapper و reducer، JavaScript function هستند.

شکل 10-1 dataflow یک Hadoop MapReduce job را نشان می‌دهد. parallelization آن بر partitioning متکی است: ورودی job معمولاً directoryای در HDFS است و هر file یا file block داخل آن، partition مستقلی محسوب می‌شود که یک map task جداگانه می‌تواند پردازش کند. در شکل، سه map task با `m1`، `m2` و `m3` مشخص شده‌اند.

هر input file معمولاً صدها megabyte حجم دارد. MapReduce scheduler می‌کوشد mapper را روی یکی از machineهایی اجرا کند که replica ورودی را دارند، به‌شرط آنکه RAM و CPU کافی برای اجرای map task موجود باشد. این اصل **computation نزدیک به data** نام دارد: نیازی نیست input file از network کپی شود، در نتیجه هم network load کمتر و هم data locality بهتر می‌شود.

در بیشتر مواقع application codeای که باید در map task اجرا شود هنوز روی machine انتخاب‌شده نیست؛ پس framework ابتدا code را—مثلاً Java JAR fileها—به machineهای لازم کپی می‌کند. سپس map task را آغاز می‌کند، input file را می‌خواند و هر بار یک record را به mapper callback می‌دهد. خروجی mapper مجموعه‌ای از key-value pairهاست.

سمت reduce نیز partition می‌شود. تعداد map taskها با تعداد input file blockها تعیین می‌شود، اما تعداد reduce taskها را نویسندهٔ job تنظیم می‌کند و لازم نیست با تعداد map taskها برابر باشد. برای اینکه تمام pairهای دارای key یکسان به یک reducer برسند، framework hash مربوط به key را محاسبه می‌کند تا reduce task مقصد را تعیین کند.

key-value pairها باید مرتب شوند، اما dataset معمولاً برای sort با الگوریتم متعارف روی یک machine بیش از حد بزرگ است. بنابراین sort در چند مرحله انجام می‌شود. ابتدا هر map task خروجی‌اش را با توجه به hash keyها میان reducerها partition می‌کند. سپس هر partition را با روشی شبیه SSTable و LSM-Tree، در file مرتب‌شده‌ای روی disk محلی mapper می‌نویسد.

وقتی mapper خواندن input و نوشتن fileهای مرتب‌شدهٔ خروجی را تمام کرد، scheduler به reducerها اطلاع می‌دهد که می‌توانند fileها را از آن mapper دریافت کنند. هر reducer به mapperها وصل می‌شود و fileهای key-value pair مرتب‌شدهٔ partition خود را download می‌کند. فرایند partition کردن بر اساس reducer، sort و کپی partitionهای data از mapperها به reducerها **shuffle** نام دارد. این واژه کمی گمراه‌کننده است: برخلاف shuffle کردن کارت، در MapReduce randomnessای وجود ندارد.

Reduce task fileهای mapperها را می‌گیرد و با حفظ ترتیب sort آن‌ها را merge می‌کند. بنابراین اگر mapperهای مختلف recordهایی با key یکسان تولید کرده باشند، در ورودی mergeشدهٔ reducer کنار هم قرار می‌گیرند.

Reducer با یک key و iteratorای فراخوانی می‌شود که recordهای همان key را به‌تدریج scan می‌کند؛ ممکن است همهٔ آن‌ها در memory جا نشوند. Reducer می‌تواند با logic دلخواه این recordها را پردازش و هر تعداد output record تولید کند. این recordها در fileای روی distributed filesystem نوشته می‌شوند؛ معمولاً یک copy روی disk محلی machine اجراکنندهٔ reducer و replicaهایی روی machineهای دیگر.

#### MapReduce workflowها

با یک MapReduce job به‌تنهایی فقط می‌توان محدوده‌ای از مسئله‌ها را حل کرد. در مثال log، یک job می‌تواند تعداد page view هر URL را به‌دست آورد، اما نمی‌تواند محبوب‌ترین URLها را مرتب کند؛ این کار به مرحلهٔ sort دیگری نیاز دارد.

به همین دلیل MapReduce jobها اغلب در workflowهایی به هم زنجیر می‌شوند و خروجی یک job ورودی job بعدی می‌شود. Hadoop MapReduce پشتیبانی ویژه‌ای برای workflow ندارد؛ زنجیره‌سازی به‌طور ضمنی با نام directory انجام می‌شود: job اول خروجی را در directory مشخصی از HDFS می‌نویسد و job دوم همان directory را ورودی می‌گیرد. از دید framework این دو job مستقل‌اند.

پس زنجیرهٔ MapReduce jobها کمتر شبیه Unix pipeline است—که خروجی یک process را با buffer کوچک memory مستقیماً به ورودی process بعدی می‌دهد—و بیشتر به چند command شباهت دارد که هرکدام خروجی را در file موقت می‌نویسند تا command بعدی آن را بخواند. این طراحی مزایا و معایبی دارد که در بخش بعدی chapter بررسی می‌شود.

خروجی batch job فقط وقتی معتبر است که job با موفقیت تمام شود؛ MapReduce خروجی ناقص job شکست‌خورده را کنار می‌گذارد. پس یک job از workflow فقط وقتی می‌تواند آغاز شود که jobهای قبلیِ تولیدکنندهٔ directory ورودی‌اش موفق شده باشند. برای مدیریت این dependencyها، workflow schedulerهای مختلفی برای Hadoop ساخته شده‌اند، از جمله Oozie، Azkaban، Luigi، Airflow و Pinball.

این schedulerها برای مدیریت مجموعه‌های بزرگ batch job هم قابلیت‌هایی دارند. در ساخت recommendation systemها، workflowهای شامل ۵۰ تا ۱۰۰ MapReduce job رایج‌اند؛ در سازمان بزرگ نیز ممکن است teamهای مختلف jobهایی اجرا کنند که خروجی یکدیگر را می‌خوانند. ابزار مناسب برای مدیریت چنین dataflow پیچیده‌ای اهمیت دارد. ابزارهای سطح‌بالاتر Hadoop مانند Pig، Hive، Cascading، Crunch و FlumeJava نیز workflowهای چندمرحله‌ای MapReduce را می‌سازند و stageها را به‌طور خودکار به هم متصل می‌کنند.

### Reduce-Side Joins and Grouping

در فصل ۲ دربارهٔ joinها در چارچوب data model و query language صحبت کردیم، اما نحوهٔ پیاده‌سازی آن‌ها را بررسی نکردیم. حالا به آن بحث برمی‌گردیم.

در بسیاری از datasetها یک record با record دیگری association دارد: foreign key در relational model، reference به document دیگر در document model یا edge در graph model. هرگاه code لازم داشته باشد به هر دو سوی association دسترسی پیدا کند—هم record دارای reference و هم record موردارجاع—join لازم می‌شود. Denormalization نیاز به join را کم می‌کند، اما معمولاً آن را کاملاً حذف نمی‌کند.

اگر query database فقط تعداد کمی record را لازم داشته باشد، database معمولاً با index آن‌ها را سریع پیدا می‌کند. query دارای join ممکن است چند lookup روی index نیاز داشته باشد. اما MapReduce مفهوم index، دست‌کم به معنای متعارف database، ندارد. وقتی MapReduce job مجموعه‌ای از fileها را ورودی می‌گیرد، تمام محتوای آن‌ها را می‌خواند؛ database به این کار full table scan می‌گوید. اگر فقط تعداد کمی record لازم باشد، full scan در مقایسه با index lookup بسیار پرهزینه است. اما query تحلیلی معمولاً aggregate را روی تعداد زیادی record حساب می‌کند و در این حالت scan کامل input می‌تواند منطقی باشد، به‌خصوص وقتی پردازش میان چند machine موازی شود.

Join در batch processing یعنی پیدا کردن تمام occurrenceهای یک association در کل dataset. فرض کنید job data تمام userها را هم‌زمان پردازش می‌کند، نه اینکه data یک user مشخص را با index پیدا کند.

#### مثال: تحلیل eventهای فعالیت user

شکل 10-2 نمونه‌ای از join در batch job را نشان می‌دهد: در یک سمت logای از eventهای کارهای userهای واردشده به وب‌سایت قرار دارد (activity event یا clickstream data) و در سمت دیگر database userها. در یک star schema، log eventها fact table و database userها یکی از dimensionهاست.

کار analytics ممکن است بخواهد فعالیت user را با profile او مرتبط کند؛ مثلاً اگر profile سن یا تاریخ تولد را داشته باشد، system می‌تواند مشخص کند کدام صفحه برای کدام گروه سنی محبوب‌تر است. Activity event فقط user ID را دارد، نه تمام profile؛ افزودن اطلاعات profile به تک‌تک eventها احتمالاً اتلاف data است. بنابراین باید eventها را با database profileها join کرد.

ساده‌ترین راه این است که eventها را یکی‌یکی بخوانیم و برای هر user ID از database روی server راه دور query بزنیم. این روش ممکن است، اما performance ضعیفی دارد: throughput پردازش به زمان round-trip تا database server محدود می‌شود، کارایی cache محلی به توزیع data بستگی دارد و queryهای موازی زیاد می‌توانند database را از پا درآورند.

برای throughput خوب در batch processing، computation تا جای ممکن باید روی همان machine انجام شود. درخواست‌های random-access از طریق network برای هر record بسیار کند است. افزون بر آن، query کردن database راه دور batch job را nondeterministic می‌کند، چون data آن database ممکن است در حین job تغییر کند.

راه بهتر این است که copyای از database userها—مثلاً با ETL از database backup—تهیه و کنار log فعالیت userها در همان distributed filesystem قرار دهیم. به این ترتیب HDFS یک دسته file برای database userها و دسته‌ای دیگر برای activity recordها دارد و MapReduce می‌تواند recordهای مرتبط را کنار هم بیاورد و کارآمد پردازش کند.

#### Sort-merge join

Mapper از هر input record یک key و value استخراج می‌کند. در مثال شکل 10-2، key برابر user ID است: یک دسته mapper روی activity eventها اجرا می‌شوند و user ID را key و خود event را value می‌گیرند؛ دسته‌ای دیگر روی database userها اجرا شده و user ID را key و تاریخ تولد user را value می‌گیرند. شکل 10-3 این reduce-side sort-merge join را نشان می‌دهد؛ اگر input datasetها چند file داشته باشند، هرکدام را می‌توان با چند mapper موازی پردازش کرد.

MapReduce خروجی mapperها را بر اساس key partition و sort می‌کند؛ در نتیجه تمام activity eventها و record مربوط به user با user ID یکسان در ورودی reducer کنار هم قرار می‌گیرند. Framework حتی می‌تواند ترتیب را طوری تنظیم کند که reducer ابتدا record database user و سپس activity eventها را بر اساس timestamp ببیند. این روش **secondary sort** نام دارد.

Reducer برای هر user ID یک‌بار اجرا می‌شود. به‌لطف secondary sort، انتظار می‌رود value نخست، record تاریخ تولد user باشد. Reducer تاریخ تولد را در local variable نگه می‌دارد، سپس eventهای همان user ID را پیمایش می‌کند و جفت‌های URL مشاهده‌شده و سن بیننده را خروجی می‌دهد. MapReduce jobهای بعدی می‌توانند توزیع سن بینندگان هر URL را محاسبه و آن‌ها را بر اساس گروه سنی دسته‌بندی کنند.

چون reducer تمام recordهای یک user ID را در یک نوبت پردازش می‌کند، در هر لحظه فقط باید یک record user را در memory نگه دارد و هیچ درخواست networkای لازم ندارد. این algorithm **sort-merge join** نام دارد، چون خروجی mapper بر اساس key مرتب است و reducer فهرست‌های مرتب‌شدهٔ دو سوی join را با هم merge می‌کند.

#### آوردن data مرتبط به یک محل

در sort-merge join، mapperها و مرحلهٔ sort تضمین می‌کنند تمام data لازم برای join یک user ID در یک محل جمع شود: یک فراخوانی reducer. وقتی data مرتبط از پیش هم‌ردیف شده باشد، reducer می‌تواند code ساده و تک‌نخی‌ای باشد که recordها را با throughput بالا و memory کم پردازش می‌کند.

می‌توان معماری را این‌طور دید که mapperها برای reducerها «message می‌فرستند». وقتی mapper یک key-value pair تولید می‌کند، key مانند آدرس مقصدی عمل می‌کند که value باید به آن برسد. هرچند key رشته‌ای دلخواه است، نه network address واقعی مانند IP address و port، اما رفتارش شبیه آدرس است: تمام pairهای دارای key یکسان به یک مقصد، یعنی یک فراخوانی reducer، تحویل می‌شوند.

مدل برنامه‌نویسی MapReduce جنبه‌های فیزیکی ارتباط network—رساندن data به machine درست—را از logic application—پردازش data پس از رسیدن—جدا می‌کند. در databaseها درخواست دریافت data اغلب جایی در عمق application code اجرا می‌شود. چون MapReduce ارتباط network را مدیریت می‌کند، application code را از نگرانی دربارهٔ partial failure هم معاف می‌کند؛ مثلاً اگر node دیگری crash کند، framework task شکست‌خورده را بی‌آنکه logic application تغییر کند retry می‌کند.

#### GROUP BY

علاوه بر join، کاربرد رایج دیگر الگوی «آوردن data مرتبط به یک محل»، گروه‌بندی recordها بر اساس key است؛ مانند `GROUP BY` در SQL. تمام recordهای دارای key یکسان یک group تشکیل می‌دهند و معمولاً در مرحلهٔ بعد aggregation انجام می‌شود؛ برای مثال:

- شمردن تعداد recordهای هر group، مانند `COUNT(*)` برای شمردن page viewها.
- جمع‌زدن مقدارهای یک field خاص، مانند `SUM(fieldname)` در SQL.
- انتخاب top `k` record بر اساس تابع ranking.

ساده‌ترین راه گروه‌بندی در MapReduce این است که mapperها key-value pair را با key گروه‌بندی موردنظر تولید کنند. سپس partitioning و sorting تمام recordهای دارای key یکسان را به یک reducer می‌رساند. بنابراین پیاده‌سازی grouping و join در MapReduce شباهت زیادی دارد.

کاربرد دیگر grouping، کنار هم آوردن تمام activity eventهای یک user session برای یافتن توالی کارهای user است؛ فرایندی که **sessionization** نام دارد. چنین تحلیلی می‌تواند نشان دهد آیا userهایی که نسخهٔ جدید وب‌سایت را دیده‌اند بیشتر از userهای نسخهٔ قدیمی خرید می‌کنند (A/B testing)، یا ارزش یک فعالیت بازاریابی را محاسبه کند. اگر چند web server درخواست‌های user را پاسخ دهند، eventهای یک user احتمالاً در log fileهای serverهای مختلف پخش شده‌اند. می‌توان session cookie، user ID یا شناسه‌ای مشابه را grouping key قرار داد تا eventهای هر user به یک محل بیایند و eventهای userهای مختلف بین partitionها پخش شوند.

#### مدیریت skew

الگوی «آوردن تمام recordهای دارای key یکسان به یک محل» وقتی می‌شکند که data مربوط به یک key خاص بسیار زیاد باشد. مثلاً در social network، بیشتر userها چندصد connection دارند اما تعداد کمی celebrity ممکن است میلیون‌ها follower داشته باشند. به چنین recordهای به‌طور نامتناسب فعالی **linchpin object** یا **hot key** می‌گویند.

جمع‌آوری همهٔ فعالیت‌های مربوط به یک celebrity—مثلاً پاسخ‌ها به post او—در یک reducer می‌تواند skew یا hot spot جدی ایجاد کند؛ یعنی یک reducer به‌مراتب بیشتر از بقیه record پردازش کند. MapReduce job فقط وقتی تمام می‌شود که تمام mapperها و reducerها تمام شده باشند؛ پس jobهای بعدی باید تا پایان کندترین reducer صبر کنند.

اگر join input، hot key داشته باشد، چند روش برای جبران وجود دارد. روش skewed join در Pig ابتدا job نمونه‌گیری اجرا می‌کند تا keyهای داغ را پیدا کند. هنگام join واقعی، mapperها recordهای مربوط به hot key را به‌صورت تصادفی میان چند reducer می‌فرستند؛ برخلاف MapReduce عادی که بر اساس hash key، reducer را deterministic انتخاب می‌کند. در طرف دیگر join، recordهای مربوط به hot key باید برای تمام reducerهای مسئول آن key replicate شوند.

این روش کار hot key را بین چند reducer پخش و parallelism را بهتر می‌کند، اما هزینه‌اش replicate کردن input دیگر join به چند reducer است. روش sharded join در Crunch مشابه است، با این تفاوت که hot keyها باید دستی مشخص شوند و job نمونه‌گیری ندارد. این تکنیک به استفاده از randomization برای کاهش hot spot در partitioned database شباهت دارد.

Hive برای skewed join راه دیگری دارد: hot keyها باید صریحاً در table metadata ثبت شوند و recordهای آن‌ها جدا از بقیه در fileهای جداگانه نگهداری شوند. هنگام join روی آن table، برای hot keyها از map-side join استفاده می‌شود.

برای aggregation روی recordهای یک hot key می‌توان grouping را در دو مرحله انجام داد. MapReduce job اول recordها را به reducerهای تصادفی می‌فرستد؛ هر reducer بخشی از recordهای hot key را group می‌کند و value تجمیع‌شده و فشرده‌تری تولید می‌کند. MapReduce job دوم valueهای خروجی همهٔ reducerهای مرحلهٔ اول را به یک value برای هر key تبدیل می‌کند.

### Map-Side Joins

در joinهای بخش قبل، منطق اصلی join در reducer اجرا می‌شد؛ به همین دلیل آن‌ها را **reduce-side join** می‌نامند. Mapperها input را آماده می‌کنند: از هر record key و value استخراج می‌کنند، pairها را به partition reducer مناسب می‌فرستند و بر اساس key مرتب می‌کنند.

مزیت این روش آن است که فرضی دربارهٔ ویژگی‌های input لازم ندارد؛ mapperها هر ساختار و شکلی را برای join آماده می‌کنند. عیبش هزینهٔ sort، کپی data به reducerها و merge کردن ورودی reducerهاست. بسته به bufferهای memory موجود، ممکن است data هنگام عبور از stageهای MapReduce چند بار روی disk نوشته شود.

اگر بتوان دربارهٔ input فرض‌هایی داشت، **map-side join** می‌تواند سریع‌تر باشد. این روش MapReduce job ساده‌تری دارد که reducer و sort ندارد. هر mapper یک input file block را از distributed filesystem می‌خواند و یک output file در filesystem می‌نویسد؛ همین.

#### Broadcast hash join

ساده‌ترین map-side join زمانی است که dataset بزرگی با dataset کوچکی join شود و dataset کوچک به‌اندازه‌ای باشد که در memory هر mapper جا بگیرد.

در مثال شکل 10-2 فرض کنید database userها در memory جا می‌شود. هر mapper هنگام شروع، database userها را از distributed filesystem در hash table داخل memory می‌خواند. سپس activity eventها را scan می‌کند و user ID هر event را در hash table lookup می‌کند.

برای input بزرگ هنوز چند map task وجود دارد؛ برای هر file block از input بزرگ یک task اجرا می‌شود. در مثال، activity eventها input بزرگ‌اند و هر mapper کل input کوچک را در memory بارگذاری می‌کند.

این algorithm ساده اما مؤثر **broadcast hash join** نام دارد. «broadcast» به این اشاره دارد که mapper هر partition از input بزرگ، تمام input کوچک را می‌خواند؛ در نتیجه input کوچک عملاً به تمام partitionها broadcast شده است. «hash» نیز به استفاده از hash table اشاره دارد. Pig با نام replicated join، Hive با نام MapJoin، و نیز Cascading و Crunch از این روش پشتیبانی می‌کنند. این روش در data warehouse query engineهایی مانند Impala هم استفاده می‌شود.

به‌جای نگه‌داشتن input کوچک در hash table داخل memory، می‌توان آن را در read-only index روی disk محلی ذخیره کرد. بخش‌های پرکاربرد index در page cache سیستم‌عامل می‌مانند؛ بنابراین lookup با دسترسی تصادفی تقریباً به‌سرعت hash table داخل memory انجام می‌شود، بدون اینکه کل dataset در memory جا بگیرد.

این مثال فرض می‌کند در hash table برای هر key دقیقاً یک entry وجود دارد؛ در database userها احتمالاً درست است، چون user ID هر user را یکتا مشخص می‌کند. در حالت کلی ممکن است چند entry یک key داشته باشند و join باید تمام matchها را خروجی دهد.

#### Partitioned hash join

اگر inputهای map-side join به یک شکل partition شده باشند، می‌توان hash join را مستقل روی هر partition اجرا کرد. در مثال شکل 10-2 می‌توان activity eventها و database userها را بر اساس رقم یکان user ID partition کرد؛ در این صورت هر طرف ده partition دارد. مثلاً mapper شمارهٔ ۳ ابتدا userهایی را که IDشان به ۳ ختم می‌شود در hash table می‌ریزد، سپس activity eventهای همان userها را scan می‌کند.

اگر partitioning درست انجام شده باشد، تمام recordهایی که باید join شوند در partitionهایی با شمارهٔ یکسان قرار دارند؛ پس هر mapper کافی است از هر input فقط یک partition بخواند و data کمتری در hash table بارگذاری کند.

این روش فقط وقتی کار می‌کند که هر دو input تعداد partition یکسان داشته باشند و recordها بر اساس key یکسان و hash function یکسان به partitionها تخصیص داده شده باشند. اگر inputها خروجی MapReduce jobهای قبلی باشند که همین grouping را انجام داده‌اند، فرض معقولی است. Hive به partitioned hash join، **bucketed map join** می‌گوید.

#### Map-side merge join

نوع دیگری از map-side join زمانی ممکن است که inputها علاوه بر partition شدن به یک شکل، بر اساس key یکسانی هم sort شده باشند. در این صورت مهم نیست inputها در memory جا می‌شوند یا نه؛ mapper می‌تواند همان mergeای را انجام دهد که معمولاً reducer انجام می‌دهد: هر دو file را به‌شکل incremental و بر اساس key صعودی بخواند و recordهای دارای key یکسان را match کند.

اگر map-side merge join ممکن باشد، احتمالاً MapReduce jobهای قبلی input را به همین شکل partition و sort کرده‌اند. در اصل این join می‌توانست در reduce stage همان job قبلی انجام شود؛ بااین‌حال اجرای merge join در map-only job جداگانه ممکن است مناسب باشد، مثلاً اگر datasetهای partitionشده و مرتب برای کارهای دیگری هم لازم باشند.

#### Map-side join در MapReduce workflowها

وقتی jobهای بعدی خروجی MapReduce join را مصرف می‌کنند، انتخاب map-side یا reduce-side join بر ساختار خروجی اثر دارد. خروجی reduce-side join بر اساس join key partition و sort شده است؛ خروجی map-side join همان partitioning و sorting input بزرگ را حفظ می‌کند، چون برای هر file block از input بزرگ یک map task اجرا می‌شود؛ چه join از نوع partitioned باشد چه broadcast.

Map-side join فرض‌های بیشتری دربارهٔ اندازه، sort و partitioning inputها دارد. برای بهینه‌سازی join باید layout فیزیکی datasetها در distributed filesystem را بدانیم؛ دانستن encoding و directory کافی نیست، تعداد partitionها و keyهایی که data بر اساس آن‌ها partition و sort شده نیز مهم است. در Hadoop ecosystem، HCatalog و Hive metastore اغلب این metadata را نگه می‌دارند.

### The Output of Batch Workflows

دربارهٔ algorithmهای اجرای MapReduce workflow صحبت کردیم، اما یک سؤال مهم باقی مانده است: پس از پایان پردازش، نتیجه چیست و اصلاً چرا این jobها را اجرا می‌کنیم؟

در database queryها میان هدف‌های transaction processing (OLTP) و analytics تمایز گذاشتیم. OLTP query معمولاً با index، تعداد کمی record را بر اساس key پیدا می‌کند تا مثلاً در صفحهٔ وب به user نشان دهد. اما analytic query اغلب تعداد زیادی record را scan می‌کند، grouping و aggregation انجام می‌دهد و معمولاً گزارشی تولید می‌کند: graph تغییر یک metric در طول زمان، ده مورد برتر بر اساس ranking، یا شکستن یک quantity به چند subcategory. مصرف‌کنندهٔ گزارش معمولاً analyst یا managerای است که باید تصمیم تجاری بگیرد.

Batch processing نه transaction processing است و نه دقیقاً analytics. از این جهت به analytics نزدیک است که معمولاً بخش بزرگی از input dataset را scan می‌کند؛ اما MapReduce workflow با SQL query تحلیلی یکسان نیست. خروجی batch process اغلب report نیست، بلکه ساختار دیگری از data است.

#### ساخت search index

Google در ابتدا از MapReduce برای ساخت indexهای search engine استفاده می‌کرد؛ فرایندی شامل workflowای از ۵ تا ۱۰ MapReduce job. Google بعداً برای این کار از MapReduce فاصله گرفت، اما نگاه به MapReduce از زاویهٔ ساخت search index به درک آن کمک می‌کند. حتی امروز Hadoop MapReduce راه مناسبی برای ساخت indexهای Lucene/Solr است.

یک full-text search index مانند Lucene را به‌طور ساده می‌توان fileای دانست که **term dictionary** نام دارد: با lookup یک keyword در آن می‌توان فهرست document IDهای دارای آن keyword را پیدا کرد؛ این فهرست **postings list** نامیده می‌شود. Search index واقعی برای ranking بر اساس relevance، اصلاح غلط املایی، حل synonymها و موارد دیگر به data اضافی نیاز دارد، اما اصل کار همین است.

برای full-text search روی مجموعهٔ ثابتی از documentها، batch process روش مؤثری برای ساخت index است: mapperها مجموعهٔ documentها را partition می‌کنند، هر reducer index مربوط به partition خودش را می‌سازد و fileهای index در distributed filesystem نوشته می‌شوند. ساخت indexهای document-partitioned به‌خوبی parallel می‌شود.

چون query کردن search index با keyword، read-only است، fileهای index پس از ساخت immutable می‌مانند. اگر مجموعهٔ documentهای indexشده تغییر کند، یک راه این است که کل workflow را به‌طور دوره‌ای دوباره اجرا کنیم و پس از اتمام، fileهای index جدید را یک‌جا جایگزین قبلی‌ها کنیم. اگر فقط تعداد کمی document تغییر کرده باشند، این کار از نظر computation پرهزینه است؛ اما ساده است: document وارد می‌شود و index تحویل داده می‌شود.

راه دیگر ساخت index به‌شکل incremental است. همان‌طور که در فصل ۳ دیدیم، Lucene هنگام افزودن، حذف یا update document، segment fileهای تازه می‌نویسد و آن‌ها را در پس‌زمینه به‌صورت asynchronous merge و compact می‌کند. در فصل ۱۱ به چنین پردازش incremental بازمی‌گردیم.

#### Key-value store به‌عنوان خروجی batch process

Search index فقط یکی از خروجی‌های ممکن batch workflow است. کاربرد رایج دیگر ساخت machine learning systemهایی مانند classifierها (spam filter، anomaly detection و image recognition) و recommendation systemهاست؛ مثلاً پیشنهاد افراد آشنا، محصول موردعلاقه یا search مرتبط.

خروجی این batch jobها اغلب نوعی database است: مثلاً databaseای که با user ID، دوست‌های پیشنهادی آن user را برمی‌گرداند یا با product ID فهرست محصول‌های مرتبط را می‌دهد. Web applicationای که requestهای user را پردازش می‌کند باید بتواند این database را query کند و معمولاً جدا از Hadoop infrastructure است. پس خروجی batch process چطور به database قابل‌دسترسی برای web application تبدیل می‌شود؟

راه بدیهی این است که mapper یا reducer مستقیماً از client library database استفاده کند و recordها را یکی‌یکی به database server بنویسد. این کار ممکن است—اگر firewall اجازهٔ دسترسی Hadoop به production database را بدهد—اما چند مشکل دارد:

- درخواست network برای هر record، چندین مرتبه کندتر از throughput معمول batch task است. حتی اگر client library از batch پشتیبانی کند، performance احتمالاً ضعیف خواهد بود.
- MapReduce jobها taskهای موازی زیادی اجرا می‌کنند. اگر همهٔ mapperها یا reducerها با نرخ بالای batch process هم‌زمان در یک output database بنویسند، ممکن است database اشباع شود و performance queryهایش افت کند؛ این وضعیت می‌تواند برای بخش‌های دیگر system هم مشکل عملیاتی بسازد.
- MapReduce برای job output تضمین all-or-nothing فراهم می‌کند: اگر job موفق شود، نتیجه معادل اجرای هر task دقیقاً یک‌بار است، حتی اگر بعضی taskها شکست خورده و retry شده باشند؛ اگر کل job شکست بخورد، outputی تولید نمی‌شود. اما نوشتن در system بیرونی side effectای قابل‌مشاهده ایجاد می‌کند که نمی‌توان آن را پنهان کرد. در نتیجه باید نگران قابل‌مشاهده‌شدن نتیجهٔ job نیمه‌کاره و پیچیدگی task attemptها و speculative execution در Hadoop باشید.

راه بهتر این است که batch job یک database جدید بسازد و آن را مثل search index در directory خروجی job روی distributed filesystem به‌شکل file بنویسد. پس از نوشتن، fileها immutable هستند و می‌توان آن‌ها را bulk در serverهایی بارگذاری کرد که queryهای read-only را پاسخ می‌دهند. Voldemort، Terrapin، ElephantDB و bulk loading در HBase از ساخت database fileها در MapReduce پشتیبانی می‌کنند.

ساخت این fileها کاربرد خوبی برای MapReduce است: استخراج key با mapper و sort کردن بر اساس key، بخش بزرگی از کار ساخت index را تشکیل می‌دهد. چون بیشتر این key-value storeها read-only هستند—file را batch job یک‌بار می‌نویسد و پس از آن تغییرناپذیر است—ساختار data ساده‌تر است و مثلاً به WAL نیاز ندارد.

هنگام بارگذاری data در Voldemort، server تا زمانی که fileهای جدید از distributed filesystem به disk محلی کپی شوند، همچنان requestها را با fileهای قدیمی پاسخ می‌دهد. پس از پایان کپی، server به‌شکل atomic به fileهای جدید سوییچ می‌کند. اگر مشکلی پیش بیاید، چون fileهای قبلی هنوز موجود و immutable هستند، برگشت به آن‌ها آسان است.

#### فلسفهٔ خروجی batch process

Unix Philosophy که پیش‌تر بررسی کردیم، آزمایش را با شفاف‌کردن dataflow تشویق می‌کند: program input را می‌خواند و output را می‌نویسد؛ input دست‌نخورده می‌ماند، output قبلی کاملاً با output جدید جایگزین می‌شود و side effect دیگری رخ نمی‌دهد. بنابراین می‌توان command را بارها با تغییر یا debug اجرا کرد، بدون آنکه state system به‌هم بخورد.

مدیریت خروجی MapReduce هم از همین فلسفه پیروی می‌کند. immutable نگه‌داشتن input و پرهیز از side effect—مانند نوشتن مستقیم در database بیرونی—هم performance را بهتر می‌کند و هم نگهداری job را آسان‌تر:

- اگر bugای باعث output اشتباه یا خراب شود، می‌توان به نسخهٔ قبلی code برگشت و job را دوباره اجرا کرد؛ یا ساده‌تر، output قبلی را در directory دیگری نگه داشت و به آن سوییچ کرد. Database دارای read-write transaction این ویژگی را ندارد: rollback کردن codeای که data بد نوشته، data database را اصلاح نمی‌کند. توانایی بازیابی از code معیوب را **human fault tolerance** نامیده‌اند.
- همین آسان‌بودن rollback به توسعهٔ سریع‌تر feature کمک می‌کند، چون اشتباه‌ها آسیب برگشت‌ناپذیر ایجاد نمی‌کنند. کمینه‌کردن irreversibility برای Agile software development هم مفید است.
- اگر map یا reduce task شکست بخورد، framework آن را خودکار دوباره زمان‌بندی و با همان input اجرا می‌کند. اگر bug code علت باشد، task باز هم crash می‌کند و پس از چند تلاش job شکست می‌خورد؛ اگر مشکل موقت باشد، system آن را تحمل می‌کند. این retry خودکار فقط به‌دلیل immutable بودن inputها و کنارگذاشتن output task شکست‌خورده امن است.
- یک مجموعه file می‌تواند input چند job مختلف باشد؛ از جمله monitoring jobهایی که metric حساب می‌کنند و بررسی می‌کنند output ویژگی موردانتظار را دارد یا نه، مثلاً با مقایسهٔ آن با اجرای قبلی و اندازه‌گیری اختلاف‌ها.
- MapReduce مانند ابزارهای Unix، logic را از wiring—تنظیم directoryهای input و output—جدا می‌کند. این separation of concerns امکان reuse code را می‌دهد: یک team jobی می‌سازد که یک کار را خوب انجام دهد و teamهای دیگر زمان و محل اجرای آن را انتخاب می‌کنند.

اصول طراحی موفق Unix در Hadoop هم خوب عمل می‌کنند، اما این دو تفاوت‌هایی دارند. ابزارهای Unix اغلب file متنی بدون type فرض می‌کنند و باید parsing زیادی انجام دهند؛ مثل `awk '{print $7}'` در مثال ابتدای chapter. در Hadoop، formatهای ساخت‌یافته‌ای مانند Avro و Parquet بعضی تبدیل‌های نحوی کم‌ارزش را حذف می‌کنند: آن‌ها encoding کارآمد مبتنی بر schema دارند و evolution schema را هم ممکن می‌سازند.

### Comparing Hadoop to Distributed Databases

همان‌طور که دیدیم، Hadoop تا حدی نسخهٔ distributed Unix است: HDFS filesystem آن است و MapReduce پیاده‌سازی خاصی از Unix process است که همیشه بین map و reduce، ابزار sort را اجرا می‌کند. می‌توان انواع join و grouping را بر پایهٔ این primitiveها پیاده‌سازی کرد.

وقتی مقالهٔ MapReduce منتشر شد، ایده در بعضی جنبه‌ها تازه نبود. تمام الگوریتم‌های processing و parallel join که بررسی کردیم بیش از یک دهه پیش‌تر در massively parallel processing (MPP) databaseها پیاده‌سازی شده بودند؛ از پیشگامان این حوزه Gamma database machine، Teradata و Tandem NonStop SQL بودند.

تفاوت اصلی این است که MPP databaseها روی اجرای موازی analytic SQL query در cluster متمرکزند، درحالی‌که ترکیب MapReduce و distributed filesystem چیزی شبیه operating system همه‌منظوره‌تری است که program دلخواه را اجرا می‌کند.

#### تنوع در storage

Database شما را ملزم می‌کند data را بر اساس model مشخصی—مثلاً relational یا document—ساختاربندی کنید. اما fileهای distributed filesystem فقط دنباله‌ای از byte هستند و می‌توان آن‌ها را با هر data model و encoding نوشت: recordهای database، متن، تصویر، ویدئو، sensor reading، sparse matrix، feature vector، genome sequence یا هر نوع data دیگر.

به‌بیان صریح، Hadoop امکان ریختن بی‌تمایز data در HDFS و تصمیم‌گیری دربارهٔ پردازش آن در زمانی بعد را فراهم کرد. در مقابل، MPP database معمولاً پیش از واردکردن data به format ذخیره‌سازی اختصاصی خود، به modeling دقیق data و query patternها نیاز دارد.

از دید purist شاید modeling و import دقیق مطلوب باشد، چون userهای database با data باکیفیت‌تری کار می‌کنند. اما در عمل، سریع در دسترس قراردادن data—حتی اگر raw، نامتعارف و سخت‌استفاده باشد—اغلب از تعیین model ایده‌آل از همان ابتدا ارزشمندتر است.

این ایده به data warehouse شباهت دارد: گردآوردن data از بخش‌های مختلف سازمان در یک محل ارزشمند است، چون join میان datasetهایی را ممکن می‌کند که قبلاً جدا بودند. Schema design دقیق در MPP database، جمع‌آوری متمرکز data را کند می‌کند؛ جمع‌آوری raw data و موکول‌کردن schema design به بعد، سرعت این کار را بالا می‌برد. این روش گاهی **data lake** یا **enterprise data hub** نامیده می‌شود.

ریختن data بدون مدل‌سازی اولیه، مسئولیت تفسیر data را جابه‌جا می‌کند: به‌جای آنکه producer را مجبور کنیم data را به قالب استاندارد ببرد، consumer باید آن را تفسیر کند؛ این رویکرد **schema-on-read** است. اگر producer و consumer teamهای متفاوتی با اولویت‌های مختلف باشند، این می‌تواند مزیت باشد. شاید حتی یک data model ایده‌آل وجود نداشته باشد و برای هدف‌های مختلف viewهای متفاوت لازم باشد. نگهداری raw data امکان transformationهای مختلف را فراهم می‌کند. این رویکرد را **sushi principle** نامیده‌اند: «raw data بهتر است».

بنابراین Hadoop اغلب برای اجرای ETL استفاده شده است: data سیستم‌های transaction processing به‌شکل raw در distributed filesystem dump می‌شود؛ سپس MapReduce jobها آن را پاک‌سازی، به شکل relational تبدیل و برای analytics به MPP data warehouse وارد می‌کنند. Data modeling همچنان انجام می‌شود، اما در مرحله‌ای جدا از data collection. پشتیبانی distributed filesystem از formatهای گوناگون، این جداسازی را ممکن می‌کند.

#### تنوع در processing modelها

MPP databaseها softwareهای یکپارچه و monolithic هستند که layout ذخیره‌سازی روی disk، query planning، scheduling و execution را با هم مدیریت می‌کنند. چون این componentها برای نیازهای database مشخص بهینه می‌شوند، system می‌تواند روی queryهای هدف performance بالایی داشته باشد. SQL هم queryهای expressive با semantics روشن می‌دهد و نیاز به code را کم می‌کند؛ بنابراین ابزارهای گرافیکی business analystها مانند Tableau می‌توانند از آن استفاده کنند.

اما همهٔ processingها را نمی‌توان به‌طور مناسب با SQL بیان کرد. برای ساخت machine learning و recommendation system، full-text search index با relevance ranking یا تحلیل تصویر، معمولاً data processing عمومی‌تری لازم است. این کارها به application خاص وابسته‌اند—مانند feature engineering برای machine learning، مدل زبان برای machine translation یا تابع برآورد ریسک برای شناسایی تقلب—و ناگزیر به نوشتن code نیاز دارند، نه فقط query.

MapReduce به engineerها امکان داد code خود را به‌سادگی روی dataset بزرگ اجرا کنند. اگر HDFS و MapReduce داشته باشید، می‌توانید SQL query execution engineای روی آن بسازید؛ پروژهٔ Hive همین کار را کرد. اما می‌توان batch processهای بسیاری هم نوشت که مناسب بیان به‌شکل SQL query نیستند.

بعدتر مشخص شد MapReduce برای بعضی processingها بیش‌ازحد محدود و کم‌کارایی است؛ بنابراین processing modelهای دیگری روی Hadoop ساخته شدند که در بخش بعدی chapter می‌بینیم. داشتن فقط دو model، SQL و MapReduce، کافی نبود و modelهای متفاوت‌تری لازم شد. انعطاف platform باز Hadoop امکان ساخت طیف وسیعی از این روش‌ها را می‌داد؛ کاری که در چارچوب MPP database monolithic ممکن نبود.

نکتهٔ مهم این است که این processing modelهای متفاوت می‌توانند روی یک shared-use cluster اجرا شوند و همگی به همان fileهای distributed filesystem دسترسی داشته باشند. در رویکرد Hadoop لازم نیست data را برای هر نوع پردازش به چند system تخصصی منتقل کنیم؛ cluster می‌تواند workloadهای متنوع را پشتیبانی کند. جابه‌جانکردن data استخراج ارزش از آن را آسان‌تر و آزمایش modelهای پردازشی تازه را ساده‌تر می‌کند.

Hadoop ecosystem هم databaseهای OLTP با random access مانند HBase و هم analytic databaseهای MPP مانند Impala را دربر می‌گیرد. نه HBase و نه Impala از MapReduce استفاده نمی‌کنند، اما هر دو HDFS را برای storage به‌کار می‌برند. آن‌ها روش‌های متفاوتی برای دسترسی و پردازش data دارند، ولی می‌توانند در یک system کنار هم کار و یکپارچه شوند.

#### طراحی برای failureهای پرتکرار

در مقایسهٔ MapReduce و MPP database دو تفاوت دیگر مهم است: نحوهٔ برخورد با failure و استفاده از memory و disk. Batch process از online system نسبت به failure حساسیت کمتری دارد؛ اگر شکست بخورد، فوراً روی user اثر نمی‌گذارد و همیشه می‌توان دوباره اجرا کرد.

اگر nodeای هنگام اجرای query در MPP database crash کند، اغلب کل query abort می‌شود و user باید دوباره آن را submit کند یا system خودکار آن را از ابتدا اجرا می‌کند. چون query معمولاً چند ثانیه یا حداکثر چند دقیقه طول می‌کشد، هزینهٔ retry زیاد نیست. MPP databaseها همچنین می‌کوشند تا حد امکان data را در memory نگه دارند—مثلاً با hash join—تا هزینهٔ disk read کاهش یابد.

MapReduce می‌تواند failure یک map یا reduce task را بدون شکست کل job تحمل کند؛ کار را در سطح همان task retry می‌کند. همچنین data را با eagerness زیادی روی disk می‌نویسد؛ بخشی به‌خاطر fault tolerance و بخشی با این فرض که dataset از memory بزرگ‌تر است.

رویکرد MapReduce برای jobهای بزرگ مناسب‌تر است؛ jobهایی که آن‌قدر data پردازش می‌کنند و طولانی‌اند که احتمالاً در مسیرشان دست‌کم یک task failure رخ می‌دهد. در این حالت اجرای مجدد کل job به‌دلیل failure یک task هدررفت است. حتی اگر recovery در سطح task، اجرای بدون failure را کندتر کند، در صورت نرخ بالای failure می‌تواند trade-off معقولی باشد.

اما این فرض‌ها چقدر واقع‌بینانه‌اند؟ در بیشتر clusterها machine failure رخ می‌دهد، ولی چندان پرتکرار نیست؛ شاید آن‌قدر نادر که بیشتر jobها اصلاً failure machine نبینند. آیا تحمل failure ارزش overhead قابل‌توجه را دارد؟

برای فهم مصرف محدود memory و recovery در سطح task در MapReduce باید به محیط اولیهٔ طراحی آن نگاه کرد. Google از datacenterهای چندمنظوره استفاده می‌کرد که online production serviceها و offline batch jobها روی machineهای یکسان اجرا می‌شدند. هر task سهمیهٔ منبعی مانند CPU core، RAM و disk داشت که با container اعمال می‌شد. هر task priority هم داشت؛ اگر task با priority بالاتر به منبع بیشتری نیاز پیدا می‌کرد، taskهای کم‌اولویت‌تر روی همان machine می‌توانستند برای آزادکردن منابع terminate یا preempt شوند. Priority قیمت منابع محاسباتی را نیز تعیین می‌کرد: teamها برای منبع مصرفی هزینه می‌دادند و processهای با priority بالاتر گران‌تر بودند.

این معماری اجازه می‌دهد منابع محاسباتی non-production با priority پایین بیش‌ازحد تخصیص داده شوند، چون system می‌داند در صورت نیاز می‌تواند آن‌ها را پس بگیرد. Overcommit کردن منابع utilization و efficiency machineها را نسبت به جداسازی taskهای production و non-production بهتر می‌کند. اما MapReduce jobها low-priority هستند و هر لحظه ممکن است به‌دلیل نیاز process پر‌اولویت‌تر preempt شوند. Batch jobها عملاً از منابع باقی‌مانده پس از مصرف processهای مهم‌تر استفاده می‌کنند.

در Google، MapReduce taskای که یک ساعت اجرا می‌شود حدود ۵٪ احتمال دارد برای آزادکردن جا برای process پر‌اولویت‌تر terminate شود. این نرخ بیش از یک مرتبهٔ بزرگی از failure ناشی از hardware، reboot machine یا دلیل‌های دیگر بیشتر است. با چنین نرخ preemptionی، اگر job شامل ۱۰۰ task باشد که هرکدام ۱۰ دقیقه اجرا می‌شوند، احتمال اینکه پیش از پایان دست‌کم یک task terminate شود بیش از ۵۰٪ است.

به همین دلیل MapReduce برای تحمل terminate شدن غیرمنتظرهٔ taskها طراحی شده است: نه به‌خاطر غیرقابل‌اعتماد بودن خاص hardware، بلکه چون امکان terminate دلخواه processها utilization منابع cluster را بهتر می‌کند.

در میان open source cluster schedulerها، preemption کمتر رایج است. YARN CapacityScheduler برای متعادل‌کردن تخصیص resource میان queueهای مختلف از preemption پشتیبانی می‌کند، اما در زمان نگارش کتاب، preemption عمومی بر اساس priority در YARN، Mesos یا Kubernetes پشتیبانی نمی‌شد. در محیطی که taskها کمتر terminate می‌شوند، تصمیم‌های طراحی MapReduce کمتر منطقی‌اند. بخش بعدی چند جایگزین MapReduce با تصمیم‌های طراحی متفاوت را بررسی می‌کند.

## Key Terms

- `HDFS` — distributed filesystem در Hadoop که fileها را به block تقسیم می‌کند و با استفاده از diskهای چند machine ذخیره و replicate می‌کند.
- `NameNode` — server مرکزی HDFS که محل قرارگیری file blockها روی machineهای cluster را track می‌کند.
- `Mapper` — function یا taskای که هر input record را مستقل پردازش و صفر یا چند key-value pair تولید می‌کند.
- `Reducer` — functionی که valueهای متعلق به یک key را دریافت، پردازش و output record تولید می‌کند.
- `Shuffle` — فرایند partition کردن، sort و انتقال خروجی mapperها به reducerهای مسئول keyهای مربوطه.
- `Data Locality` — اجرای computation نزدیک به machine نگهدارندهٔ input، برای کاهش انتقال network و بهبود کارایی.
- `Reduce-Side Join` — joinی که در آن mapperها data را بر اساس join key مرتب و partition می‌کنند و reducerها recordهای متناظر را ترکیب می‌کنند.
- `Sort-Merge Join` — الگوریتم join که دو ورودی مرتب‌شده را با پیمایش و merge هم‌زمان بر اساس key تطبیق می‌دهد.
- `Map-Side Join` — joinی که بدون reducer و shuffle در mapper انجام می‌شود و به اندازه یا layout مشخص inputها متکی است.
- `Broadcast Hash Join` — map-side join که input کوچک را در hash table هر mapper بارگذاری و در تمام partitionهای input بزرگ lookup می‌کند.
- `Hot Key` — keyای که حجم نامتناسبی از recordها را به یک reducer می‌فرستد و باعث skew و کندشدن کل job می‌شود.
- `Sessionization` — گروه‌بندی activity eventها بر اساس شناسهٔ session یا user برای بازسازی توالی رفتارها.
- `Data Lake` — مخزن متمرکزی برای نگهداری raw data در formatهای متنوع، پیش از تعیین schema یا کاربرد نهایی آن.
- `Schema-on-Read` — رویکردی که در آن data با schema مشخصی هنگام نوشتن محدود نمی‌شود و consumer هنگام خواندن آن را تفسیر می‌کند.
- `Human Fault Tolerance` — امکان بازگشت از اشتباه انسانی یا bug با حفظ input و تولید مجدد output، بدون دست‌کاری مخرب state قبلی.
- `MPP Database` — databaseای که queryهای تحلیلی را به‌شکل موازی روی cluster اجرا می‌کند و storage، planning و execution را یکپارچه مدیریت می‌کند.
- `Preemption` — خاتمه‌دادن task کم‌اولویت برای آزادسازی منابع جهت اجرای task پر‌اولویت‌تر.

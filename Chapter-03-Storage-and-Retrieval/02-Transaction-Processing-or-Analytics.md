# Chapter 3 — Storage and Retrieval

## Transaction Processing or Analytics?

در روزهای ابتدایی business data processing، یک write در database معمولاً با رخ دادن یک commercial transaction مرتبط بود: انجام یک sale، ثبت order نزد supplier، پرداخت salary یک employee و موارد مشابه. بعدتر که databaseها در حوزه‌هایی استفاده شدند که در آن‌ها پولی جابه‌جا نمی‌شد، واژهٔ transaction همچنان باقی ماند و به گروهی از read و write اشاره کرد که یک logical unit را تشکیل می‌دهند.

یک transaction الزاماً نباید propertyهای ACID، یعنی atomicity، consistency، isolation و durability، را داشته باشد. Transaction processing فقط به معنای اجازه دادن به clientها برای انجام read و write با low latency است؛ در مقابل batch processing jobهایی قرار دارد که فقط به‌صورت دوره‌ای اجرا می‌شوند، مثلاً روزی یک بار. Propertyهای ACID را در Chapter 7 و batch processing را در Chapter 10 بررسی می‌کنیم.

با اینکه databaseها برای typeهای مختلف data، مانند commentهای blog post، actionهای یک game و contactهای address book، استفاده شدند، access pattern پایه همچنان شبیه پردازش business transactionها باقی ماند. یک application معمولاً تعداد کمی record را با استفاده از یک key و یک index lookup می‌کند. Recordها نیز بر اساس input کاربر insert یا update می‌شوند. چون این applicationها interactive هستند، این access pattern به نام online transaction processing یا OLTP شناخته شد.

در عین حال، databaseها به‌طور فزاینده‌ای برای data analytics نیز استفاده شدند؛ کاری که access patternهای بسیار متفاوتی دارد. یک analytic query معمولاً باید تعداد بسیار زیادی record را scan کند، از هر record فقط چند column را بخواند و به‌جای برگرداندن raw data به user، statisticهای aggregate مانند count، sum یا average را محاسبه کند. برای مثال، اگر data شما tableای از sales transactionها باشد، analytic queryها ممکن است چنین سؤال‌هایی را پاسخ دهند:

- درآمد کل هر store در ماه ژانویه چقدر بوده است؟
- در آخرین promotion خود چند banana بیشتر از مقدار معمول فروختیم؟
- کدام brand از baby food بیشتر از همه همراه با brand X از diaper خریداری شده است؟

این queryها اغلب توسط business analystها نوشته می‌شوند و به reportهایی feed می‌شوند که به management شرکت برای تصمیم‌گیری بهتر کمک می‌کنند؛ این حوزه business intelligence نام دارد. برای متمایز کردن این الگوی استفاده از database از transaction processing، این روش online analytical processing یا OLAP نامیده شده است [47].

تفاوت OLTP و OLAP همیشه کاملاً شفاف نیست، اما چند ویژگی معمول آن‌ها در جدول ۳-۱ آمده است.

*پاورقی: معنای online در OLAP کاملاً روشن نیست؛ احتمالاً به این واقعیت اشاره دارد که queryها فقط برای reportهای از پیش تعریف‌شده نیستند و analystها از OLAP به‌صورت interactive برای queryهای exploratory استفاده می‌کنند.*

**جدول ۳-۱. مقایسهٔ ویژگی‌های transaction processing system و analytic system**

| ویژگی | Transaction processing system (OLTP) | Analytic system (OLAP) |
|---|---|---|
| الگوی اصلی read | تعداد کمی record در هر query که با key دریافت می‌شوند | aggregate کردن تعداد زیادی record |
| الگوی اصلی write | writeهای random-access و low-latency ناشی از input کاربر | bulk import با ETL یا event stream |
| استفاده‌کنندهٔ اصلی | end user یا customer از طریق web application | analyst داخلی برای decision support |
| data چه چیزی را نشان می‌دهد؟ | آخرین state داده، یعنی وضعیت فعلی | history رویدادهایی که در طول زمان رخ داده‌اند |
| اندازهٔ dataset | از gigabyte تا terabyte | از terabyte تا petabyte |

در ابتدا از همان databaseها هم برای transaction processing و هم برای analytic query استفاده می‌شد. SQL از این نظر انعطاف‌پذیر بود و هم برای queryهای نوع OLTP و هم برای queryهای نوع OLAP به‌خوبی کار می‌کرد. بااین‌حال، در اواخر دههٔ ۱۹۸۰ و اوایل دههٔ ۱۹۹۰، روندی شکل گرفت که شرکت‌ها استفاده از OLTP systemهای خود برای analytics را متوقف کنند و analytics را روی database جداگانه‌ای اجرا کنند. این database جداگانه data warehouse نام گرفت.

### Data Warehousing

یک enterprise ممکن است ده‌ها transaction processing system متفاوت داشته باشد: systemهایی برای web site مشتریان، کنترل point-of-sale یا checkout در storeهای فیزیکی، tracking موجودی در warehouseها، برنامه‌ریزی route وسایل نقلیه، مدیریت supplierها، ادارهٔ employeeها و موارد دیگر. هرکدام از این systemها پیچیده‌اند و برای نگهداری آن‌ها به teamهای مختلفی نیاز است؛ بنابراین در نهایت این systemها عمدتاً مستقل از یکدیگر کار می‌کنند.

معمولاً از این OLTP systemها انتظار می‌رود availability بالایی داشته باشند و transactionها را با low latency پردازش کنند، چون اغلب برای operationهای business حیاتی هستند. به همین دلیل database administratorها به‌دقت از OLTP databaseها محافظت می‌کنند. آن‌ها معمولاً تمایل ندارند به business analystها اجازه دهند queryهای analytic ad hoc را روی OLTP database اجرا کنند، چون این queryها اغلب پرهزینه‌اند و بخش بزرگی از dataset را scan می‌کنند؛ در نتیجه ممکن است به performance transactionهایی که هم‌زمان در حال اجرا هستند آسیب بزنند.

در مقابل، data warehouse یک database جداگانه است که analystها می‌توانند queryهای خود را آزادانه روی آن اجرا کنند، بدون اینکه operationهای OLTP تحت تأثیر قرار بگیرند [48]. Data warehouse یک copy فقط‌خواندنی از data تمام OLTP systemهای مختلف شرکت را نگهداری می‌کند.

Data از OLTP databaseها استخراج می‌شود؛ این کار می‌تواند با یک data dump دوره‌ای یا یک continuous stream از updateها انجام شود. سپس data به schemaای مناسب برای analysis transform و clean می‌شود و در نهایت در data warehouse load می‌گردد. این فرآیند انتقال data به warehouse، Extract-Transform-Load یا ETL نام دارد و در شکل ۳-۸ نشان داده شده است.

*شکل ۳-۸. نمای ساده‌شدهٔ ETL به یک data warehouse.*

امروزه تقریباً تمام enterpriseهای بزرگ data warehouse دارند، اما در شرکت‌های کوچک این systemها تقریباً ناشناخته‌اند. احتمالاً دلیل آن این است که بیشتر شرکت‌های کوچک OLTP systemهای زیادی ندارند و مقدار data آن‌ها نیز کم است؛ آن‌قدر کم که می‌توان آن را با یک conventional SQL database query کرد یا حتی در یک spreadsheet تحلیل کرد. در یک شرکت بزرگ، برای انجام کاری که در شرکت کوچک ساده است، به تلاش و infrastructure بسیار بیشتری نیاز است.

یکی از مزیت‌های بزرگ استفاده از data warehouse جداگانه، در مقایسه با query کردن مستقیم OLTP systemها برای analytics، این است که data warehouse می‌تواند برای analytic access patternها optimize شود. مشخص شده است که indexing algorithmهایی که در نیمهٔ اول این chapter بررسی کردیم برای OLTP به‌خوبی کار می‌کنند، اما برای پاسخ دادن به analytic queryها چندان مناسب نیستند.

در ادامهٔ این chapter، storage engineهایی را بررسی می‌کنیم که برای analytics optimize شده‌اند.

#### The divergence between OLTP databases and data warehouses

Data model یک data warehouse معمولاً relational است، چون SQL عموماً برای analytic queryها fit خوبی دارد. ابزارهای graphical زیادی برای data analysis وجود دارند که SQL query تولید می‌کنند، resultها را visualize می‌کنند و به analystها اجازه می‌دهند data را با operationهایی مانند drill-down و slicing and dicing به‌صورت exploratory بررسی کنند.

در ظاهر، data warehouse و relational OLTP database شبیه هم هستند، چون هر دو SQL query interface دارند. بااین‌حال، internals آن‌ها می‌تواند کاملاً متفاوت باشد، چون برای query patternهای بسیار متفاوت optimize شده‌اند. بسیاری از database vendorها اکنون روی پشتیبانی از transaction processing یا analytics تمرکز می‌کنند، نه هر دو.

بعضی databaseها، مانند Microsoft SQL Server و SAP HANA، از transaction processing و data warehousing در یک product واحد پشتیبانی می‌کنند. اما این قابلیت‌ها به‌طور فزاینده‌ای به دو storage و query engine جداگانه تبدیل می‌شوند که از طریق یک SQL interface مشترک در دسترس هستند [49, 50, 51].

Vendorهای data warehouse مانند Teradata، Vertica، SAP HANA و ParAccel معمولاً productهای خود را با commercial licenseهای گران‌قیمت می‌فروشند. Amazon RedShift نسخهٔ hosted مربوط به ParAccel است. در سال‌های اخیر، مجموعهٔ بزرگی از projectهای open source با عنوان SQL-on-Hadoop به‌وجود آمده‌اند. این projectها جوان هستند، اما هدفشان رقابت با commercial data warehouse systemهاست. از جملهٔ آن‌ها می‌توان به Apache Hive، Spark SQL، Cloudera Impala، Facebook Presto، Apache Tajo و Apache Drill اشاره کرد [52, 53]. بعضی از این projectها بر ایده‌های Google Dremel بنا شده‌اند [54].

### Stars and Snowflakes: Schemas for Analytics

همان‌طور که در Chapter 2 بررسی کردیم، در حوزهٔ transaction processing، بسته به نیازهای application از data modelهای متنوعی استفاده می‌شود. در analytics، در مقابل، تنوع data model بسیار کمتر است. بسیاری از data warehouseها با الگوی نسبتاً مشخصی استفاده می‌شوند که star schema یا dimensional modeling نام دارد [55].

Schema نمونه در شکل ۳-۹ یک data warehouse را نشان می‌دهد که ممکن است در یک grocery retailer پیدا شود. در مرکز schema، جدولی قرار دارد که fact table نامیده می‌شود؛ در این مثال نام آن fact_sales است. هر row از fact table یک event را نشان می‌دهد که در زمان مشخصی رخ داده است؛ در اینجا هر row خرید یک product توسط یک customer را نمایش می‌دهد. اگر به‌جای retail sale در حال تحلیل web traffic بودیم، هر row می‌توانست یک page view یا click توسط user را نشان دهد.

*شکل ۳-۹. نمونه‌ای از star schema برای استفاده در یک data warehouse.*

معمولاً factها به‌صورت eventهای منفرد capture می‌شوند، چون این روش بیشترین flexibility را برای analysisهای آینده فراهم می‌کند. بااین‌حال، در این صورت fact table می‌تواند بسیار بزرگ شود. یک enterprise بزرگ مانند Apple، Walmart یا eBay ممکن است ده‌ها petabyte transaction history در data warehouse خود داشته باشد که بخش عمدهٔ آن در fact tableها قرار دارد [56].

بعضی columnهای fact table attribute هستند؛ مانند priceای که product با آن فروخته شده و cost خرید آن از supplier، که امکان محاسبهٔ profit margin را فراهم می‌کنند. columnهای دیگر fact table، referenceهای foreign key به tableهای دیگری هستند که dimension table نام دارند. چون هر row از fact table یک event را نشان می‌دهد، dimensionها مشخص می‌کنند event مربوط به چه کسی، چه چیزی، کجا، چه زمانی، چگونه و چرا بوده است.

برای مثال، در شکل ۳-۹ یکی از dimensionها product فروخته‌شده است. هر row در tableِ dim_product نمایندهٔ یک type از productهای قابل‌فروش است و اطلاعاتی مانند stock-keeping unit یا SKU، description، brand name، category، fat content، package size و موارد دیگر را شامل می‌شود. هر row در tableِ fact_sales با استفاده از foreign key مشخص می‌کند که در آن transaction مشخص کدام product فروخته شده است. برای سادگی، اگر customer چند product متفاوت را هم‌زمان بخرد، هر product به‌صورت row جداگانه‌ای در fact table نمایش داده می‌شود.

حتی date و time نیز اغلب با استفاده از dimension table نمایش داده می‌شوند، چون این کار اجازه می‌دهد اطلاعات اضافی دربارهٔ dateها، مانند public holidayها، encode شود و queryها بتوانند میان saleهای روزهای تعطیل و روزهای غیرتعطیل تفاوت بگذارند.

نام star schema از اینجا می‌آید که وقتی relationship میان tableها را visualize می‌کنیم، fact table در مرکز قرار دارد و dimension tableها آن را احاطه کرده‌اند؛ connectionهای این tableها مانند rayهای یک star هستند.

یکی از variantهای این template snowflake schema نام دارد که در آن dimensionها بیشتر به subdimensionها تقسیم می‌شوند. برای مثال، می‌توان tableهای جداگانه‌ای برای brandها و product categoryها داشت و هر row در dim_product به‌جای ذخیره کردن brand و category به‌صورت string، با foreign key به tableهای brand و category reference دهد. Snowflake schemaها نسبت به star schemaها normalizedتر هستند، اما star schemaها اغلب ترجیح داده می‌شوند، چون کار کردن analystها با آن‌ها ساده‌تر است [55].

در یک data warehouse معمولی، tableها اغلب بسیار wide هستند: fact tableها معمولاً بیش از ۱۰۰ column و گاهی چندصد column دارند [51]. Dimension tableها نیز می‌توانند بسیار wide باشند، چون تمام metadataای را شامل می‌شوند که ممکن است برای analysis مرتبط باشد. برای مثال، tableِ dim_store می‌تواند جزئیات serviceهای ارائه‌شده در هر store، داشتن یا نداشتن in-store bakery، مساحت store، تاریخ افتتاح اولیه، تاریخ آخرین بازسازی، فاصله تا نزدیک‌ترین highway و موارد دیگر را شامل شود.

## Key Terms

- `Transaction Processing` — پردازش read و writeهایی که یک logical unit را تشکیل می‌دهند؛ معمولاً برای requestهای interactive با low latency استفاده می‌شود.
- `Online Transaction Processing (OLTP)` — الگوی پردازش transactionهای interactive در applicationهای عملیاتی؛ معمولاً شامل lookup تعداد کمی record با key و update بر اساس input user است.
- `Analytics` — تحلیل حجم بزرگی از data برای استخراج aggregate و insight، به‌جای برگرداندن raw recordها.
- `Online Analytical Processing (OLAP)` — الگوی اجرای analytic queryهای interactive روی حجم بزرگی از data، معمولاً برای decision support و business intelligence.
- `Data Warehouse` — database جداگانه و معمولاً read-only برای نگهداری data گردآوری‌شده از چند operational system و اجرای analytics.
- `Extract-Transform-Load (ETL)` — فرآیند استخراج data از source system، transform و clean کردن آن و load کردن result در data warehouse.
- `Fact Table` — table مرکزی در star schema که هر row آن یک event یا اندازه‌گیری business را نشان می‌دهد.
- `Dimension Table` — tableای که context و ویژگی‌های fact را نگه می‌دارد؛ مانند product، customer، store یا date.
- `Star Schema` — schema تحلیلی که fact table در مرکز و dimension tableها در اطراف آن قرار دارند.
- `Snowflake Schema` — variant نرمال‌شده‌تر star schema که dimensionها را به subdimensionهای جداگانه تقسیم می‌کند.
- `Analytical Query` — queryای که معمولاً تعداد زیادی record را scan و aggregateهایی مانند count، sum یا average محاسبه می‌کند.
- `Operational System` — systemی که workload جاری business را اجرا می‌کند؛ معمولاً شامل OLTP databaseها و serviceهای customer-facing است.
- `Reporting` — تولید report از data برای پایش وضعیت و پشتیبانی از تصمیم‌گیری.
- `Business Intelligence` — استفاده از data، report و analysis برای کمک به management در تصمیم‌گیری business.
- `Dimensional Modeling` — روش مدل‌سازی data برای analytics با استفاده از fact و dimension tableها.

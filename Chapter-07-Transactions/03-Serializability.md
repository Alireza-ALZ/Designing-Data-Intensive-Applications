# Chapter 7 — Transactions

## Serializability

در این chapter چند example از transactionهایی دیدیم که مستعد race condition هستند. بعضی race conditionها توسط read committed و snapshot isolation prevent می‌شوند، اما بعضی دیگر نه. با exampleهای بسیار tricky مربوط به write skew و phantom نیز روبه‌رو شدیم. این وضعیت تأسف‌آور است:

- درک isolation levelها دشوار است و implementation آن‌ها در databaseهای مختلف consistency ندارد؛ برای مثال، معنای «repeatable read» به‌طور قابل‌توجهی متفاوت است.
- اگر به application code نگاه کنید، تشخیص اینکه اجرای آن در یک isolation level مشخص safe است یا نه دشوار است؛ به‌خصوص در application بزرگ، جایی که ممکن است از تمام operationهایی که به‌صورت concurrent رخ می‌دهند خبر نداشته باشید.
- toolهای خوبی برای detect کردن race condition نداریم. در تئوری static analysis می‌تواند کمک کند [26]، اما techniqueهای پژوهشی هنوز به استفادهٔ عملی نرسیده‌اند. Testing برای concurrency issueها دشوار است، چون این مشکل‌ها معمولاً nondeterministic هستند و فقط وقتی رخ می‌دهند که timing نامساعد باشد.

این مسئله جدید نیست و از دههٔ ۱۹۷۰، یعنی زمانی که weak isolation levelها برای نخستین‌بار معرفی شدند، وجود داشته است [2]. در تمام این مدت پاسخ پژوهشگران ساده بوده است: از serializable isolation استفاده کنید.

Serializable isolation معمولاً قوی‌ترین isolation level در نظر گرفته می‌شود. این level تضمین می‌کند که اگرچه transactionها ممکن است به‌صورت parallel اجرا شوند، result نهایی همان result حالتی باشد که transactionها یکی‌یکی و به‌صورت serial، بدون concurrency، اجرا شده باشند. بنابراین database تضمین می‌کند اگر transactionها هنگام اجرای مستقل correct باشند، هنگام اجرای concurrent نیز correct باقی بمانند؛ به بیان دیگر، database جلوی تمام race conditionهای ممکن را می‌گیرد.

اما اگر serializable isolation تا این حد بهتر از آشفتگی weak isolation levelهاست، چرا همه از آن استفاده نمی‌کنند؟ برای پاسخ به این سؤال باید optionهای implement کردن serializability و performance آن‌ها را بررسی کنیم. بیشتر databaseهایی که امروز serializability ارائه می‌کنند از یکی از سه technique زیر استفاده می‌کنند:

- اجرای literal transactionها به‌ترتیب serial (به بخش «Actual Serial Execution» در صفحهٔ ۲۵۲ مراجعه کنید)
- **Two-phase locking** (به بخش «Two-Phase Locking (2PL)» در صفحهٔ ۲۵۷ مراجعه کنید) که برای چند دهه تنها option عملی بود
- techniqueهای optimistic concurrency control مانند serializable snapshot isolation (به بخش «Serializable Snapshot Isolation (SSI)» در صفحهٔ ۲۶۱ مراجعه کنید)

فعلاً این techniqueها را عمدتاً در context مربوط به single-node database بررسی می‌کنیم؛ در Chapter 9 خواهیم دید چگونه می‌توان آن‌ها را برای transactionهایی generalize کرد که چند node را در یک distributed system درگیر می‌کنند.

### Actual Serial Execution

ساده‌ترین راه برای جلوگیری از concurrency problem این است که concurrency را کاملاً حذف کنیم: فقط یک transaction را در هر لحظه و به‌ترتیب serial روی یک thread اجرا کنیم. با این کار مسئلهٔ detect و prevent کردن conflict میان transactionها را به‌طور کامل دور می‌زنیم؛ Isolation حاصل، بنا به definition، serializable است.

با اینکه این idea واضح به نظر می‌رسد، database designerها نسبتاً به‌تازگی—حدود سال ۲۰۰۷—به این نتیجه رسیدند که یک loop تک‌threadی برای اجرای transactionها feasible است [45]. اگر در ۳۰ سال قبل، multi-threaded concurrency برای performance خوب essential در نظر گرفته می‌شد، چه چیزی تغییر کرد که اجرای single-threadی ممکن شد؟

دو development باعث این بازنگری شدند:

- RAM آن‌قدر ارزان شد که برای بسیاری از use caseها اکنون نگه‌داشتن کل active dataset در memory feasible است (به بخش «Keeping everything in memory» در صفحهٔ ۸۸ مراجعه کنید). وقتی تمام data موردنیاز transaction در memory باشد، transactionها بسیار سریع‌تر از حالتی اجرا می‌شوند که مجبور باشند برای load شدن data از disk صبر کنند.
- Database designerها متوجه شدند OLTP transactionها معمولاً کوتاه‌اند و فقط تعداد کمی read و write انجام می‌دهند (به بخش «Transaction Processing or Analytics?» در صفحهٔ ۹۰ مراجعه کنید). در مقابل، analytic queryهای طولانی معمولاً read-only هستند و می‌توان آن‌ها را با استفاده از snapshot isolation، خارج از serial execution loop و روی snapshot consistent اجرا کرد.

رویکرد اجرای serial transactionها در VoltDB/H-Store، Redis و Datomic implement شده است [46, 47, 48]. Systemی که برای اجرای single-threadی طراحی شده است، گاهی می‌تواند بهتر از systemی performance داشته باشد که concurrency را پشتیبانی می‌کند، چون overhead مربوط به coordination و locking را ندارد. بااین‌حال، throughput آن به throughput یک CPU core محدود می‌شود. برای استفادهٔ حداکثری از آن single thread، transactionها باید متفاوت از شکل سنتی خود structure شوند.

#### Encapsulating Transactions in Stored Procedures

در روزهای ابتدایی databaseها، این هدف وجود داشت که یک database transaction کل flow مربوط به activity یک user را در بر بگیرد. برای مثال، خرید ticket هواپیما فرایندی چندمرحله‌ای است: search کردن route، fare و seatهای available؛ تصمیم‌گیری دربارهٔ itinerary؛ booking کردن seatهای هر flight در itinerary؛ وارد کردن جزئیات passenger؛ و پرداخت. Database designerها تصور می‌کردند اگر کل این process یک transaction باشد خوب است، تا بتوان آن را به‌صورت atomic commit کرد.

متأسفانه انسان‌ها در تصمیم‌گیری و response دادن بسیار کندند. اگر database transaction مجبور باشد برای input یک user صبر کند، database باید تعداد potentially بسیار زیادی transaction concurrent را پشتیبانی کند که بیشتر آن‌ها idle هستند. بیشتر databaseها نمی‌توانند این کار را به‌صورت efficient انجام دهند؛ بنابراین تقریباً تمام OLTP applicationها transactionها را کوتاه نگه می‌دارند و از انتظار interactive برای user درون transaction اجتناب می‌کنند. در web، این یعنی transaction در همان HTTP request commit می‌شود و transaction چند request را در بر نمی‌گیرد. هر HTTP request جدید، transaction جدیدی را start می‌کند.

بااینکه user از critical path کنار گذاشته شده است، transactionها همچنان به سبک interactive client/server اجرا می‌شوند، یعنی هر بار یک statement.

application یک query ارسال می‌کند، result را read می‌کند، شاید بر اساس result query اول query دیگری بفرستد و همین‌طور ادامه دهد. queryها و resultها میان application code که روی یک machine اجرا می‌شود و database server که روی machine دیگری قرار دارد، رفت‌وبرگشت می‌کنند.

در این سبک interactive transaction، زمان زیادی صرف network communication میان application و database می‌شود. اگر concurrency را در database ممنوع کنید و فقط یک transaction را در هر لحظه process کنید، throughput بسیار بد خواهد بود، چون database بیشتر زمان خود را صرف انتظار برای ارسال query بعدی توسط application در transaction فعلی می‌کند. در چنین databaseای برای دستیابی به performance قابل‌قبول لازم است چند transaction به‌صورت concurrent process شوند.

به همین دلیل، systemهایی که serial transaction processing تک‌threadی دارند، multi-statement transactionهای interactive را اجازه نمی‌دهند. در عوض، application باید کل code مربوط به transaction را از قبل به database submit کند؛ این code به‌صورت **stored procedure** اجرا می‌شود. تفاوت این دو approach در شکل ۷-۹ نشان داده شده است. اگر تمام data موردنیاز transaction در memory باشد، stored procedure می‌تواند بسیار سریع execute شود، بدون اینکه برای network یا disk I/O منتظر بماند.

**شکل ۷-۹.** تفاوت میان interactive transaction و stored procedure با استفاده از transaction مثال شکل ۷-۸.

#### Pros and Cons of Stored Procedures

Stored procedureها مدتی طولانی در relational databaseها وجود داشته‌اند و از سال ۱۹۹۹ بخشی از SQL standard با نام SQL/PSM بوده‌اند. این featureها به دلایل مختلفی reputation چندان خوبی به دست نیاورده‌اند:

- هر database vendor زبان خودش را برای stored procedure دارد؛ Oracle دارای PL/SQL، SQL Server دارای T-SQL و PostgreSQL دارای PL/pgSQL است. این زبان‌ها هم‌پای development زبان‌های programming عمومی پیش نرفته‌اند، بنابراین از دیدگاه امروز زشت و archaic به نظر می‌رسند و ecosystem مربوط به libraryهایی را که در بیشتر programming languageها پیدا می‌کنید ندارند.
- مدیریت codeای که در database اجرا می‌شود دشوار است: در مقایسه با application server، debug کردن آن سخت‌تر، نگه‌داری آن در version control و deploy کردنش awkwardتر، testing آن trickyتر و integration آن با metrics collection system برای monitoring دشوارتر است.
- Database معمولاً بسیار performance-sensitiveتر از application server است، چون یک database instance اغلب میان application serverهای زیادی share می‌شود. یک stored procedure بد نوشته‌شده—برای مثال، procedureای که memory یا CPU زیادی مصرف می‌کند—می‌تواند دردسر بسیار بیشتری از code بد مشابه در application server ایجاد کند.

بااین‌حال، این مشکل‌ها قابل‌حل هستند. implementationهای جدید stored procedure از PL/SQL فاصله گرفته‌اند و به‌جای آن از programming languageهای عمومی موجود استفاده می‌کنند: VoltDB از Java یا Groovy، Datomic از Java یا Clojure و Redis از Lua استفاده می‌کند.

با stored procedure و data در memory، اجرای تمام transactionها روی یک thread feasible می‌شود. چون این transactionها لازم نیست برای I/O صبر کنند و overhead مربوط به دیگر mechanismهای concurrency control را نیز ندارند، می‌توانند روی یک thread به throughput خوبی برسند.

VoltDB برای replication نیز از stored procedure استفاده می‌کند: به‌جای copy کردن writeهای transaction از یک node به node دیگر، همان stored procedure را روی هر replica execute می‌کند. بنابراین VoltDB نیاز دارد stored procedureها deterministic باشند؛ یعنی هنگام اجرا روی nodeهای مختلف result یکسانی تولید کنند. برای مثال، اگر transaction لازم باشد از date و time فعلی استفاده کند، باید این کار را از طریق deterministic APIهای special انجام دهد.

#### Partitioning

اجرای تمام transactionها به‌صورت serial، concurrency control را بسیار ساده می‌کند، اما transaction throughput database را به سرعت یک CPU core روی یک machine محدود می‌کند. Read-only transactionها می‌توانند با استفاده از snapshot isolation در محل دیگری اجرا شوند؛ اما برای applicationهایی با write throughput بالا، transaction processor تک‌threadی می‌تواند bottleneck جدی باشد.

برای scale کردن روی چند CPU core و چند node، می‌توانید data را partition کنید (به Chapter 6 مراجعه کنید)؛ VoltDB از این روش پشتیبانی می‌کند. اگر بتوانید dataset را طوری partition کنید که هر transaction فقط data درون یک partition را read و write کند، هر partition می‌تواند thread پردازش transaction خودش را داشته باشد و مستقل از partitionهای دیگر اجرا شود. در این حالت می‌توانید به هر CPU core یک partition بدهید؛ در نتیجه transaction throughput با تعداد CPU coreها به‌صورت linearly scale می‌شود [47].

بااین‌حال، برای هر transactionای که باید به چند partition دسترسی داشته باشد، database باید transaction را میان تمام partitionهای درگیر coordinate کند. Stored procedure باید روی تمام partitionها به‌صورت lock-step اجرا شود تا serializability در کل system حفظ شود.

از آنجا که cross-partition transactionها overhead مربوط به coordination دارند، بسیار کندتر از single-partition transactionها هستند. VoltDB throughput حدود ۱۰۰۰ cross-partition write در هر ثانیه گزارش می‌کند؛ این مقدار چند order of magnitude کمتر از single-partition throughput است و با اضافه کردن machineهای بیشتر افزایش پیدا نمی‌کند [49]. اینکه transactionها بتوانند single-partition باشند، تا حد زیادی به structure data مورد استفادهٔ application بستگی دارد. Simple key-value data اغلب به‌سادگی partition می‌شود، اما data با secondary indexهای متعدد احتمالاً به مقدار زیادی cross-partition coordination نیاز دارد (به بخش «Partitioning and Secondary Indexes» در صفحهٔ ۲۰۶ مراجعه کنید).

#### Summary of Serial Execution

Serial execution transactionها، تحت constraintهای مشخص، به روشی viable برای دستیابی به serializable isolation تبدیل شده است:

- هر transaction باید کوچک و سریع باشد، چون فقط یک transaction کند کافی است تا کل transaction processing متوقف شود.
- این روش فقط برای use caseهایی مناسب است که active dataset در memory جا شود. Dataای که به‌ندرت access می‌شود ممکن است به disk منتقل شود، اما اگر لازم باشد در یک single-threaded transaction به آن دسترسی پیدا شود، system بسیار کند خواهد شد.x
- Write throughput باید آن‌قدر پایین باشد که روی یک CPU core قابل‌پردازش باشد؛ یا باید transactionها را طوری partition کرد که به cross-partition coordination نیاز نداشته باشند.
- Cross-partition transaction ممکن است، اما میزان استفاده از آن محدودیت سختی دارد.

*پاورقی:* اگر transaction نیاز داشته باشد به dataای دسترسی پیدا کند که در memory نیست، بهترین solution ممکن است abort کردن transaction، fetch کردن asynchronous data به memory درحالی‌که process کردن transactionهای دیگر ادامه دارد، و سپس restart کردن transaction پس از load شدن data باشد. این approach، همان‌طور که پیش‌تر در بخش «Keeping everything in memory» در صفحهٔ ۸۸ اشاره شد، **anti-caching** نام دارد.

### Two-Phase Locking (2PL)

برای حدود ۳۰ سال، تنها algorithm پرکاربرد serializability در databaseها **two-phase locking (2PL)** بود.xi

> #### 2PL is not 2PC
>
> توجه کنید که با وجود شباهت نام two-phase locking (2PL) و two-phase commit (2PC)، این دو کاملاً متفاوت‌اند. 2PC را در Chapter 9 بررسی خواهیم کرد.

پیش‌تر دیدیم که lockها اغلب برای جلوگیری از dirty write استفاده می‌شوند (به بخش «No Dirty Writes» در صفحهٔ ۲۳۵ مراجعه کنید): اگر دو transaction به‌صورت concurrent تلاش کنند روی object یکسانی write کنند، lock تضمین می‌کند writer دوم باید تا پایان transaction اول—چه abort شده باشد و چه commit—صبر کند و سپس ادامه دهد.

Two-phase locking مشابه این روش است، اما requirementهای بسیار قوی‌تری برای lock دارد. چند transaction می‌توانند تا زمانی که هیچ‌کدام در حال write نیستند، object یکسانی را به‌صورت concurrent read کنند. اما به‌محض اینکه transactionای بخواهد objectی را write، modify یا delete کند، دسترسی exclusive لازم می‌شود:

- اگر transaction A objectی را read کرده باشد و transaction B بخواهد آن object را write کند، B باید تا commit یا abort شدن A صبر کند. این کار تضمین می‌کند B نتواند object را بدون اطلاع A به‌طور غیرمنتظره تغییر دهد.
- اگر transaction A objectی را write کرده باشد و transaction B بخواهد آن object را read کند، B باید تا commit یا abort شدن A صبر کند. در 2PL read کردن version قدیمی object، مانند شکل ۷-۱، قابل‌قبول نیست.

در 2PL، writerها فقط writerهای دیگر را block نمی‌کنند؛ آن‌ها readerها را نیز block می‌کنند و برعکس. Snapshot isolation این mantra را دارد که readerها هیچ‌وقت writerها را block نمی‌کنند و writerها نیز هیچ‌وقت readerها را block نمی‌کنند (به بخش «Implementing Snapshot Isolation» در صفحهٔ ۲۳۹ مراجعه کنید). این اصل تفاوت کلیدی snapshot isolation با two-phase locking است. از سوی دیگر، چون 2PL serializability را فراهم می‌کند، در برابر تمام race conditionهایی که پیش‌تر بررسی کردیم—از جمله lost update و write skew—محافظت می‌کند.

#### Implementation of Two-Phase Locking

2PL در MySQL با storage engine مربوط به InnoDB و در SQL Server برای serializable isolation level استفاده می‌شود؛ همچنین در DB2 برای repeatable read isolation level به‌کار می‌رود [23, 36].

Block شدن readerها و writerها با داشتن lock روی هر object در database implement می‌شود. lock می‌تواند در shared mode یا exclusive mode باشد:

- اگر transaction بخواهد objectی را read کند، ابتدا باید lock آن را در shared mode acquire کند. چند transaction می‌توانند هم‌زمان lock را در shared mode در اختیار داشته باشند، اما اگر transaction دیگری از قبل exclusive lock روی object داشته باشد، این transactionها باید صبر کنند.
- اگر transaction بخواهد objectی را write کند، ابتدا باید lock آن را در exclusive mode acquire کند. هیچ transaction دیگری نمی‌تواند هم‌زمان lock را در اختیار داشته باشد—نه در shared mode و نه در exclusive mode—بنابراین اگر هر lock موجودی روی object باشد، transaction باید صبر کند.
- اگر transaction ابتدا objectی را read و سپس write کند، می‌تواند shared lock خود را به exclusive lock upgrade کند. این upgrade همانند گرفتن مستقیم exclusive lock عمل می‌کند.
- پس از آنکه transaction lock را acquire کرد، باید تا پایان transaction—یعنی commit یا abort—آن را نگه دارد. دلیل نام‌گذاری «two-phase» همین است: phase اول، هنگام اجرای transaction، زمان acquire شدن lockهاست و phase دوم، در پایان transaction، زمان release شدن تمام lockها.

از آنجا که lockهای زیادی در حال استفاده‌اند، به‌سادگی ممکن است transaction A در انتظار release شدن lock توسط transaction B گیر کند و transaction B نیز برعکس در انتظار A باشد. این وضعیت **deadlock** نام دارد. Database به‌صورت automatic deadlock میان transactionها را detect می‌کند و یکی از آن‌ها را abort می‌کند تا transactionهای دیگر بتوانند progress کنند. Application باید transaction abortشده را retry کند.

#### Performance of Two-Phase Locking

عیب بزرگ two-phase locking و دلیلی که باعث شده از دههٔ ۱۹۷۰ همه از آن استفاده نکنند، performance است: transaction throughput و query response time در two-phase locking به‌طور قابل‌توجهی بدتر از weak isolation است.

این مسئله تا حدی ناشی از overhead acquire و release کردن تمام lockهاست، اما دلیل مهم‌تر reduced concurrency است. بر اساس design، اگر دو transaction concurrent بخواهند کاری انجام دهند که به هر شکلی ممکن است به race condition منجر شود، یکی باید تا پایان دیگری صبر کند.

Relational databaseهای سنتی مدت transaction را محدود نمی‌کنند، چون برای applicationهای interactive طراحی شده‌اند که برای input انسان منتظر می‌مانند. بنابراین وقتی transactionای باید منتظر transaction دیگری بماند، limitی برای مدت انتظار وجود ندارد. حتی اگر مطمئن شوید transactionها را کوتاه نگه می‌دارید، وقتی چند transaction بخواهند به object یکسانی دسترسی پیدا کنند queue تشکیل می‌شود و یک transaction ممکن است مجبور شود تا completion چند transaction دیگر صبر کند.

به همین دلیل، databaseهایی که از 2PL استفاده می‌کنند ممکن است latency بسیار unstable داشته باشند و در percentileهای بالا بسیار کند شوند (به بخش «Describing Performance» در صفحهٔ ۱۳ مراجعه کنید)، به‌خصوص اگر workload contention داشته باشد. فقط یک transaction کند یا transactionای که data زیادی access می‌کند و lockهای زیادی می‌گیرد، کافی است تا کل system را به توقف نزدیک کند. این instability برای operation robust مشکل‌ساز است.

Deadlock در read committed isolation مبتنی بر lock نیز ممکن است رخ دهد، اما در 2PL serializable isolation بسیار رایج‌تر است؛ میزان آن به access patternهای transaction شما بستگی دارد. این موضوع می‌تواند performance problem دیگری ایجاد کند: وقتی transaction به‌دلیل deadlock abort و retry می‌شود، باید تمام work خود را از ابتدا انجام دهد. اگر deadlockها frequent باشند، مقدار قابل‌توجهی effort هدر می‌رود.

#### Predicate Locks

در توصیف قبلی lockها، یک جزئیات ظریف اما مهم را نادیده گرفتیم. در بخش «Phantoms Causing Write Skew» در صفحهٔ ۲۵۰ دربارهٔ phantom صحبت کردیم؛ یعنی یک transaction result search query مربوط به transaction دیگری را تغییر می‌دهد. Databaseای با serializable isolation باید از phantom جلوگیری کند.

در example مربوط به meeting room booking، اگر transactionای bookingهای موجود برای یک room را در یک time window مشخص search کرده باشد (به Example 7-2 مراجعه کنید)، transaction دیگری نباید بتواند به‌صورت concurrent booking دیگری را برای همان room و time range insert یا update کند. Insert کردن booking برای roomهای دیگر یا برای همان room در زمان دیگری که روی booking پیشنهادی اثر ندارد، مشکلی ندارد.

چگونه این کار را implement کنیم؟ از نظر مفهومی به **predicate lock** نیاز داریم [3]. این lock مانند shared/exclusive lockی که پیش‌تر توضیح دادیم کار می‌کند، اما به یک object مشخص—مثلاً یک row از table—تعلق ندارد؛ بلکه به تمام objectهایی تعلق دارد که با search condition مشخصی match می‌شوند، مانند query زیر:

```sql
SELECT * FROM bookings
  WHERE room_id = 123 AND
    end_time   > '2018-01-01 12:00' AND
    start_time < '2018-01-01 13:00';
```

Predicate lock به شکل زیر access را محدود می‌کند:

- اگر transaction A بخواهد objectهایی را read کند که با condition مشخصی match می‌شوند—مانند query بالا—باید predicate lock را روی conditionهای query در shared mode acquire کند. اگر transaction B در حال حاضر روی هر objectی که با این conditionها match می‌شود exclusive lock داشته باشد، A باید تا release شدن lock توسط B صبر کند و سپس query خود را اجرا کند.
- اگر transaction A بخواهد objectی را insert، update یا delete کند، ابتدا باید check کند آیا value قدیمی یا جدید با predicate lock موجودی match می‌شود یا نه. اگر predicate lock مطابقی در اختیار transaction B باشد، A باید تا commit یا abort شدن B صبر کند و سپس ادامه دهد.

ایدهٔ اصلی این است که predicate lock حتی روی objectهایی اعمال می‌شود که هنوز در database وجود ندارند اما ممکن است در آینده اضافه شوند؛ یعنی phantomها. اگر two-phase locking شامل predicate lock باشد، database از تمام شکل‌های write skew و دیگر race conditionها جلوگیری می‌کند و در نتیجه Isolation آن serializable می‌شود.

#### Index-Range Locks

متأسفانه predicate lockها performance خوبی ندارند: اگر transactionهای active lockهای زیادی داشته باشند، check کردن lockهای matching زمان‌بر می‌شود. به همین دلیل، بیشتر databaseهایی که 2PL دارند در واقع **index-range locking** یا **next-key locking** را implement می‌کنند که approximation ساده‌شده‌ای از predicate locking است [41, 50].

ساده کردن predicate با match کردن مجموعهٔ بزرگ‌تری از objectها safe است. برای مثال، اگر predicate lock برای bookingهای room شمارهٔ ۱۲۳ از ظهر تا ساعت ۱ بعدازظهر داشته باشید، می‌توانید آن را با lock کردن bookingهای room شمارهٔ ۱۲۳ در هر زمان، یا با lock کردن تمام roomها—نه فقط room ۱۲۳—بین ظهر و ساعت ۱ approximate کنید. این کار safe است، چون هر writeای که با predicate اصلی match شود، حتماً با approximationها نیز match می‌شود.

در database مربوط به room booking، احتمالاً روی column مربوط به `room_id` index دارید و/یا روی `start_time` و `end_time` index دارید؛ در غیر این صورت query قبلی روی database بزرگ بسیار کند خواهد بود:

- فرض کنید index شما روی `room_id` است و database از این index برای پیدا کردن bookingهای موجود در room شمارهٔ ۱۲۳ استفاده می‌کند. در این حالت database می‌تواند به‌سادگی shared lock را به index entry مربوطه attach کند؛ این lock نشان می‌دهد transaction bookingهای room شمارهٔ ۱۲۳ را search کرده است.
- alternatively، اگر database از index مبتنی بر time برای پیدا کردن bookingهای موجود استفاده کند، می‌تواند shared lock را به rangeای از valueها در آن index attach کند؛ این lock نشان می‌دهد transaction bookingهایی را search کرده است که با time period ظهر تا ساعت ۱ بعدازظهر در ۱ ژانویهٔ ۲۰۱۸ overlap دارند.

در هر دو حالت، approximation مربوط به search condition به یکی از indexها attach می‌شود. حالا اگر transaction دیگری بخواهد bookingای را برای همان room و/یا time period overlapping insert، update یا delete کند، باید همان بخش index را update کند. در این فرایند با shared lock برخورد می‌کند و مجبور می‌شود تا release شدن lock صبر کند.

این روش از phantom و write skew به‌صورت effective محافظت می‌کند. Index-range lockها به‌اندازهٔ predicate lock دقیق نیستند—ممکن است range بزرگ‌تری از objectها را نسبت به مقدار strictly necessary برای حفظ serializability lock کنند—اما چون overhead بسیار کمتری دارند، compromise خوبی هستند.

اگر index مناسبی وجود نداشته باشد که range lock را به آن attach کنیم، database می‌تواند به shared lock روی کل table fallback کند. این کار برای performance مناسب نیست، چون تمام transactionهای دیگر را از write کردن در table متوقف می‌کند، اما fallback safeای است.

### Serializable Snapshot Isolation (SSI)

این chapter تصویری نه‌چندان امیدوارکننده از concurrency control در databaseها ارائه کرده است. از یک طرف، implementationهای serializability یا performance خوبی ندارند (two-phase locking) یا به‌خوبی scale نمی‌شوند (serial execution). از طرف دیگر، weak isolation levelها performance خوبی دارند، اما مستعد race conditionهای مختلفی مانند lost update، write skew و phantom هستند. آیا serializable isolation و performance خوب ذاتاً با یکدیگر ناسازگارند؟

شاید نه. Algorithmی به نام **serializable snapshot isolation (SSI)** بسیار promising است. SSI serializability کامل ارائه می‌کند، اما در مقایسه با snapshot isolation فقط penalty performance کوچکی دارد. SSI نسبتاً جدید است: نخستین‌بار در سال ۲۰۰۸ توصیف شد [40] و موضوع رسالهٔ PhD مربوط به Michael Cahill بود [51].

امروزه SSI هم در single-node databaseها—به‌عنوان serializable isolation level در PostgreSQL از version 9.1 به بعد [41]—و هم در distributed databaseها استفاده می‌شود؛ FoundationDB از algorithm مشابهی استفاده می‌کند. از آنجا که SSI در مقایسه با دیگر concurrency control mechanismها بسیار جوان است، performance آن هنوز در عمل در حال اثبات شدن است، اما این possibility را دارد که در آینده آن‌قدر fast شود که به default جدید تبدیل شود.

#### Pessimistic Versus Optimistic Concurrency Control

Two-phase locking یک mechanism از نوع **pessimistic concurrency control** است: بر این اصل بنا شده که اگر چیزی ممکن است اشتباه پیش برود—برای مثال lockای که transaction دیگری در اختیار دارد—بهتر است تا safe شدن situation صبر کنیم و بعد کاری انجام دهیم. این approach شبیه mutual exclusion در multi-threaded programming است که برای محافظت از data structureها استفاده می‌شود.

Serial execution از یک نظر pessimistic در extremeترین حالت است: در اصل معادل این است که هر transaction برای مدت اجرای خود یک exclusive lock روی کل database یا یک partition از database داشته باشد. ما این pessimism را با بسیار سریع اجرا کردن هر transaction جبران می‌کنیم تا transaction فقط برای مدت کوتاهی این «lock» را نگه دارد.

در مقابل، serializable snapshot isolation یک technique از نوع **optimistic concurrency control** است. Optimistic در این context یعنی به‌جای block کردن transaction وقتی چیزی potentially dangerous رخ می‌دهد، اجازه دهیم transactionها ادامه دهند و امیدوار باشیم همه‌چیز درست پیش برود. وقتی transaction می‌خواهد commit شود، database check می‌کند آیا اتفاق بدی رخ داده است یا نه؛ یعنی آیا Isolation نقض شده است. اگر نقض شده باشد، transaction abort و retry می‌شود. فقط transactionهایی اجازهٔ commit دارند که به‌صورت serializable اجرا شده باشند.

Optimistic concurrency control ایدهٔ قدیمی‌ای است [52] و مزایا و معایب آن مدت‌هاست مورد بحث قرار گرفته‌اند [53]. اگر contention بالا باشد—یعنی transactionهای زیادی تلاش کنند به objectهای یکسان دسترسی داشته باشند—این approach بد عمل می‌کند، چون درصد زیادی از transactionها باید abort شوند. اگر system همین حالا نزدیک maximum throughput خود باشد، load اضافی ناشی از retry شدن transactionها می‌تواند performance را بدتر کند.

بااین‌حال، اگر capacity آزاد کافی وجود داشته باشد و contention میان transactionها زیاد نباشد، techniqueهای optimistic concurrency control معمولاً بهتر از techniqueهای pessimistic عمل می‌کنند. می‌توان contention را با atomic operationهای commutative کاهش داد: برای مثال، اگر چند transaction به‌صورت concurrent بخواهند counterای را increment کنند، order اعمال incrementها اهمیتی ندارد—تا زمانی که counter در همان transaction read نشود—بنابراین تمام incrementهای concurrent را می‌توان بدون conflict apply کرد.

همان‌طور که از نام SSI پیداست، این algorithm بر snapshot isolation بنا شده است؛ یعنی تمام readهای درون transaction از snapshot consistent database انجام می‌شوند (به بخش «Snapshot Isolation and Repeatable Read» در صفحهٔ ۲۳۷ مراجعه کنید). تفاوت اصلی SSI با techniqueهای optimistic concurrency control قبلی همین است. SSI علاوه بر snapshot isolation، algorithmی برای detect کردن serialization conflict میان writeها و تعیین transactionهایی که باید abort شوند اضافه می‌کند.

#### Decisions Based on an Outdated Premise

هنگامی که write skew را در snapshot isolation بررسی کردیم (به بخش «Write Skew and Phantoms» در صفحهٔ ۲۴۶ مراجعه کنید)، یک pattern تکرارشونده دیدیم: transaction dataای را از database read می‌کند، result query را بررسی می‌کند و بر اساس result مشاهده‌شده تصمیم می‌گیرد actionی انجام دهد؛ یعنی در database write کند. بااین‌حال، در snapshot isolation ممکن است result query اولیه تا زمان commit transaction دیگر up to date نباشد، چون data در این فاصله modify شده است.

به بیان دیگر، transaction بر اساس یک **premise** عمل می‌کند؛ یعنی factی که در ابتدای transaction true بوده است، مانند «در حال حاضر دو doctor on call هستند». بعدتر، وقتی transaction می‌خواهد commit شود، ممکن است data اصلی تغییر کرده باشد و premise دیگر true نباشد.

وقتی application queryای اجرا می‌کند—برای مثال «در حال حاضر چند doctor on call هستند؟»—database نمی‌داند application از result query در logic خود چگونه استفاده می‌کند. برای safe بودن، database باید فرض کند هر تغییری در result query یا premise به این معناست که writeهای آن transaction ممکن است invalid باشند. به بیان دیگر، ممکن است میان queryها و writeهای transaction یک causal dependency وجود داشته باشد. برای ارائهٔ serializable isolation، database باید situationهایی را detect کند که transaction ممکن است بر اساس premiseای outdated عمل کرده باشد و در این صورت transaction را abort کند.

database چگونه می‌فهمد result query ممکن است تغییر کرده باشد؟ دو case وجود دارد:

- detect کردن read یک version قدیمی و stale از object در MVCC؛ یعنی write commit‌نشده‌ای پیش از read رخ داده است.
- detect کردن writeهایی که روی readهای قبلی اثر می‌گذارند؛ یعنی write پس از read رخ داده است.

#### Detecting Stale MVCC Reads

به یاد بیاورید که snapshot isolation معمولاً با multi-version concurrency control پیاده‌سازی می‌شود (MVCC؛ به شکل ۷-۱۰ مراجعه کنید). وقتی transaction از snapshot consistent در یک MVCC database read می‌کند، writeهایی را که transactionهای دیگر در زمان گرفته شدن snapshot هنوز commit نکرده بودند نادیده می‌گیرد. در شکل ۷-۱۰، transaction شمارهٔ ۴۳ می‌بیند Alice دارای `on_call = true` است، چون transaction شمارهٔ ۴۲—که status مربوط به on-call بودن Alice را modify کرده—هنوز commit نشده است. بااین‌حال، تا زمانی که transaction ۴۳ می‌خواهد commit شود، transaction ۴۲ commit شده است. یعنی writeای که هنگام read کردن snapshot consistent نادیده گرفته شده بود، اکنون effect خود را گذاشته و premise مربوط به transaction ۴۳ دیگر true نیست.

**شکل ۷-۱۰.** تشخیص زمانی که transaction از snapshot مربوط به MVCC valueهای outdated را read می‌کند.

برای جلوگیری از این anomaly، database باید track کند که transactionی به‌دلیل MVCC visibility ruleها writeهای transaction دیگری را نادیده گرفته است. وقتی transaction می‌خواهد commit شود، database check می‌کند آیا هیچ‌کدام از writeهای نادیده‌گرفته‌شده اکنون commit شده‌اند یا نه. اگر چنین باشد، transaction باید abort شود.

چرا تا زمان commit صبر کنیم؟ چرا transaction ۴۳ را به‌محض detect شدن stale read abort نکنیم؟ اگر transaction ۴۳ read-only باشد، لازم نیست abort شود، چون خطری از جانب write skew وجود ندارد. در زمان انجام read توسط transaction ۴۳، database هنوز نمی‌داند آیا transaction بعداً write خواهد کرد یا نه. علاوه بر این، transaction ۴۲ ممکن است بعداً abort شود یا هنگام commit شدن transaction ۴۳ همچنان commit‌نشده باقی بماند؛ در این صورت read ممکن است در نهایت stale نباشد. با پرهیز از abortهای غیرضروری، SSI پشتیبانی snapshot isolation از readهای طولانی روی snapshot consistent را حفظ می‌کند.

#### Detecting Writes That Affect Prior Reads

دومین case زمانی است که transaction دیگری پس از read شدن data آن را modify کند. شکل ۷-۱۱ این case را نشان می‌دهد.

**شکل ۷-۱۱.** در serializable snapshot isolation، تشخیص زمانی که یک transaction data خوانده‌شده توسط transaction دیگری را modify می‌کند.

در context مربوط به two-phase locking دربارهٔ index-range lock صحبت کردیم (به بخش «Index-Range Locks» در صفحهٔ ۲۶۰ مراجعه کنید). این lockها اجازه می‌دهند database دسترسی به تمام rowهایی را که با search query مشخصی match می‌شوند lock کند، مانند `WHERE shift_id = 1234`. در اینجا می‌توانیم technique مشابهی به‌کار ببریم، با این تفاوت که SSI lockها transactionهای دیگر را block نمی‌کنند.

در شکل ۷-۱۱، transactionهای ۴۲ و ۴۳ هر دو doctorهای on-call مربوط به shift شمارهٔ ۱۲۳۴ را search می‌کنند. اگر روی `shift_id` index وجود داشته باشد، database می‌تواند از index entry مربوط به ۱۲۳۴ استفاده کند تا ثبت کند transactionهای ۴۲ و ۴۳ این data را read کرده‌اند. (اگر index وجود نداشته باشد، این information را می‌توان در سطح table track کرد.) این information فقط برای مدتی لازم است: پس از پایان یک transaction—چه commit شده باشد و چه abort—و پس از پایان تمام transactionهای concurrent، database می‌تواند آنچه را که transaction read کرده فراموش کند.

وقتی transactionای در database write می‌کند، باید در indexها به دنبال transactionهای دیگری بگردد که اخیراً data تحت تأثیر write را read کرده‌اند. این process شبیه acquire کردن write lock روی key range تحت تأثیر است، اما به‌جای block کردن تا commit شدن readerها، lock مانند یک **tripwire** عمل می‌کند: فقط به transactionها notify می‌دهد که dataای که read کرده‌اند ممکن است دیگر up to date نباشد.

در شکل ۷-۱۱، transaction ۴۳ به transaction ۴۲ اطلاع می‌دهد که read قبلی آن outdated شده است و برعکس. Transaction ۴۲ زودتر commit می‌شود و موفق است: با اینکه write مربوط به transaction ۴۳ روی transaction ۴۲ اثر گذاشته، transaction ۴۳ هنوز commit نشده است و بنابراین write آن هنوز effect خود را نگذاشته است. اما وقتی transaction ۴۳ می‌خواهد commit شود، write متعارض transaction ۴۲ قبلاً commit شده است؛ بنابراین ۴۳ باید abort شود.

#### Performance of Serializable Snapshot Isolation

مانند همیشه، جزئیات engineering زیادی بر عملکرد algorithm در عمل اثر می‌گذارند. برای مثال، یکی از trade-offها granularityای است که با آن read و writeهای transactionها track می‌شوند. اگر database activity هر transaction را با جزئیات زیادی track کند، می‌تواند دقیق‌تر تعیین کند کدام transactionها باید abort شوند، اما bookkeeping overhead ممکن است significant شود. Tracking با جزئیات کمتر سریع‌تر است، اما ممکن است باعث شود transactionهایی بیشتر از مقدار strictly necessary abort شوند.

در بعضی caseها مشکلی ندارد که transaction informationای را read کند که transaction دیگری آن را overwrite کرده است؛ بسته به اتفاق‌های دیگر، گاهی می‌توان ثابت کرد result execution همچنان serializable است. PostgreSQL از این theory برای کاهش abortهای غیرضروری استفاده می‌کند [11, 41].

مزیت بزرگ serializable snapshot isolation در مقایسه با two-phase locking این است که یک transaction لازم نیست برای lockهای در اختیار transaction دیگر block شود. مانند snapshot isolation، writerها readerها را block نمی‌کنند و readerها نیز writerها را block نمی‌کنند. این design principle باعث می‌شود query latency predictableتر و variable بودن آن کمتر شود. به‌خصوص، read-only queryها می‌توانند روی snapshot consistent و بدون نیاز به هیچ lockی اجرا شوند؛ ویژگی‌ای که برای workloadهای read-heavy بسیار جذاب است.

در مقایسه با serial execution، serializable snapshot isolation به throughput یک CPU core محدود نیست: FoundationDB detection مربوط به serialization conflict را میان چند machine distribute می‌کند و اجازه می‌دهد system به throughput بسیار بالا scale شود. حتی اگر data میان چند machine partition شده باشد، transactionها می‌توانند data را در چند partition read و write کنند و همچنان serializable isolation را حفظ کنند [54].

نرخ abortها اثر significantی بر performance کلی SSI دارد. برای مثال، transactionای که data را در مدت طولانی read و write می‌کند، احتمالاً با conflict روبه‌رو و abort می‌شود؛ بنابراین SSI نیاز دارد read-write transactionها نسبتاً کوتاه باشند. (Read-only transactionهای طولانی‌مدت ممکن است مشکلی نداشته باشند.) بااین‌حال، احتمالاً SSI نسبت به transactionهای کند، حساسیت کمتری از two-phase locking یا serial execution دارد.

## Key Terms

- `Serializability` — guaranteeای که result transactionهای concurrent را معادل اجرای serial آن‌ها می‌کند.
- `Serial Execution` — اجرای transactionها یکی‌یکی و روی یک thread یا execution stream.
- `Serializable Transaction` — transactionای که execution آن با یک schedule serial سازگار است.
- `Two-Phase Locking (2PL)` — protocolای که lockها را تا پایان transaction نگه می‌دارد و shared و exclusive access را برای serializability کنترل می‌کند.
- `Shared Lock` — lockای که چند reader می‌توانند هم‌زمان داشته باشند، اما با exclusive writer تعارض دارد.
- `Exclusive Lock` — lockای که دسترسی دیگر transactionها را تا release شدن محدود می‌کند.
- `Predicate Lock` — lock روی تمام objectهایی که با یک search condition match می‌شوند، حتی phantomهای آینده.
- `Index-Range Lock` — approximation کم‌هزینه‌تر predicate lock که روی rangeای از index اعمال می‌شود.
- `Serializable Snapshot Isolation (SSI)` — optimistic concurrency control مبتنی بر snapshot isolation که serialization conflictها را detect می‌کند.
- `Serialization Conflict` — تعارضی که نشان می‌دهد ترتیب concurrent operationها با هیچ schedule serial سازگار نیست.
- `Pessimistic Concurrency Control` — رویکردی که پیش از احتمال failure یا conflict، transaction را block می‌کند.
- `Optimistic Concurrency Control` — رویکردی که transaction را ادامه می‌دهد و در زمان commit conflict را check می‌کند.
- `Transaction Scheduling` — تعیین ترتیب و interleaving اجرای operationهای transactionها.
- `Deadlock` — انتظار چرخه‌ای چند transaction برای lockهایی که یکدیگر در اختیار دارند.
- `Lock Contention` — رقابت transactionها برای acquire کردن lock یکسان.
- `Lock Upgrade` — تبدیل shared lock به exclusive lock پس از read و پیش از write.
- `Stored Procedure` — code کامل transaction که از قبل به database submit می‌شود و درون database اجرا می‌گردد.
- `Cross-Partition Transaction` — transactionای که به data چند partition دسترسی دارد و به coordination میان آن‌ها نیازمند است.
- `Performance Overhead` — هزینهٔ اضافی CPU، memory، network یا coordination ناشی از mechanismهای consistency و concurrency control.
- `Tripwire` — mechanismی که conflict احتمالی را notify می‌کند، بدون اینکه transactionهای دیگر را مستقیماً block کند.

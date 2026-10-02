# Chapter 7 — Transactions

## The Slippery Concept of a Transaction

تقریباً تمام relational databaseهای امروزی و بعضی nonrelational databaseها از transaction پشتیبانی می‌کنند. بیشتر آن‌ها از سبکی پیروی می‌کنند که IBM System R، نخستین SQL database، در سال ۱۹۷۵ معرفی کرد [1, 2, 3]. با اینکه بعضی جزئیات implementation تغییر کرده‌اند، ایدهٔ کلی طی ۴۰ سال تقریباً ثابت مانده است: transaction support در MySQL، PostgreSQL، Oracle، SQL Server و غیره شباهت عجیبی به System R دارد.

در اواخر دههٔ ۲۰۰۰، nonrelational یا NoSQL databaseها محبوب شدند. آن‌ها می‌خواستند با ارائهٔ data modelهای جدید (به Chapter 2 مراجعه کنید) و با ارائهٔ replication (Chapter 5) و partitioning (Chapter 6) به‌صورت پیش‌فرض، وضعیت موجود relational را بهبود دهند. Transactionها قربانی اصلی این movement بودند: بسیاری از databaseهای نسل جدید transactionها را کاملاً کنار گذاشتند یا این واژه را برای توصیف مجموعه‌ای بسیار ضعیف‌تر از guaranteeهایی که پیش‌تر از آن understood می‌شد به‌کار بردند [4].

هم‌زمان با hype مربوط به این نسل جدید از distributed databaseها، این باور عمومی شکل گرفت که transactionها antithesis مربوط به scalability هستند و هر system بزرگ‌مقیاسی برای حفظ performance خوب و High Availability باید transactionها را کنار بگذارد [5, 6]. از سوی دیگر، database vendorها گاهی transactional guaranteeها را برای «applicationهای جدی» با «data ارزشمند» یک requirement ضروری معرفی می‌کردند. هر دو دیدگاه pure hyperbole هستند.

حقیقت به این سادگی نیست: transactionها نیز مانند هر design choice فنی دیگری مزایا و محدودیت‌هایی دارند. برای درک این trade-offها، باید جزئیات guaranteeهایی را که transactionها می‌توانند ارائه کنند—هم در operation عادی و هم در شرایط extreme اما واقع‌گرایانه—بررسی کنیم.

### The Meaning of ACID

Safety guaranteeهایی که transactionها فراهم می‌کنند، اغلب با acronym شناخته‌شدهٔ **ACID** توصیف می‌شوند که از Atomicity، Consistency، Isolation و Durability تشکیل شده است. این acronym را Theo Härder و Andreas Reuter در سال ۱۹۸۳ ابداع کردند [7] تا terminology دقیقی برای mechanismهای Fault Tolerance در databaseها ایجاد کنند.

بااین‌حال، در عمل implementation مربوط به ACID در یک database با implementation آن در database دیگر برابر نیست. برای مثال، همان‌طور که خواهیم دید، دربارهٔ معنای Isolation ابهام زیادی وجود دارد [8]. ایدهٔ سطح‌بالا sound است، اما جزئیات تعیین‌کننده‌اند. امروزه وقتی systemی ادعا می‌کند «ACID compliant» است، مشخص نیست دقیقاً چه guaranteeهایی می‌توان از آن انتظار داشت. متأسفانه ACID تا حد زیادی به یک اصطلاح marketing تبدیل شده است.

Systemهایی که معیارهای ACID را برآورده نمی‌کنند گاهی **BASE** نامیده می‌شوند؛ BASE مخفف Basically Available، Soft state و Eventual consistency است [9]. این تعریف حتی از ACID نیز مبهم‌تر است. به نظر می‌رسد تنها تعریف sensible برای BASE این باشد: «not ACID»؛ یعنی می‌تواند تقریباً هر معنایی داشته باشد.

بیایید تعریف‌های Atomicity، Consistency، Isolation و Durability را با جزئیات بررسی کنیم؛ این کار به ما اجازه می‌دهد ایدهٔ خود از transaction را دقیق‌تر کنیم.

#### Atomicity

به‌طور کلی، atomic به چیزی گفته می‌شود که نمی‌توان آن را به بخش‌های کوچک‌تر تقسیم کرد. این واژه در شاخه‌های مختلف computing معناهای مشابه اما اندکی متفاوتی دارد. برای مثال، در multi-threaded programming، اگر یک thread یک operation اتمیک اجرا کند، هیچ راهی وجود ندارد که thread دیگری result نیمه‌تمام آن operation را ببیند. system یا در state قبل از operation قرار دارد یا در state بعد از آن؛ نه چیزی بین این دو.

در مقابل، در context مربوط به ACID، Atomicity دربارهٔ concurrency نیست. این مفهوم توضیح نمی‌دهد وقتی چند process هم‌زمان تلاش می‌کنند به یک data دسترسی پیدا کنند چه اتفاقی رخ می‌دهد، چون آن موضوع زیر letter مربوط به Isolation پوشش داده می‌شود (به بخش «Isolation» در صفحهٔ ۲۲۵ مراجعه کنید).

Atomicity در ACID دربارهٔ این است که اگر client بخواهد چند write انجام دهد و پس از پردازش بعضی از آن writeها faultی رخ دهد چه اتفاقی باید بیفتد؛ برای مثال، process crash کند، network connection قطع شود، disk پر شود یا یک integrity constraint نقض شود. اگر writeها در یک atomic transaction گروه‌بندی شده باشند و transaction به‌دلیل fault نتواند complete یا commit شود، transaction abort می‌شود و database باید تمام writeهایی را که تا آن لحظه در آن transaction انجام داده discard یا undo کند.

بدون Atomicity، اگر هنگام ایجاد چند change در میانهٔ کار error رخ دهد، تشخیص اینکه کدام changeها اعمال شده‌اند و کدام اعمال نشده‌اند دشوار است. application می‌تواند دوباره تلاش کند، اما این کار خطر اجرای دوبارهٔ همان change را به‌همراه دارد و ممکن است به data duplicate یا incorrect منجر شود. Atomicity این مسئله را ساده می‌کند: اگر transaction abort شده باشد، application مطمئن است که هیچ تغییری ایجاد نکرده است و می‌تواند با خیال راحت آن را retry کند.

توانایی abort کردن transaction هنگام error و discard کردن تمام writeهای آن transaction، ویژگی تعیین‌کنندهٔ Atomicity در ACID است. شاید **abortability** واژهٔ بهتری از Atomicity بود، اما چون Atomicity واژهٔ رایج است، همان را به‌کار می‌بریم.

#### Consistency

واژهٔ consistency بیش از حد overloaded است:

- در Chapter 5 دربارهٔ replica consistency و مسئلهٔ eventual consistency که در systemهای asynchronously replicated ایجاد می‌شود صحبت کردیم (به بخش «Problems with Replication Lag» در صفحهٔ ۱۶۱ مراجعه کنید).
- Consistent hashing یک approach برای partitioning است که بعضی systemها برای rebalancing استفاده می‌کنند (به بخش «Consistent Hashing» در صفحهٔ ۲۰۴ مراجعه کنید).
- در CAP theorem (به Chapter 9 مراجعه کنید)، واژهٔ consistency به معنای linearizability به‌کار می‌رود (به بخش «Linearizability» در صفحهٔ ۳۲۴ مراجعه کنید).
- در context مربوط به ACID، consistency به مفهومی application-specific از قرار داشتن database در یک «state خوب» اشاره دارد.

متأسفانه یک واژه برای دست‌کم چهار معنای متفاوت استفاده می‌شود.

ایدهٔ Consistency در ACID این است که statementهای مشخصی دربارهٔ data شما وجود دارد—که به آن‌ها **invariant** می‌گوییم—و این statementها باید همیشه true باشند. برای مثال، در یک accounting system، creditها و debitها در تمام accountها باید همیشه balance باشند. اگر transaction با databaseای شروع شود که بر اساس این invariantها valid است و تمام writeهای transaction نیز validity آن را حفظ کنند، می‌توانید مطمئن باشید که invariantها همیشه برقرار می‌مانند.

بااین‌حال، این ایدهٔ consistency به تعریف application از invariantها وابسته است و این مسئولیت application است که transactionهای خود را به‌درستی تعریف کند تا consistency را حفظ کنند. database نمی‌تواند این موضوع را guarantee کند: اگر data بدی write کنید که invariantهای شما را نقض کند، database نمی‌تواند جلوی آن را بگیرد. (database می‌تواند بعضی invariantهای مشخص را check کند؛ برای مثال با استفاده از foreign key constraint یا uniqueness constraint. اما در حالت کلی، application تعریف می‌کند چه dataای valid یا invalid است و database فقط آن را ذخیره می‌کند.)

Atomicity، Isolation و Durability propertyهای database هستند، اما Consistency—در معنای ACID—property مربوط به application است. application ممکن است برای دستیابی به consistency به propertyهای Atomicity و Isolation database تکیه کند، اما این موضوع فقط بر عهدهٔ database نیست. بنابراین letter مربوط به C در ACID واقعاً نباید بخشی از ACID باشد.i

#### Isolation

بیشتر databaseها به‌صورت هم‌زمان توسط چند client access می‌شوند. اگر clientها بخش‌های متفاوتی از database را read و write کنند، مشکلی وجود ندارد؛ اما اگر به همان database recordها دسترسی داشته باشند، ممکن است با concurrency problem یا race condition روبه‌رو شوید.

شکل ۷-۱ نمونهٔ ساده‌ای از این نوع problem است. فرض کنید دو client به‌صورت هم‌زمان counterای را که در database ذخیره شده increment می‌کنند. هر client باید value فعلی را read کند، ۱ به آن اضافه کند و value جدید را write کند؛ البته فرض می‌کنیم database operation داخلی برای increment ندارد. در شکل ۷-۱، counter باید از ۴۲ به ۴۴ افزایش پیدا کند، چون دو increment رخ داده است، اما به‌دلیل race condition در واقع فقط به ۴۳ می‌رسد.

Isolation در معنای ACID یعنی transactionهایی که به‌صورت concurrent اجرا می‌شوند از یکدیگر isolated باشند و مزاحم یکدیگر نشوند. کتاب‌های کلاسیک database، Isolation را به‌صورت **serializability** formalize می‌کنند؛ یعنی هر transaction می‌تواند فرض کند تنها transaction در حال اجرای کل database است. database تضمین می‌کند پس از commit شدن transactionها، result همان result حالتی باشد که transactionها به‌صورت serial—یکی پس از دیگری—اجرا شده باشند، حتی اگر در reality به‌صورت concurrent اجرا شده باشند [10].

**شکل ۷-۱.** race condition میان دو client که به‌صورت concurrent یک counter را increment می‌کنند.

بااین‌حال، در عمل serializable isolation به‌ندرت استفاده می‌شود، چون هزینهٔ performance دارد. بعضی databaseهای محبوب، مانند Oracle 11g، حتی آن را implement نمی‌کنند. در Oracle یک isolation level به نام «serializable» وجود دارد، اما در واقع چیزی به نام snapshot isolation را implement می‌کند که guarantee ضعیف‌تری از serializability دارد [8, 11]. Snapshot isolation و شکل‌های دیگر Isolation را در بخش «Weak Isolation Levels» در صفحهٔ ۲۳۳ بررسی خواهیم کرد.

#### Durability

هدف database system فراهم کردن مکانی امن برای storage کردن data است؛ مکانی که بتوان data را بدون ترس از دست دادن ذخیره کرد. Durability این promise است که پس از commit موفق یک transaction، هیچ dataای که آن transaction write کرده است فراموش نشود، حتی اگر hardware fault رخ دهد یا database crash کند.

در یک single-node database، Durability معمولاً به این معناست که data روی nonvolatile storage مانند hard drive یا SSD نوشته شده باشد. این کار معمولاً به write-ahead log یا mechanism مشابهی نیز نیاز دارد (به بخش «Making B-trees reliable» در صفحهٔ ۸۲ مراجعه کنید) تا در صورت corrupted شدن data structureهای روی disk امکان recovery وجود داشته باشد. در یک replicated database، Durability ممکن است به این معنا باشد که data با موفقیت روی تعداد مشخصی node copy شده است. برای ارائهٔ Durability guarantee، database باید تا کامل شدن این writeها یا replicationها صبر کند و سپس transaction را successfully committed گزارش دهد.

همان‌طور که در بخش «Reliability» در صفحهٔ ۶ بحث کردیم، Durability کامل وجود ندارد: اگر تمام hard diskها و تمام backupهای شما هم‌زمان نابود شوند، database آشکارا هیچ کاری برای نجات data نمی‌تواند انجام دهد.

#### Replication and Durability

از نظر تاریخی، Durability به معنای write کردن روی archive tape بود. بعدتر به معنای write کردن روی disk یا SSD فهمیده شد. در سال‌های اخیر، این مفهوم برای replication نیز به‌کار رفته است. کدام implementation بهتر است؟

حقیقت این است که هیچ‌چیز perfect نیست:

- اگر data را روی disk write کنید و machine از کار بیفتد، data شما از دست نرفته است، اما تا زمانی که machine را repair کنید یا disk را به machine دیگری منتقل کنید، data inaccessible خواهد بود. systemهای replicated می‌توانند available باقی بمانند.
- یک correlated fault—مانند قطعی برق یا bugای که همهٔ nodeها را در برابر input مشخصی crash می‌کند—می‌تواند تمام replicaها را هم‌زمان از کار بیندازد (به بخش «Reliability» در صفحهٔ ۶ مراجعه کنید) و هر dataای را که فقط در memory قرار دارد از بین ببرد. بنابراین write کردن روی disk همچنان برای in-memory databaseها مهم است.
- در یک system با asynchronous replication، وقتی leader unavailable می‌شود ممکن است writeهای اخیر از دست بروند (به بخش «Handling Node Outages» در صفحهٔ ۱۵۶ مراجعه کنید).
- وقتی برق ناگهان قطع می‌شود، به‌خصوص دربارهٔ SSDها نشان داده شده است که گاهی guaranteeهایی را که supposedly باید ارائه کنند نقض می‌کنند؛ حتی `fsync` نیز تضمین نمی‌کند که همیشه درست کار کند [12]. firmware مربوط به disk نیز مانند هر software دیگری ممکن است bug داشته باشد [13, 14].
- interactionهای ظریف میان storage engine و filesystem implementation می‌توانند به bugهایی منجر شوند که پیدا کردنشان دشوار است و ممکن است پس از crash باعث corrupted شدن fileهای روی disk شوند [15, 16].
- data روی disk ممکن است به‌تدریج corrupt شود، بدون اینکه این corruption detect شود [17]. اگر data برای مدتی corrupt بوده باشد، replicaها و backupهای اخیر نیز ممکن است corrupt شده باشند. در این حالت باید تلاش کنید data را از یک historical backup restore کنید.
- یک مطالعه دربارهٔ SSDها نشان داد که بین ۳۰٪ تا ۸۰٪ driveها در چهار سال اول operation دست‌کم یک bad block ایجاد می‌کنند [18]. magnetic hard driveها نرخ پایین‌تری از bad sector دارند، اما نرخ failure کامل آن‌ها از SSDها بیشتر است.
- اگر یک SSD از power جدا شود، بسته به temperature ممکن است ظرف چند هفته شروع به از دست دادن data کند [19].

در عمل هیچ technique واحدی نمی‌تواند guaranteeهای absolute فراهم کند. فقط techniqueهای مختلفی برای risk reduction وجود دارند، از جمله write کردن روی disk، replicate کردن روی remote machineها و backup گرفتن؛ این techniqueها باید در کنار هم استفاده شوند و می‌توانند یکدیگر را تکمیل کنند. مانند همیشه، بهتر است هر «guarantee» نظری را با مقدار مناسبی احتیاط بپذیرید.

### Single-Object and Multi-Object Operations

برای جمع‌بندی، در ACID، Atomicity و Isolation توضیح می‌دهند database وقتی client در یک transaction چند write انجام می‌دهد چه رفتاری باید داشته باشد:

**Atomicity**

اگر در میانهٔ یک sequence از writeها error رخ دهد، transaction باید abort شود و writeهایی که تا آن نقطه انجام شده‌اند discard شوند. به بیان دیگر، database با ارائهٔ guaranteeای all-or-nothing شما را از نگرانی دربارهٔ partial failure نجات می‌دهد.

**Isolation**

transactionهایی که به‌صورت concurrent اجرا می‌شوند نباید با یکدیگر interference داشته باشند. برای مثال، اگر یک transaction چند write انجام دهد، transaction دیگری باید یا تمام آن writeها را ببیند یا هیچ‌کدام را؛ نه اینکه فقط subsetی از آن‌ها را مشاهده کند.

این تعریف‌ها فرض می‌کنند که می‌خواهید چند object—مانند row، document یا record—را هم‌زمان تغییر دهید. چنین **multi-object transaction**هایی اغلب زمانی لازم‌اند که چند بخش data باید با یکدیگر sync باقی بمانند. شکل ۷-۲ نمونه‌ای از یک email application را نشان می‌دهد.

برای نمایش تعداد messageهای unread یک user می‌توانید queryای مانند query زیر را اجرا کنید:

```sql
SELECT COUNT(*) FROM emails WHERE recipient_id = 2 AND unread_flag = true
```

بااین‌حال، اگر emailهای زیادی وجود داشته باشند، ممکن است این query بیش از حد کند باشد و تصمیم بگیرید تعداد messageهای unread را در یک field جداگانه ذخیره کنید؛ این کار نوعی denormalization است. حالا هر زمان message جدیدی دریافت می‌شود، باید counter مربوط به unread را نیز increment کنید و هر زمان messageای read علامت‌گذاری می‌شود، باید counter مربوط به unread را decrement کنید.

در شکل ۷-۲، user شمارهٔ ۲ با یک anomaly روبه‌رو می‌شود: mailbox listing یک unread message را نشان می‌دهد، اما counter صفر unread message را نشان می‌دهد، چون increment مربوط به counter هنوز انجام نشده است.ii Isolation با تضمین اینکه user شمارهٔ ۲ یا email درج‌شده و counter updateشده را با هم ببیند یا هیچ‌کدام را نبیند—اما نه یک وضعیت نصفه‌نیمه و inconsistent—از این مشکل جلوگیری می‌کرد.

**شکل ۷-۲.** نقض Isolation: یک transaction writeهای commitنشدهٔ transaction دیگری را read می‌کند؛ این وضعیت «dirty read» نام دارد.

شکل ۷-۳ نیاز به Atomicity را نشان می‌دهد: اگر در هر نقطه از اجرای transaction error رخ دهد، ممکن است mailbox و unread counter از sync خارج شوند. در یک atomic transaction، اگر update مربوط به counter fail شود، transaction abort و email درج‌شده rollback می‌شود.

**شکل ۷-۳.** Atomicity تضمین می‌کند اگر error رخ دهد، writeهای قبلی همان transaction undo شوند تا system به state inconsistent نرسد.

Multi-object transactionها به روشی برای تعیین اینکه کدام read و write operationها به یک transaction تعلق دارند نیاز دارند. در relational databaseها این کار معمولاً بر اساس TCP connection مربوط به client با database server انجام می‌شود: روی هر connection مشخص، هر چیزی که میان statementهای `BEGIN TRANSACTION` و `COMMIT` قرار بگیرد بخشی از همان transaction در نظر گرفته می‌شود.iii

از سوی دیگر، بسیاری از nonrelational databaseها راهی برای گروه‌بندی operationها در کنار یکدیگر ندارند. حتی اگر یک multi-object API وجود داشته باشد—برای مثال، یک key-value store ممکن است operationای مانند multi-put داشته باشد که چند key را در یک operation update می‌کند—این موضوع الزاماً به معنای وجود transaction semantics نیست: command ممکن است برای بعضی keyها موفق و برای keyهای دیگر fail شود و database را در stateای partially updated باقی بگذارد.

#### Single-Object Writes

Atomicity و Isolation زمانی که فقط یک object در حال تغییر است نیز کاربرد دارند. برای مثال، تصور کنید یک JSON document بیست کیلوبایتی را در database write می‌کنید:

- اگر network connection پس از ارسال شدن ۱۰ کیلوبایت اول قطع شود، آیا database آن fragment ده‌کیلوبایتی و unparseable از JSON را ذخیره می‌کند؟
- اگر هنگام overwrite کردن value قبلی روی disk، power fail شود، آیا در نهایت ترکیبی spliceشده از value قدیمی و جدید خواهید داشت؟
- اگر client دیگری هنگام انجام write این document را read کند، آیا valueای partially updated را خواهد دید؟

این مسائل بسیار گیج‌کننده خواهند بود؛ بنابراین storage engineها تقریباً همیشه تلاش می‌کنند Atomicity و Isolation را در سطح یک object واحد—مانند یک key-value pair—روی یک node فراهم کنند. Atomicity را می‌توان با استفاده از log برای crash recovery implement کرد (به بخش «Making B-trees reliable» در صفحهٔ ۸۲ مراجعه کنید) و Isolation را می‌توان با lock کردن هر object implement کرد؛ به‌طوری‌که در هر لحظه فقط یک thread اجازهٔ access به object را داشته باشد.

بعضی databaseها operationهای atomic پیچیده‌تری نیز ارائه می‌کنند،iv مانند operation مربوط به increment که نیاز به چرخهٔ read-modify-write مشابه شکل ۷-۱ را از بین می‌برد. operation محبوب دیگر **compare-and-set** است که فقط در صورتی اجازهٔ write می‌دهد که value به‌صورت concurrent توسط فرد دیگری تغییر نکرده باشد (به بخش «Compare-and-set» در صفحهٔ ۲۴۵ مراجعه کنید).

این single-object operationها مفیدند، چون می‌توانند از lost update جلوگیری کنند؛ یعنی زمانی که چند client تلاش می‌کنند به‌صورت concurrent روی یک object یکسان write کنند (به بخش «Preventing Lost Updates» در صفحهٔ ۲۴۲ مراجعه کنید). بااین‌حال، آن‌ها در معنای معمول کلمه transaction نیستند. compare-and-set و دیگر single-object operationها برای اهداف marketing گاهی «lightweight transaction» یا حتی «ACID» نامیده شده‌اند [20, 21, 22]، اما این terminology گمراه‌کننده است. transaction معمولاً mechanismی برای گروه‌بندی چند operation روی چند object در یک واحد execution محسوب می‌شود.

#### The Need for Multi-Object Transactions

بسیاری از distributed datastoreها multi-object transactionها را کنار گذاشته‌اند، چون implement کردن آن‌ها میان partitionها دشوار است و در بعضی scenarioهایی که availability یا performance بسیار بالا لازم است می‌توانند مانع ایجاد کنند. بااین‌حال، هیچ مانع fundamentalای برای transactionها در یک distributed database وجود ندارد و implementation مربوط به distributed transactionها را در Chapter 9 بررسی خواهیم کرد.

اما آیا اصلاً به multi-object transaction نیاز داریم؟ آیا می‌توان هر applicationای را فقط با key-value data model و single-object operationها implement کرد؟

در بعضی use caseها، single-object insert، update و delete کافی هستند. بااین‌حال، در بسیاری از موارد دیگر باید writeهای چند object متفاوت را با یکدیگر coordinate کرد:

- در یک relational data model، یک row در یک table اغلب به rowای در table دیگر با استفاده از foreign key reference می‌دهد. (به‌طور مشابه، در graph-like data model، یک vertex به vertexهای دیگر edge دارد.) Multi-object transaction به شما اجازه می‌دهد مطمئن شوید این referenceها valid باقی می‌مانند: هنگام insert کردن چند record که به یکدیگر reference می‌دهند، foreign keyها باید صحیح و up to date باشند؛ در غیر این صورت data بی‌معنا می‌شود.
- در یک document data model، fieldهایی که باید با هم update شوند اغلب درون یک document قرار دارند و document مانند یک object واحد در نظر گرفته می‌شود؛ بنابراین برای update کردن یک document واحد به multi-object transaction نیاز نداریم. بااین‌حال، document databaseهایی که join functionality ندارند، denormalization را نیز تشویق می‌کنند (به بخش «Relational Versus Document Databases Today» در صفحهٔ ۳۸ مراجعه کنید). وقتی dataی denormalized باید update شود، مانند مثال شکل ۷-۲، لازم است چند document را در یک operation update کنید. Transactionها در این وضعیت برای جلوگیری از out of sync شدن data denormalized بسیار مفیدند.
- در databaseهایی که secondary index دارند—یعنی تقریباً همهٔ databaseها به‌جز pure key-value storeها—هر بار که یک value را change می‌کنید، indexها نیز باید update شوند. این indexها از دید transaction، database objectهای متفاوتی هستند: برای مثال، بدون transaction Isolation ممکن است recordی در یک index ظاهر شود اما در index دیگری ظاهر نشود، چون update مربوط به index دوم هنوز انجام نشده است.

این applicationها را همچنان می‌توان بدون transaction implement کرد. بااین‌حال، بدون Atomicity، error handling بسیار پیچیده‌تر می‌شود و نبود Isolation می‌تواند به concurrency problem منجر شود. این مشکل‌ها را در بخش «Weak Isolation Levels» در صفحهٔ ۲۳۳ بررسی و در Chapter 12 approachهای جایگزین را بررسی خواهیم کرد.

#### Handling Errors and Aborts

یک ویژگی کلیدی transaction این است که اگر error رخ دهد می‌توان آن را abort و با اطمینان retry کرد. ACID databaseها بر این philosophy بنا شده‌اند: اگر database در معرض نقض guarantee مربوط به Atomicity، Isolation یا Durability باشد، ترجیح می‌دهد transaction را کاملاً abandon کند تا اینکه اجازه دهد transaction در وضعیتی نیمه‌تمام باقی بماند.

بااین‌حال، همهٔ systemها از این philosophy پیروی نمی‌کنند. به‌طور خاص، datastoreهایی که از leaderless replication استفاده می‌کنند (به بخش «Leaderless Replication» در صفحهٔ ۱۷۷ مراجعه کنید) بیشتر بر مبنای «best effort» کار می‌کنند. این رویکرد را می‌توان این‌طور خلاصه کرد: «database هرچه بتواند انجام می‌دهد و اگر با error روبه‌رو شود، چیزی را که قبلاً انجام داده undo نمی‌کند»؛ بنابراین recovery از error بر عهدهٔ application است.

Errorها اجتناب‌ناپذیرند، اما بسیاری از software developerها ترجیح می‌دهند فقط به happy path فکر کنند و وارد جزئیات error handling نشوند. برای مثال، ORM frameworkهای محبوبی مانند `Rails ActiveRecord` و `Django` transactionهای abortشده را retry نمی‌کنند؛ error معمولاً به‌صورت exception در stack بالا می‌آید، input مربوط به user دور ریخته می‌شود و user یک error message دریافت می‌کند. این تأسف‌آور است، چون هدف اصلی abort امکان retry امن است.

اگرچه retry کردن transaction abortشده mechanism ساده و مؤثری برای error handling است، perfect نیست:

- اگر transaction در واقع successful شده باشد، اما network هنگام تلاش server برای acknowledge کردن commit موفق به client fail شده باشد—به‌طوری‌که client تصور کند transaction fail شده است—retry کردن transaction باعث می‌شود transaction دو بار اجرا شود؛ مگر اینکه mechanism اضافی برای deduplication در سطح application داشته باشید.
- اگر error به‌دلیل overload رخ داده باشد، retry کردن transaction مشکل را بهتر نمی‌کند و آن را بدتر خواهد کرد. برای جلوگیری از این feedback cycle می‌توانید تعداد retryها را محدود کنید، از **exponential backoff** استفاده کنید و errorهای ناشی از overload را—در صورت امکان—متفاوت از errorهای دیگر handle کنید.
- retry کردن فقط پس از transient error ارزش دارد؛ برای مثال error ناشی از deadlock، isolation violation، temporary network interruption و failover. پس از permanent error—برای مثال constraint violation—retry بی‌فایده است.
- اگر transaction side effectهایی خارج از database نیز داشته باشد، این side effectها ممکن است حتی در صورت abort شدن transaction رخ داده باشند. برای مثال، اگر emailی ارسال می‌کنید، نباید هر بار که transaction را retry می‌کنید دوباره همان email را ارسال کنید. اگر می‌خواهید مطمئن شوید چند system متفاوت یا با هم commit می‌شوند یا با هم abort، two-phase commit می‌تواند کمک کند (این موضوع را در بخش «Atomic Commit and Two-Phase Commit (2PC)» در صفحهٔ ۳۵۴ بررسی خواهیم کرد).
- اگر client process هنگام retry کردن fail شود، هر dataای که تلاش می‌کرد در database write کند از دست خواهد رفت.

## Key Terms

- `Transaction` — واحد منطقی اجرای چند read و write که به‌صورت یک operation all-or-nothing مدیریت می‌شود.
- `ACID` — مجموعه‌ای از guaranteeهای Atomicity، Consistency، Isolation و Durability برای transactionها.
- `BASE` — رویکردی مبهم‌تر از ACID، مبتنی بر Basically Available، Soft state و Eventual consistency.
- `Atomicity` — تضمین اینکه transaction یا به‌طور کامل commit شود یا تمام writeهای آن discard و abort شوند.
- `ACID Consistency` — حفظ invariantهای application-specific که state معتبر database را تعریف می‌کنند.
- `Isolation` — جلوگیری از interference میان transactionهای concurrent و پنهان کردن stateهای میانی آن‌ها.
- `Commit` — نهایی کردن موفق transaction و اعلام اینکه writeهای آن پذیرفته شده‌اند.
- `Abort` — متوقف کردن transaction پیش از تکمیل و discard کردن writeهای آن.
- `Rollback` — undo کردن writeهای transaction abortشده و بازگرداندن database به state پیش از آن transaction.
- `Integrity Constraint` — قاعده‌ای که valid بودن data را محدود می‌کند و ممکن است نقض شدن آن transaction را fail کند.
- `Referential Integrity` — حفظ valid بودن referenceهایی مانند foreign key میان recordها یا tableها.
- `Uniqueness Constraint` — constraintای که duplicate نبودن یک value یا key را تضمین می‌کند.
- `Invariant` — شرطی که باید در تمام stateهای معتبر database برقرار بماند.
- `Concurrency` — اجرای هم‌زمان operationها یا transactionهای متعدد.
- `Serializability` — guaranteeای که result transactionهای concurrent را معادل اجرای serial آن‌ها می‌کند.
- `Durability` — تضمین باقی ماندن data پس از commit، حتی در صورت crash یا hardware failure.
- `Crash Recovery` — فرایند بازگرداندن database به state معتبر پس از crash با استفاده از log یا mechanismهای مشابه.
- `Single-Object Operation` — operation اتمیک روی یک object مانند یک key-value pair یا document.
- `Multi-Object Transaction` — transactionای که read و write چند row، document یا record را در یک واحد هماهنگ می‌کند.
- `Compare-and-Set` — operationای که فقط در صورت تغییر نکردن concurrent value، write را انجام می‌دهد.
- `Atomic Increment` — incrementای که read-modify-write را به یک operation اتمیک تبدیل می‌کند.
- `Dirty Read` — مشاهدهٔ writeای که transaction مربوط به آن هنوز commit نشده است.
- `Two-Phase Commit (2PC)` — protocol هماهنگ‌کننده‌ای برای commit یا abort کردن چند system به‌صورت مشترک.
- `Atomic Commit` — تصمیم هماهنگ برای اینکه چند participant همگی commit شوند یا همگی abort.
- `Exponential Backoff` — افزایش تدریجی فاصلهٔ زمانی میان retryها برای کاهش فشار هنگام overload یا failure.

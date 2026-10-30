# Chapter 9 — Consistency and Consensus

## Distributed Transactions and Consensus

Consensus یکی از بنیادی‌ترین مسئله‌ها در distributed computing است. در نگاه اول ساده به‌نظر می‌رسد: چند node باید بر سر چیزی توافق کنند. اما systemهای زیادی بر پایهٔ این تصور اشتباه ساخته شده‌اند که حل این مسئله آسان است.

این موضوع با وجود اهمیتش، دیرتر در کتاب مطرح می‌شود؛ چون ظرافت‌های آن را بدون دانش قبلی نمی‌توان درک کرد. فهم consensus در جامعهٔ پژوهشی نیز طی چند دهه و همراه با سوءبرداشت‌های فراوان شکل گرفت. حالا که replication، transactionها، system modelها، linearizability و total order broadcast را بررسی کرده‌ایم، می‌توانیم سراغ آن برویم.

مواردی مانند انتخاب leader و atomic commit به توافق نیاز دارند. در databaseای با single-leader replication، همهٔ nodeها باید بر سر leader توافق کنند؛ وگرنه ممکن است دو node خود را leader بدانند، هر دو write بپذیرند و data از هم واگرا شود. در transactionای که چند node یا partition را دربر می‌گیرد نیز باید همه دربارهٔ نتیجه توافق کنند: همه commit یا همه abort.

در انتخاب leader، network fault ممکن است باعث شود nodeها دربارهٔ رهبر اختلاف نظر پیدا کنند. Consensus از failover نادرست و split brain جلوگیری می‌کند؛ در split brain دو node خود را leader می‌دانند، هر دو write می‌پذیرند و در نتیجه data ناسازگار و حتی گم می‌شود.

Atomic commit نمونه‌ای از consensus است، اما صورت‌بندی آن کمی متفاوت است: transaction فقط وقتی می‌تواند commit شود که تمام participantها رأی commit داده باشند و اگر هر participant نیاز به abort داشته باشد، transaction باید abort شود. Consensus معمولی می‌تواند هر valueای را که یکی از participantها پیشنهاد کرده انتخاب کند. این دو مسئله را می‌توان به یکدیگر تبدیل کرد؛ nonblocking atomic commit از consensus هم دشوارتر است.

نتیجهٔ FLP—به‌نام Fischer، Lynch و Paterson—می‌گوید اگر احتمال crash شدن node وجود داشته باشد، هیچ algorithmی در تمام اجراها قادر به دستیابی به consensus نیست. این نتیجه در asynchronous system model ثابت شده؛ مدلی محدودکننده که algorithm deterministic است و نمی‌تواند از clock یا timeout استفاده کند. اگر algorithm مجاز باشد timeout یا روش دیگری برای شناسایی nodeهای مشکوک به crash را به‌کار بگیرد—حتی اگر گاهی اشتباه کند—consensus قابل‌حل می‌شود. استفاده از random number نیز برای دور زدن ناممکن‌بودن کافی است. پس FLP از نظر نظری مهم است، اما distributed systemها معمولاً در عمل می‌توانند به consensus برسند.

در این بخش ابتدا atomic commit و به‌خصوص Two-Phase Commit (2PC) را بررسی می‌کنیم؛ روش رایجی که در databaseها، messaging systemها و application serverها پیاده‌سازی شده است. 2PC نوعی consensus algorithm است، اما چندان خوب نیست. از محدودیت‌های آن به سمت consensus algorithmهای بهتر، مانند Zab در ZooKeeper و Raft در etcd، پیش می‌رویم.

### Atomic Commit and Two-Phase Commit (2PC)

در فصل ۷ دیدیم هدف transaction atomicity ارائهٔ semantics ساده در برابر failure میان چند write است. نتیجهٔ transaction یا commit موفق است که تمام writeهای آن را durable می‌کند، یا abort است که تمام writeها را rollback می‌کند؛ یعنی خنثی یا دور انداخته می‌شوند.

Atomicity نمی‌گذارد transaction ناموفق، نتیجه‌ای نیمه‌کاره یا stateای نیمه‌به‌روز در database باقی بگذارد. این ویژگی برای multi-object transactionها و databaseهایی که secondary index دارند مهم است. هر secondary index ساختار data جداگانه‌ای از primary data دارد؛ پس با تغییر data، index مربوطه هم باید تغییر کند. Atomicity تضمین می‌کند secondary index با primary data سازگار بماند.

#### از atomic commit تک‌گرهی تا distributed

برای transactionهایی که روی یک database node اجرا می‌شوند، atomicity را معمولاً storage engine پیاده‌سازی می‌کند. وقتی client از node می‌خواهد transaction را commit کند، database ابتدا writeها را durable می‌کند—معمولاً در write-ahead log—و سپس commit record را به log روی disk اضافه می‌کند. اگر database وسط کار crash کند، هنگام راه‌اندازی مجدد transaction از log بازیابی می‌شود: اگر commit record پیش از crash با موفقیت روی disk نوشته شده باشد، transaction commitشده است؛ در غیر این صورت writeهای آن rollback می‌شوند.

پس در یک node، commit شدن transaction به ترتیب durable نوشتن data روی disk وابسته است: اول data و سپس commit record. لحظهٔ تعیین‌کننده، وقتی است که disk نوشتن commit record را تمام می‌کند. پیش از آن هنوز ممکن است transaction به‌دلیل crash abort شود؛ پس از آن transaction commit شده است، حتی اگر database crash کند. در این حالت یک device—کنترلر یک disk مشخص متصل به یک node مشخص—commit را atomic می‌کند.

اما اگر transaction چند node را درگیر کند چه؟ مثلاً multi-object transaction در database پارتیشن‌شده یا term-partitioned secondary index که index entry آن روی nodeای جدا از primary data قرار دارد. بیشتر distributed datastoreهای NoSQL از این نوع distributed transaction پشتیبانی نمی‌کنند، اما بعضی clustered relational systemها پشتیبانی می‌کنند.

فرستادن commit request به همهٔ nodeها و commit مستقل روی هرکدام کافی نیست؛ ممکن است commit روی بعضی nodeها موفق و روی بعضی دیگر ناموفق شود:

- بعضی nodeها ممکن است constraint violation یا conflict تشخیص دهند و مجبور به abort شوند، درحالی‌که nodeهای دیگر می‌توانند commit کنند.
- ممکن است بعضی commit requestها در network گم شوند و در نهایت به‌دلیل timeout abort شوند، اما requestهای دیگر برسند.
- بعضی nodeها ممکن است پیش از کامل‌شدن نوشتن commit record crash کنند و هنگام recovery، transaction را rollback کنند؛ درحالی‌که nodeهای دیگر موفق به commit شده‌اند.

اگر بعضی nodeها commit و بقیه abort کنند، state آن‌ها ناسازگار می‌شود. پس از commit شدن transaction روی یک node نیز دیگر نمی‌توان آن را پس گرفت، حتی اگر بعداً معلوم شود روی node دیگری abort شده است. به همین دلیل یک node فقط وقتی باید commit کند که مطمئن باشد تمام nodeهای دیگر transaction نیز commit خواهند کرد.

Commit باید برگشت‌ناپذیر باشد: پس از commit نمی‌توان نظر را عوض کرد و transaction را به‌صورت بازنگرانه abort کرد. پس از commit، data برای transactionهای دیگر قابل‌مشاهده است و clientهای دیگر ممکن است به آن تکیه کنند؛ این اصل پایهٔ read committed isolation است. اگر transaction پس از commit قابل abort بود، transactionهایی که data آن را خوانده‌اند بر پایهٔ dataای قرار می‌گرفتند که بعداً اعلام شده هرگز وجود نداشته است؛ در نتیجه آن‌ها هم باید برگردانده می‌شدند.

البته می‌توان اثر transaction commitشده را بعداً با یک compensating transaction خنثی کرد. اما از دید database این یک transaction جداگانه است و هر نیازمندی مربوط به correctness میان دو transaction بر عهدهٔ application است.

#### آشنایی با Two-Phase Commit

Two-Phase Commit algorithmی برای atomic commit میان چند node است؛ یعنی تضمین می‌کند یا همهٔ nodeها commit کنند یا همه abort. 2PC در بعضی databaseها داخلی استفاده می‌شود و از طریق XA transaction یا WS-AtomicTransaction برای SOAP web serviceها نیز در دسترس applicationها قرار دارد.

2PC به‌جای یک commit request، فرایند commit/abort را به دو phase تقسیم می‌کند. component تازه‌ای به‌نام **coordinator** (یا transaction manager) وارد کار می‌شود. coordinator اغلب libraryای در همان application process است که transaction را درخواست کرده، اما می‌تواند process یا service جداگانه‌ای هم باشد؛ از نمونه‌های آن Narayana، JOTM، BTM و MSDTC هستند.

ابتدا application طبق معمول data را از چند database node می‌خواند و در آن‌ها می‌نویسد. این nodeها **participant** transaction هستند. وقتی application آمادهٔ commit شد:

1. در **phase 1 (prepare)**، coordinator از همهٔ participantها می‌پرسد آیا می‌توانند commit کنند.
2. اگر همه پاسخ «بله» بدهند و آمادگی commit را اعلام کنند، coordinator در **phase 2** به همه commit request می‌فرستد.
3. اگر حتی یکی از participantها پاسخ «نه» بدهد، coordinator در phase 2 به همه abort request می‌فرستد.

این فرایند را می‌توان به مراسم ازدواج سنتی تشبیه کرد: برگزارکننده جداگانه از دو طرف می‌پرسد آیا با ازدواج موافق‌اند. پس از دریافت هر دو پاسخ مثبت، ازدواج را اعلام می‌کند و خبر به حاضران می‌رسد؛ اگر یکی «بله» نگوید، مراسم لغو می‌شود.

2PC را با Two-Phase Locking (2PL) اشتباه نگیرید: 2PC در distributed database atomic commit فراهم می‌کند، اما 2PL serializable isolation فراهم می‌کند. این دو مفهوم مستقل‌اند.

#### سازوکار promiseها

چرا 2PC atomicity را حفظ می‌کند، درحالی‌که یک‌مرحله‌ای commit کردن چند node این کار را نمی‌کند؟ مگر prepare و commit requestها هم ممکن نیست در network گم شوند؟ پاسخ در promiseهایی است که protocol برقرار می‌کند:

1. وقتی application می‌خواهد distributed transactionای را آغاز کند، از coordinator یک transaction ID سراسری و یکتا می‌گیرد.
2. Application روی هر participant یک single-node transaction آغاز می‌کند و transaction ID یکتا را به آن الصاق می‌کند. تمام read و writeها درون همین transactionهای محلی انجام می‌شوند. اگر در این مرحله مشکلی رخ دهد—مثلاً node crash یا request timeout شود—coordinator یا هر participant می‌تواند transaction را abort کند.
3. وقتی application آمادهٔ commit شد، coordinator برای تمام participantها prepare requestای با transaction ID سراسری می‌فرستد. اگر هرکدام از requestها شکست بخورد یا timeout شود، coordinator برای همان ID به همه abort request می‌فرستد.
4. Participant پس از دریافت prepare باید مطمئن شود تحت هر شرایطی می‌تواند commit کند. این کار شامل نوشتن تمام transaction data روی disk و بررسی conflictها و constraint violationهاست. crash، قطعی برق یا پرشدن disk بعداً عذر قابل‌قبولی برای commit نکردن نیست. با پاسخ «بله»، node قول می‌دهد اگر از او خواسته شد، transaction را بدون خطا commit کند. به‌عبارتی حق abort کردن را واگذار می‌کند، اما هنوز commit نکرده است.
5. پس از دریافت پاسخ همهٔ participantها، coordinator تصمیم قطعی commit یا abort را می‌گیرد؛ فقط وقتی commit می‌کند که همه رأی «بله» داده باشند. این تصمیم را روی disk در transaction log خود می‌نویسد تا اگر بعداً crash کرد بداند چه تصمیمی گرفته است. این لحظه **commit point** است.
6. پس از ثبت تصمیم روی disk، coordinator درخواست commit یا abort را برای همهٔ participantها می‌فرستد. اگر request شکست بخورد یا timeout شود، باید تا موفقیت، آن را retry کند. دیگر بازگشتی وجود ندارد: تصمیم commit باید اجرا شود، حتی اگر retryهای زیادی لازم باشد. اگر participant در این فاصله crash کرده باشد، پس از recovery transaction را commit می‌کند؛ چون قبلاً رأی «بله» داده و دیگر نمی‌تواند از commit سر باز زند.

پس protocol دو «نقطهٔ بی‌بازگشت» دارد: participant با رأی «بله» قول می‌دهد بعداً بتواند commit کند، هرچند coordinator هنوز می‌تواند abort را انتخاب کند؛ و تصمیم coordinator پس از ثبت‌شدن برگشت‌ناپذیر است. همین promiseها atomicity مربوط به 2PC را تضمین می‌کنند. در single-node atomic commit این دو رویداد در نوشتن یک commit record در transaction log ادغام شده‌اند.

در تشبیه ازدواج، پیش از گفتن «بله» هنوز می‌توانید با «هرگز!» از transaction خارج شوید. اما پس از گفتن «بله» نمی‌توانید گفته‌تان را پس بگیرید. اگر بعد از آن بیهوش شوید و اعلام نهایی برگزارکننده را نشنوید، transaction همچنان commit شده است. بعد از به‌هوش‌آمدن می‌توانید با پرس‌وجو از coordinator دربارهٔ وضعیت transaction ID سراسری بفهمید ازدواج انجام شده یا منتظر retry بعدی commit request بمانید.

#### Failure در coordinator

اگر prepare request شکست بخورد یا timeout شود، coordinator transaction را abort می‌کند؛ اگر commit یا abort request شکست بخورد، آن را بی‌نهایت retry می‌کند. اما crash کردن خود coordinator مشکل‌ساز است.

اگر coordinator پیش از ارسال prepare request از کار بیفتد، participant می‌تواند transaction را با اطمینان abort کند. اما وقتی participant prepare request را دریافت کرده و «بله» گفته باشد، دیگر نمی‌تواند به‌تنهایی abort کند؛ باید منتظر بماند coordinator بگوید transaction commit شده یا abort. اگر coordinator crash کند یا network از کار بیفتد، participant کاری جز انتظار ندارد. transaction آن در وضعیت **in doubt** یا uncertain است.

ممکن است coordinator در واقع تصمیم به commit گرفته و commit request را به database 2 رسانده باشد، اما پیش از فرستادن request به database 1 crash کند. database 1 نمی‌داند commit کند یا abort. حتی timeout هم کمکی نمی‌کند: اگر database 1 به‌تنهایی abort کند، با database 2 که commit کرده ناسازگار می‌شود. commit کردن مستقل نیز امن نیست، چون participant دیگری شاید abort کرده باشد.

بدون پاسخ coordinator، participant راهی برای دانستن تصمیم ندارد. در تئوری participantها می‌توانند با هم ارتباط بگیرند و رأی‌های یکدیگر را بررسی کنند، اما چنین کاری جزئی از 2PC نیست. تنها راه تکمیل 2PC انتظار برای recovery coordinator است. به همین دلیل coordinator باید پیش از ارسال commit یا abort request، تصمیم خود را در transaction log روی disk ثبت کند. هنگام بازیابی، log وضعیت transactionهای in-doubt را مشخص می‌کند؛ transactionهایی که commit record ندارند abort می‌شوند. بنابراین commit point در 2PC در نهایت به single-node atomic commit روی coordinator وابسته است.

#### Three-Phase Commit

2PC را **blocking atomic commit protocol** می‌نامند، چون اگر coordinator از کار بیفتد، participantها ممکن است در انتظار recovery آن گیر کنند. از نظر نظری می‌توان protocol nonblocking ساخت که با failure یک node متوقف نشود، اما عملی‌کردن آن ساده نیست.

به‌عنوان جایگزین، الگوریتم Three-Phase Commit (3PC) پیشنهاد شده است. اما 3PC فرض می‌کند network delay و زمان پاسخ nodeها bounded هستند. در systemهای عملی که network delay و process pause می‌توانند unbounded باشند، 3PC نمی‌تواند atomicity را تضمین کند.

به‌طور کلی، nonblocking atomic commit به **perfect failure detector** نیاز دارد؛ یعنی سازوکاری قابل‌اعتماد برای تشخیص crash کردن node. در network با delay نامحدود، timeout failure detector قابل‌اعتمادی نیست، چون request ممکن است به‌دلیل مشکل network timeout شود، حتی اگر هیچ nodeای crash نکرده باشد. به همین علت با وجود مشکل شناخته‌شدهٔ coordinator failure، همچنان از 2PC استفاده می‌شود.

### Distributed Transactions in Practice

Distributed transactionها، به‌ویژه آن‌هایی که با 2PC پیاده‌سازی شده‌اند، reputation دوگانه‌ای دارند. از یک سو safety guarantee مهمی می‌دهند که دستیابی به آن دشوار است؛ از سوی دیگر به‌خاطر مشکل‌های عملیاتی، افت شدید performance و وعده‌هایی فراتر از توانشان مورد انتقادند. بسیاری از cloud serviceها به‌دلیل مشکل‌های عملیاتی سراغ distributed transaction نمی‌روند.

بعضی پیاده‌سازی‌ها جریمهٔ performance سنگینی دارند؛ گزارش شده distributed transaction در MySQL بیش از ۱۰ برابر transaction تک‌گرهی کندتر است. بخش زیادی از هزینهٔ 2PC ناشی از disk forcing اضافی (`fsync`) برای crash recovery و round-tripهای بیشتر در network است.

بااین‌حال، نباید بی‌درنگ distributed transaction را کنار گذاشت؛ ابتدا باید دو نوع متفاوت آن را از هم جدا کرد:

- **Database-internal distributed transaction:** بعضی distributed databaseها که در پیکربندی عادی از replication و partitioning استفاده می‌کنند، از transaction داخلی میان nodeهای خود پشتیبانی می‌کنند. برای نمونه، VoltDB و storage engine با نام MySQL Cluster NDB چنین قابلیتی دارند. تمام nodeهای شرکت‌کننده یک database software واحد را اجرا می‌کنند.
- **Heterogeneous distributed transaction:** participantها دو یا چند فناوری متفاوت‌اند؛ مثلاً databaseهای vendorهای مختلف یا حتی systemهای غیر database مانند message brokerها. باید atomic commit میان این systemهای متفاوت تضمین شود.

transaction داخلی database لازم نیست با systemهای دیگر سازگار باشد؛ پس می‌تواند protocol دلخواه و optimizationهای مخصوص همان فناوری را به‌کار ببرد و معمولاً بهتر عمل می‌کند. transactionهایی که فناوری‌های ناهمگون را دربر می‌گیرند دشوارترند.

#### پردازش message دقیقاً یک‌بار

Heterogeneous distributed transaction می‌تواند systemهای گوناگون را به‌شکل قدرتمندی یکپارچه کند. برای مثال، message queue فقط وقتی message را processed acknowledge می‌کند که transaction مربوط به پردازش آن در database با موفقیت commit شده باشد. این کار با commit اتمیک acknowledgment پیام و writeهای database در یک transaction انجام می‌شود؛ حتی اگر message broker و database فناوری‌های نامرتبطی باشند و روی machineهای جدا اجرا شوند.

اگر تحویل message یا database transaction شکست بخورد، هر دو abort می‌شوند و message broker می‌تواند با اطمینان message را دوباره تحویل دهد. با commit اتمیک خود message و side effectهای پردازش آن، می‌توان تضمین کرد message عملاً دقیقاً یک‌بار پردازش شود؛ حتی اگر پیش از موفقیت چند retry لازم شده باشد. Abort اثرهای transaction نیمه‌کاره را کنار می‌گذارد.

این روش فقط وقتی ممکن است که همهٔ systemهای متأثر از transaction از یک atomic commit protocol مشترک استفاده کنند. مثلاً اگر side effect پردازش message ارسال email باشد و email server از 2PC پشتیبانی نکند، با شکست و retry پردازش ممکن است email چند بار ارسال شود. اما اگر تمام side effectها هنگام abort rollback شوند، می‌توان پردازش را با اطمینان retry کرد، انگار هیچ اتفاقی نیفتاده است.

#### XA transactionها

استاندارد X/Open XA (مخفف eXtended Architecture) برای پیاده‌سازی 2PC میان فناوری‌های ناهمگون تعریف شده است. این استاندارد در سال ۱۹۹۱ معرفی شد و بسیاری از relational databaseهای سنتی—از جمله PostgreSQL، MySQL، DB2، SQL Server و Oracle—و message brokerهایی مانند ActiveMQ، HornetQ، MSMQ و IBM MQ از آن پشتیبانی می‌کنند.

XA network protocol نیست؛ C APIای برای ارتباط با transaction coordinator است. برای زبان‌های دیگر binding وجود دارد. مثلاً در Java EE، XA transaction از طریق Java Transaction API (JTA) پیاده‌سازی می‌شود؛ JTA نیز از سوی driverهای database مبتنی بر Java Database Connectivity (JDBC) و driverهای message broker مبتنی بر Java Message Service (JMS) پشتیبانی می‌شود.

XA فرض می‌کند application از network driver یا client library برای ارتباط با databaseها یا messaging serviceهای participant استفاده می‌کند. اگر driver از XA پشتیبانی کند، از XA API می‌پرسد آیا operation باید بخشی از distributed transaction باشد؛ اگر بله، اطلاعات لازم را به database server می‌فرستد. Driver همچنین callbackهایی در اختیار coordinator می‌گذارد تا از participant بخواهد prepare، commit یا abort کند.

Transaction coordinator، XA API را پیاده‌سازی می‌کند. استاندارد نحوهٔ پیاده‌سازی coordinator را تعیین نمی‌کند؛ در عمل اغلب libraryای است که در همان process application بارگذاری می‌شود، نه serviceای جداگانه. coordinator participantهای هر transaction را track می‌کند، پاسخ‌های آن‌ها به prepare را جمع‌آوری می‌کند و با log روی disk محلی، تصمیم commit یا abort را برای هر transaction نگه می‌دارد.

اگر application process یا machine آن crash کند، coordinator نیز از دسترس خارج می‌شود. participantهایی که transaction آماده‌شده اما commitنشده دارند در وضعیت in doubt می‌مانند. چون log coordinator روی disk محلی application server است، server باید دوباره راه‌اندازی شود و library coordinator log را بخواند تا نتیجهٔ transactionها را بازیابی کند. سپس coordinator از طریق callbackهای XA در driver به participantها می‌گوید commit یا abort کنند. Database server نمی‌تواند مستقیماً با coordinator تماس بگیرد؛ همهٔ ارتباط باید از client library عبور کند.

#### نگه‌داشتن lockها در وضعیت in doubt

مشکل transaction گیرکرده در وضعیت in doubt از lockها ناشی می‌شود. Database transactionها معمولاً برای جلوگیری از dirty write روی rowهایی که تغییر می‌دهند row-level exclusive lock می‌گیرند. اگر serializable isolation بخواهید، database مبتنی بر Two-Phase Locking روی rowهایی که transaction می‌خواند shared lock هم می‌گیرد.

Database تا commit یا abort شدن transaction نمی‌تواند این lockها را آزاد کند. در نتیجه، 2PC باعث می‌شود transaction در کل مدتی که in doubt است lockها را نگه دارد. اگر coordinator crash کند و ۲۰ دقیقه طول بکشد تا بالا بیاید، lockها هم ۲۰ دقیقه نگه داشته می‌شوند. اگر log coordinator کاملاً از دست برود، lockها ممکن است برای همیشه—یا تا حل دستی مدیر system—باقی بمانند.

در این مدت transaction دیگری نمی‌تواند آن rowها را تغییر دهد و بسته به database، حتی ممکن است نتواند آن‌ها را بخواند. پس بخش‌های دیگری از application که به همان data نیاز دارند block می‌شوند و بخش بزرگی از application تا حل‌شدن transaction in-doubt ممکن است unavailable بماند.

#### بازیابی از failure در coordinator

از نظر نظری coordinator پس از restart باید state را از log بازیابی و transactionهای in-doubt را حل کند. اما در عمل transactionهای یتیم in-doubt هم رخ می‌دهند؛ مثلاً coordinator به‌دلیل گم‌شدن log یا خراب‌شدن آن در اثر software bug نمی‌تواند نتیجه را تعیین کند. این transactionها خودکار حل نمی‌شوند و می‌توانند برای همیشه lock نگه دارند و transactionهای دیگر را block کنند.

Restart کردن database serverها مشکل را رفع نمی‌کند؛ پیاده‌سازی درست 2PC باید lockهای transaction in-doubt را حتی پس از restart حفظ کند، وگرنه atomicity را به‌خطر می‌اندازد. راه خروج این است که administrator دستی دربارهٔ commit یا rollback تصمیم بگیرد: participantهای هر transaction را بررسی کند، ببیند آیا یکی از آن‌ها قبلاً commit یا abort کرده است و همان نتیجه را روی بقیه اعمال کند. این کار ممکن است تلاش دستی زیادی بخواهد و احتمالاً باید هنگام outage جدی production، زیر فشار و کمبود وقت انجام شود.

بعضی پیاده‌سازی‌های XA راه اضطراری‌ای به‌نام **heuristic decision** دارند: participant می‌تواند بدون تصمیم قطعی coordinator، یک‌طرفه transaction in-doubt را commit یا abort کند. در اینجا heuristic تعبیری ملایم برای احتمال شکستن atomicity است، چون این کار promiseهای 2PC را نقض می‌کند. پس فقط باید برای خروج از وضعیت فاجعه‌بار به‌کار رود، نه در کار عادی.

#### محدودیت‌های distributed transaction

XA مسئلهٔ واقعی و مهم سازگار نگه‌داشتن چند data system را حل می‌کند، اما مشکل‌های عملیاتی مهمی هم ایجاد می‌کند. نکتهٔ کلیدی این است که transaction coordinator خودش نوعی database است که نتیجهٔ transactionها را نگه می‌دارد؛ پس باید با همان دقتی با آن برخورد کرد که با هر database مهم دیگری:

- اگر coordinator فقط روی یک machine اجرا شود و replicate نشده باشد، single point of failure کل system است؛ failure آن باعث می‌شود application serverهای دیگر پشت lockهای transactionهای in-doubt block شوند. بااین‌حال، بسیاری از coordinatorها به‌طور پیش‌فرض highly available نیستند یا replication ابتدایی دارند.
- بسیاری از server-side applicationها با مدل stateless موردعلاقهٔ HTTP ساخته می‌شوند: state پایدار در database می‌ماند و application serverها را می‌توان آزادانه اضافه یا حذف کرد. اما وقتی coordinator بخشی از application server است، logهای آن به state پایدار و حیاتی system تبدیل می‌شوند؛ برای recovery transactionهای in-doubt پس از crash به آن‌ها نیاز است. در نتیجه application server دیگر stateless نیست.
- XA برای سازگاری با data systemهای بسیار متنوع، ناگزیر lowest common denominator است. مثلاً نمی‌تواند deadlock را میان systemهای متفاوت تشخیص دهد، چون لازم است protocol استانداردی برای تبادل اطلاعات lockهای موردانتظار وجود داشته باشد. با SSI نیز کار نمی‌کند، چون تشخیص conflict میان systemهای مختلف به protocol نیاز دارد.
- محدودیت transactionهای distributed داخلی database کمتر است و مثلاً distributed SSI ممکن است؛ اما 2PC برای commit موفق به پاسخ همهٔ participantها نیاز دارد. بنابراین اگر هر بخشی از system خراب باشد، transaction هم شکست می‌خورد. Distributed transactionها تمایل دارند failure را تشدید کنند؛ خلاف هدف ساختن systemهای fault-tolerant.

آیا باید از تلاش برای سازگار نگه‌داشتن چند system دست بکشیم؟ نه؛ روش‌های دیگری هم وجود دارند که بدون دشواری heterogeneous distributed transaction به نتیجهٔ مشابه می‌رسند. اما پیش از آن، consensus را کامل می‌کنیم.

### Fault-Tolerant Consensus

به‌طور غیررسمی، consensus یعنی چند node بر سر چیزی توافق کنند. اگر چند نفر هم‌زمان بخواهند آخرین صندلی هواپیما، یک صندلی مشخص در سالن تئاتر یا username یکسانی را رزرو کنند، consensus algorithm می‌تواند تعیین کند کدام operation ناسازگار با بقیه برنده شود.

صورت‌بندی معمول consensus چنین است: یک یا چند node می‌توانند valueهایی را پیشنهاد دهند و algorithm یکی از آن valueها را انتخاب می‌کند. در مثال صندلی، هر node که درخواست مشتری‌ای را رسیدگی می‌کند ممکن است ID همان مشتری را پیشنهاد دهد و تصمیم مشخص می‌کند چه کسی صندلی را گرفته است.

Consensus algorithm باید این propertyها را داشته باشد:

- **Uniform agreement:** هیچ دو nodeای تصمیم متفاوت نمی‌گیرند.
- **Integrity:** هیچ nodeای بیش از یک‌بار تصمیم نمی‌گیرد.
- **Validity:** اگر nodeای valueای را انتخاب کند، آن value را یکی از nodeها پیشنهاد داده است.
- **Termination:** هر nodeای که crash نکرده باشد، سرانجام تصمیمی می‌گیرد.

Uniform agreement و integrity هستهٔ ایدهٔ consensus هستند: همه بر یک نتیجه توافق می‌کنند و پس از تصمیم‌گیری نظر عوض نمی‌شود. Validity راه‌حل‌های بدیهی و بی‌فایده را رد می‌کند؛ مثلاً algorithmی که هرچه پیشنهاد شده باشد همیشه `null` انتخاب کند، agreement و integrity را دارد اما validity را نه.

اگر fault tolerance مهم نباشد، رعایت سه property نخست آسان است: یک node را «دیکتاتور» کنید تا همهٔ تصمیم‌ها را بگیرد. اما با failure آن node، system دیگر تصمیمی نمی‌گیرد. همین وضعیت را در 2PC دیدیم: اگر coordinator از کار بیفتد، participantهای in-doubt نمی‌توانند دربارهٔ commit یا abort تصمیم بگیرند.

Termination ایدهٔ fault tolerance را دقیق می‌کند: consensus algorithm نباید برای همیشه بی‌عمل بماند و باید پیشرفت کند؛ حتی اگر بعضی nodeها failure داشته باشند، nodeهای دیگر باید به تصمیم برسند. Termination یک liveness property است، درحالی‌که سه مورد دیگر safety property هستند.

System model مربوط به consensus فرض می‌کند node پس از crash ناگهان ناپدید می‌شود و دیگر برنمی‌گردد؛ مانند زلزله‌ای که datacenter را زیر گل‌ولای دفن کند. هر algorithmی که منتظر recovery چنین nodeای بماند، نمی‌تواند termination را تضمین کند. 2PC نیز این شرط را برآورده نمی‌کند.

اگر تمام nodeها crash کنند و هیچ‌کدام فعال نباشند، هیچ algorithmی نمی‌تواند تصمیم بگیرد. تعداد failureهایی که algorithm تحمل می‌کند محدود است: برای تضمین termination دست‌کم باید اکثریت nodeها درست کار کنند. این اکثریت می‌تواند quorum تشکیل دهد. پس termination مشروط به این است که کمتر از نصف nodeها crash کرده یا unreachable باشند.

بااین‌حال، بیشتر پیاده‌سازی‌های consensus حتی اگر اکثریت nodeها از کار بیفتند یا network مشکل جدی داشته باشد، safety propertyها—agreement، integrity و validity—را حفظ می‌کنند. outage بزرگ ممکن است پردازش requestها را متوقف کند، اما نباید باعث تصمیم نامعتبر یا corruption در consensus شود.

بیشتر consensus algorithmها Byzantine fault را فرض نمی‌کنند. اگر nodeای protocol را درست اجرا نکند—مثلاً به nodeهای مختلف messageهای متناقض بفرستد—ممکن است safety propertyها شکسته شوند. می‌توان consensus را تا وقتی کمتر از یک‌سوم nodeها Byzantine باشند در برابر این faultها مقاوم کرد، اما جزئیات آن algorithmها در این کتاب نمی‌گنجد.

#### Consensus algorithmها و total order broadcast

از consensus algorithmهای fault-tolerant شناخته‌شده می‌توان به Viewstamped Replication (VSR)، Paxos، Raft و Zab اشاره کرد. این algorithmها شباهت‌های زیادی دارند، اما یکسان نیستند. جزئیات کاملشان را بررسی نمی‌کنیم؛ دانستن ایده‌های کلی کافی است، مگر اینکه خودتان بخواهید consensus system پیاده‌سازی کنید—کاری که دشوار است و معمولاً توصیه نمی‌شود.

بیشتر این algorithmها مستقیماً مدل رسمی یک value را پیشنهاد و انتخاب نمی‌کنند. در عوض، دنباله‌ای از valueها را انتخاب می‌کنند؛ یعنی total order broadcast پیاده‌سازی می‌کنند. Total order broadcast مستلزم آن است که همهٔ nodeها messageهای یکسان را دقیقاً یک‌بار و با ترتیب یکسان تحویل بگیرند. این معادل چند دور consensus است: در هر دور، nodeها message بعدی موردنظر را پیشنهاد می‌کنند و سپس بر سر message بعدی در total order تصمیم می‌گیرند.

پس total order broadcast معادل دورهای تکراری consensus است و هر تصمیم consensus متناظر با تحویل یک message است:

- به‌دلیل agreement، همهٔ nodeها تصمیم می‌گیرند messageهای یکسان را با ترتیب یکسان تحویل دهند.
- به‌دلیل integrity، messageها duplicate نمی‌شوند.
- به‌دلیل validity، messageها خراب یا از هیچ ساخته نمی‌شوند.
- به‌دلیل termination، messageها گم نمی‌شوند.

Viewstamped Replication، Raft و Zab مستقیماً total order broadcast را پیاده‌سازی می‌کنند؛ چون از اجرای مکرر consensus تک‌مقداری کارآمدتر است. در Paxos، این optimization با نام Multi-Paxos شناخته می‌شود.

#### Single-leader replication و consensus

در فصل ۵ دیدیم single-leader replication همهٔ writeها را به leader می‌فرستد و آن‌ها را با ترتیب یکسان روی followerها apply می‌کند. آیا این همان total order broadcast نیست؟ پس چرا در فصل ۵ به consensus نیاز پیدا نکردیم؟ پاسخ به نحوهٔ انتخاب leader بستگی دارد.

اگر انسان‌های تیم عملیات leader را دستی انتخاب و پیکربندی کنند، عملاً یک «consensus algorithm» دیکتاتوری داریم: فقط یک node اجازهٔ پذیرش write دارد و ترتیب writeها در replication log را تعیین می‌کند. اگر از کار بیفتد، تا وقتی operatorها دستی node دیگری را leader کنند، writeها unavailable هستند. چنین systemی ممکن است در عمل مناسب باشد، اما termination مربوط به consensus را ندارد، چون پیشرفت به مداخلهٔ انسان وابسته است.

بعضی databaseها leader election و failover خودکار دارند و هنگام failure رهبر، follower را به‌عنوان leader جدید ارتقا می‌دهند. این به fault-tolerant total order broadcast و در نتیجه حل consensus نزدیک‌تر است.

اما مسئلهٔ split brain باقی می‌ماند: همهٔ nodeها باید بر سر اینکه leader کیست توافق کنند؛ وگرنه دو node ممکن است هر دو خود را leader بدانند و database را ناسازگار کنند. پس برای انتخاب leader به consensus نیاز داریم. از طرف دیگر، consensus algorithmهای اینجا total order broadcast هستند، total order broadcast به single-leader replication شباهت دارد و single-leader replication هم leader می‌خواهد. ظاهراً برای انتخاب leader، ابتدا باید leader داشته باشیم؛ برای حل consensus باید consensus را حل کرده باشیم. این دور با epoch و quorum شکسته می‌شود.

#### Epoch number و quorum

تمام consensus protocolهای مطرح‌شده به‌نوعی از leader استفاده می‌کنند، اما یکتابودن leader را تضمین نمی‌کنند. در عوض، تضمین ضعیف‌تری می‌دهند: هر epoch یک leader یکتا دارد. Epoch number در Paxos، ballot number؛ در Viewstamped Replication، view number؛ و در Raft، term number نامیده می‌شود.

هر بار که تصور شود leader فعلی مرده است، nodeها برای انتخاب leader جدید رأی‌گیری می‌کنند. election جدید epoch number افزایشی می‌گیرد؛ در نتیجه epochها total order و افزایش monotonic دارند. اگر دو leader در دو epoch متفاوت با هم تعارض کنند—شاید رهبر قبلی واقعاً از کار نیفتاده باشد—leader دارای epoch بزرگ‌تر برنده است.

پیش از تصمیم‌گیری، leader باید بررسی کند که leader دیگری با epoch بزرگ‌تر وجود ندارد که تصمیم متعارضی بگیرد. node نمی‌تواند صرفاً به تشخیص خودش اعتماد کند: اینکه خود را leader می‌داند، لزوماً به معنی پذیرفته‌شدنش از سوی nodeهای دیگر نیست.

برای هر تصمیم پیشنهادی، leader باید از quorum رأی بگیرد: value پیشنهادی را برای nodeهای دیگر می‌فرستد و منتظر می‌ماند تا quorum آن را تأیید کند. Quorum اغلب اکثریت nodeهاست، اما همیشه نه. Node فقط وقتی به proposal رأی مثبت می‌دهد که از leader دیگری با epoch بالاتر خبر نداشته باشد.

در نتیجه دو دور رأی‌گیری داریم: نخست برای انتخاب leader و سپس برای تصویب proposal آن. Quorumهای این دو رأی‌گیری باید با هم overlap داشته باشند. اگر رأی‌گیری proposal موفق شود، دست‌کم یکی از nodeهایی که به آن رأی داده در آخرین election leader نیز شرکت کرده است. بنابراین اگر رأی‌گیری proposal هیچ epoch بالاتری نشان ندهد، leader فعلی نتیجه می‌گیرد election جدیدتری رخ نداده و هنوز leader است؛ پس می‌تواند value پیشنهادی را با اطمینان انتخاب کند.

این رأی‌گیری در ظاهر به 2PC شبیه است، اما تفاوت‌های مهمی دارد: در 2PC coordinator انتخاب نمی‌شود؛ consensus به رأی اکثریت nodeها نیاز دارد، ولی 2PC از تمام participantها رأی «بله» می‌خواهد. افزون بر این، consensus protocolها recovery را تعریف می‌کنند تا nodeها پس از انتخاب leader جدید به state سازگار برسند و safety propertyها همواره حفظ شوند. این تفاوت‌ها برای correctness و fault tolerance حیاتی‌اند.

#### محدودیت‌های consensus

Consensus دستاورد بزرگی برای distributed systemهاست: safety propertyهای مشخص—agreement، integrity و validity—را در systemی فراهم می‌کند که بقیه‌چیزهایش نامطمئن‌اند؛ و تا وقتی اکثریت nodeها فعال و reachable باشند، امکان پیشرفت دارد. Consensus total order broadcast می‌دهد و از این طریق می‌تواند atomic operationهای خطی‌پذیر را به‌شکل fault-tolerant پیاده‌سازی کند.

اما این مزایا هزینه دارند. رأی‌گیری nodeها دربارهٔ proposalها نوعی synchronous replication است. Databaseها اغلب asynchronous replication را انتخاب می‌کنند؛ در این حالت ممکن است هنگام failover مقداری data commitشده از دست برود، اما بسیاری این ریسک را برای performance بهتر می‌پذیرند.

Consensus برای کار به اکثریت strict نیاز دارد: برای تحمل یک failure دست‌کم سه node لازم است تا دو node باقی‌مانده اکثریت باشند؛ برای تحمل دو failure دست‌کم پنج node لازم است تا سه node باقی بمانند. اگر network بخشی از nodeها را از بقیه جدا کند، فقط بخش اکثریت می‌تواند پیشرفت کند و بقیه block می‌شوند.

بیشتر consensus algorithmها مجموعهٔ ثابتی از nodeهای رأی‌دهنده را فرض می‌کنند؛ بنابراین نمی‌توان nodeها را آزادانه به cluster اضافه یا از آن حذف کرد. extensionهای dynamic membership اجازه می‌دهند مجموعهٔ nodeها در طول زمان تغییر کند، اما نسبت به algorithmهای static membership کمتر شناخته شده‌اند.

Consensus systemها معمولاً برای تشخیص nodeهای failureکرده به timeout متکی‌اند. در محیط‌هایی با network delay بسیار متغیر، به‌ویژه systemهای جغرافیایی، ممکن است node به‌اشتباه و به‌دلیل اختلال موقت network تصور کند leader از کار افتاده است. این اشتباه safety را نقض نمی‌کند، اما electionهای پی‌درپی performance را به‌شدت خراب می‌کنند؛ system ممکن است بیشتر وقتش را صرف انتخاب leader کند تا انجام کار مفید.

بعضی consensus algorithmها به‌ویژه به مشکل‌های network حساس‌اند. برای نمونه، در Raft اگر همهٔ network درست باشد به‌جز یک link دائماً غیرقابل‌اعتماد، ممکن است leadership پیوسته میان دو node جابه‌جا شود یا leader فعلی مرتب مجبور به کناره‌گیری شود؛ در نتیجه system عملاً پیشرفتی نمی‌کند. algorithmهای دیگر هم مشکل‌های مشابهی دارند و طراحی algorithmهایی مقاوم‌تر در برابر network غیرقابل‌اعتماد، هنوز مسئله‌ای پژوهشی است.

### Membership and Coordination Services

پروژه‌هایی مانند ZooKeeper و etcd اغلب «distributed key-value store» یا «coordination and configuration service» نامیده می‌شوند. API آن‌ها شبیه database است: می‌توان value مربوط به key را خواند و نوشت و روی keyها پیمایش کرد. پس چرا برایشان consensus algorithm پیاده‌سازی شده است؟

برنامه‌نویس application معمولاً مستقیم از ZooKeeper استفاده نمی‌کند، چون برای database عمومی مناسب نیست؛ بلکه غیرمستقیم از طریق پروژه‌هایی مانند HBase، Hadoop YARN، OpenStack Nova و Kafka به آن وابسته می‌شود.

ZooKeeper و etcd برای نگهداری مقدار کمی data طراحی شده‌اند که می‌تواند در memory جا بگیرد؛ البته برای durability روی disk هم نوشته می‌شود. این data محدود با total order broadcast مقاوم در برابر fault میان nodeها replicate می‌شود. چون هر message را می‌توان writeای به database دانست، apply کردن writeهای یکسان با ترتیب یکسان، replicaها را سازگار نگه می‌دارد.

ZooKeeper بر اساس lock service گوگل به‌نام Chubby ساخته شده و علاوه بر total order broadcast و در نتیجه consensus، چند feature مفید برای distributed systemها دارد:

- **Linearizable atomic operations:** با compare-and-set اتمیک می‌توان lock ساخت. اگر چند node هم‌زمان operation یکسانی را انجام دهند فقط یکی موفق می‌شود. Consensus protocol تضمین می‌کند operation حتی هنگام failure node یا قطع network اتمیک و linearizable باشد. Distributed lock معمولاً به‌شکل lease با زمان انقضا پیاده‌سازی می‌شود تا اگر client از کار افتاد، lock سرانجام آزاد شود.
- **Total ordering of operations:** وقتی resource با lock یا lease محافظت می‌شود، برای جلوگیری از تداخل clientها هنگام process pause به fencing token نیاز است. این token عددی است که هر بار lock گرفته می‌شود افزایش می‌یابد. ZooKeeper با مرتب‌کردن total همهٔ operationها و تخصیص transaction ID افزایشی (`zxid`) و version number (`cversion`) این قابلیت را فراهم می‌کند.
- **Failure detection:** clientها session بلندمدتی با serverهای ZooKeeper نگه می‌دارند و با heartbeatهای دوره‌ای زنده‌بودن یکدیگر را می‌سنجند. قطع موقت اتصال یا failure یک ZooKeeper node لزوماً session را تمام نمی‌کند. اما اگر heartbeatها بیشتر از session timeout قطع شوند، ZooKeeper session را مرده اعلام می‌کند. می‌توان تنظیم کرد lockهای session پس از timeout خودکار آزاد شوند؛ ZooKeeper این nodeها را ephemeral node می‌نامد.
- **Change notification:** client می‌تواند lock و valueهایی را که client دیگری ساخته بخواند و تغییرات آن‌ها را watch کند. از این راه می‌فهمد client دیگری به cluster پیوسته یا session آن timeout شده و ephemeral nodeهایش ناپدید شده‌اند. Subscription به notification نیاز به polling مکرر را کاهش می‌دهد.

از میان این featureها فقط linearizable atomic operationها واقعاً به consensus نیاز دارند؛ اما ترکیب آن‌هاست که ZooKeeper را برای coordination توزیع‌شده مفید می‌کند.

#### تخصیص کار به nodeها

یک کاربرد مناسب مدل ZooKeeper/Chubby انتخاب leader یا primary میان چند instance از یک process یا service است. اگر leader از کار بیفتد، node دیگری مسئولیت را می‌گیرد. این قابلیت برای single-leader databaseها و نیز job schedulerها و systemهای stateful مفید است.

نمونهٔ دیگر، تخصیص partitionهای یک resource—مانند database، message stream، file storage یا distributed actor system—به nodeهاست. با پیوستن nodeهای جدید ممکن است لازم باشد برای rebalance کردن load بعضی partitionها از nodeهای قبلی منتقل شوند. با حذف یا failure nodeها نیز nodeهای دیگر باید کار آن‌ها را بر عهده بگیرند.

با استفادهٔ سنجیده از atomic operationها، ephemeral nodeها و notificationها می‌توان این کارها را انجام داد و application را قادر ساخت بدون دخالت انسان از faultها recover کند. این کار آسان نیست؛ حتی با وجود libraryهایی مانند Apache Curator که ابزارهای سطح‌بالاتر ارائه می‌کنند. بااین‌حال، از پیاده‌سازی consensus algorithmهای لازم از صفر—که سابقهٔ موفقیت ضعیفی دارد—بهتر است.

اگرچه application ممکن است ابتدا روی یک node اجرا شود و بعد به هزاران node برسد، رأی‌گیری اکثریت میان این تعداد node بسیار ناکارآمد خواهد بود. در عوض ZooKeeper روی تعداد ثابتی node—معمولاً سه یا پنج—اجرا می‌شود و رأی‌گیری را میان همان‌ها انجام می‌دهد، درحالی‌که از clientهای بسیار بیشتری پشتیبانی می‌کند. به این ترتیب می‌توان بخشی از کار coordination nodeها—consensus، ترتیب operationها و failure detection—را به service بیرونی واگذار کرد.

data تحت مدیریت ZooKeeper معمولاً آهسته تغییر می‌کند؛ مثلاً اطلاعاتی مانند «node واقع در `10.1.1.23`، leader مربوط به partition شمارهٔ 7 است» که ممکن است هر چند دقیقه یا ساعت عوض شود. ZooKeeper برای runtime stateای طراحی نشده که هزاران یا میلیون‌ها بار در ثانیه تغییر کند. برای replicate کردن application state از nodeای به node دیگر می‌توان از ابزارهایی مانند Apache BookKeeper استفاده کرد.

#### Service discovery

ZooKeeper، etcd و Consul اغلب برای service discovery به‌کار می‌روند؛ یعنی پیدا کردن IP address لازم برای اتصال به یک service. در cloud datacenterها، virtual machineها پیوسته ایجاد و حذف می‌شوند و معمولاً IP address serviceها از قبل معلوم نیست. به‌جای آن، service هنگام شروع، network endpoint خود را در service registry ثبت می‌کند تا serviceهای دیگر بتوانند آن را پیدا کنند.

اما روشن نیست service discovery واقعاً به consensus نیاز داشته باشد. روش سنتی پیدا کردن IP address یک service از روی نام آن DNS است؛ DNS برای performance و availability مناسب از چند لایهٔ cache استفاده می‌کند. readهای DNS قطعاً linearizable نیستند و معمولاً کمی stale بودن نتیجه مشکل‌ساز تلقی نمی‌شود. مهم‌تر آن است که DNS همیشه در دسترس و در برابر network interruption مقاوم باشد.

Service discovery به consensus نیاز ندارد، اما leader election نیازمند آن است. پس اگر consensus system از قبل leader را می‌شناسد، منطقی است اطلاعات آن را برای کشف leader از سوی serviceهای دیگر هم به‌کار بگیریم. بعضی consensus systemها برای این منظور read-only caching replica دارند: این replicaها log تصمیم‌های consensus را به‌شکل asynchronous دریافت می‌کنند، اما در رأی‌گیری شرکت نمی‌کنند؛ بنابراین می‌توانند readهایی را پاسخ دهند که linearizable بودن لازم ندارند.

#### Membership serviceها

ZooKeeper و ابزارهای مشابه را می‌توان بخشی از تاریخچهٔ طولانی پژوهش دربارهٔ membership serviceها دانست؛ پژوهشی که به دههٔ ۱۹۸۰ برمی‌گردد و برای ساخت systemهای بسیار قابل‌اعتماد، از جمله systemهای کنترل ترافیک هوایی، اهمیت داشته است.

Membership service مشخص می‌کند کدام nodeها در حال حاضر عضو فعال و زندهٔ cluster هستند. به‌دلیل network delay نامحدود، تشخیص قابل‌اعتماد failure node دیگر ممکن نیست. اما اگر failure detection را با consensus ترکیب کنیم، nodeها می‌توانند توافق کنند کدام nodeها زنده محسوب شوند.

ممکن است consensus یک node زنده را اشتباهاً مرده اعلام کند؛ بااین‌حال توافق بر سر اعضای فعلی cluster بسیار مفید است. مثلاً می‌توان leader را کم‌شماره‌ترین node میان اعضای فعلی انتخاب کرد، اما این روش وقتی کار نمی‌کند که nodeهای مختلف دربارهٔ اعضای فعلی نظرهای متفاوت داشته باشند.

## Key Terms

- `Transaction Coordinator` — componentی که participantهای distributed transaction را هماهنگ می‌کند، رأی‌های prepare را می‌گیرد و تصمیم نهایی commit یا abort را ثبت و منتشر می‌کند.
- `Participant` — database node یا systemی که بخشی از distributed transaction را اجرا می‌کند و به درخواست prepare، commit یا abort پاسخ می‌دهد.
- `In-Doubt Transaction` — transaction آماده‌شده‌ای که participant نمی‌داند باید commit شود یا abort، چون تصمیم coordinator به آن نرسیده است.
- `Heuristic Decision` — تصمیم یک‌طرفهٔ participant برای commit یا abort کردن transaction in-doubt، بدون تصمیم قطعی coordinator؛ راهی اضطراری که ممکن است atomicity را بشکند.
- `Blocking Atomic Commit` — atomic commit protocolی که در failure coordinator ممکن است participantها را تا recovery یا مداخلهٔ دستی متوقف نگه دارد؛ 2PC نمونهٔ آن است.
- `Fault-Tolerant Consensus` — توافق nodeها بر یک value با حفظ safety و امکان پیشرفت تا وقتی quorum کافی از nodeها فعال و reachable باشد.
- `Consensus Termination` — liveness propertyای که می‌گوید nodeهای crashنکرده سرانجام باید تصمیم بگیرند، نه اینکه برای همیشه منتظر بمانند.
- `Epoch Number` — شمارهٔ افزایشی برای دورهٔ رهبری که تعارض رهبرهای متعلق به دوره‌های مختلف را حل می‌کند؛ نام‌های دیگر آن ballot number، view number و term number هستند.
- `Membership Service` — serviceای که فهرست اعضای فعال cluster را مدیریت می‌کند و مبنای coordination و تخصیص کار میان nodeهاست.
- `Failure Detector` — mechanismی برای تشخیص یا مشکوک‌شدن به failure node؛ در شبکهٔ دارای delay نامحدود، timeout به‌تنهایی تشخیص قطعی نمی‌دهد.
- `Three-Phase Commit (3PC)` — protocol پیشنهادی برای nonblocking atomic commit که به فرض‌های bounded network delay و bounded response time متکی است.

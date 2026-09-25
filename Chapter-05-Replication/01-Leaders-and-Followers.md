# Chapter 5 — Replication

## Leaders and Followers

هر nodeای که یک copy از database را نگه می‌دارد `replica` نامیده می‌شود. وقتی replicaهای متعددی داریم، ناگزیر این سؤال مطرح می‌شود: چگونه مطمئن شویم تمام data در نهایت به تمام replicaها می‌رسد؟

هر write در database باید توسط تمام replicaها پردازش شود؛ در غیر این صورت replicaها دیگر data یکسانی نخواهند داشت. رایج‌ترین راه‌حل برای این مسئله `leader-based replication` است (که `active/passive` یا `master-slave replication` نیز نامیده می‌شود). این روش در شکل ۵-۱ نشان داده شده و به شکل زیر کار می‌کند:

1. یکی از replicaها به‌عنوان `leader` (که `master` یا `primary` نیز نامیده می‌شود) تعیین می‌شود. وقتی clientها می‌خواهند در database write کنند، باید requestهای خود را به leader بفرستند؛ leader ابتدا data جدید را در local storage خود write می‌کند.
2. replicaهای دیگر `follower` نامیده می‌شوند (read replica، slave، secondary یا hot standby).* هر زمان که leader data جدیدی در local storage خود write می‌کند، change مربوط به data را نیز به‌عنوان بخشی از `replication log` یا `change stream` برای تمام followerها می‌فرستد. هر follower log را از leader می‌گیرد و با apply کردن تمام writeها به همان orderی که روی leader پردازش شده‌اند، local copy خود از database را update می‌کند.
3. وقتی client می‌خواهد data را از database read کند، می‌تواند leader یا هرکدام از followerها را query کند. بااین‌حال، writeها فقط روی leader پذیرفته می‌شوند (از دید client، followerها read-only هستند).

**شکل ۵-۱.** replication مبتنی بر leader (master-slave).

این mode از replication، feature built-in بسیاری از relational databaseهاست؛ از جمله `PostgreSQL` (از version 9.0)، `MySQL`، `Oracle Data Guard` [2] و `AlwaysOn Availability Groups` در SQL Server [3]. این روش در برخی databaseهای غیررابطه‌ای مانند `MongoDB`، `RethinkDB` و `Espresso` [4] نیز استفاده می‌شود. در نهایت، leader-based replication فقط به databaseها محدود نیست: message brokerهای distributed مانند `Kafka` [5] و queueهای highly available در `RabbitMQ` [6] نیز از آن استفاده می‌کنند. برخی network filesystemها و replicated block deviceهایی مانند `DRBD` نیز مشابه همین روش هستند.

*پاورقی: افراد مختلف تعریف‌های متفاوتی از hot، warm و cold standby server دارند. برای مثال، در PostgreSQL، hot standby به replicaای گفته می‌شود که readهای client را می‌پذیرد، درحالی‌که warm standby changeهای leader را پردازش می‌کند اما queryهای client را پردازش نمی‌کند. برای هدف این کتاب، این تفاوت مهم نیست.*

### Synchronous Versus Asynchronous Replication

یک جزئیات مهم در replicated system این است که replication به‌صورت synchronous انجام می‌شود یا asynchronous. (در relational databaseها این موضوع اغلب یک configuration option است؛ systemهای دیگر معمولاً از ابتدا به یکی از این دو روش hardcode شده‌اند.)

به اتفاقی فکر کنید که در شکل ۵-۱ رخ می‌دهد؛ user یک website تصویر profile خود را update می‌کند. در زمانی مشخص، client request مربوط به update را به leader می‌فرستد و کمی بعد leader آن را دریافت می‌کند. سپس leader در زمانی دیگر change مربوط به data را برای followerها forward می‌کند. در نهایت، leader به client اطلاع می‌دهد که update موفق بوده است.

شکل ۵-۲ ارتباط میان componentهای مختلف system را نشان می‌دهد: client کاربر، leader و دو follower. زمان از چپ به راست حرکت می‌کند. request یا response message با یک arrow ضخیم نشان داده شده است.

**شکل ۵-۲.** leader-based replication با یک follower synchronous و یک follower asynchronous.

در مثال شکل ۵-۲، replication به follower 1، synchronous است: leader منتظر می‌ماند تا follower 1 دریافت write را تأیید کند، سپس موفقیت را به user گزارش می‌دهد و write را برای clientهای دیگر قابل مشاهده می‌کند. replication به follower 2، asynchronous است: leader message را می‌فرستد، اما منتظر response از follower نمی‌ماند.

نمودار نشان می‌دهد که پیش از پردازش message توسط follower 2، delay قابل‌توجهی وجود دارد. معمولاً replication سریع است و بیشتر database systemها changeها را در کمتر از یک second روی followerها apply می‌کنند. بااین‌حال، هیچ guaranteeای دربارهٔ مدت زمان این کار وجود ندارد. در شرایطی ممکن است followerها چندین دقیقه یا بیشتر از leader عقب بمانند؛ برای مثال، وقتی follower در حال recovery از یک failure است، system نزدیک maximum capacity کار می‌کند یا میان nodeها network problem وجود دارد.

مزیت synchronous replication این است که تضمین می‌شود follower یک copy به‌روز و consistent با leader داشته باشد. اگر leader ناگهان fail شود، مطمئن هستیم data همچنان روی follower در دسترس است. نقطه‌ضعف این است که اگر followerِ synchronous response ندهد (چون crash کرده، network fault رخ داده یا هر دلیل دیگری وجود دارد)، write نمی‌تواند پردازش شود. Leader باید تمام writeها را block کند و منتظر بماند تا replicaی synchronous دوباره available شود.

به همین دلیل synchronous بودن تمام followerها عملی نیست: از کار افتادن هر node باعث می‌شود کل system متوقف شود. در عمل، اگر synchronous replication را در یک database فعال کنید، معمولاً یکی از followerها synchronous و بقیه asynchronous هستند. اگر followerِ synchronous unavailable یا slow شود، یکی از followerهای asynchronous به‌عنوان synchronous انتخاب می‌شود. این کار تضمین می‌کند که دست‌کم روی دو node، یعنی leader و یک followerِ synchronous، copy به‌روزی از data داشته باشید. این configuration گاهی `semi-synchronous` نیز نامیده می‌شود [7].

اغلب leader-based replication به‌صورت کاملاً asynchronous configure می‌شود. در این حالت، اگر leader fail کند و قابل recovery نباشد، هر writeای که هنوز به followerها replicate نشده است از دست می‌رود. یعنی حتی اگر write به client تأیید شده باشد، durable بودن آن تضمین نمی‌شود. بااین‌حال، configuration کاملاً asynchronous این مزیت را دارد که leader می‌تواند به پردازش writeها ادامه دهد، حتی اگر تمام followerهای آن عقب افتاده باشند.

ممکن است تضعیف durability شبیه trade-off بدی به نظر برسد، اما asynchronous replication همچنان به‌طور گسترده استفاده می‌شود؛ به‌خصوص وقتی followerهای زیادی دارید یا followerها از نظر جغرافیایی distributed هستند. در بخش «Problems with Replication Lag» در صفحهٔ ۱۶۱ دوباره به این مسئله برمی‌گردیم.

#### Research on Replication

از دست دادن data در صورت fail شدن leader می‌تواند برای systemهای asynchronous replicated مشکل بسیار جدی باشد. به همین دلیل، پژوهشگران به بررسی روش‌های replicationای ادامه داده‌اند که data را از دست نمی‌دهند اما همچنان performance و availability خوبی ارائه می‌کنند. برای مثال، `chain replication` [8, 9] variantای از synchronous replication است که در چند system مانند `Microsoft Azure Storage` [10, 11] با موفقیت implement شده است.

میان consistency مربوط به replication و consensus (به توافق رسیدن چند node بر سر یک value) ارتباط نزدیکی وجود دارد و در Chapter 9 این حوزهٔ تئوری را با جزئیات بیشتری بررسی خواهیم کرد. در این chapter روی formهای ساده‌تر replication تمرکز می‌کنیم که در databaseهای عملی به‌طور معمول استفاده می‌شوند.

### Setting Up New Followers

هر از گاهی لازم است followerهای جدیدی setup کنید؛ شاید برای افزایش تعداد replicaها یا جایگزین کردن nodeهای failed. چگونه مطمئن می‌شوید follower جدید copy دقیقی از data مربوط به leader دارد؟

صرفاً copy کردن data fileها از یک node به node دیگر معمولاً کافی نیست: clientها دائماً در database write می‌کنند و data همیشه در حال تغییر است، بنابراین یک file copy معمولی بخش‌های مختلف database را در زمان‌های متفاوتی مشاهده می‌کند. نتیجه ممکن است هیچ معنای معتبری نداشته باشد.

می‌توانید با lock کردن database (و unavailable کردن آن برای writeها) fileهای روی disk را consistent کنید، اما این کار با هدف high availability ما سازگار نیست. خوشبختانه setup کردن follower معمولاً بدون downtime امکان‌پذیر است. این فرآیند در سطح مفهومی به شکل زیر است:

1. در یک نقطهٔ زمانی مشخص، از database مربوط به leader یک `snapshot` consistent بگیرید؛ در صورت امکان، بدون گرفتن lock روی کل database. بیشتر databaseها این feature را دارند، چون برای backup نیز به آن نیاز است. در برخی موارد به toolهای third-party نیاز دارید؛ مانند `innobackupex` برای MySQL [12].
2. snapshot را به node مربوط به follower جدید copy کنید.
3. follower به leader connect می‌شود و تمام data changeهایی را که از زمان گرفته شدن snapshot رخ داده‌اند request می‌کند. برای این کار snapshot باید به position دقیقی در replication log مربوط باشد. این position نام‌های مختلفی دارد: برای مثال، PostgreSQL آن را `log sequence number` و MySQL آن را `binlog coordinates` می‌نامد.
4. وقتی follower backlog مربوط به data changeهای پس از snapshot را پردازش کرد، می‌گوییم follower catch up کرده است. حالا می‌تواند به پردازش data changeها از leader، هم‌زمان با رخ دادن آن‌ها، ادامه دهد.

گام‌های عملی setup کردن follower بسته به database به‌طور قابل‌توجهی متفاوت است. در برخی systemها این فرآیند کاملاً automated است، درحالی‌که در برخی دیگر workflow چندمرحله‌ای و نسبتاً پیچیده‌ای است که administrator باید آن را به‌صورت دستی انجام دهد.

### Handling Node Outages

هر nodeای در system ممکن است از کار بیفتد؛ شاید به‌صورت unexpected به‌دلیل fault، اما به همان اندازه ممکن است به‌دلیل maintenance برنامه‌ریزی‌شده باشد (برای مثال، reboot کردن machine برای نصب kernel security patch). توانایی reboot کردن nodeها به‌صورت جداگانه و بدون downtime مزیت بزرگی برای operations و maintenance است. بنابراین هدف ما این است که کل system با وجود failureهای nodeهای منفرد به کار خود ادامه دهد و اثر outage هر node تا حد امکان کوچک باشد.

چگونه با leader-based replication به high availability می‌رسید؟

#### Follower failure: Catch-up recovery

هر follower روی local disk خود log مربوط به data changeهایی را که از leader دریافت کرده نگه می‌دارد. اگر follower crash کند و restart شود، یا network میان leader و follower به‌طور موقت قطع شود، follower می‌تواند نسبتاً ساده recovery کند: از روی log خود می‌داند پیش از رخ دادن fault، آخرین transaction پردازش‌شده کدام بوده است. بنابراین follower می‌تواند به leader connect شود و تمام data changeهایی را request کند که در مدت disconnected بودن رخ داده‌اند. پس از apply کردن این changeها، follower به leader catch up می‌کند و می‌تواند مانند قبل stream مربوط به data changeها را دریافت کند.

#### Leader failure: Failover

مدیریت failure مربوط به leader دشوارتر است: یکی از followerها باید به‌عنوان leader جدید promote شود، clientها باید reconfigure شوند تا writeهای خود را به leader جدید بفرستند و followerهای دیگر باید شروع به consume کردن data changeها از leader جدید کنند. این فرآیند `failover` نامیده می‌شود.

Failover می‌تواند به‌صورت manual انجام شود (administrator از failure leader مطلع می‌شود و گام‌های لازم را برای تعیین leader جدید انجام می‌دهد) یا automatic باشد. یک فرآیند automatic failover معمولاً شامل گام‌های زیر است:

1. مشخص کردن اینکه leader fail شده است. چیزهای زیادی ممکن است به‌درستی کار نکنند: crash، power outage، network issue و موارد دیگر. هیچ روش foolproofی برای تشخیص علت دقیق وجود ندارد، بنابراین بیشتر systemها صرفاً از timeout استفاده می‌کنند: nodeها مرتباً messageهایی را برای یکدیگر رفت‌وبرگشت می‌دهند و اگر nodeای برای مدتی - مثلاً ۳۰ second - response ندهد، فرض می‌شود مرده است. (اگر leader عمداً برای planned maintenance از کار انداخته شده باشد، این حالت صدق نمی‌کند.)
2. انتخاب leader جدید. این کار می‌تواند با یک election process انجام شود (که در آن leader با رأی اکثریت replicaهای باقی‌مانده انتخاب می‌شود) یا یک controller node که قبلاً انتخاب شده است می‌تواند leader جدید را appoint کند. بهترین candidate برای leadership معمولاً replicaای است که به‌روزترین data changeهای leader قدیمی را دارد تا data loss به حداقل برسد. به توافق رسیدن تمام nodeها بر سر leader جدید یک consensus problem است که در Chapter 9 با جزئیات بررسی می‌شود.
3. reconfigure کردن system برای استفاده از leader جدید. حالا clientها باید write requestهای خود را به leader جدید بفرستند (این موضوع را در بخش «Request Routing» در صفحهٔ ۲۱۴ بررسی می‌کنیم). اگر leader قدیمی دوباره برگردد، ممکن است همچنان تصور کند leader است و متوجه نشده باشد که replicaهای دیگر آن را مجبور به step down کرده‌اند. system باید تضمین کند leader قدیمی به follower تبدیل شود و leader جدید را به رسمیت بشناسد.

در failover احتمال رخ دادن مشکل‌های زیادی وجود دارد:

- اگر از asynchronous replication استفاده شود، ممکن است leader جدید پیش از fail شدن leader قدیمی، تمام writeهای آن را دریافت نکرده باشد. اگر leader قبلی پس از انتخاب leader جدید دوباره به cluster ملحق شود، با آن writeها چه باید کرد؟ ممکن است leader جدید در این فاصله writeهای متناقضی دریافت کرده باشد. رایج‌ترین راه‌حل این است که writeهای replicate‌نشدهٔ leader قدیمی به‌سادگی discard شوند؛ کاری که ممکن است انتظار clientها از durability را نقض کند.
- دور انداختن writeها به‌خصوص زمانی خطرناک است که لازم باشد systemهای storage دیگری خارج از database را با محتوای database هماهنگ کنید. برای مثال، در یک incident در `GitHub` [13]، یک follower قدیمی MySQL به‌عنوان leader promote شد. Database برای assign کردن primary key به rowهای جدید از counterای auto-incrementing استفاده می‌کرد، اما چون counter در leader جدید از leader قدیمی عقب‌تر بود، برخی primary keyهایی را که قبلاً توسط leader قدیمی assign شده بودند دوباره استفاده کرد. این primary keyها در یک Redis store نیز استفاده می‌شدند؛ بنابراین reuse شدن primary keyها باعث inconsistency میان MySQL و Redis شد و در نتیجه بخشی از private data به userهای اشتباه disclose شد.
- در برخی fault scenarioها (به Chapter 8 مراجعه کنید) ممکن است دو node هر دو تصور کنند leader هستند. این وضعیت `split brain` نام دارد و خطرناک است: اگر هر دو leader write را بپذیرند و processی برای resolve کردن conflictها وجود نداشته باشد (به بخش «Multi-Leader Replication» در صفحهٔ ۱۶۸ مراجعه کنید)، احتمالاً data از دست می‌رود یا corrupt می‌شود. به‌عنوان safety catch، برخی systemها mechanismی دارند که اگر دو leader شناسایی شوند یکی از nodeها را shutdown می‌کند.† بااین‌حال، اگر این mechanism با دقت طراحی نشده باشد، ممکن است در نهایت هر دو node shutdown شوند [14].
- timeout مناسب پیش از اعلام dead بودن leader چقدر است؟ timeout طولانی‌تر در صورت failure leader، زمان recovery را افزایش می‌دهد. بااین‌حال، اگر timeout بیش از حد کوتاه باشد، ممکن است failoverهای غیرضروری رخ دهند. برای مثال، یک load spike موقت می‌تواند response time یک node را از timeout بالاتر ببرد یا یک network glitch باعث تأخیر packetها شود. اگر system از قبل با load زیاد یا network problem دست‌وپنجه نرم کند، failover غیرضروری احتمالاً وضعیت را بدتر می‌کند، نه بهتر.

برای این مشکل‌ها راه‌حل ساده‌ای وجود ندارد. به همین دلیل، برخی teamهای operations ترجیح می‌دهند failover را manual انجام دهند، حتی اگر software از automatic failover پشتیبانی کند.

این مسئله‌ها - failure nodeها، networkهای unreliable و trade-offهای مربوط به consistency، durability، availability و latency replica - در واقع مشکل‌های بنیادی distributed systemها هستند. در Chapter 8 و Chapter 9 آن‌ها را با عمق بیشتری بررسی خواهیم کرد.

*پاورقی: این approach `fencing` نام دارد یا با تأکید بیشتر `Shoot The Other Node In The Head (STONITH)` نامیده می‌شود. در بخش «The leader and the lock» در صفحهٔ ۳۰۱، fencing را با جزئیات بیشتری بررسی خواهیم کرد.*

### Implementation of Replication Logs

leader-based replication در پشت صحنه چگونه کار می‌کند؟ چند روش مختلف replication در عمل استفاده می‌شود، بنابراین هرکدام را به‌صورت کوتاه بررسی کنیم.

#### Statement-based replication

در ساده‌ترین حالت، leader هر write request (یعنی statement)ای را که execute می‌کند log می‌کند و statement log را برای followerها می‌فرستد. در یک relational database، این یعنی هر statement از نوع `INSERT`، `UPDATE` یا `DELETE` به followerها forward می‌شود و هر follower آن SQL statement را طوری parse و execute می‌کند که گویی آن را از یک client دریافت کرده است.

بااین‌حال، این approach به روش‌های مختلفی ممکن است شکست بخورد:

- هر statementای که یک nondeterministic function مانند `NOW()` برای دریافت date و time فعلی یا `RAND()` برای تولید random number call کند، احتمالاً در هر replica value متفاوتی تولید می‌کند.
- اگر statementها از columnهای auto-incrementing استفاده کنند یا به data موجود در database وابسته باشند (برای مثال `UPDATE … WHERE <some condition>`)، باید دقیقاً به همان order روی هر replica execute شوند؛ در غیر این صورت ممکن است effect متفاوتی داشته باشند. این موضوع وقتی چند transaction به‌صورت concurrent اجرا می‌شوند محدودکننده است.
- statementهایی که side effect دارند (برای مثال trigger، stored procedure یا user-defined function) ممکن است باعث ایجاد side effectهای متفاوت در هر replica شوند، مگر اینکه side effectها کاملاً deterministic باشند.

می‌توان برای این مشکل‌ها workaroundهایی ایجاد کرد؛ برای مثال، leader می‌تواند هنگام log کردن statement، هر nondeterministic function call را با return value ثابتی جایگزین کند تا تمام followerها همان value را دریافت کنند. بااین‌حال، چون edge caseهای بسیار زیادی وجود دارد، امروزه معمولاً روش‌های replication دیگری ترجیح داده می‌شوند.

پیش از version 5.1، MySQL از statement-based replication استفاده می‌کرد. این روش هنوز هم گاهی استفاده می‌شود، چون نسبتاً compact است؛ اما MySQL اکنون به‌صورت default، اگر statement شامل nondeterminism باشد، به row-based replication (که کمی بعد بررسی می‌شود) switch می‌کند. `VoltDB` از statement-based replication استفاده می‌کند و با اجباری کردن deterministic بودن transactionها، آن را safe می‌کند [15].

#### Write-ahead log (WAL) shipping

در Chapter 3 بررسی کردیم که storage engineها data را روی disk چگونه represent می‌کنند و دیدیم که معمولاً هر write به یک log append می‌شود:

- در log-structured storage engine (به بخش «SSTables and LSM-Trees» در صفحهٔ ۷۶ مراجعه کنید)، این log محل اصلی storage است. log segmentها در background compact و garbage-collect می‌شوند.
- در B-tree (به بخش «B-Trees» در صفحهٔ ۷۹ مراجعه کنید) که blockهای منفرد disk را overwrite می‌کند، هر modification ابتدا در یک `write-ahead log` write می‌شود تا پس از crash، index به state consistent restore شود.

در هر دو حالت، log یک sequence append-only از byteهاست که تمام writeهای database را شامل می‌شود. می‌توانیم دقیقاً از همین log برای ساختن replica روی node دیگری استفاده کنیم: leader علاوه بر write کردن log روی disk، آن را از طریق network برای followerها نیز می‌فرستد.

وقتی follower این log را پردازش می‌کند، copyای از همان data structureهای موجود روی leader می‌سازد.

این روش replication، در میان سیستم‌های دیگر، در PostgreSQL و Oracle استفاده می‌شود [16]. نقطه‌ضعف اصلی این است که log، data را در سطح بسیار پایینی توصیف می‌کند: یک WAL شامل جزئیات byteهایی است که در کدام disk blockها تغییر کرده‌اند. این موضوع replication را به storage engine وابسته می‌کند. اگر database storage format خود را از یک version به version دیگر تغییر دهد، معمولاً نمی‌توان versionهای متفاوت database software را روی leader و followerها اجرا کرد.

ممکن است این موضوع یک implementation detail جزئی به نظر برسد، اما می‌تواند اثر عملیاتی بزرگی داشته باشد. اگر replication protocol اجازه دهد follower از software version جدیدتری نسبت به leader استفاده کند، می‌توانید database software را بدون downtime upgrade کنید: ابتدا followerها را upgrade کنید و سپس با failover، یکی از nodeهای upgradeشده را به‌عنوان leader جدید انتخاب کنید. اگر replication protocol این version mismatch را اجازه ندهد، همان‌طور که اغلب در WAL shipping رخ می‌دهد، چنین upgradeهایی به downtime نیاز خواهند داشت.

#### Logical (row-based) log replication

یک alternative این است که برای replication و storage engine از log formatهای متفاوتی استفاده کنیم؛ در این صورت replication log از internals مربوط به storage engine decouple می‌شود. به چنین replication logای `logical log` می‌گوییم تا آن را از data representation فیزیکی storage engine متمایز کنیم.

یک logical log برای relational database معمولاً sequenceای از recordهاست که writeهای tableهای database را در granularity مربوط به row توصیف می‌کند:

- برای rowای که insert شده، log شامل valueهای جدید تمام columnهاست.
- برای rowای که delete شده، log اطلاعات کافی برای identify کردن یکتای row حذف‌شده را نگه می‌دارد. معمولاً این اطلاعات primary key است؛ اما اگر table primary key نداشته باشد، valueهای قدیمی تمام columnها باید log شوند.
- برای rowای که update شده، log اطلاعات کافی برای identify کردن یکتای row updateشده و valueهای جدید تمام columnها (یا دست‌کم valueهای جدید تمام columnهایی که تغییر کرده‌اند) را نگه می‌دارد.

Transactionای که چند row را modify می‌کند چند log record از این نوع تولید می‌کند و پس از آن recordی می‌آید که نشان می‌دهد transaction commit شده است. `binlog` در MySQL، وقتی برای row-based replication configure شده باشد، از همین approach استفاده می‌کند [17].

از آنجا که logical log از internals storage engine decouple است، می‌توان آن را آسان‌تر backward compatible نگه داشت؛ در نتیجه leader و follower می‌توانند versionهای متفاوتی از database software یا حتی storage engineهای متفاوتی را اجرا کنند.

یک logical log format برای parse شدن توسط external applicationها نیز ساده‌تر است. این ویژگی زمانی مفید است که بخواهید محتوای database را به system خارجی بفرستید؛ برای مثال، به data warehouseای برای offline analysis یا برای ساختن custom index و cache [18]. این technique `change data capture` نام دارد و در Chapter 11 دوباره به آن برمی‌گردیم.

#### Trigger-based replication

روش‌های replication که تاکنون توضیح دادیم توسط خود database system implement می‌شوند و application code در آن‌ها دخالت ندارد. در بسیاری از موارد همین چیزی است که می‌خواهید، اما در برخی شرایط flexibility بیشتری لازم است. برای مثال، اگر بخواهید فقط subsetای از data را replicate کنید، یا بخواهید از یک نوع database به نوع دیگری replicate کنید، یا به logicای برای conflict resolution نیاز داشته باشید (به بخش «Handling Write Conflicts» در صفحهٔ ۱۷۱ مراجعه کنید)، ممکن است لازم باشد replication را به application layer منتقل کنید.

برخی toolها، مانند `Oracle GoldenGate` [19]، می‌توانند با read کردن database log، data changeها را در اختیار application قرار دهند. راه‌حل دیگر استفاده از featureهایی است که در بسیاری از relational databaseها وجود دارد: trigger و stored procedure.

یک `trigger` به شما اجازه می‌دهد custom application codeای را register کنید که هر زمان data change (یعنی write transaction) در database system رخ داد، به‌صورت automatic execute شود. Trigger می‌تواند این change را در table جداگانه‌ای log کند و external process بتواند آن را از آن table read کند. سپس external process می‌تواند application logic لازم را apply کند و data change را به system دیگری replicate کند. `Databus` برای Oracle [20] و `Bucardo` برای Postgres [21] نمونه‌هایی از این روش هستند.

Trigger-based replication معمولاً overhead بیشتری نسبت به روش‌های دیگر replication دارد و در مقایسه با replication built-in database بیشتر مستعد bug و limitation است. بااین‌حال، به‌دلیل flexibility خود همچنان می‌تواند مفید باشد.

## Key Terms

- `Replication` — نگهداری copy یکسانی از data روی چند machine یا node.
- `Leader` — replicaای که writeها را می‌پذیرد و changeها را برای سایر replicaها منتشر می‌کند.
- `Follower` — replicaای که changeهای leader را دریافت و apply می‌کند و معمولاً readها را پاسخ می‌دهد.
- `Primary` — نام دیگر leader در leader-based replication.
- `Replica` — nodeای که copyای از database یا dataset را نگه می‌دارد.
- `Leader-Based Replication` — replicationای که در آن یک leader writeها را دریافت و followerها را update می‌کند.
- `Synchronous Replication` — replicationای که leader پیش از اعلام موفقیت write منتظر تأیید follower می‌ماند.
- `Asynchronous Replication` — replicationای که leader بدون انتظار برای response follower، write را ادامه می‌دهد.
- `Semi-Synchronous Replication` — configurationای که دست‌کم یک follower synchronous و followerهای دیگر asynchronous هستند.
- `Replication Lag` — فاصلهٔ زمانی یا مقداری میان state leader و state follower.
- `Replication Log` — logای از data changeها که leader برای ساخت یا update کردن replicaها منتشر می‌کند.
- `Snapshot` — تصویر consistent از data در یک نقطهٔ زمانی مشخص.
- `Catch-Up Recovery` — recovery follower با دریافت و apply کردن changeهای رخ‌داده در زمان disconnected بودن.
- `Failover` — فرآیند انتقال نقش leader به replicaی دیگر پس از failure leader.
- `Node Outage` — unavailable شدن یک node، چه به‌دلیل fault و چه به‌دلیل maintenance.
- `Write-Ahead Log (WAL)` — logی که modification را پیش از تغییر data اصلی ثبت می‌کند و می‌تواند برای replication یا recovery استفاده شود.
- `Statement-Based Replication` — replication با ارسال statement یا write request اجراشده از leader به follower.
- `Write-Ahead Log Shipping` — replication با ارسال WAL سطح پایین storage engine از leader به follower.
- `Logical Replication` — replication بر اساس logای مستقل از internals storage engine و نزدیک به تغییرات منطقی rowها.
- `Change Data Capture (CDC)` — استخراج data changeها از logical log برای ارسال به systemهای خارجی.
- `Consistency` — میزان هماهنگی state replicaها و guarantee مربوط به ترتیب و مشاهدهٔ data.
- `Durability` — تضمین باقی ماندن write پذیرفته‌شده پس از crash یا failure.
- `Network Partition` — قطع یا اختلالی که ارتباط بخشی از nodeها را با بخش دیگر مختل می‌کند.
- `Consensus` — توافق چند node بر سر value یا leader مشترک.
- `Split Brain` — وضعیتی که دو node هم‌زمان خود را leader می‌دانند و ممکن است writeهای متناقض بپذیرند.
- `Fencing` — mechanismی برای جلوگیری از ادامهٔ فعالیت node قدیمی یا غیرمجاز پس از تغییر leader.

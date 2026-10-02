# Chapter 7 — Transactions

## Weak Isolation Levels

اگر دو transaction به data یکسانی دست نزنند، می‌توان آن‌ها را با خیال راحت به‌صورت parallel اجرا کرد، چون هیچ‌کدام به دیگری وابسته نیست. Concurrency issue یا race condition زمانی وارد ماجرا می‌شود که یک transaction dataای را read کند که هم‌زمان توسط transaction دیگری در حال تغییر است، یا دو transaction هم‌زمان تلاش کنند data یکسانی را تغییر دهند.

پیدا کردن concurrency bugها با testing دشوار است، چون چنین bugهایی فقط وقتی trigger می‌شوند که timing به‌طور نامساعدی اتفاق بیفتد. این مشکل‌های زمانی ممکن است بسیار نادر رخ دهند و معمولاً reproduce کردنشان دشوار است. Reasoning دربارهٔ concurrency نیز بسیار سخت است، به‌خصوص در application بزرگی که الزاماً نمی‌دانید کدام بخش‌های دیگر code به database دسترسی دارند. توسعهٔ application حتی با یک user در هر لحظه به‌اندازهٔ کافی دشوار است؛ داشتن userهای concurrent کار را بسیار سخت‌تر می‌کند، چون هر dataای ممکن است در هر لحظه به‌صورت غیرمنتظره تغییر کند.

به همین دلیل، databaseها مدت‌ها تلاش کرده‌اند با ارائهٔ transaction isolation، concurrency issueها را از application developerها پنهان کنند. در تئوری، Isolation باید زندگی شما را ساده‌تر کند، چون اجازه می‌دهد وانمود کنید concurrency وجود ندارد: serializable isolation یعنی database تضمین می‌کند transactionها همان اثری را داشته باشند که اگر به‌صورت serial—یعنی یکی‌یکی و بدون concurrency—اجرا می‌شدند.

در عمل، Isolation متأسفانه به این سادگی نیست. Serializable isolation هزینهٔ performance دارد و بسیاری از databaseها نمی‌خواهند این هزینه را بپردازند [8]. بنابراین systemها معمولاً از isolation levelهای ضعیف‌تری استفاده می‌کنند که در برابر بعضی concurrency issueها محافظت ایجاد می‌کنند، اما نه همهٔ آن‌ها. درک این levelها دشوارتر است و ممکن است به bugهای ظریف منجر شوند، بااین‌حال در عمل بسیار استفاده می‌شوند [23].

Concurrency bugهایی که از weak transaction isolation ناشی می‌شوند فقط یک مشکل تئوری نیستند. این bugها باعث loss قابل‌توجه money شده‌اند [24, 25]، به investigation توسط financial auditorها منجر شده‌اند [26] و data مشتریان را corrupt کرده‌اند [27]. پس از آشکار شدن چنین مشکل‌هایی، معمولاً گفته می‌شود: «اگر با data مالی کار می‌کنید از ACID database استفاده کنید!» اما این حرف اصل مسئله را از دست می‌دهد. حتی بسیاری از relational database systemهای محبوب—که معمولاً ACID در نظر گرفته می‌شوند—از weak isolation استفاده می‌کنند و بنابراین الزاماً جلوی این bugها را نمی‌گرفتند.

به‌جای تکیهٔ blind بر toolها، باید درک خوبی از انواع concurrency problemهای موجود و روش جلوگیری از آن‌ها به دست آوریم. سپس می‌توانیم با استفاده از toolهایی که در اختیار داریم، applicationهایی reliable و correct بسازیم.

در این section چند isolation level ضعیف یا nonserializable را که در عمل استفاده می‌شوند بررسی می‌کنیم و با جزئیات توضیح می‌دهیم چه نوع race conditionهایی ممکن است رخ دهند و چه نوع‌هایی نمی‌توانند رخ دهند؛ تا بتوانید level مناسب application خود را انتخاب کنید. پس از آن، serializability را با جزئیات بررسی خواهیم کرد (به بخش «Serializability» در صفحهٔ ۲۵۱ مراجعه کنید). بحث ما دربارهٔ isolation levelها informal و مبتنی بر example خواهد بود. اگر به definition و analysis دقیق propertyهای آن‌ها نیاز دارید، می‌توانید به literature دانشگاهی مراجعه کنید [28, 29, 30].

### Read Committed

Read committed ابتدایی‌ترین level مربوط به transaction isolation است.v این level دو guarantee ارائه می‌کند:

1. هنگام read کردن از database، فقط dataای را می‌بینید که commit شده است؛ یعنی **dirty read** رخ نمی‌دهد.
2. هنگام write کردن در database، فقط dataای را overwrite می‌کنید که commit شده است؛ یعنی **dirty write** رخ نمی‌دهد.

بیایید این دو guarantee را با جزئیات بیشتری بررسی کنیم.

#### No Dirty Reads

تصور کنید یک transaction dataای را در database write کرده، اما هنوز commit یا abort نشده است. آیا transaction دیگری می‌تواند آن dataٔ commit‌نشده را ببیند؟ اگر پاسخ مثبت باشد، به آن **dirty read** گفته می‌شود [2].

Transactionهایی که در read committed isolation level اجرا می‌شوند باید جلوی dirty read را بگیرند. این یعنی writeهای یک transaction فقط زمانی برای transactionهای دیگر visible می‌شوند که آن transaction commit شود؛ و در آن لحظه تمام writeهای آن transaction با هم visible می‌شوند. شکل ۷-۴ این رفتار را نشان می‌دهد: user شمارهٔ ۱ مقدار `x = 3` را set کرده، اما تا زمانی که transaction او commit نشده است، `get x` مربوط به user شمارهٔ ۲ همچنان value قدیمی یعنی ۲ را برمی‌گرداند.

**شکل ۷-۴.** نبود dirty read: user شمارهٔ ۲ فقط پس از commit شدن transaction user شمارهٔ ۱، value جدید `x` را می‌بیند.

*پاورقی:* بعضی databaseها از isolation level ضعیف‌تری به نام read uncommitted پشتیبانی می‌کنند. این level جلوی dirty write را می‌گیرد، اما مانع dirty read نمی‌شود.

جلوگیری از dirty read چند دلیل دارد:

- اگر transaction لازم باشد چند object را update کند، dirty read باعث می‌شود transaction دیگری بعضی updateها را ببیند و بعضی دیگر را نبیند. برای مثال، در شکل ۷-۲ user email جدید را می‌بیند، اما counter updateشده را نمی‌بیند. این یک dirty read از email است. دیدن database در stateای partially updated برای userها گیج‌کننده است و ممکن است باعث شود transactionهای دیگر تصمیم‌های نادرست بگیرند.
- اگر transaction abort شود، تمام writeهایی که انجام داده باید rollback شوند (مانند شکل ۷-۳). اگر database اجازهٔ dirty read بدهد، یک transaction ممکن است dataای را ببیند که بعداً rollback می‌شود؛ یعنی dataای که هرگز واقعاً commit نشده است. Reasoning دربارهٔ پیامدهای چنین وضعیتی خیلی زود ذهن‌فرسا می‌شود.

#### No Dirty Writes

اگر دو transaction به‌صورت concurrent تلاش کنند یک object یکسان را در database update کنند چه اتفاقی می‌افتد؟ ما نمی‌دانیم writeها به چه ترتیبی رخ خواهند داد، اما معمولاً فرض می‌کنیم write بعدی write قبلی را overwrite می‌کند.

اما اگر write قبلی بخشی از transactionای باشد که هنوز commit نشده و write بعدی value commit‌نشده را overwrite کند چه؟ به این وضعیت **dirty write** گفته می‌شود [28]. Transactionهایی که در read committed isolation level اجرا می‌شوند باید جلوی dirty write را بگیرند؛ معمولاً با delay کردن write دوم تا زمانی که transaction مربوط به write اول commit یا abort شود.

جلوگیری از dirty write از بعضی concurrency problemها جلوگیری می‌کند:

- اگر transactionها چند object را update کنند، dirty write می‌تواند به outcome بدی منجر شود. برای مثال، شکل ۷-۵ را در نظر بگیرید که یک website فروش car دست‌دوم را نشان می‌دهد و دو نفر، Alice و Bob، هم‌زمان تلاش می‌کنند یک car یکسان را بخرند. خرید car به دو database write نیاز دارد: listing موجود در website باید برای نشان دادن buyer update شود و sales invoice باید برای buyer ارسال شود. در شکل ۷-۵، فروش به Bob واگذار می‌شود، چون او update برنده را روی table مربوط به listing انجام می‌دهد، اما invoice برای Alice ارسال می‌شود، چون او update برنده را روی table مربوط به invoice انجام می‌دهد. Read committed از چنین اتفاق‌هایی جلوگیری می‌کند.
- بااین‌حال، read committed از race condition میان دو counter increment در شکل ۷-۱ جلوگیری نمی‌کند. در این حالت write دوم پس از commit شدن transaction اول رخ می‌دهد، بنابراین dirty write نیست. این وضعیت همچنان نادرست است، اما دلیل متفاوتی دارد؛ در بخش «Preventing Lost Updates» در صفحهٔ ۲۴۲ توضیح می‌دهیم چگونه چنین counter incrementهایی را safe کنیم.

**شکل ۷-۵.** در حضور dirty write، writeهای متناقض transactionهای مختلف ممکن است با یکدیگر قاطی شوند.

#### Implementing Read Committed

Read committed isolation level بسیار محبوب است. این level در Oracle 11g، PostgreSQL، SQL Server 2012، MemSQL و بسیاری databaseهای دیگر، setting پیش‌فرض است [8].

Databaseها معمولاً با استفاده از row-level lock جلوی dirty write را می‌گیرند: وقتی transaction می‌خواهد object مشخصی—row یا document—را modify کند، ابتدا باید lock آن object را acquire کند. سپس باید این lock را تا commit یا abort شدن transaction نگه دارد. برای هر object مشخص فقط یک transaction می‌تواند lock را در اختیار داشته باشد؛ اگر transaction دیگری بخواهد همان object را write کند، باید تا commit یا abort شدن transaction اول صبر کند، سپس lock را acquire کند و ادامه دهد. این locking در read committed mode یا isolation levelهای قوی‌تر، به‌صورت automatic توسط database انجام می‌شود.

اما چگونه از dirty read جلوگیری کنیم؟ یک option این است که از همان lock استفاده کنیم و از هر transactionای که می‌خواهد object را read کند بخواهیم lock را برای مدت کوتاهی acquire کند و بلافاصله پس از read کردن release کند. در این صورت read نمی‌تواند زمانی انجام شود که object دارای valueای dirty و commit‌نشده است، چون در آن مدت lock در اختیار transactionای است که write را انجام داده است.

بااین‌حال، approach مبتنی بر read lock در عمل خوب کار نمی‌کند، چون یک write transaction طولانی می‌تواند transactionهای read-only زیادی را مجبور کند تا پایان آن transaction صبر کنند. این کار response time مربوط به read-only transactionها را بدتر می‌کند و برای operability نامناسب است: کند شدن یک بخش application، به‌دلیل انتظار برای lockها، روی بخش کاملاً متفاوتی از application اثر زنجیره‌ای می‌گذارد.

به همین دلیل، بیشتر databaseهاvi با approach نشان‌داده‌شده در شکل ۷-۴ از dirty read جلوگیری می‌کنند: برای هر objectای که write شده است، database هم value قدیمی و commit‌شده و هم value جدیدی را که transaction دارندهٔ write lock set کرده است نگه می‌دارد. تا زمانی که transaction در حال اجراست، هر transaction دیگری که object را read کند value قدیمی را دریافت می‌کند. فقط پس از commit شدن value جدید است که transactionها به read کردن value جدید switch می‌کنند.

*پاورقی:* در زمان نگارش کتاب، تنها databaseهای mainstreamای که برای read committed isolation از lock استفاده می‌کردند IBM DB2 و Microsoft SQL Server در configuration مربوط به `read_committed_snapshot=off` بودند [23, 36].

### Snapshot Isolation and Repeatable Read

اگر سطحی به read committed نگاه کنید، شاید بتوانید database را ببخشید اگر فکر کنید این level تمام نیازهای transaction را برآورده می‌کند: abortها را ممکن می‌کند (که برای Atomicity لازم‌اند)، مانع read کردن resultهای ناقص transactionها می‌شود و از intermingled شدن writeهای concurrent جلوگیری می‌کند. این‌ها featureهای مفیدی هستند و guaranteeهایی بسیار قوی‌تر از system بدون transaction به شمار می‌روند.

بااین‌حال، هنگام استفاده از این isolation level هنوز راه‌های زیادی برای رخ دادن concurrency bug وجود دارد. برای مثال، شکل ۷-۶ مشکلی را نشان می‌دهد که می‌تواند با read committed رخ دهد.

**شکل ۷-۶.** read skew: Alice database را در stateای inconsistent مشاهده می‌کند.

فرض کنید Alice در یک bank، ۱۰۰۰ دلار savings دارد که میان دو account و هرکدام با balance برابر ۵۰۰ دلار تقسیم شده است. حالا transactionای ۱۰۰ دلار را از یکی از accountهای او به دیگری transfer می‌کند. اگر Alice بدشانس باشد و دقیقاً هنگام پردازش این transaction فهرست balanceهای account خود را نگاه کند، ممکن است balance یک account را پیش از رسیدن incoming payment ببیند؛ یعنی ۵۰۰ دلار. سپس account دیگر را پس از انجام outgoing transfer ببیند؛ یعنی balance جدید ۴۰۰ دلار. در این حالت برای Alice به نظر می‌رسد مجموع accountهایش فقط ۹۰۰ دلار است و ۱۰۰ دلار در هوا ناپدید شده است.

این anomaly **nonrepeatable read** یا **read skew** نام دارد: اگر Alice در پایان transaction دوباره balance account 1 را read کند، value متفاوتی—۶۰۰ دلار—از value قبلی خواهد دید. Read skew در read committed isolation قابل‌قبول تلقی می‌شود، چون balanceهایی که Alice دیده در لحظهٔ read شدن commit شده بودند.

واژهٔ skew متأسفانه overloaded است: پیش‌تر از آن برای اشاره به workload نامتوازن و hot spot استفاده کردیم (به بخش «Skewed Workloads and Relieving Hot Spots» در صفحهٔ ۲۰۵ مراجعه کنید)، اما در اینجا به یک timing anomaly اشاره دارد.

در مورد Alice، این مشکل دائمی نیست، چون به احتمال زیاد اگر چند ثانیه بعد website online banking را reload کند balanceهای consistent را خواهد دید. بااین‌حال، بعضی situationها چنین inconsistency موقتی را تحمل نمی‌کنند:

**Backupها**

گرفتن backup نیازمند copy کردن کل database است و این کار در یک database بزرگ ممکن است ساعت‌ها طول بکشد. در زمانی که backup process در حال اجراست، writeها به database ادامه پیدا می‌کنند. در نتیجه ممکن است بعضی بخش‌های backup شامل version قدیمی‌تر data و بخش‌های دیگر شامل version جدیدتر باشند. اگر لازم باشد از چنین backupای restore کنید، inconsistencyها—مانند ناپدید شدن پول—دائمی می‌شوند.

**Analytic queryها و integrity checkها**

گاهی می‌خواهید queryای اجرا کنید که بخش بزرگی از database را scan می‌کند. چنین queryهایی در analytics رایج‌اند (به بخش «Transaction Processing or Analytics?» در صفحهٔ ۹۰ مراجعه کنید) یا ممکن است بخشی از integrity check دوره‌ای باشند که بررسی می‌کند همه‌چیز درست است؛ مانند monitoring برای data corruption. اگر این queryها بخش‌های مختلف database را در pointهای زمانی متفاوت ببینند، احتمالاً resultهای nonsensical برمی‌گردانند.

**Snapshot isolation** رایج‌ترین راه‌حل این مشکل است [28]. ایده این است که هر transaction از یک snapshot consistent از database read کند؛ یعنی transaction تمام dataای را می‌بیند که در ابتدای transaction در database commit شده بود. حتی اگر data بعداً توسط transaction دیگری تغییر کند، هر transaction فقط data قدیمی مربوط به همان point in time را می‌بیند.

Snapshot isolation برای queryهای طولانی‌مدت و read-only مانند backup و analytics بسیار سودمند است. وقتی dataای که query روی آن کار می‌کند هم‌زمان با اجرای query در حال تغییر است، reasoning دربارهٔ معنای query بسیار دشوار می‌شود. وقتی transaction بتواند snapshot consistent و freezeشده در یک point in time از database ببیند، درک آن بسیار آسان‌تر است.

Snapshot isolation feature محبوبی است و PostgreSQL، MySQL با storage engine مربوط به InnoDB، Oracle، SQL Server و databaseهای دیگر از آن پشتیبانی می‌کنند [23, 31, 32].

#### Implementing Snapshot Isolation

مانند read committed isolation، implementationهای snapshot isolation نیز معمولاً برای جلوگیری از dirty write از write lock استفاده می‌کنند (به بخش «Implementing read committed» در صفحهٔ ۲۳۶ مراجعه کنید). این یعنی transactionای که write انجام می‌دهد می‌تواند جلوی progress transaction دیگری را که همان object را write می‌کند بگیرد. بااین‌حال، readها به lock نیاز ندارند.

از نظر performance، اصل کلیدی snapshot isolation این است که readerها هرگز writerها را block نمی‌کنند و writerها نیز هرگز readerها را block نمی‌کنند. این ویژگی به database اجازه می‌دهد هم‌زمان با پردازش عادی writeها، queryهای read طولانی را روی snapshot consistent اجرا کند، بدون اینکه میان آن دو lock contention ایجاد شود.

برای implement کردن snapshot isolation، databaseها باید چند version مختلف و commit‌شده از یک object را نگه دارند، چون transactionهای مختلفی که در حال اجرا هستند ممکن است لازم باشد state database را در pointهای زمانی متفاوت ببینند. از آنجا که این technique چند version از object را کنار هم نگه می‌دارد، **multi-version concurrency control (MVCC)** نامیده می‌شود.

اگر database فقط لازم بود read committed isolation را فراهم کند و snapshot isolation لازم نبود، نگه‌داشتن دو version از object کافی بود: version commit‌شده و version overwriteشده اما هنوز commit‌نشده. بااین‌حال، storage engineهایی که snapshot isolation را پشتیبانی می‌کنند معمولاً برای read committed isolation خود نیز از MVCC استفاده می‌کنند. یک approach رایج این است که read committed برای هر query یک snapshot جدا داشته باشد، درحالی‌که snapshot isolation از یک snapshot یکسان برای کل transaction استفاده می‌کند.

شکل ۷-۷ نشان می‌دهد snapshot isolation مبتنی بر MVCC در PostgreSQL چگونه implement می‌شود [31]؛ implementationهای دیگر نیز مشابه‌اند. هنگام شروع transaction، یک transaction ID یکتا و همیشه افزایشیvi به آن داده می‌شود که `txid` نام دارد. هر زمان transaction چیزی در database write کند، dataای که write کرده با transaction ID مربوط به writer tag می‌شود.

*پاورقی:* دقیق‌تر بگوییم، transaction IDها integerهای ۳۲بیتی هستند و پس از تقریباً ۴ میلیارد transaction overflow می‌کنند. process مربوط به vacuum در PostgreSQL cleanup لازم را انجام می‌دهد تا overflow روی data اثر نگذارد.

**شکل ۷-۷.** implement کردن snapshot isolation با استفاده از objectهای multi-version.

هر row در table یک field به نام `created_by` دارد که ID مربوط به transactionای را نگه می‌دارد که row را insert کرده است. علاوه بر این، هر row یک field به نام `deleted_by` دارد که در ابتدا خالی است. اگر transactionای row را delete کند، row واقعاً از database delete نمی‌شود؛ بلکه با set کردن field `deleted_by` روی ID مربوط به transactionی که deletion را درخواست کرده، برای deletion mark می‌شود. مدتی بعد، وقتی مطمئن شدیم هیچ transactionای دیگر نمی‌تواند به data حذف‌شده دسترسی داشته باشد، یک garbage collection process در database rowهای markشده برای deletion را remove و فضای آن‌ها را آزاد می‌کند.

یک update در داخل به delete و create تبدیل می‌شود. برای مثال، در شکل ۷-۷، transaction شمارهٔ ۱۳ مبلغ ۱۰۰ دلار از account شمارهٔ ۲ کم می‌کند و balance را از ۵۰۰ به ۴۰۰ تغییر می‌دهد. حالا table مربوط به account در واقع دو row برای account شمارهٔ ۲ دارد: rowای با balance برابر ۵۰۰ که transaction ۱۳ آن را برای deletion mark کرده و rowای با balance برابر ۴۰۰ که transaction ۱۳ آن را create کرده است.

#### Visibility Rules for Observing a Consistent Snapshot

وقتی transaction از database read می‌کند، از transaction IDها برای تصمیم‌گیری دربارهٔ objectهای visible و invisible استفاده می‌شود. database با تعریف دقیق visibility ruleها می‌تواند snapshot consistentای از database را به application ارائه دهد. این فرایند به شکل زیر کار می‌کند:

1. در شروع هر transaction، database فهرستی از تمام transactionهای دیگری را که در آن لحظه in progress هستند—یعنی هنوز commit یا abort نشده‌اند—تهیه می‌کند. هر writeای که این transactionها انجام داده‌اند نادیده گرفته می‌شود، حتی اگر آن transactionها بعداً commit شوند.
2. تمام writeهای transactionهای abortشده نادیده گرفته می‌شوند.
3. تمام writeهای transactionهایی که transaction ID بزرگ‌تری دارند—یعنی پس از شروع transaction فعلی start شده‌اند—نادیده گرفته می‌شوند، صرف‌نظر از اینکه آن transactionها commit شده‌اند یا نه.
4. تمام writeهای دیگر برای queryهای application visible هستند.

این ruleها هم برای create و هم برای delete کردن objectها اعمال می‌شوند. در شکل ۷-۷، وقتی transaction شمارهٔ ۱۲ از account شمارهٔ ۲ read می‌کند، balance برابر ۵۰۰ را می‌بیند، چون deletion مربوط به balance برابر ۵۰۰ توسط transaction شمارهٔ ۱۳ انجام شده است و طبق rule شمارهٔ ۳، transaction ۱۲ نمی‌تواند deletion انجام‌شده توسط transaction ۱۳ را ببیند. Creation مربوط به balance برابر ۴۰۰ نیز طبق همان rule هنوز visible نیست.

به بیان دیگر، object زمانی visible است که هر دو شرط زیر برقرار باشند:

- در زمان start شدن transaction مربوط به reader، transactionای که object را create کرده قبلاً commit شده باشد.
- object برای deletion mark نشده باشد؛ یا اگر mark شده، transactionای که deletion را درخواست کرده در زمان start شدن transaction مربوط به reader هنوز commit نشده باشد.

یک transaction طولانی‌مدت ممکن است برای مدت زیادی از یک snapshot استفاده کند و به read کردن valueهایی ادامه دهد که از دید transactionهای دیگر مدت‌هاست overwrite یا delete شده‌اند. database با update نکردن valueها در جای خود و در عوض با ایجاد version جدید در هر تغییر، می‌تواند snapshot consistent فراهم کند و فقط overhead اندکی متحمل شود.

#### Indexes and Snapshot Isolation

indexها در یک multi-version database چگونه کار می‌کنند؟ یک option این است که index صرفاً به تمام versionهای یک object اشاره کند و از index query بخواهد versionهای objectی را که برای transaction فعلی visible نیستند filter کند. وقتی garbage collection، versionهای قدیمی object را که دیگر برای هیچ transactionای visible نیستند remove می‌کند، index entryهای متناظر نیز می‌توانند remove شوند.

در عمل، جزئیات implementation زیادی performance مربوط به multi-version concurrency control را تعیین می‌کنند. برای مثال، PostgreSQL برای جلوگیری از update کردن index زمانی که versionهای مختلف یک object بتوانند در یک page جا بگیرند، optimizationهایی دارد [31].

approach دیگری در CouchDB، Datomic و LMDB استفاده می‌شود. این systemها نیز از B-tree استفاده می‌کنند (به بخش «B-Trees» در صفحهٔ ۷۹ مراجعه کنید)، اما variantای append-only یا copy-on-write به‌کار می‌برند که هنگام update، pageهای tree را overwrite نمی‌کند؛ بلکه copy جدیدی از هر page تغییرکرده ایجاد می‌کند. pageهای parent تا root tree copy می‌شوند و برای اشاره به versionهای جدید pageهای child update می‌شوند. pageهایی که تحت تأثیر write قرار نگرفته‌اند لازم نیست copy شوند و immutable باقی می‌مانند [33, 34, 35].

در append-only B-tree، هر write transaction یا batchای از transactionها root جدیدی برای B-tree ایجاد می‌کند و هر root، snapshot consistentای از database در point in time ایجاد شدن خود است. دیگر نیازی نیست objectها را بر اساس transaction ID filter کنیم، چون writeهای بعدی نمی‌توانند B-tree موجود را modify کنند؛ آن‌ها فقط می‌توانند root جدیدی برای tree بسازند. بااین‌حال، این approach به background processای برای compaction و garbage collection نیز نیاز دارد.

#### Repeatable Read and Naming Confusion

Snapshot isolation isolation level مفیدی است، به‌خصوص برای read-only transactionها. بااین‌حال، databaseهای مختلفی که آن را implement می‌کنند، نام‌های متفاوتی برایش به‌کار می‌برند. در Oracle به آن serializable گفته می‌شود و در PostgreSQL و MySQL آن را repeatable read می‌نامند [23].

دلیل این naming confusion آن است که SQL standard مفهومی به نام snapshot isolation ندارد، چون standard بر اساس تعریف ۱۹۷۵ System R از isolation levelها بنا شده [2] و snapshot isolation در آن زمان هنوز اختراع نشده بود. در عوض، standard مفهوم repeatable read را تعریف می‌کند که در نگاه اول شبیه snapshot isolation است. PostgreSQL و MySQL isolation level مربوط به snapshot خود را repeatable read می‌نامند، چون requirementهای standard را برآورده می‌کند و در نتیجه می‌توانند ادعا کنند با standard سازگارند.

متأسفانه تعریف isolation levelها در SQL standard flawed است—ambiguous و imprecise است و به‌اندازه‌ای که از یک standard انتظار می‌رود implementation-independent نیست [28]. با اینکه چند database repeatable read را implement می‌کنند، guaranteeهای واقعی آن‌ها با وجود standardized بودن ظاهری بسیار متفاوت است [23]. در literature پژوهشی definition رسمی‌ای برای repeatable read وجود دارد [29, 30]، اما بیشتر implementationها آن definition رسمی را satisfy نمی‌کنند. برای پیچیده‌تر شدن ماجرا، IBM DB2 از عبارت «repeatable read» برای اشاره به serializability استفاده می‌کند [8].

در نتیجه، واقعاً هیچ‌کس نمی‌داند repeatable read دقیقاً چه معنایی دارد.

### Preventing Lost Updates

Read committed و snapshot isolation که تا اینجا بررسی کردیم، عمدتاً دربارهٔ guaranteeهایی بودند که یک read-only transaction در حضور writeهای concurrent می‌تواند دریافت کند. ما مسئلهٔ write هم‌زمان دو transaction را تا حد زیادی نادیده گرفتیم و فقط dirty write را بررسی کردیم (به بخش «No Dirty Writes» در صفحهٔ ۲۳۵ مراجعه کنید)؛ dirty write یکی از انواع write-write conflict است که ممکن است رخ دهد.

انواع جالب دیگری از conflict نیز میان transactionهایی که به‌صورت concurrent write می‌کنند رخ می‌دهد. شناخته‌شده‌ترین آن‌ها **lost update** است که در شکل ۷-۱ با مثال دو counter increment concurrent نشان داده شد.

Lost update زمانی رخ می‌دهد که application valueای را از database read کند، آن را modify کند و value تغییرکرده را دوباره write کند؛ یعنی یک read-modify-write cycle. اگر دو transaction این کار را به‌صورت concurrent انجام دهند، ممکن است یکی از modificationها از دست برود، چون write دوم شامل modification اول نیست. گاهی می‌گوییم write بعدی، write قبلی را clobber می‌کند. این pattern در scenarioهای مختلفی رخ می‌دهد:

- increment کردن counter یا update کردن account balance؛ این کار به read کردن value فعلی، محاسبهٔ value جدید و write کردن آن نیاز دارد.
- ایجاد یک local change در valueای پیچیده؛ برای مثال، اضافه کردن element به list داخل یک JSON document که نیازمند parse کردن document، ایجاد change و write کردن document تغییرکرده است.
- edit کردن هم‌زمان یک wiki page توسط دو user، به‌طوری‌که هر user changeهای خود را با ارسال کل محتوای page به server save می‌کند و هرچه در آن لحظه در database وجود دارد overwrite می‌شود.

چون این مسئله بسیار رایج است، solutionهای متنوعی برای آن توسعه داده شده است.

#### Atomic Write Operations

بسیاری از databaseها atomic update operation ارائه می‌کنند که نیاز به پیاده‌سازی read-modify-write cycle در application code را از بین می‌برند. اگر بتوانید code خود را بر اساس این operationها express کنید، معمولاً بهترین solution همین است. برای مثال، instruction زیر در بیشتر relational databaseها concurrency-safe است:

```sql
UPDATE counters SET value = value + 1 WHERE key = 'foo';
```

به‌طور مشابه، document databaseهایی مانند MongoDB operationهای atomicی برای ایجاد local modification در بخشی از JSON document ارائه می‌کنند و Redis operationهای atomicی برای modify کردن data structureهایی مانند priority queue فراهم می‌کند. همهٔ writeها را نمی‌توان به‌سادگی با operationهای atomic express کرد؛ برای مثال، update کردن wiki page شامل text editing دلخواه است.viii اما در situationهایی که operation atomic قابل‌استفاده است، معمولاً بهترین choice محسوب می‌شود.

Atomic operationها معمولاً با گرفتن exclusive lock روی object هنگام read شدن implement می‌شوند، به‌طوری‌که transaction دیگری تا apply شدن update نمی‌تواند object را read کند. این technique گاهی **cursor stability** نامیده می‌شود [36, 37]. گزینهٔ دیگر این است که تمام atomic operationها را مجبور کنیم روی یک thread واحد اجرا شوند.

متأسفانه ORM frameworkها به‌سادگی اجازه می‌دهند به‌طور تصادفی codeای بنویسید که read-modify-write cycle ناامن اجرا می‌کند، به‌جای اینکه از atomic operationهای فراهم‌شده توسط database استفاده کند [38]. اگر بدانید چه می‌کنید، این مسئله مشکل‌ساز نیست، اما potential source باگ‌های ظریفی است که پیدا کردنشان با testing دشوار خواهد بود.

*پاورقی:* می‌توان edit کردن یک text document را به‌صورت streamای از atomic mutationها express کرد، اما این کار نسبتاً پیچیده است. برای چند reference به بخش «Automatic Conflict Resolution» در صفحهٔ ۱۷۴ مراجعه کنید.

#### Explicit Locking

اگر atomic operationهای built-in database functionality لازم را فراهم نکنند، گزینهٔ دیگر برای جلوگیری از lost update این است که application به‌صورت explicit objectهایی را که قرار است update شوند lock کند. سپس application می‌تواند read-modify-write cycle را انجام دهد و اگر transaction دیگری به‌صورت concurrent تلاش کند همان object را read کند، مجبور شود تا پایان read-modify-write cycle اول صبر کند.

برای مثال، یک multiplayer game را در نظر بگیرید که چند player می‌توانند یک figure یکسان را به‌صورت concurrent حرکت دهند. در این حالت ممکن است atomic operation کافی نباشد، چون application علاوه بر update باید مطمئن شود move یک player با ruleهای game سازگار است؛ این logic چیزی نیست که بتوان آن را به‌شکل sensible در database query implement کرد. در عوض می‌توانید از lock استفاده کنید تا دو player نتوانند هم‌زمان همان piece را حرکت دهند، مانند Example 7-1.

**Example 7-1.** lock کردن explicit rowها برای جلوگیری از lost update

```sql
BEGIN TRANSACTION;

SELECT * FROM figures
  WHERE name = 'robot' AND game_id = 222
  FOR UPDATE;

-- Check whether move is valid, then update the position
-- of the piece that was returned by the previous SELECT.
UPDATE figures SET position = 'c4' WHERE id = 1234;

COMMIT;
```

عبارت `FOR UPDATE` به database می‌گوید تمام rowهایی را که این query برمی‌گرداند lock کند.

این روش کار می‌کند، اما برای درست انجام دادن آن باید با دقت دربارهٔ application logic فکر کنید. به‌سادگی ممکن است فراموش کنید جایی از code یک lock لازم را اضافه کنید و در نتیجه race condition ایجاد شود.

#### Automatically Detecting Lost Updates

Atomic operationها و lockها با مجبور کردن read-modify-write cycleها به اجرای sequential، از lost update جلوگیری می‌کنند. یک alternative این است که اجازه دهیم آن‌ها به‌صورت parallel اجرا شوند و اگر transaction manager lost update را detect کرد، transaction را abort کند و آن را مجبور کند read-modify-write cycle خود را retry کند.

مزیت این approach آن است که databaseها می‌توانند این check را به‌صورت efficient و همراه با snapshot isolation انجام دهند. در واقع، PostgreSQL در repeatable read، Oracle در serializable و SQL Server در snapshot isolation به‌صورت automatic تشخیص می‌دهند lost update رخ داده است و transaction خاطی را abort می‌کنند. بااین‌حال، repeatable read در MySQL/InnoDB lost update را detect نمی‌کند [23]. بعضی نویسنده‌ها [28, 30] معتقدند database برای اینکه snapshot isolation ارائه دهد باید جلوی lost update را بگیرد؛ بر اساس این definition، MySQL snapshot isolation ارائه نمی‌کند.

تشخیص lost update feature بسیار خوبی است، چون application code را مجبور نمی‌کند از feature خاصی در database استفاده کند. ممکن است استفاده از lock یا atomic operation را فراموش کنید و bug ایجاد شود، اما lost update detection به‌صورت automatic انجام می‌شود و در نتیجه خطاپذیری کمتری دارد.

#### Compare-and-Set

در databaseهایی که transaction ارائه نمی‌کنند، گاهی atomic compare-and-set operation پیدا می‌کنید؛ این operation پیش‌تر در بخش «Single-Object Writes» در صفحهٔ ۲۳۰ ذکر شد. هدف آن جلوگیری از lost update است: update فقط زمانی انجام می‌شود که value از آخرین read شما تغییر نکرده باشد. اگر value فعلی با چیزی که قبلاً read کرده‌اید match نکند، update هیچ اثری ندارد و read-modify-write cycle باید retry شود.

برای مثال، برای جلوگیری از update هم‌زمان یک wiki page توسط دو user، ممکن است چیزی شبیه query زیر را امتحان کنید و انتظار داشته باشید update فقط در صورتی رخ دهد که content page از زمان شروع edit توسط user تغییر نکرده باشد:

```sql
-- This may or may not be safe, depending on the database implementation
UPDATE wiki_pages SET content = 'new content'
  WHERE id = 1234 AND content = 'old content';
```

اگر content تغییر کرده و دیگر با `old content` match نکند، این update هیچ اثری نخواهد داشت؛ بنابراین باید check کنید update واقعاً اثر کرده است یا نه و در صورت نیاز retry کنید. بااین‌حال، اگر database اجازه دهد `WHERE` clause از یک old snapshot read کند، این statement ممکن است جلوی lost update را نگیرد، چون condition ممکن است true باشد، حتی اگر write concurrent دیگری در حال رخ دادن باشد. پیش از تکیه کردن بر compare-and-set operation database خود، مطمئن شوید که این operation safe است.

#### Conflict Resolution and Replication

در replicated databaseها (به Chapter 5 مراجعه کنید)، جلوگیری از lost update بُعد دیگری پیدا می‌کند: چون copyهایی از data روی چند node وجود دارد و data ممکن است به‌صورت concurrent روی nodeهای مختلف modify شود، برای جلوگیری از lost update به stepهای بیشتری نیاز داریم.

Lock و compare-and-set فرض می‌کنند یک copy واحد و up-to-date از data وجود دارد. بااین‌حال، databaseهایی با multi-leader یا leaderless replication معمولاً اجازه می‌دهند چند write به‌صورت concurrent رخ دهد و آن‌ها را asynchronously replicate می‌کنند؛ بنابراین نمی‌توانند تضمین کنند فقط یک copy up-to-date از data وجود دارد. در این context، techniqueهای مبتنی بر lock یا compare-and-set کاربرد ندارند. (در بخش «Linearizability» در صفحهٔ ۳۲۴ دوباره به این موضوع می‌پردازیم.)

در عوض، همان‌طور که در بخش «Detecting Concurrent Writes» در صفحهٔ ۱۸۴ بحث کردیم، approach رایج در چنین replicated databaseهایی این است که اجازه دهیم writeهای concurrent چند version متناقض از value ایجاد کنند—که **sibling** نیز نامیده می‌شوند—و سپس با application code یا data structureهای special، این versionها را بعداً resolve و merge کنیم.

Atomic operationها در replicated context می‌توانند به‌خوبی کار کنند، به‌خصوص اگر **commutative** باشند؛ یعنی بتوان آن‌ها را به ترتیب متفاوت روی replicaهای مختلف apply کرد و همچنان به result یکسانی رسید. برای مثال، increment کردن counter یا اضافه کردن element به یک set، operationهای commutative هستند. این ایده در datatypeهای Riak 2.0 به‌کار رفته است که از lost update میان replicaها جلوگیری می‌کنند. وقتی valueای به‌صورت concurrent توسط clientهای مختلف update شود، Riak updateها را به‌صورت automatic و طوری merge می‌کند که هیچ updateای از دست نرود [39].

در مقابل، روش conflict resolution مربوط به **last write wins (LWW)** مستعد lost update است؛ همان‌طور که در بخش «Last write wins (discarding concurrent writes)» در صفحهٔ ۱۸۶ بحث شد. متأسفانه LWW در بسیاری از replicated databaseها default است.

### Write Skew and Phantoms

در بخش‌های قبلی dirty write و lost update را دیدیم؛ این دو نوع race condition زمانی رخ می‌دهند که transactionهای مختلف به‌صورت concurrent تلاش کنند objectهای یکسانی را write کنند. برای جلوگیری از data corruption باید این race conditionها را prevent کرد؛ یا database به‌صورت automatic این کار را انجام دهد، یا application با safeguardهای دستی مانند lock و atomic write operation از آن‌ها جلوگیری کند.

بااین‌حال، این تمام فهرست race conditionهای احتمالی میان writeهای concurrent نیست. در این بخش exampleهای ظریف‌تری از conflict را می‌بینیم.

برای شروع، این example را در نظر بگیرید: applicationای برای doctorها می‌نویسید تا shiftهای on-call خود را در یک hospital مدیریت کنند. Hospital معمولاً تلاش می‌کند در هر لحظه چند doctor on call داشته باشد، اما حتماً باید دست‌کم یک doctor on call وجود داشته باشد. Doctorها می‌توانند shift خود را رها کنند—مثلاً اگر خودشان بیمار باشند—به شرطی که دست‌کم یک colleague در آن shift on call باقی بماند [40, 41].

حالا فرض کنید Alice و Bob دو doctor on-call برای shift مشخصی هستند. هر دو حال خوبی ندارند و تصمیم می‌گیرند leave بگیرند. متأسفانه تقریباً هم‌زمان روی button مربوط به off call شدن click می‌کنند. شکل ۷-۸ نشان می‌دهد بعد از آن چه رخ می‌دهد.

**شکل ۷-۸.** نمونه‌ای از write skew که باعث bug در application می‌شود.

در هر transaction، application ابتدا check می‌کند که در حال حاضر دو doctor یا بیشتر on call هستند؛ اگر چنین باشد، فرض می‌کند off call شدن یک doctor safe است. چون database از snapshot isolation استفاده می‌کند، هر دو check مقدار ۲ را برمی‌گردانند و هر دو transaction به مرحلهٔ بعد می‌روند. Alice record مربوط به خودش را update می‌کند تا off call شود و Bob نیز record خودش را به همین شکل update می‌کند. هر دو transaction commit می‌شوند و حالا هیچ doctorای on call نیست.

در نتیجه requirement مربوط به وجود دست‌کم یک doctor on call نقض شده است.

#### Characterizing Write Skew

این anomaly **write skew** نام دارد [28]. این وضعیت نه dirty write است و نه lost update، چون دو transaction دو object متفاوت را update می‌کنند؛ record مربوط به on-call بودن Alice و record مربوط به on-call بودن Bob. وجود conflict در اینجا کمتر obvious است، اما بدون شک یک race condition وجود دارد: اگر دو transaction یکی پس از دیگری اجرا می‌شدند، doctor دوم اجازه نداشت off call شود. رفتار anomalous فقط به این دلیل ممکن شد که transactionها به‌صورت concurrent اجرا شدند.

می‌توانید write skew را generalization مسئلهٔ lost update در نظر بگیرید. Write skew زمانی رخ می‌دهد که دو transaction objectهای یکسانی را read کنند و سپس بعضی از همان objectها را update کنند؛ البته transactionهای مختلف ممکن است objectهای متفاوتی را update کنند. در حالت خاصی که transactionهای مختلف یک object یکسان را update کنند، بسته به timing با anomaly مربوط به dirty write یا lost update روبه‌رو می‌شویم.

دیدیم که برای جلوگیری از lost update روش‌های متفاوتی وجود دارد. در مورد write skew، گزینه‌های ما محدودترند:

- Atomic single-object operationها کمک نمی‌کنند، چون چند object درگیر هستند.
- تشخیص automatic lost update که در بعضی implementationهای snapshot isolation وجود دارد نیز متأسفانه کمک نمی‌کند: write skew در repeatable read مربوط به PostgreSQL، repeatable read مربوط به MySQL/InnoDB، serializable مربوط به Oracle یا snapshot isolation مربوط به SQL Server به‌صورت automatic detect نمی‌شود [23]. جلوگیری automatic از write skew به true serializable isolation نیاز دارد (به بخش «Serializability» در صفحهٔ ۲۵۱ مراجعه کنید).
- بعضی databaseها اجازه می‌دهند constraintهایی configure کنید که بعداً توسط database enforce شوند؛ برای مثال uniqueness، foreign key constraint یا محدودیت روی value مشخص. اما برای بیان این requirement که دست‌کم یک doctor باید on call باشد، به constraintای نیاز دارید که چند object را درگیر کند. بیشتر databaseها برای چنین constraintهایی built-in support ندارند، اما بسته به database ممکن است بتوانید آن‌ها را با trigger یا materialized view implement کنید [42].
- اگر نمی‌توانید از serializable isolation level استفاده کنید، second-best option در این case احتمالاً lock کردن explicit rowهایی است که transaction به آن‌ها وابسته است. در example doctorها می‌توانید چیزی شبیه query زیر بنویسید:

```sql
BEGIN TRANSACTION;

SELECT * FROM doctors
  WHERE on_call = true
  AND shift_id = 1234 FOR UPDATE;

UPDATE doctors
  SET on_call = false
  WHERE name = 'Alice'
  AND shift_id = 1234;

COMMIT;
```

مانند قبل، `FOR UPDATE` به database می‌گوید تمام rowهایی را که query برمی‌گرداند lock کند.

#### More Examples of Write Skew

Write skew ممکن است در ابتدا مسئله‌ای esoteric به نظر برسد، اما پس از آشنا شدن با آن، situationهای بیشتری را می‌بینید که می‌تواند در آن رخ دهد. چند example دیگر:

**Meeting room booking system**

فرض کنید می‌خواهید enforce کنید که برای یک meeting room، دو booking در یک زمان وجود نداشته باشد [43]. وقتی کسی می‌خواهد booking ایجاد کند، ابتدا bookingهای متعارض را check می‌کنید؛ یعنی bookingهای همان room که time range آن‌ها overlap دارد. اگر هیچ booking متعارضی پیدا نشد، meeting را create می‌کنید (Example 7-2).ix

**Example 7-2.** meeting room booking system تلاش می‌کند double-booking را جلوگیری کند، اما این code تحت snapshot isolation safe نیست:

```sql
BEGIN TRANSACTION;

-- Check for any existing bookings that overlap with the period of noon-1pm
SELECT COUNT(*) FROM bookings
  WHERE room_id = 123 AND
    end_time > '2015-01-01 12:00' AND start_time < '2015-01-01 13:00';

-- If the previous query returned zero:
INSERT INTO bookings
  (room_id, start_time, end_time, user_id)
  VALUES (123, '2015-01-01 12:00', '2015-01-01 13:00', 666);

COMMIT;
```

متأسفانه snapshot isolation جلوی insert کردن concurrent یک meeting متعارض توسط user دیگری را نمی‌گیرد. برای تضمین اینکه conflict در scheduling رخ نمی‌دهد، دوباره به serializable isolation نیاز دارید.

**Multiplayer game**

در Example 7-1 از lock برای جلوگیری از lost update استفاده کردیم؛ یعنی مطمئن شدیم دو player نمی‌توانند هم‌زمان یک figure یکسان را حرکت دهند. بااین‌حال، lock جلوی حرکت دادن دو figure متفاوت به یک position یکسان روی board را نمی‌گیرد و ممکن است مانع move دیگری که ruleهای game را نقض می‌کند نیز نشود. بسته به نوع ruleای که enforce می‌کنید، ممکن است بتوانید از unique constraint استفاده کنید؛ در غیر این صورت در برابر write skew آسیب‌پذیر هستید.

**Claiming a username**

در websiteای که هر user باید username یکتایی داشته باشد، ممکن است دو user هم‌زمان تلاش کنند accountهایی با username یکسان بسازند. می‌توانید از transaction استفاده کنید تا check کنید name قبلاً گرفته شده است یا نه و اگر گرفته نشده، account را با آن name create کنید. بااین‌حال، مانند exampleهای قبلی، این code تحت snapshot isolation safe نیست.

خوشبختانه unique constraint در اینجا solution ساده‌ای است؛ transaction دومی که تلاش می‌کند username را register کند، به‌دلیل نقض constraint abort خواهد شد.

**Preventing double-spending**

سرویسی که به userها اجازه می‌دهد money یا point خرج کنند باید check کند user بیش از موجودی خود خرج نکند. ممکن است این کار را با insert کردن یک spending item موقت در account user، فهرست کردن تمام itemهای account و check کردن positive بودن sum پیاده کنید [44]. در صورت رخ دادن write skew، ممکن است دو spending item به‌صورت concurrent insert شوند و مجموع آن‌ها balance را منفی کند، بدون اینکه هیچ‌کدام از transactionها متوجه دیگری شود.

#### Phantoms Causing Write Skew

تمام این exampleها pattern مشابهی دارند:

1. یک `SELECT` query check می‌کند آیا requirementای با search کردن rowهایی که با یک search condition match می‌شوند برقرار است یا نه؛ برای مثال دست‌کم دو doctor on call هستند، هیچ bookingای برای آن room در آن زمان وجود ندارد، position روی board از قبل توسط figure دیگری اشغال نشده، username قبلاً گرفته نشده یا هنوز money کافی در account وجود دارد.
2. application code بر اساس result query اول تصمیم می‌گیرد چگونه ادامه دهد؛ شاید operation را انجام دهد یا به user error گزارش کند و transaction را abort کند.
3. اگر application تصمیم بگیرد ادامه دهد، یک write—یعنی `INSERT`، `UPDATE` یا `DELETE`—در database انجام می‌دهد و transaction را commit می‌کند.

اثر این write، precondition مربوط به تصمیم مرحلهٔ ۲ را تغییر می‌دهد. به بیان دیگر، اگر پس از commit کردن write، `SELECT` query مرحلهٔ ۱ را دوباره اجرا کنید، result متفاوتی خواهید گرفت، چون write مجموعهٔ rowهایی را که با search condition match می‌شوند تغییر داده است: حالا یک doctor کمتر on call است، meeting room برای آن زمان booking شده، position روی board توسط figure جابه‌جاشده اشغال شده، username گرفته شده یا money موجود در account کمتر شده است.

این stepها می‌توانند با order متفاوتی نیز رخ دهند. برای مثال، می‌توان ابتدا write را انجام داد، سپس `SELECT` query را اجرا کرد و در پایان بر اساس result query تصمیم گرفت transaction را abort یا commit کند.

در example مربوط به doctor on call، rowای که در step ۳ modify می‌شد یکی از rowهایی بود که در step ۱ برگردانده شده بود. بنابراین می‌توانستیم با lock کردن rowها در step ۱—یعنی `SELECT FOR UPDATE`—transaction را safe کنیم و از write skew جلوگیری کنیم. اما چهار example دیگر متفاوت‌اند: آن‌ها نبود rowهایی را check می‌کنند که با یک search condition match شوند و write نیز rowای اضافه می‌کند که با همان condition match می‌شود. اگر query مرحلهٔ ۱ هیچ rowای برنگرداند، `SELECT FOR UPDATE` نمی‌تواند lock را به چیزی attach کند.

به این effect که write در یک transaction result یک search query را در transaction دیگری تغییر می‌دهد **phantom** گفته می‌شود [3]. Snapshot isolation در read-only queryها از phantom جلوگیری می‌کند، اما در read-write transactionهایی مانند exampleهای بالا، phantom می‌تواند به caseهای بسیار tricky از write skew منجر شود.

#### Materializing Conflicts

اگر مشکل phantom این است که objectی وجود ندارد تا lock را به آن attach کنیم، شاید بتوانیم به‌صورت مصنوعی یک lock object در database ایجاد کنیم.

برای مثال، در مسئلهٔ meeting room booking می‌توانید tableای از time slotها و roomها بسازید. هر row این table متناظر با یک room مشخص در یک time period مشخص است؛ مثلاً بازه‌های ۱۵ دقیقه‌ای. برای تمام combinationهای ممکن از room و time period از قبل row ایجاد می‌کنید؛ برای مثال، برای شش ماه آینده.

حالا transactionای که می‌خواهد booking ایجاد کند می‌تواند rowهای مربوط به room و time period موردنظر را lock کند (`SELECT FOR UPDATE`). پس از acquire کردن lockها، می‌تواند bookingهای overlapping را check کند و مانند قبل booking جدید را insert کند. توجه کنید که table اضافی برای storage کردن information مربوط به booking استفاده نمی‌شود؛ این table صرفاً مجموعه‌ای از lockهاست که برای جلوگیری از modify شدن concurrent bookingهای مربوط به room و time range یکسان به‌کار می‌رود.

این approach **materializing conflicts** نام دارد، چون phantom را می‌گیرد و آن را به lock conflict روی مجموعه‌ای مشخص از rowهای موجود در database تبدیل می‌کند [11]. متأسفانه فهمیدن اینکه conflictها را چگونه materialize کنیم می‌تواند دشوار و error-prone باشد و اینکه mechanism مربوط به concurrency control به data model application نشت کند، design خوشایندی نیست. به همین دلیل materializing conflicts باید زمانی به‌عنوان last resort در نظر گرفته شود که هیچ alternativeای ممکن نباشد. در بیشتر caseها serializable isolation level بسیار preferable است.

## Key Terms

- `Weak Isolation` — isolation levelی که فقط در برابر بعضی concurrency anomalyها محافظت می‌کند و همهٔ تداخل‌ها را حذف نمی‌کند.
- `Isolation Level` — مجموعهٔ guaranteeهای database دربارهٔ visibility و interference میان transactionهای concurrent.
- `Read Committed` — isolation levelی که dirty read و dirty write را منع می‌کند، اما الزاماً read skew یا write skew را نه.
- `Dirty Write` — overwrite کردن valueای که transaction مربوط به آن هنوز commit نشده است.
- `Snapshot Isolation` — isolation levelی که هر transaction را روی snapshot consistent مربوط به شروع آن اجرا می‌کند.
- `Repeatable Read` — نامی که بعضی databaseها برای snapshot isolation به‌کار می‌برند، با guaranteeهایی که ممکن است میان implementationها متفاوت باشد.
- `Multi-Version Concurrency Control (MVCC)` — نگه‌داری چند version از objectها برای ارائهٔ snapshot consistent به transactionها.
- `Read Skew` — مشاهدهٔ valueهای مرتبط در pointهای زمانی متفاوت که یک state موقتاً inconsistent ایجاد می‌کند.
- `Nonrepeatable Read` — read دوبارهٔ object در یک transaction و مشاهدهٔ value متفاوت نسبت به read قبلی.
- `Lost Update` — از دست رفتن modification یک transaction چون write concurrent بعدی آن را overwrite کرده است.
- `Write Skew` — anomalyای که transactionهای concurrent پس از read کردن data مشترک، objectهای متفاوتی را update می‌کنند و invariant نقض می‌شود.
- `Phantom` — تغییر result یک search query در transaction دیگر، به‌دلیل insert، update یا delete کردن rowهای matching.
- `Predicate` — شرط یا expressionای که مشخص می‌کند کدام rowها در یک query باید match شوند.
- `Concurrency Control` — مجموعهٔ mechanismهایی برای مدیریت interaction میان operationهای concurrent.
- `Row-Level Lock` — lock روی یک row یا object که دسترسی هم‌زمان transactionهای دیگر را محدود می‌کند.
- `Cursor Stability` — techniqueای که هنگام read-modify-write object را با lock محافظت می‌کند تا update اتمیک انجام شود.
- `Commutative Operation` — operationای که order اجرای آن روی replicaهای مختلف result نهایی را تغییر نمی‌دهد.
- `Read Uncommitted` — isolation level ضعیفی که dirty read را اجازه می‌دهد، اما dirty write را منع می‌کند.
- `Materializing Conflicts` — تبدیل phantom به lock conflict روی rowهای مصنوعی و از پیش ایجادشده.

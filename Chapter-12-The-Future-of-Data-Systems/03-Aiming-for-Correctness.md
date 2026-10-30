# Chapter 12 — The Future of Data Systems

## Aiming for Correctness

اگر در یک stateless service که فقط data را می‌خواند مشکلی پیش بیاید، معمولاً مسئلهٔ بزرگی نیست: bug را برطرف می‌کنید، service را restart می‌کنید و همه‌چیز به حالت عادی برمی‌گردد. اما systemهای stateful مانند database چنین نیستند؛ آن‌ها طوری طراحی شده‌اند که چیزها را تقریباً برای همیشه به خاطر بسپارند. پس اگر مشکلی رخ دهد، اثرش نیز ممکن است برای همیشه باقی بماند و باید با دقت بیشتری به آن فکر کرد.

می‌خواهیم applicationهایی reliable و درست بسازیم؛ یعنی semantics آن‌ها حتی در حضور faultهای گوناگون روشن و قابل‌فهم باشد. نزدیک به چهار دهه، ویژگی‌های transaction یعنی atomicity، isolation و durability ابزارهای اصلی ساخت applicationهای درست بوده‌اند. اما این پایه‌ها ضعیف‌تر از چیزی هستند که به‌نظر می‌رسند؛ برای نمونه، ابهام و سردرگمی دربارهٔ weak isolation levelها را در نظر بگیرید.

در بعضی حوزه‌ها، transactionها کنار گذاشته می‌شوند و modelهایی جای آن‌ها را می‌گیرند که performance و scalability بهتری دارند، اما semantics آشفته‌تری ارائه می‌کنند. از consistency زیاد صحبت می‌شود، اما تعریف آن اغلب دقیق نیست. بعضی می‌گویند برای دسترس‌پذیری بهتر باید «weak consistency را پذیرفت»، بی‌آنکه روشن باشد این حرف در عمل چه معنایی دارد.

با وجود اهمیت این موضوع، درک و روش‌های مهندسی ما به‌طرز غافلگیرکننده‌ای سست‌اند. برای مثال، تشخیص اینکه اجرای یک application با transaction isolation level یا replication configuration مشخص امن است یا نه، بسیار دشوار است. راه‌حل‌های ساده ممکن است هنگام concurrency کم و نبود fault درست به‌نظر برسند، اما در شرایط دشوارتر bugهای ظریف زیادی نشان دهند.

آزمایش‌های Jepsen از Kyle Kingsbury اختلاف چشمگیر میان safety guaranteeهای ادعاشدهٔ بعضی محصول‌ها و رفتار واقعی‌شان هنگام مشکل شبکه و crash را آشکار کرده‌اند. حتی اگر محصول‌های زیرساختی مانند databaseها بی‌نقص باشند، application code همچنان باید قابلیت‌هایشان را درست به‌کار ببرد؛ کاری که اگر فهم configuration دشوار باشد (مثلاً برای weak isolation levelها یا quorum configurationها) مستعد خطاست.

اگر application شما گاهی می‌تواند data را به‌شکلی پیش‌بینی‌ناپذیر خراب یا گم کند، زندگی ساده‌تر است و شاید بتوانید فقط امیدوار باشید که مشکلی پیش نمی‌آید. اما اگر به guaranteeهای قوی‌تری برای correctness نیاز دارید، serializability و atomic commit روش‌های جاافتاده‌ای هستند؛ البته هزینه دارند: معمولاً فقط در یک datacenter کار می‌کنند (پس معماری geographically distributed را کنار می‌گذارند) و scale و fault tolerance قابل‌دستیابی را محدود می‌کنند.

رویکرد سنتی transaction از میان نمی‌رود، اما به‌نظر من آخرین حرف دربارهٔ ساخت applicationهای درست و مقاوم در برابر fault هم نیست. در این بخش چند راه برای اندیشیدن دربارهٔ correctness در معماری‌های dataflow پیشنهاد می‌کنم.

### The End-to-End Argument for Databases

اینکه application از data systemای با safety propertyهای نسبتاً قوی، مثلاً serializable transaction، استفاده می‌کند تضمین نمی‌کند که data گم یا خراب نشود. اگر bugای در application باعث نوشتن data نادرست یا حذف data از database شود، serializable transaction نجاتتان نمی‌دهد.

این مثال شاید پیش‌پاافتاده به‌نظر برسد، اما باید جدی‌اش گرفت: applicationها bug دارند و آدم‌ها اشتباه می‌کنند. از این مثال در بحث «State, Streams, and Immutability» برای دفاع از data immutable و append-only استفاده کردم، چون اگر code معیوب نتواند data سالم را نابود کند، بازیابی از چنین اشتباه‌هایی آسان‌تر است.

Immutability به‌تنهایی درمان همه‌چیز نیست. مثال ظریف‌تری از data corruption را بررسی کنیم.

#### اجرای exactly-once یک operation

در بحث Fault Tolerance با مفهومی به نام exactly-once (یا effectively-once) semantics آشنا شدیم. اگر هنگام پردازش message مشکلی پیش بیاید، می‌توانید منصرف شوید و message را حذف کنید (یعنی data loss را بپذیرید)، یا دوباره تلاش کنید. در حالت دوم این خطر وجود دارد که بار اول operation واقعاً موفق شده باشد، اما شما از موفقیتش باخبر نشده باشید؛ در نتیجه message دوبار پردازش می‌شود.

پردازش دوباره نوعی data corruption است: نمی‌خواهیم بابت یک service دوبار از مشتری پول بگیریم یا counter را دوبار افزایش دهیم و metric را بیش از مقدار واقعی گزارش کنیم. در اینجا exactly-once یعنی computation را طوری تنظیم کنیم که اثر نهایی همان باشد که اگر faultای رخ نداده بود؛ حتی اگر به‌علت fault، operation واقعاً retry شده باشد. پیش‌تر چند روش رسیدن به این هدف را بررسی کردیم.

یکی از مؤثرترین روش‌ها idempotent کردن operation است؛ یعنی تضمین کنیم اجرای آن یک‌بار یا چندبار اثر یکسانی داشته باشد. اما idempotent کردن operationای که ذاتاً چنین نیست، به تلاش و دقت نیاز دارد: ممکن است لازم باشد metadata اضافی، مانند مجموعهٔ operation IDهایی که یک مقدار را تغییر داده‌اند، نگه دارید و هنگام failover از یک node به node دیگر fencing را رعایت کنید.

#### جلوگیری از duplicate

نیاز به حذف duplicate فقط در stream processing رخ نمی‌دهد. برای مثال، TCP از sequence number بسته‌ها استفاده می‌کند تا آن‌ها را در سمت گیرنده مرتب کند و مشخص کند بسته‌ای در network گم یا تکراری شده است. TCP packetهای گم‌شده را دوباره می‌فرستد و پیش از تحویل data به application، duplicateها را در TCP stack حذف می‌کند.

اما این حذف duplicate فقط در محدودهٔ یک TCP connection کار می‌کند. فرض کنید connection، اتصال client به database است و transaction مثال 12-1 در حال اجراست. در بسیاری از databaseها transaction به client connection وابسته است: اگر client چند query بفرستد، database می‌فهمد همه متعلق به یک transaction هستند، چون از همان TCP connection آمده‌اند. اگر پس از فرستادن `COMMIT` و پیش از دریافت پاسخ database server، اتصال client قطع شود و timeout رخ دهد، client نمی‌داند transaction commit شده یا abort (Figure 8-1).

**Example 12-1. انتقال پول non-idempotent از یک حساب به حساب دیگر**

```sql
BEGIN TRANSACTION;
UPDATE accounts SET balance = balance + 11.00 WHERE account_id = 1234;
UPDATE accounts SET balance = balance - 11.00 WHERE account_id = 4321;
COMMIT;
```

client می‌تواند دوباره به database وصل شود و transaction را retry کند؛ اما دیگر در محدودهٔ حذف duplicate توسط TCP نیست. چون transaction مثال 12-1 idempotent نیست، ممکن است به‌جای ۱۱ دلار، ۲۲ دلار منتقل شود. بنابراین با اینکه این مثال نمونهٔ متداولی برای atomicity در transaction است، در واقع درست نیست؛ بانک‌های واقعی هم این‌طور کار نمی‌کنند.

Two-phase commit protocolها رابطهٔ یک‌به‌یک میان TCP connection و transaction را می‌شکنند؛ چون transaction coordinator باید بتواند پس از fault شبکه دوباره به database وصل شود و دربارهٔ commit یا abort کردن transaction نامشخص تصمیم بگیرد. آیا این برای تضمین اجرای یک‌بارهٔ transaction کافی است؟ متأسفانه نه.

حتی اگر بتوانیم transactionهای تکراری میان client و server پایگاه داده را حذف کنیم، باید network میان device کاربر نهایی و application server را هم در نظر بگیریم. مثلاً browser احتمالاً برای ارسال دستور به server از HTTP POST استفاده می‌کند. شاید کاربر اتصال cellular ضعیفی داشته باشد؛ POST به server برسد، اما پیش از دریافت پاسخ، سیگنال قطع شود. کاربر احتمالاً error می‌بیند و ممکن است دستی دوباره تلاش کند. browser می‌پرسد «مطمئنید می‌خواهید این فرم را دوباره بفرستید؟» و کاربر تأیید می‌کند، چون می‌خواسته operation انجام شود. الگوی Post/Redirect/Get در حالت عادی جلوی این پیام را می‌گیرد، اما وقتی POST timeout شود کمکی نمی‌کند. از دید web server، retry یک request جداست و از دید database نیز transaction جداگانه‌ای است؛ سازوکارهای معمول deduplication این مسئله را حل نمی‌کنند.

#### شناسه‌های operation

برای idempotent کردن operation در چندین مرحلهٔ ارتباط شبکه‌ای، تکیه بر transaction mechanism خود database کافی نیست؛ باید جریان request را از ابتدا تا انتها در نظر گرفت.

برای نمونه، می‌توانید برای operation یک شناسهٔ یکتایی مانند UUID بسازید و آن را در hidden form field در client قرار دهید، یا با hash کردن همهٔ fieldهای مرتبط فرم، operation ID را به دست آورید. اگر browser درخواست POST را دوبار بفرستد، هر دو request یک operation ID دارند. سپس می‌توانید این ID را تا database منتقل کنید و مطمئن شوید هر ID فقط یک‌بار اجرا می‌شود؛ مانند Example 12-2.

**Example 12-2. حذف requestهای تکراری با شناسهٔ یکتا**

```sql
ALTER TABLE requests ADD UNIQUE (request_id);

BEGIN TRANSACTION;

INSERT INTO requests
  (request_id, from_account, to_account, amount)
  VALUES('0286FDB8-D7E1-423F-B40B-792B3608036C', 4321, 1234, 11.00);

UPDATE accounts SET balance = balance + 11.00 WHERE account_id = 1234;
UPDATE accounts SET balance = balance - 11.00 WHERE account_id = 4321;

COMMIT;
```

Example 12-2 به uniqueness constraint روی ستون `request_id` متکی است. اگر transactionای بخواهد IDای را insert کند که از قبل وجود دارد، `INSERT` شکست می‌خورد و transaction abort می‌شود؛ بنابراین operation دوبار اعمال نخواهد شد. relational databaseها معمولاً حتی در weak isolation levelها هم می‌توانند uniqueness constraint را درست حفظ کنند؛ درحالی‌که check-then-insert در سطح application ممکن است تحت isolation غیرserializable شکست بخورد.

علاوه بر حذف requestهای تکراری، table مربوط به `requests` در Example 12-2 نوعی event log هم هست و به event sourcing شباهت دارد. لازم نیست update کردن balance حساب‌ها در همان transaction ثبت event انجام شود، چون این updateها redundant هستند و یک downstream consumer می‌تواند آن‌ها را از request event مشتق کند؛ به شرط آنکه event دقیقاً یک‌بار پردازش شود، که آن را هم می‌توان با request ID تضمین کرد.

#### استدلال end-to-end

سناریوی حذف transactionهای تکراری فقط یک نمونه از اصل عمومی‌تری به نام end-to-end argument است که Saltzer، Reed و Clark در سال ۱۹۸۴ صورت‌بندی کردند:

> یک قابلیت فقط زمانی می‌تواند به‌طور کامل و درست پیاده‌سازی شود که application واقع در endpointهای سیستم ارتباطی از آن آگاه باشد و در پیاده‌سازی‌اش مشارکت کند. بنابراین فراهم‌کردن آن قابلیت صرفاً به‌عنوان ویژگی خود سیستم ارتباطی ممکن نیست. (گاهی نسخه‌ای ناقص از آن قابلیت در سیستم ارتباطی برای بهبود performance مفید است.)

در مثال ما، قابلیت موردنظر حذف duplicate بود. TCP بسته‌های تکراری را در سطح TCP connection حذف می‌کند و بعضی stream processorها در سطح پردازش message به‌اصطلاح exactly-once semantics می‌دهند؛ اما هیچ‌کدام برای جلوگیری از فرستادن مجدد request توسط user، وقتی request اول timeout شده، کافی نیستند. TCP، transactionهای database و stream processorها به‌تنهایی نمی‌توانند همهٔ duplicateها را حذف کنند. راه‌حل باید end-to-end باشد: transaction ID باید از client کاربر نهایی تا database منتقل شود.

این استدلال دربارهٔ بررسی integrity data نیز صدق می‌کند: checksumهای تعبیه‌شده در Ethernet، TCP و TLS می‌توانند corruption بسته‌ها در network را تشخیص دهند، اما فساد ناشی از bug نرم‌افزار در دو سر ارتباط یا خرابی data روی disk را پیدا نمی‌کنند. برای کشف همهٔ منبع‌های احتمالی data corruption، به end-to-end checksum هم نیاز است.

استدلال مشابهی دربارهٔ encryption وجود دارد: password شبکهٔ WiFi خانه جلوی شنود ترافیک WiFi را می‌گیرد، اما از حمله‌گرانی که جای دیگری در اینترنت‌اند محافظت نمی‌کند؛ TLS/SSL میان client و server از شما در برابر حمله‌گر network محافظت می‌کند، اما جلوی compromise شدن server را نمی‌گیرد. فقط end-to-end encryption و authentication می‌توانند در برابر همهٔ این موارد محافظت کنند.

قابلیت‌های low-level مانند حذف duplicate در TCP، checksumهای Ethernet و encryption در WiFi به‌تنهایی ویژگی end-to-end موردنظر را فراهم نمی‌کنند، اما همچنان مفیدند چون احتمال بروز مشکل در لایه‌های بالاتر را کاهش می‌دهند. مثلاً اگر TCP بسته‌ها را به ترتیب درست کنار هم نمی‌گذاشت، HTTP requestها اغلب خراب می‌شدند. فقط باید به یاد داشته باشیم که قابلیت‌های reliability در لایهٔ پایین، به‌تنهایی برای تضمین end-to-end correctness کافی نیستند.

#### به‌کارگیری تفکر end-to-end در data systemها

این ما را به ادعای اصلی برمی‌گرداند: استفادهٔ application از data systemای با safety propertyهای نسبتاً قوی، مانند serializable transaction، تضمین نمی‌کند data گم یا خراب نشود. خود application نیز باید اقدام‌های end-to-end، مانند حذف duplicate، انجام دهد.

این وضعیت تأسف‌بار است، چون پیاده‌سازی درست fault tolerance دشوار است. مکانیزم‌های reliability در لایهٔ پایین، مانند TCP، خوب کار می‌کنند و به همین دلیل faultهای باقی‌مانده در لایه‌های بالاتر نسبتاً کم رخ می‌دهند. خوب می‌شد سازوکارهای باقی‌ماندهٔ fault tolerance را در abstractionای جمع کنیم تا application code لازم نباشد نگرانشان باشد؛ اما گمان می‌کنم هنوز abstraction مناسبی پیدا نکرده‌ایم.

مدت‌هاست transaction را abstraction خوبی می‌دانند و من هم باور دارم مفید است. همان‌طور که در مقدمهٔ Chapter 7 گفتیم، transaction مسئله‌های گوناگونی مانند writeهای هم‌زمان، نقض constraint، crash، قطع network و خرابی disk را به دو نتیجهٔ ممکن تقلیل می‌دهد: commit یا abort. این ساده‌سازی بزرگی برای مدل برنامه‌نویسی است، اما به‌گمانم کافی نیست.

transactionها پرهزینه‌اند، به‌خصوص وقتی فناوری‌های storage ناهمگون را درگیر کنند. اگر چون distributed transaction گران است از آن صرف‌نظر کنیم، ناچار می‌شویم مکانیزم‌های fault tolerance را در application code دوباره پیاده‌سازی کنیم. مثال‌های متعدد کتاب نشان داده‌اند که استدلال دربارهٔ concurrency و partial failure دشوار و خلاف شهود است؛ بنابراین احتمال می‌دهم بیشتر این مکانیزم‌های سطح application درست کار نکنند و نتیجه، data گم‌شده یا خراب باشد.

به همین دلیل، ارزش دارد abstractionهای fault-toleranceای بررسی شوند که هم فراهم‌کردن end-to-end correctness propertyهای خاص application را آسان کنند و هم performance و ویژگی‌های عملیاتی مناسبی در محیط distributed بزرگ‌مقیاس داشته باشند.

### Enforcing Constraints

بیایید correctness را در چارچوب ایده‌های unbundling database بررسی کنیم. دیدیم حذف duplicate به‌صورت end-to-end با request IDای ممکن است که از client تا database ثبت‌کنندهٔ write منتقل می‌شود. constraintهای دیگر چطور؟

به‌طور خاص، uniqueness constraint را در نظر بگیریم؛ مانند constraintی که در Example 12-2 به آن تکیه کردیم. نمونه‌های دیگری از قابلیت‌های application که uniqueness می‌خواهند عبارت‌اند از: username یا email address باید user را به‌طور یکتا مشخص کند؛ file storage service نباید دو file هم‌نام داشته باشد؛ و دو نفر نباید بتوانند یک صندلی هواپیما یا سالن تئاتر را رزرو کنند.

constraintهای دیگر نیز مشابه‌اند: مثلاً تضمین اینکه balance حساب منفی نشود، بیشتر از موجودی انبار کالا نفروشیم، یا اتاق جلسه رزروهای هم‌پوشان نداشته باشد. روش‌های اعمال uniqueness را اغلب می‌توان برای این constraintها هم به کار برد.

#### uniqueness constraintها به consensus نیاز دارند

در Chapter 9 دیدیم که در محیط distributed، اعمال uniqueness constraint به consensus نیاز دارد: اگر چند request هم‌زمان مقدار یکسانی داشته باشند، system باید تصمیم بگیرد کدام operation متعارض پذیرفته شود و بقیه را به‌دلیل نقض constraint رد کند.

رایج‌ترین راه رسیدن به این consensus، انتخاب یک node به‌عنوان leader و سپردن همهٔ تصمیم‌ها به آن است. این روش تا وقتی خوب کار می‌کند که مشکلی با عبور همهٔ requestها از یک node نداشته باشید (حتی اگر client آن سوی دنیا باشد) و آن node از کار نیفتد. اگر باید خرابی leader را هم تحمل کنید، دوباره با مسئلهٔ consensus روبه‌رو می‌شوید.

می‌توان uniqueness checking را با partition کردن بر اساس مقداری که باید یکتا باشد scale کرد. مثلاً برای تضمین یکتایی request ID، می‌توان همهٔ requestهای دارای یک ID را به یک partition فرستاد. برای یکتا بودن username نیز می‌توان بر اساس hash username partition کرد.

اما asynchronous multi-master replication ممکن نیست، چون masterهای مختلف ممکن است هم‌زمان writeهای متعارض را بپذیرند و در نتیجه مقدارها دیگر یکتا نباشند. اگر می‌خواهید هر write ناقض constraint را فوراً رد کنید، هماهنگی synchronous اجتناب‌ناپذیر است.

#### uniqueness در log-based messaging

log تضمین می‌کند همهٔ consumerها messageها را با ترتیب یکسان ببینند؛ این guarantee به‌طور رسمی total order broadcast نام دارد و معادل consensus است. در رویکرد unbundled database با log-based messaging می‌توان از رویکردی مشابه برای اعمال uniqueness constraint استفاده کرد.

یک stream processor همهٔ messageهای یک log partition را به‌ترتیب و روی یک thread مصرف می‌کند. بنابراین اگر log بر اساس مقداری partition شده باشد که باید یکتا باشد، stream processor می‌تواند به‌شکلی قطعی و بدون ابهام مشخص کند کدام operation متعارض زودتر رسیده است. فرض کنید چند user می‌خواهند username یکسانی را claim کنند:

1. هر request برای username به message تبدیل و به partitionای append می‌شود که با hash username تعیین شده است.
2. یک stream processor، requestها را به‌ترتیب از log می‌خواند و با استفاده از database محلی، usernameهای گرفته‌شده را نگه می‌دارد. اگر username آزاد باشد، آن را گرفته‌شده ثبت و message موفقیت را در output stream منتشر می‌کند؛ اگر قبلاً گرفته شده باشد، message رد درخواست را می‌فرستد.
3. client درخواست‌کننده output stream را دنبال می‌کند و منتظر message موفقیت یا رد متناظر با request خودش می‌ماند.

این الگوریتم اساساً همان الگوریتم پیاده‌سازی storage خطی‌پذیر با total order broadcast است. با افزایش تعداد partitionها، throughput بالایی از requestها را به‌آسانی scale می‌کند، چون هر partition مستقل پردازش می‌شود.

این روش فقط برای uniqueness constraint نیست؛ بسیاری از constraintهای دیگر را هم می‌توان با آن اعمال کرد. اصل بنیادین این است که writeهایی که ممکن است با هم تعارض داشته باشند به یک partition فرستاده و به‌ترتیب پردازش شوند. تعریف conflict ممکن است به application وابسته باشد، اما stream processor می‌تواند برای اعتبارسنجی request از منطق دلخواه استفاده کند. این ایده به روشی شباهت دارد که Bayou در دههٔ ۱۹۹۰ پیشگام آن بود.

#### پردازش request چند-partitionای

اجرای اتمیک operation همراه با رعایت constraintها وقتی چند partition درگیرند جالب‌تر می‌شود. در Example 12-2 ممکن است سه partition داشته باشیم: یکی برای request ID، یکی برای حساب دریافت‌کننده و دیگری برای حساب پرداخت‌کننده. دلیلی ندارد این سه مورد در یک partition باشند؛ مستقل از یکدیگرند.

در رویکرد سنتی database، اجرای چنین transactionای به atomic commit در هر سه partition نیاز دارد؛ در نتیجه transaction باید در total order نسبت به همهٔ transactionهای دیگر در هر یک از partitionها قرار بگیرد. از آنجا که هماهنگی میان partitionها لازم می‌شود، دیگر نمی‌توان آن‌ها را مستقل پردازش کرد و احتمالاً throughput افت می‌کند.

بااین‌حال، می‌توان با partitioned logها و بدون atomic commit به correctness معادل رسید:

1. client به request انتقال پول از حساب A به حساب B، یک request ID یکتا می‌دهد و آن را به log partitionای append می‌کند که بر اساس request ID انتخاب شده است.
2. یک stream processor log requestها را می‌خواند. برای هر request، دو message به output streamها می‌فرستد: دستور debit برای حساب پرداخت‌کننده A (partitionشده بر اساس A) و دستور credit برای حساب دریافت‌کننده B (partitionشده بر اساس B). request ID اصلی نیز در هر دو message قرار می‌گیرد.
3. processorهای بعدی streamهای دستورهای credit و debit را مصرف می‌کنند، بر اساس request ID duplicateها را حذف می‌کنند و balance حساب‌ها را تغییر می‌دهند.

مرحله‌های ۱ و ۲ لازم‌اند، چون اگر client مستقیماً دستورهای credit و debit را بفرستد، برای تضمین اینکه هر دو انجام شوند یا هیچ‌کدام انجام نشوند، atomic commit میان دو partition لازم است. برای پرهیز از distributed transaction، ابتدا request را به‌صورت یک message واحد و پایدار در log ثبت می‌کنیم و سپس دستورهای credit و debit را از آن مشتق می‌سازیم. writeهای تک‌شیئی تقریباً در همهٔ data systemها atomic هستند؛ پس request یا در log ظاهر می‌شود یا نمی‌شود و نیازی به multi-partition atomic commit نیست.

اگر stream processor مرحلهٔ ۲ crash کند، از آخرین checkpoint پردازش را از سر می‌گیرد. هیچ requestای را جا نمی‌اندازد، اما ممکن است بعضی requestها را چند بار پردازش کند و دستورهای credit و debit تکراری بسازد. چون processor deterministic است، همان دستورها را دوباره تولید می‌کند و processorهای مرحلهٔ ۳ می‌توانند با request ID سرتاسری آن‌ها را به‌آسانی deduplicate کنند.

اگر می‌خواهید مطمئن شوید این انتقال باعث overdraft حساب پرداخت‌کننده نمی‌شود، می‌توانید stream processor دیگری داشته باشید که بر اساس شمارهٔ حساب پرداخت‌کننده partition شده، balanceها را نگه دارد و transactionها را اعتبارسنجی کند. در آن صورت فقط transactionهای معتبر وارد request log مرحلهٔ ۱ می‌شوند.

با شکستن transaction چند-partitionای به دو مرحله با partitioning متفاوت و استفاده از request ID سرتاسری، به همان خاصیت correctness می‌رسیم: هر request دقیقاً یک‌بار روی هر دو حساب پرداخت‌کننده و دریافت‌کننده اعمال می‌شود، حتی در حضور fault و بدون atomic commit protocol. ایدهٔ چند مرحله با partitioningهای متفاوت به multi-partition data processing و concurrency control نیز شباهت دارد.

### Timeliness and Integrity

ویژگی مناسبی که transactionها معمولاً دارند linearizability است: نویسنده تا commit شدن transaction منتظر می‌ماند و پس از آن، writeهایش فوراً برای همهٔ readerها قابل‌مشاهده است.

وقتی یک operation را میان چند مرحلهٔ stream processor تقسیم می‌کنیم، چنین تضمینی نداریم: consumerهای log ذاتاً asynchronous هستند و فرستنده منتظر نمی‌ماند تا consumerها message را پردازش کنند. بااین‌حال، client می‌تواند منتظر ظاهر شدن message در output stream بماند؛ همین کار را در مثال بررسی uniqueness با log-based messaging انجام دادیم.

در آن مثال، correctness بررسی uniqueness به منتظر ماندن یا نماندن فرستنده برای نتیجه وابسته نیست. انتظار فقط برای آن است که به‌صورت synchronous به فرستنده اطلاع دهیم بررسی uniqueness موفق بوده یا نه؛ این notification را می‌توان از effectهای پردازش message جدا کرد.

به‌طور کلی‌تر، به‌نظر من واژهٔ consistency دو نیاز متفاوت را با هم درمی‌آمیزد که بهتر است جداگانه بررسی شوند:

**Timeliness**

Timeliness یعنی userها system را در state به‌روز مشاهده کنند. اگر user از copyای stale از data بخواند، ممکن است stateای ناسازگار ببیند؛ اما این ناسازگاری موقتی است و معمولاً با صبر کردن و دوباره تلاش کردن برطرف می‌شود.

CAP theorem از consistency به معنای linearizability استفاده می‌کند که راهی قوی برای رسیدن به timeliness است. خاصیت‌های ضعیف‌تری مانند read-after-write consistency هم می‌توانند مفید باشند.

**Integrity**

Integrity یعنی نبود corruption؛ یعنی نه dataای از دست برود و نه data متناقض یا نادرستی وجود داشته باشد. اگر derived datasetای به‌عنوان viewای از data زیربنایی نگهداری می‌شود، derivation باید درست باشد. برای مثال، database index باید محتوای database را درست بازتاب دهد؛ indexای که بعضی recordها را جا انداخته باشد چندان مفید نیست.

اگر integrity نقض شود، ناسازگاری دائمی است: در بیشتر موارد صبر کردن و دوباره تلاش کردن database corruption را درست نمی‌کند. باید صریحاً آن را بررسی و repair کرد. در زمینهٔ ACID transactionها، consistency معمولاً نوعی تعریف application-specific از integrity است. Atomicity و durability ابزارهای مهم حفظ integrity هستند.

به‌شکل شعاری: نقض timeliness یعنی «eventual consistency»، اما نقض integrity یعنی «perpetual inconsistency».

ادعای من این است که در بیشتر applicationها integrity بسیار مهم‌تر از timeliness است. نقض timeliness ممکن است آزاردهنده و گیج‌کننده باشد، اما نقض integrity می‌تواند فاجعه‌بار باشد.

برای مثال، عجیب نیست اگر transactionای که طی ۲۴ ساعت گذشته با کارت اعتباری انجام داده‌اید هنوز در صورتحساب نیامده باشد؛ چنین systemهایی معمولاً مقداری lag دارند. بانک‌ها transactionها را asynchronous تطبیق و تسویه می‌کنند و timeliness در اینجا چندان مهم نیست. اما اگر balance صورتحساب با جمع transactionها به‌علاوهٔ balance صورتحساب قبلی برابر نباشد (خطای محاسبه)، یا از شما بابت transactionای پول گرفته شود ولی پول به فروشنده نرسد (ناپدید شدن پول)، مشکل جدی است؛ این‌ها نقض integrity system هستند.

#### correctness در dataflow systemها

ACID transactionها معمولاً هم timeliness (مثلاً linearizability) و هم integrity (مثلاً atomic commit) را تضمین می‌کنند. بنابراین اگر correctness application را از دید ACID بررسی کنید، تفاوت timeliness و integrity چندان به چشم نمی‌آید.

اما یکی از ویژگی‌های جالب event-based dataflow systemهایی که در این chapter بررسی کردیم، جدا کردن این دو است. در پردازش asynchronous event stream، timeliness تضمین نمی‌شود؛ مگر اینکه consumerهایی بسازید که پیش از پاسخ دادن منتظر رسیدن message بمانند. در عوض، integrity برای streaming systemها محوری است.

Exactly-once یا effectively-once semantics مکانیزمی برای حفظ integrity است. اگر event گم شود یا اثر event دوبار اعمال شود، integrity data system ممکن است نقض شود. بنابراین تحویل fault-tolerant message و حذف duplicate (مثلاً با operationهای idempotent) برای حفظ integrity system در حضور fault مهم‌اند.

همان‌طور که در بخش قبل دیدیم، stream processing systemهای قابل‌اعتماد می‌توانند integrity را بدون distributed transaction و atomic commit protocol حفظ کنند؛ بنابراین ممکن است correctnessای قابل‌مقایسه را با performance و operational robustness بسیار بهتر فراهم کنند. این integrity را با ترکیبی از سازوکارها به دست آوردیم:

- محتوای write را در قالب یک message واحد نمایش می‌دهیم که به‌آسانی atomic نوشته می‌شود؛ رویکردی سازگار با event sourcing.
- همهٔ updateهای دیگر state را با تابع‌های مشتق‌سازی deterministic از همان message می‌سازیم؛ مشابه stored procedure.
- request ID تولیدشده در client را در همهٔ مراحل پردازش منتقل می‌کنیم تا حذف duplicate و idempotence به‌صورت end-to-end ممکن شود.
- messageها را immutable نگه می‌داریم و هر از گاهی derived data را دوباره پردازش می‌کنیم تا recovery از bug آسان‌تر شود.

به‌نظر من این ترکیب، مسیری بسیار امیدبخش برای ساخت applicationهای fault-tolerant در آینده است.


#### constraintهایی که می‌توانند موقتاً نقض شوند

همان‌طور که گفتیم، اعمال uniqueness constraint به consensus نیاز دارد که معمولاً با عبور دادن همهٔ eventهای یک partition از یک node انجام می‌شود. اگر شکل سنتی uniqueness را بخواهیم، این محدودیت اجتناب‌ناپذیر است و stream processing هم نمی‌تواند حذفش کند.

اما بسیاری از applicationهای واقعی با تعریف بسیار سهل‌گیرانه‌تری از uniqueness هم کنار می‌آیند:

- اگر دو نفر هم‌زمان username یکسانی ثبت کنند یا یک صندلی را رزرو کنند، می‌توان از یکی عذرخواهی کرد و خواست انتخاب دیگری داشته باشد. چنین تغییری برای اصلاح خطا compensating transaction نام دارد.
- اگر مشتریان بیش از موجودی انبار سفارش دهند، می‌توان کالای بیشتری سفارش داد، بابت تأخیر عذرخواهی کرد و تخفیف داد. همین کاری است که اگر مثلاً forklift بخشی از کالاها را خراب کند و موجودی واقعی کمتر از مقدار ثبت‌شده شود، باید انجام دهید. پس workflow عذرخواهی از قبل بخشی از فرایند کسب‌وکار است و شاید لازم نباشد constraint تعداد موجودی linearizable باشد.
- بسیاری از airlineها بیشتر از ظرفیت هواپیما بلیت می‌فروشند، چون انتظار دارند بعضی مسافران پرواز را از دست بدهند؛ بسیاری از hotelها نیز بیش از اتاق‌هایشان رزرو می‌گیرند، چون بعضی مهمان‌ها لغو می‌کنند. در این موارد، constraint «یک نفر برای هر صندلی» عمداً و به دلایل تجاری نقض می‌شود و برای تقاضای بیش از عرضه، فرایند جبران (بازپرداخت، ارتقای کلاس یا فراهم کردن اتاق رایگان در hotel مجاور) وجود دارد. حتی بدون overbooking هم برای لغو پرواز به‌علت آب‌وهوای بد یا اعتصاب کارکنان به عذرخواهی و جبران نیاز است؛ بازیابی از چنین اتفاق‌هایی بخشی عادی از کسب‌وکار است.
- اگر کسی بیش از موجودی حسابش پول برداشت کند، بانک می‌تواند جریمهٔ overdraft بگیرد و بازپرداخت بدهی را مطالبه کند. با محدود کردن مجموع برداشت روزانه، ریسک بانک محدود می‌شود.

در بسیاری از زمینه‌های کسب‌وکار، پذیرفتنی است که constraint موقتاً نقض شود و بعد با عذرخواهی اصلاحش کنیم. هزینهٔ عذرخواهی از نظر مالی یا اعتباری فرق می‌کند، اما اغلب کم است: نمی‌توان email را پس گرفت، ولی می‌توان email اصلاحی فرستاد. اگر تصادفاً از کارت اعتباری دوبار پول بگیرید، می‌توانید یکی را refund کنید؛ هزینه‌اش کارمزد پردازش و شاید شکایت مشتری است. پولی را که از ATM پرداخت شده مستقیماً نمی‌توان پس گرفت؛ هرچند اگر حساب overdraft شده باشد و مشتری بدهی را نپردازد، در اصل می‌توان از وصول‌کنندهٔ بدهی کمک گرفت.

پذیرفتنی بودن هزینهٔ عذرخواهی تصمیمی تجاری است. اگر پذیرفتنی باشد، مدل سنتیِ بررسی همهٔ constraintها پیش از نوشتن data بیش از حد محدودکننده است و نیازی به constraint خطی‌پذیر نیست. شاید منطقی باشد write را با خوش‌بینی انجام دهیم و constraint را پس از آن بررسی کنیم. می‌توان validation را پیش از اقدام‌هایی انجام داد که بازیابی از آن‌ها پرهزینه است؛ اما این به این معنا نیست که باید data را پیش از هر writeای اعتبارسنجی کرد.

این applicationها همچنان به integrity نیاز دارند: نباید reservation گم شود یا به‌علت ناهماهنگی debit و credit پول ناپدید شود. اما برای اعمال constraint به timeliness نیاز ندارند: اگر بیش از موجودی انبار کالا فروخته‌اید، می‌توانید بعداً با عذرخواهی مشکل را جبران کنید. این شبیه روش‌های conflict resolution در بحث Handling Write Conflicts است.

#### data systemهای پرهیزکننده از coordination

تا اینجا به دو مشاهدهٔ مهم رسیدیم:

1. Dataflow systemها می‌توانند integrity guaranteeهای derived data را بدون atomic commit، linearizability یا هماهنگی synchronous میان partitionها حفظ کنند.
2. هرچند uniqueness constraint سخت‌گیرانه به timeliness و coordination نیاز دارد، بسیاری از applicationها constraintهای سهل‌گیرانه‌تری را می‌پذیرند که شاید موقتاً نقض و بعد اصلاح شوند؛ به‌شرط آنکه integrity در تمام مدت حفظ شود.

این دو مشاهده در کنار هم یعنی dataflow systemها می‌توانند برای بسیاری از applicationها data management فراهم کنند، بی‌آنکه coordination لازم باشد و درعین‌حال integrity قوی بدهند. این data systemهای coordination-avoiding جذاب‌اند: نسبت به systemهایی که به هماهنگی synchronous نیاز دارند، می‌توانند performance و fault tolerance بهتری داشته باشند.

برای مثال، چنین systemی می‌تواند روی چند datacenter و با پیکربندی multi-leader اجرا شود و regionها را به‌شکل asynchronous replicate کند. هر datacenter می‌تواند مستقل از بقیه به کار ادامه دهد، چون هماهنگی synchronous میان regionها لازم نیست. چنین systemی timeliness ضعیفی دارد و بدون coordination نمی‌تواند linearizable باشد، اما همچنان می‌تواند integrity قوی داشته باشد.

در این معماری، serializable transactionها برای نگهداری derived state همچنان مفیدند، اما می‌توان آن‌ها را در scope کوچکی به کار برد که خوب عمل می‌کنند. distributed transactionهای ناهمگون مانند XA لازم نیستند. هر جا نیاز باشد، می‌توان coordination synchronous را اضافه کرد (مثلاً برای اعمال constraint سخت‌گیرانه پیش از عملیاتی که بازیابی از آن ممکن نیست)؛ اما اگر فقط بخش کوچکی از application به آن نیاز دارد، لازم نیست همهٔ اجزای system هزینهٔ coordination را بپردازند.

از زاویه‌ای دیگر، coordination و constraintها تعداد عذرخواهی‌هایی را که به‌علت ناسازگاری لازم است کم می‌کنند، اما ممکن است performance و availability system را هم کاهش دهند و در نتیجه تعداد عذرخواهی‌ها به‌علت outage را افزایش دهند. نمی‌توان تعداد عذرخواهی‌ها را به صفر رساند؛ هدف باید یافتن trade-off مناسب نیاز شما باشد: نقطه‌ای که نه ناسازگاری زیادی وجود دارد و نه مشکل availability بیش از حد.

### Trust, but Verify

تمام بحث ما دربارهٔ correctness، integrity و fault tolerance بر این فرض بنا شده که بعضی چیزها ممکن است خراب شوند و بعضی دیگر نه. این فرض‌ها را system model می‌نامیم. برای مثال، باید فرض کنیم processها crash می‌کنند، machineها ناگهان برقشان را از دست می‌دهند و network ممکن است messageها را دلخواهانه با تأخیر برساند یا حذف کند. در مقابل، شاید فرض کنیم data نوشته‌شده روی disk پس از `fsync` از بین نمی‌رود، data در memory خراب نمی‌شود و دستور ضرب CPU همیشه نتیجهٔ درست می‌دهد.

این فرض‌ها معقول‌اند، چون بیشتر اوقات درست هستند و اگر دائماً نگران اشتباه رایانه‌ها باشیم، انجام هر کاری دشوار می‌شود. system modelهای سنتی دربارهٔ faultها رویکردی دودویی دارند: فرض می‌کنیم بعضی اتفاق‌ها ممکن‌اند و بعضی هرگز رخ نمی‌دهند. در واقع، مسئله بیشتر احتمالاتی است: بعضی اتفاق‌ها محتمل‌تر و بعضی کم‌احتمال‌ترند. پرسش این است که آیا نقض فرض‌ها آن‌قدر رخ می‌دهد که در عمل با آن روبه‌رو شویم یا نه.

دیدیم data ممکن است وقتی بی‌استفاده روی disk قرار دارد خراب شود و corruption شبکه نیز گاهی از checksumهای TCP عبور کند. شاید لازم باشد بیشتر به این مسئله توجه کنیم.

یکی از applicationهایی که قبلاً روی آن کار می‌کردم، crash reportهای clientها را جمع‌آوری می‌کرد. بعضی reportها را فقط می‌شد با bit-flip تصادفی در memory آن deviceها توضیح داد. بعید به‌نظر می‌رسد، اما وقتی تعداد زیادی device نرم‌افزار شما را اجرا می‌کنند، اتفاق‌های بسیار نامحتمل هم رخ می‌دهند. علاوه بر خرابی تصادفی memory بر اثر نقص سخت‌افزار یا radiation، بعضی الگوهای آسیب‌زای دسترسی به memory می‌توانند حتی در memory سالم bitها را تغییر دهند؛ اثری که برای شکستن مکانیزم‌های امنیتی operating system هم به کار می‌رود و rowhammer نام دارد. وقتی دقیق‌تر نگاه کنیم، hardware آن abstraction بی‌نقصی نیست که به‌نظر می‌رسد.

برای روشن بودن موضوع، bit-flip تصادفی در سخت‌افزار مدرن همچنان بسیار نادر است. فقط می‌خواهم بگویم ناممکن نیست و سزاوار توجه است.

#### حفظ integrity در برابر bugهای نرم‌افزاری

جدا از مشکل‌های سخت‌افزاری، همیشه خطر bug نرم‌افزاری وجود دارد؛ bugهایی که checksumهای لایهٔ پایین‌تر در network، memory یا filesystem آن‌ها را پیدا نمی‌کنند. حتی نرم‌افزارهای database پرکاربرد هم bug دارند: شخصاً مواردی دیده‌ام که MySQL نتوانسته uniqueness constraint را درست حفظ کند و isolation level سریال‌پذیر PostgreSQL دچار write skew شده است؛ با وجود آنکه MySQL و PostgreSQL databaseهایی معتبر و مقاوم‌اند که سال‌ها توسط افراد بسیاری در شرایط واقعی آزموده شده‌اند. وضعیت نرم‌افزارهای کم‌تجربه‌تر احتمالاً بسیار بدتر است.

با وجود تلاش چشمگیر برای طراحی دقیق، testing و review، bugها باز هم وارد software می‌شوند. هرچند نادرند و بالاخره پیدا و رفع می‌شوند، در فاصلهٔ زمانی تا رفع‌شان ممکن است data را خراب کنند.

در application code باید bugهای بسیار بیشتری را انتظار داشته باشیم، چون بیشتر applicationها به‌اندازهٔ code database بررسی و آزمایش نمی‌شوند. بسیاری از applicationها حتی قابلیت‌های database برای حفظ integrity، مانند foreign key یا uniqueness constraint، را درست به‌کار نمی‌برند.

Consistency در معنای ACID بر این ایده بنا شده که database از stateای سازگار شروع می‌کند و transaction آن را به state سازگار دیگری می‌برد؛ پس انتظار داریم database همیشه سازگار بماند. اما این تصور فقط وقتی معنا دارد که transaction را عاری از bug فرض کنیم. اگر application به‌شکلی نادرست از database استفاده کند ــ مثلاً weak isolation level را ناایمن به کار ببرد ــ integrity database تضمین‌شدنی نیست.

#### وعده‌ها را کورکورانه باور نکنید

از آنجا که hardware و software همیشه مطابق ideal مورد انتظار ما عمل نمی‌کنند، به‌نظر می‌رسد data corruption دیر یا زود اجتناب‌ناپذیر باشد. پس دست‌کم باید راهی برای فهمیدن وقوع corruption داشته باشیم تا آن را repair و منشأ خطا را پیدا کنیم. بررسی integrity data را auditing می‌نامند.

همان‌طور که در بحث مزایای eventهای immutable گفتیم، auditing فقط برای applicationهای مالی نیست. بااین‌حال، در finance اهمیت زیادی دارد، دقیقاً چون همه می‌دانند اشتباه رخ می‌دهد و نیاز به تشخیص و اصلاح آن را می‌پذیرند.

systemهای بالغ نیز احتمال وقوع اتفاق‌های نادر را در نظر می‌گیرند و ریسک آن را مدیریت می‌کنند. برای مثال، storage systemهای بزرگ‌مقیاسی مانند HDFS و Amazon S3 کاملاً به disk اعتماد نمی‌کنند: processهای پس‌زمینه‌ای دارند که پیوسته fileها را می‌خوانند، با replicaهای دیگر مقایسه می‌کنند و برای کاهش خطر corruption خاموش، fileها را از یک disk به disk دیگر منتقل می‌کنند.

اگر می‌خواهید مطمئن شوید data هنوز وجود دارد، باید واقعاً آن را بخوانید و بررسی کنید. بیشتر وقت‌ها data سر جایش است؛ اما اگر نباشد، بهتر است زودتر باخبر شوید. به همین دلیل باید هر از گاهی بازیابی از backupها را هم امتحان کرد؛ وگرنه شاید وقتی backup خراب است فقط زمانی بفهمید که دیگر دیر شده و data را از دست داده‌اید. صرفاً کورکورانه اعتماد نکنید که همه‌چیز کار می‌کند.

#### فرهنگ verification

systemهایی مانند HDFS و S3 همچنان فرض می‌کنند disk بیشتر اوقات درست کار می‌کند؛ فرضی معقول، اما متفاوت از این باور که disk همیشه درست کار می‌کند. بااین‌حال، systemهای زیادی رویکرد «اعتماد کن، اما verify کن» ندارند و خود را پیوسته audit نمی‌کنند. بسیاری فرض می‌کنند correctness guaranteeها مطلق‌اند و برای احتمال نادر data corruption تدبیری ندارند. امیدوارم در آینده systemهای self-validating یا self-auditing بیشتری ببینیم که integrity خود را پیوسته بررسی کنند، نه اینکه به اعتماد کور متکی باشند.

نگرانم فرهنگ databaseهای ACID ما را به ساخت applicationهایی سوق داده باشد که کورکورانه به فناوری (مثلاً transaction mechanism) اعتماد می‌کنند و auditability را نادیده می‌گیرند. چون فناوری مورداعتمادمان بیشتر اوقات به‌اندازهٔ کافی خوب کار می‌کرد، سازوکارهای auditing ارزش سرمایه‌گذاری تلقی نمی‌شدند.

اما بعد landscape database تغییر کرد: consistency guaranteeهای ضعیف‌تر با نام NoSQL رایج شدند و storage technologyهای کم‌تجربه‌تر کاربرد گسترده پیدا کردند. بااین‌حال، چون سازوکارهای audit ساخته نشده بودند، همچنان applicationها را بر پایهٔ اعتماد کور می‌ساختیم؛ حال آنکه این رویکرد خطرناک‌تر شده بود. بیایید به طراحی برای auditability فکر کنیم.

#### طراحی برای auditability

اگر transactionای چند object را در database تغییر دهد، بعداً فهمیدن معنای آن دشوار است. حتی اگر transaction logها را capture کنید، insert، update و delete در tableهای مختلف لزوماً روشن نمی‌کنند چرا آن تغییرها انجام شده‌اند. فراخوانی application logicای که دربارهٔ آن تغییرها تصمیم گرفته، گذراست و قابل‌بازسازی نیست.

در مقابل، event-based systemها auditability بهتری فراهم می‌کنند. در رویکرد event sourcing، input کاربر به system به‌شکل یک event immutable واحد نمایش داده می‌شود و updateهای state حاصل از آن event مشتق می‌شوند. می‌توان derivation را deterministic و تکرارپذیر کرد؛ در نتیجه اجرای همان log از eventها با همان نسخهٔ derivation code، همان updateهای state را تولید می‌کند.

شفاف کردن dataflow، data provenance را روشن‌تر می‌کند و integrity checking را عملی‌تر می‌سازد. برای event log می‌توان از hash استفاده کرد تا بفهمیم storage مربوط به event خراب نشده است. برای هر derived state نیز می‌توان batch و stream processorهایی را که آن را از event log ساخته‌اند دوباره اجرا کرد و دید آیا همان نتیجه حاصل می‌شود؛ حتی می‌توان derivation موازی و redundant اجرا کرد.

dataflowای که deterministic و دقیق تعریف شده باشد، debug و trace کردن اجرای system را نیز آسان می‌کند تا بفهمیم چرا کاری انجام شده است. اگر اتفاق غیرمنتظره‌ای رخ دهد، داشتن قابلیت تشخیصی برای بازسازی دقیق شرایط منتهی به آن ارزشمند است؛ نوعی time-travel debugging.

#### بازگشت به استدلال end-to-end

اگر نتوانیم کاملاً به تک‌تک componentهای system اعتماد کنیم ــ اینکه همهٔ سخت‌افزارها بی‌نقص و همهٔ نرم‌افزارها بدون bug باشند ــ باید دست‌کم integrity data را دوره‌ای بررسی کنیم. اگر بررسی نکنیم، تا زمانی که corruption خسارتی پایین‌دستی ایجاد کند از آن باخبر نمی‌شویم؛ آن وقت پیدا کردن مشکل بسیار دشوارتر و پرهزینه‌تر خواهد بود.

بررسی integrity data systemها بهتر است end-to-end انجام شود: هرچه systemهای بیشتری را در integrity check بگنجانیم، احتمال کمتری وجود دارد که corruption در مرحله‌ای از فرایند پنهان بماند. اگر بتوانیم درستی کل derived data pipeline را از ابتدا تا انتها بررسی کنیم، همهٔ diskها، networkها، serviceها و algorithmهای مسیر نیز عملاً در بررسی پوشش داده می‌شوند.

integrity checkهای end-to-end و پیوسته اعتماد ما به correctness system را افزایش می‌دهند و به ما امکان می‌دهند سریع‌تر حرکت کنیم. auditing هم مانند automated testing احتمال پیدا شدن سریع bug را بیشتر و خطر آسیب‌دیدن system بر اثر تغییر یا storage technology جدید را کمتر می‌کند. اگر از تغییر دادن system نترسیم، بهتر می‌توانیم application را با نیازهای در حال تغییر تکامل دهیم.

#### ابزارهای data systemهای قابل audit

در حال حاضر، systemهای زیادی auditability را دغدغه‌ای درجه‌یک نمی‌دانند. بعضی applicationها سازوکار audit خودشان را دارند؛ مثلاً همهٔ changeها را در audit table جداگانه ثبت می‌کنند، اما تضمین integrity هم‌زمان audit log و state database همچنان دشوار است. می‌توان transaction log را با امضای دوره‌ای توسط hardware security module دست‌کاری‌ناپذیر کرد، اما این تضمین نمی‌کند که از همان ابتدا transactionهای درست وارد log شده باشند.

جالب است از ابزارهای cryptographic برای اثبات integrity یک system استفاده کنیم؛ به‌شکلی که در برابر دامنهٔ وسیعی از مشکل‌های سخت‌افزاری و نرم‌افزاری و حتی اقدام‌های مخرب مقاوم باشد. ارزهای رمزنگاری‌شده، blockchain و فناوری‌های distributed ledger مانند Bitcoin، Ethereum، Ripple و Stellar برای کاوش این حوزه پدید آمده‌اند.

من صلاحیت اظهارنظر دربارهٔ شایستگی این فناوری‌ها به‌عنوان ارز یا سازوکار توافق بر قرارداد را ندارم؛ اما از دید data systemها، ایده‌های جالبی دارند. در اصل، این‌ها distributed databaseهایی با data model و transaction mechanism هستند که replicaهایشان را سازمان‌هایی میزبانی می‌کنند که به یکدیگر اعتماد ندارند. replicaها پیوسته integrity یکدیگر را بررسی می‌کنند و با protocol consensus بر سر transactionهایی که باید اجرا شوند به توافق می‌رسند.

دربارهٔ جنبه‌های Byzantine fault tolerance این فناوری‌ها تا حدی تردید دارم و روش proof of work (مثلاً Bitcoin mining) را به‌شدت اتلاف‌گر می‌دانم. throughput تراکنش Bitcoin نسبتاً پایین است؛ هرچند علت آن بیشتر سیاسی و اقتصادی است تا فنی. بااین‌حال، قابلیت‌های integrity checking آن‌ها جالب‌اند.

Cryptographic auditing و integrity checking اغلب به Merkle tree متکی‌اند: درختی از hashها که می‌تواند به‌طور کارآمد ثابت کند recordای در dataset مشخصی وجود دارد و چند خاصیت دیگر را نیز بررسی کند. بیرون از هیاهوی cryptocurrencyها، certificate transparency یک فناوری امنیتی مبتنی بر Merkle tree است که برای بررسی اعتبار TLS/SSL certificateها به‌کار می‌رود.

می‌توان تصور کرد algorithmهای integrity checking و auditing مانند آنچه در certificate transparency و distributed ledgerها وجود دارد، در data systemهای عمومی‌تر نیز رایج شوند. برای مقیاس‌پذیر کردن آن‌ها تا حد systemهای فاقد cryptographic auditing و پایین نگه‌داشتن هزینهٔ performance هنوز کار لازم است؛ اما به‌نظر من حوزهٔ جالبی برای دنبال کردن در آینده است.

## Key Terms

- `End-to-End Argument` — اصلی که می‌گوید بعضی guaranteeها فقط با مشارکت application در دو سر مسیر ارتباطی به‌طور کامل قابل‌تحقق‌اند.
- `Idempotent Operation` — operationای که اجرای تکراری‌اش نتیجه‌ای متفاوت از اجرای یک‌باره ایجاد نمی‌کند.
- `Operation ID` — شناسهٔ یکتایی که در سراسر مسیر request منتقل می‌شود تا اجرای تکراری قابل‌شناسایی و حذف باشد.
- `Compensating Transaction` — operation اصلاحی‌ای که اثر یک operation پیشین را از نظر کسب‌وکار جبران می‌کند، وقتی rollback مستقیم ممکن یا مناسب نیست.
- `Timeliness` — خاصیت به‌روز بودن مشاهدهٔ user از system؛ با تأخیر یا stale read نقض می‌شود، اما معمولاً موقتی است.
- `Integrity` — درستی و کامل‌بودن data؛ نبود data loss، corruption یا مقدارهای متناقض و نادرست.
- `Coordination-Avoiding Data System` — data systemای که با تکیه بر log، partitioning و پردازش asynchronous نیاز به هماهنگی synchronous را کاهش می‌دهد، اما integrity را حفظ می‌کند.
- `Auditing` — بررسی مستقل و دوره‌ای integrity داده و یافتن فساد یا ناسازگاری احتمالی.
- `Merkle Tree` — ساختار درختی از hashها که بررسی عضویت و integrity recordها در dataset را کارآمد می‌کند.
- `Deterministic Dataflow` — dataflowای که اجرای دوبارهٔ همان inputها با همان منطق، همان derived state را تولید می‌کند و audit و recovery را ممکن می‌سازد.


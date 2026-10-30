# Chapter 8 — The Trouble with Distributed Systems

## Knowledge, Truth, and Lies

تا اینجا در این chapter بررسی کردیم که distributed systemها چه تفاوتی با programهای روی یک کامپیوتر دارند: memory مشترکی وجود ندارد و ارتباط فقط از طریق message passing روی networkی غیرقابل‌اعتماد و با delayهای متغیر انجام می‌شود؛ systemها نیز ممکن است با partial failure، clockهای غیرقابل‌اعتماد و pause در پردازش روبه‌رو شوند.

اگر با distributed systemها آشنا نباشید، پیامدهای این مسئله‌ها بسیار گیج‌کننده‌اند. یک node در network هیچ‌چیز را با قطعیت نمی‌داند؛ فقط بر اساس messageهایی که از طریق network دریافت می‌کند یا نمی‌کند، حدس می‌زند. یک node تنها با exchange کردن message می‌تواند از state node دیگری باخبر شود؛ مثلاً بداند چه dataای ذخیره کرده یا درست کار می‌کند یا نه. اگر remote node پاسخ ندهد، راهی برای فهمیدن state آن وجود ندارد، چون نمی‌توان مشکل network را به‌طور قابل‌اعتماد از مشکل خود node تشخیص داد.

بحث دربارهٔ این systemها گاهی به فلسفه نزدیک می‌شود: در system خود چه چیزی را درست یا نادرست می‌دانیم؟ وقتی ابزارهای مشاهده و اندازه‌گیری قابل‌اعتماد نیستند، تا چه حد می‌توانیم به دانسته‌هایمان مطمئن باشیم؟ آیا systemهای نرم‌افزاری باید از قوانینی که در جهان فیزیکی انتظار داریم، مانند علت و معلول، پیروی کنند؟

خوشبختانه لازم نیست تا کشف معنای زندگی پیش برویم. در یک distributed system می‌توانیم فرض‌هایی را که دربارهٔ رفتار system داریم—یعنی **system model**—بیان کنیم و system را طوری طراحی کنیم که با آن فرض‌ها سازگار باشد. می‌توان ثابت کرد که algorithmها در چارچوب یک system model مشخص درست کار می‌کنند. بنابراین حتی اگر system model زیرین guaranteeهای کمی بدهد، دستیابی به رفتار قابل‌اعتماد ممکن است.

البته خوش‌رفتار کردن software در system model غیرقابل‌اعتماد ساده نیست. در ادامهٔ این chapter، مفهوم knowledge و truth را در distributed systemها بیشتر بررسی می‌کنیم تا بهتر بتوانیم دربارهٔ فرض‌های قابل‌قبول و guaranteeهایی که می‌خواهیم ارائه دهیم فکر کنیم. در Chapter 9 نیز نمونه‌هایی از distributed systemها و algorithmهایی را می‌بینیم که تحت فرض‌های مشخص، guaranteeهای خاصی ارائه می‌کنند.

### The Truth Is Defined by the Majority

یک network با **asymmetric fault** را تصور کنید: node می‌تواند تمام messageهای ارسالی به خودش را دریافت کند، اما messageهای خروجی آن drop یا با تأخیر مواجه می‌شوند [19]. با اینکه node کاملاً درست کار می‌کند و requestهای nodeهای دیگر را دریافت می‌کند، آن nodeها responseهایش را نمی‌شنوند. پس از گذشت timeout، چون از node خبری نشنیده‌اند، آن را dead اعلام می‌کنند. ماجرا مثل کابوسی است که node نیمه‌جداشده را، درحالی‌که فریاد می‌زند «من نمرده‌ام!»، به‌سمت قبرستان می‌کشند؛ اما چون کسی صدایش را نمی‌شنود، مراسم خاکسپاری با آرامش ادامه پیدا می‌کند.

در سناریویی کمی کم‌کابوس‌تر، node نیمه‌جداشده متوجه می‌شود messageهای ارسالی‌اش از nodeهای دیگر acknowledgement نمی‌گیرند و درمی‌یابد که احتمالاً network faultی وجود دارد. بااین‌حال، nodeهای دیگر اشتباهاً آن را dead اعلام کرده‌اند و node نیمه‌جداشده کاری از دستش برنمی‌آید.

در سناریوی سوم، nodeای را تصور کنید که دچار stop-the-world garbage collection pause طولانی می‌شود. GC تمام threadهای node را برای یک دقیقه متوقف می‌کند؛ در نتیجه هیچ requestای پردازش نمی‌شود و هیچ responseای ارسال نمی‌شود. nodeهای دیگر منتظر می‌مانند، retry می‌کنند، بی‌حوصله می‌شوند و سرانجام node را dead اعلام می‌کنند و به‌سمت نعش‌کش می‌برند. بالاخره GC تمام می‌شود و threadهای node چنان ادامه می‌دهند که انگار هیچ اتفاقی نیفتاده است. nodeهای دیگر با تعجب می‌بینند nodeای که مرده فرض می‌شد، سالم از تابوت بیرون می‌آید و با خوشحالی با اطرافیان حرف می‌زند. ابتدا nodeای که GC روی آن اجرا شده حتی نمی‌فهمد یک دقیقه گذشته و dead اعلام شده است؛ از دید خودش، از آخرین گفت‌وگو با nodeهای دیگر تقریباً زمانی نگذشته است.

درس این ماجراها آن است که یک node نمی‌تواند همیشه به قضاوت خودش دربارهٔ وضعیت اعتماد کند. distributed system هم نمی‌تواند فقط به یک node متکی باشد، چون آن node هر لحظه ممکن است fail شود و system را در وضعیتی گیر بیندازد که نتواند recover کند. در عوض، بسیاری از distributed algorithmها به **quorum**، یعنی رأی‌گیری میان nodeها، تکیه می‌کنند؛ به بخش «Quorums for reading and writing» در صفحهٔ ۱۷۹ مراجعه کنید. برای کاهش وابستگی به یک node خاص، تصمیم‌ها به حداقل تعداد مشخصی رأی از چند node نیاز دارند.

این قاعده تصمیم‌گیری دربارهٔ dead اعلام کردن nodeها را هم شامل می‌شود. اگر quorumای از nodeها، node دیگری را dead اعلام کند، باید آن را dead در نظر گرفت؛ حتی اگر خود node هنوز کاملاً زنده احساس کند. آن node باید از تصمیم quorum پیروی کند و از نقش خود کنار برود.

رایج‌ترین quorum، اکثریت مطلق—بیش از نیمی از nodeها—است، هرچند انواع دیگری از quorum هم وجود دارد. Majority quorum اجازه می‌دهد system با وجود failure بعضی nodeها به کار ادامه دهد: از سه node می‌توان failure یک node و از پنج node می‌توان failure دو node را تحمل کرد. این روش همچنان safe است، چون در system نمی‌توان هم‌زمان دو اکثریت با تصمیم‌های متناقض داشت. هنگام بررسی consensus algorithmها در Chapter 9، quorumها را با جزئیات بیشتری می‌بینیم.

#### The leader and the lock

اغلب لازم است در system از هر چیز مشخصی فقط یک نمونه وجود داشته باشد. برای مثال:

- برای جلوگیری از split brain، فقط یک node مجاز است leader یک database partition باشد؛ به بخش «Handling Node Outages» در صفحهٔ ۱۵۶ مراجعه کنید.
- برای جلوگیری از write هم‌زمان و خراب شدن یک resource یا object، فقط یک transaction یا client مجاز است lock آن را در اختیار داشته باشد.
- چون username باید user را به‌طور یکتا مشخص کند، فقط یک user مجاز است username مشخصی را ثبت کند.

پیاده‌سازی چنین قاعده‌ای در distributed system به دقت نیاز دارد. اینکه nodeای باور داشته باشد «فرد برگزیده» است—leader یک partition، دارندهٔ lock یا request handler کاربری که username را گرفته—الزاماً به این معنا نیست که quorum هم با آن موافق است. ممکن است node قبلاً leader بوده باشد، اما nodeهای دیگر در این فاصله آن را dead اعلام کرده باشند؛ مثلاً به‌دلیل قطع network یا GC pause. در این صورت ممکن است node از leadership کنار گذاشته شده و leader دیگری انتخاب شده باشد.

اگر node همچنان نقش «فرد برگزیده» را ادامه دهد، درحالی‌که اکثریت آن را dead اعلام کرده‌اند، ممکن است در systemی که با دقت طراحی نشده مشکل ایجاد کند. چنین nodeای می‌تواند به‌نام خودش برای nodeهای دیگر message بفرستد و اگر آن‌ها حرفش را باور کنند، کل system تصمیم نادرستی بگیرد.

برای نمونه، Figure 8-4 یک bug مربوط به data corruption را نشان می‌دهد که از پیاده‌سازی نادرست locking ناشی شده است. این bug صرفاً نظری نیست؛ HBase قبلاً چنین مشکلی داشت [74, 75]. فرض کنید می‌خواهید تضمین کنید در storage service، هر لحظه فقط یک client بتواند به یک file دسترسی داشته باشد، چون write هم‌زمان چند client می‌تواند file را خراب کند. راه‌حل پیشنهادی این است که client پیش از دسترسی به file از lock service یک lease بگیرد.

**Figure 8-4.** پیاده‌سازی نادرست distributed lock: client 1 تصور می‌کند lease آن هنوز معتبر است، درحالی‌که lease منقضی شده؛ در نتیجه file ذخیره‌شده را خراب می‌کند.

این مشکل نمونه‌ای از همان مسئله‌ای است که در «Process Pauses» در صفحهٔ ۲۹۵ دیدیم: اگر client دارندهٔ lease برای مدت زیادی pause شود، lease آن منقضی می‌شود. client دیگری می‌تواند برای همان file lease بگیرد و نوشتن را آغاز کند. وقتی client متوقف‌شده برمی‌گردد، اشتباهاً تصور می‌کند lease هنوز معتبر است و آن هم شروع به نوشتن می‌کند. writeهای دو client با هم تداخل می‌کنند و file خراب می‌شود.

#### Fencing tokens

برای محافظت از resource—مانند file storage در Figure 8-4—با lock یا lease، باید مانع شویم nodeای که به‌اشتباه خود را «فرد برگزیده» می‌داند، به بقیهٔ system آسیب بزند. تکنیک نسبتاً ساده‌ای به نام **fencing** این هدف را تأمین می‌کند و در Figure 8-5 نشان داده شده است.

**Figure 8-5.** با پذیرفتن writeها فقط به‌ترتیب صعودی fencing tokenها، دسترسی به storage ایمن می‌شود.

فرض کنید هر بار lock server یک lock یا lease صادر می‌کند، یک **fencing token** هم برمی‌گرداند؛ عددی که با هر بار صدور lock افزایش می‌یابد، مثلاً lock service آن را increment می‌کند. سپس می‌توان شرط کرد که client در هر write request به storage service، fencing token فعلی خود را هم بفرستد.

در Figure 8-5، client 1 lease را با token شمارهٔ ۳۳ می‌گیرد، اما بعد برای مدت زیادی pause می‌شود و lease آن منقضی می‌شود. client 2 lease را با token شمارهٔ ۳۴ می‌گیرد—شماره همیشه افزایش می‌یابد—و write request خود را همراه token ۳۴ به storage service می‌فرستد. کمی بعد client 1 دوباره فعال می‌شود و write را با token شمارهٔ ۳۳ می‌فرستد. اما storage server به خاطر دارد که پیش‌تر writeای با token بزرگ‌تر، یعنی ۳۴، پردازش کرده است؛ بنابراین request دارای token ۳۳ را رد می‌کند.

اگر از ZooKeeper به‌عنوان lock service استفاده شود، transaction ID با نام `zxid` یا version node با نام `cversion` را می‌توان fencing token قرار داد. این مقدارها تضمین شده که به‌شکل monotonic افزایش می‌یابند و بنابراین ویژگی لازم را دارند [74].

این mechanism مستلزم آن است که خود resource در بررسی tokenها نقش فعال داشته باشد و هر writeای را که token آن از tokenهای پردازش‌شدهٔ قبلی قدیمی‌تر است رد کند. کافی نیست به clientها اعتماد کنیم که خودشان status lock را بررسی کنند. برای resourceهایی که صریحاً fencing token را پشتیبانی نمی‌کنند، شاید بتوان workaroundی به‌کار برد؛ مثلاً در file storage token را در filename قرار داد. بااین‌حال، برای جلوگیری از پردازش request خارج از محدودهٔ حفاظت lock، وجود نوعی check ضروری است.

بررسی token در سمت server شاید در نگاه اول عیب به نظر برسد، اما می‌توان آن را مزیت دانست: عاقلانه نیست service فرض کند clientهایش همیشه درست رفتار می‌کنند، چون clientها اغلب در اختیار افرادی هستند که اولویت‌هایشان با اولویت‌های گردانندگان service تفاوت زیادی دارد [76]. بنابراین بهتر است هر service از خودش در برابر clientهای ناخواسته یا مخرب محافظت کند.

### Byzantine Faults

Fencing token می‌تواند nodeای را شناسایی و متوقف کند که ناخواسته اشتباه عمل می‌کند؛ مثلاً هنوز نفهمیده lease آن منقضی شده است. اما اگر node عمداً بخواهد guaranteeهای system را دور بزند، به‌سادگی می‌تواند messageهایی با fencing token جعلی بفرستد.

در این کتاب فرض می‌کنیم nodeها unreliable اما صادق‌اند: ممکن است کند شوند یا به‌دلیل fault هیچ‌گاه پاسخ ندهند و state آن‌ها به‌دلیل GC pause یا network delay قدیمی شده باشد؛ اما فرض می‌کنیم اگر nodeای پاسخ دهد، «حقیقت» را می‌گوید؛ یعنی تا جایی که می‌داند، از قواعد protocol پیروی می‌کند.

اگر احتمال داشته باشد nodeها «دروغ بگویند» و responseهای دلخواهِ faulty یا corrupt بفرستند، مسئله‌های distributed system بسیار دشوارتر می‌شوند؛ مثلاً nodeای ادعا کند message مشخصی را دریافت کرده، درحالی‌که واقعاً آن را نگرفته است. به این رفتار **Byzantine fault** می‌گویند و مسئلهٔ رسیدن به consensus در چنین محیط بی‌اعتمادی، **Byzantine Generals Problem** نام دارد [77].

#### Byzantine Generals Problem

Byzantine Generals Problem تعمیمی از مسئلهٔ موسوم به Two Generals Problem است [78]. در آن، دو فرماندهٔ ارتش باید دربارهٔ نقشهٔ نبرد توافق کنند. چون در دو محل جدا اردو زده‌اند، تنها از طریق پیک با هم ارتباط دارند و messageهای پیک گاهی با تأخیر می‌رسند یا گم می‌شوند؛ درست مانند packetهای network. این مسئلهٔ consensus را در Chapter 9 بررسی می‌کنیم.

در نسخهٔ Byzantine مسئله، `n` فرمانده باید به توافق برسند، اما چند خائن در میانشان هستند. بیشتر فرماندهان وفادارند و messageهای درست می‌فرستند؛ خائنان ممکن است با ارسال messageهای جعلی یا نادرست، دیگران را فریب دهند و سردرگم کنند و تلاش کنند شناسایی نشوند. از پیش معلوم نیست کدام فرمانده خائن است.

Byzantium شهری باستانی در یونان بود که بعدتر Constantinople نام گرفت و در محل استانبول امروزی در ترکیه قرار داشت. هیچ مدرک تاریخی‌ای وجود ندارد که فرماندهان Byzantium بیش از دیگران مستعد دسیسه و توطئه بوده باشند. نام این مسئله از کاربرد واژهٔ Byzantine به‌معنای بیش از حد پیچیده، دیوان‌سالارانه و موذیانه گرفته شده؛ کاربردی که مدت‌ها پیش از کامپیوترها در سیاست رواج داشت [79]. Lamport می‌خواست ملیتی را انتخاب کند که به خوانندگان برنخورد و به او گفته شد انتخاب نام The Albanian Generals Problem چندان فکر خوبی نیست [80].

یک system **Byzantine fault-tolerant** است اگر حتی وقتی بعضی nodeها خراب‌اند و از protocol پیروی نمی‌کنند، یا مهاجمان مخرب در network اختلال ایجاد می‌کنند، درست به کار ادامه دهد. این مسئله در شرایط خاصی اهمیت دارد. برای مثال:

- در محیط‌های هوافضا، radiation ممکن است data در memory یا registerهای CPU را corrupt کند و باعث شود کامپیوتر responseهایی کاملاً غیرقابل‌پیش‌بینی به nodeهای دیگر بدهد. ازآنجاکه failure system هزینهٔ بسیار سنگینی دارد—مثلاً سقوط هواپیما و کشته شدن سرنشینان یا برخورد موشک با ایستگاه فضایی بین‌المللی—flight control systemها باید Byzantine fault را تحمل کنند [81, 82].
- در systemی با چند سازمان مشارکت‌کننده، ممکن است بعضی participantها بخواهند تقلب کنند یا از دیگران کلاه‌برداری کنند. در چنین شرایطی، اعتماد ساده به messageهای node دیگر امن نیست، چون ممکن است با نیت مخرب فرستاده شده باشند. برای مثال، می‌توان peer-to-peer networkهایی مانند Bitcoin و دیگر blockchainها را راهی برای توافق طرف‌هایی دانست که به یکدیگر اعتماد ندارند—مثلاً دربارهٔ رخ دادن یا ندادن یک transaction—بی‌آنکه به مرجع مرکزی تکیه کنند [83].

بااین‌حال، در systemهایی که در این کتاب بررسی می‌کنیم، معمولاً می‌توان با اطمینان فرض کرد Byzantine fault رخ نمی‌دهد. در datacenter، تمام nodeها زیر کنترل سازمان خودتان هستند و می‌توان امیدوار بود به آن‌ها اعتماد کرد؛ ضمن اینکه سطح radiation آن‌قدر کم است که memory corruption مشکل عمده‌ای نیست. Protocolهای Byzantine fault tolerance پیچیده‌اند [84] و embedded systemهای fault-tolerant به پشتیبانی در سطح hardware نیاز دارند [81]. برای بیشتر data systemهای سمت server، هزینهٔ deployment چنین راه‌حل‌هایی آن‌ها را غیرعملی می‌کند.

Web applicationها باید رفتار دلخواه و مخرب clientهایی را که تحت کنترل end user هستند—مانند web browser—پیش‌بینی کنند. به همین دلیل، input validation، sanitization و output escaping برای جلوگیری از SQL injection و cross-site scripting اهمیت دارند. اما معمولاً در اینجا از Byzantine fault-tolerant protocol استفاده نمی‌کنیم؛ server را مرجع تصمیم‌گیری دربارهٔ رفتار مجاز و غیرمجاز client قرار می‌دهیم. در peer-to-peer networkها که چنین مرجع مرکزی‌ای وجود ندارد، Byzantine fault tolerance اهمیت بیشتری دارد.

می‌توان bug در software را Byzantine fault دانست؛ اما اگر همان software را روی تمام nodeها deploy کنید، Byzantine fault-tolerant algorithm نمی‌تواند نجاتتان دهد. بیشتر این algorithmها نیاز دارند بیش از دوسوم nodeها درست کار کنند؛ یعنی اگر چهار node داشته باشید، حداکثر یکی از آن‌ها می‌تواند malfunction کند. برای استفاده از این روش در برابر bugهای software، باید چهار implementation مستقل از یک software داشته باشید و امیدوار باشید bug فقط در یکی از آن‌ها ظاهر شود.

به همین ترتیب، جذاب به نظر می‌رسد که protocol بتواند از ما در برابر vulnerability، نفوذ امنیتی و حملهٔ مخرب محافظت کند؛ اما این هم واقع‌بینانه نیست. در بیشتر systemها، اگر مهاجم بتواند یک node را compromise کند، احتمالاً می‌تواند بقیه را هم compromise کند، چون همه احتمالاً همان software را اجرا می‌کنند. بنابراین mechanismهای متداول—authentication، access control، encryption، firewall و مانند آن—همچنان ابزارهای اصلی مقابله با مهاجم‌اند.

#### شکل‌های خفیف‌تر دروغ‌گویی

با اینکه فرض می‌کنیم nodeها عموماً صادق‌اند، افزودن mechanismهایی برای مقابله با شکل‌های خفیف «دروغ‌گویی» می‌تواند مفید باشد؛ مثلاً message نامعتبری که به‌دلیل مشکل hardware، bug نرم‌افزاری یا misconfiguration ایجاد شده باشد. این محافظت‌ها Byzantine fault tolerance کامل نیستند و در برابر مهاجم مصمم دوام نمی‌آورند، اما گام‌هایی ساده و عملی برای افزایش reliability هستند. برای مثال:

- Packetهای network گاهی به‌دلیل مشکل hardware یا bug در operating system، driver یا router و مانند آن corrupt می‌شوند. معمولاً checksumهای داخلی TCP و UDP packet خراب را شناسایی می‌کنند، اما گاهی packet از تشخیص می‌گریزد [85, 86, 87]. معمولاً اقدام ساده‌ای مانند checksum در application-level protocol برای محافظت کافی است.
- Applicationای که از طریق public در دسترس است باید inputهای user را با دقت sanitize کند؛ مثلاً بررسی کند value در range معقولی باشد و طول stringها را محدود کند تا کسی با allocation بزرگ memory باعث denial of service نشود. ممکن است internal service پشت firewall بتواند input را با سخت‌گیری کمتری بررسی کند، اما انجام sanity checkهای پایه روی valueها—مثلاً هنگام protocol parsing [85]—ایدهٔ خوبی است.
- NTP client را می‌توان با address چند server configure کرد. هنگام synchronization، client با همهٔ آن‌ها تماس می‌گیرد، errorهایشان را برآورد می‌کند و بررسی می‌کند که آیا اکثریت serverها روی یک بازهٔ زمانی توافق دارند یا نه. تا وقتی بیشتر serverها درست کار کنند، NTP serverای که به‌دلیل misconfiguration زمان غلط گزارش می‌کند به‌عنوان outlier شناسایی و از synchronization کنار گذاشته می‌شود [37]. استفاده از چند server، NTP را از حالتی که فقط به یک server تکیه کند مقاوم‌تر می‌کند.

### System Model and Reality

algorithmهای بسیاری برای حل مسئله‌های distributed system طراحی شده‌اند؛ برای مثال، راه‌حل‌های مسئلهٔ consensus را در Chapter 9 بررسی می‌کنیم. برای مفید بودن، این algorithmها باید faultهای گوناگون distributed system را که در این chapter دیدیم تحمل کنند.

Algorithmها باید طوری نوشته شوند که بیش از حد به جزئیات configuration سخت‌افزار و software محل اجرا وابسته نباشند. این کار مستلزم آن است که نوع faultهایی را که انتظار داریم در system رخ دهند، به‌شکلی formal بیان کنیم. این کار را با تعریف **system model** انجام می‌دهیم؛ انتزاعی که مشخص می‌کند algorithm مجاز است چه فرض‌هایی داشته باشد.

در مورد فرض‌های زمانی، سه system model رایج‌اند:

**Synchronous model**

در synchronous model فرض می‌شود network delay، process pause و clock error همگی bound دارند. این فرض به معنای clockهای کاملاً synchronizeشده یا network بدون delay نیست؛ فقط می‌گوید network delay، pause و clock drift هرگز از یک upper bound ثابت فراتر نمی‌روند [88]. این مدل برای بیشتر systemهای عملی واقع‌بینانه نیست، چون همان‌طور که در این chapter گفتیم، delay و pause نامحدود رخ می‌دهند.

**Partially synchronous model**

Partial synchrony یعنی system بیشتر اوقات مانند system synchronous رفتار می‌کند، اما گاهی از bound مربوط به network delay، process pause و clock drift فراتر می‌رود [88]. این مدل برای بسیاری از systemها واقع‌بینانه است: networkها و processها در بیشتر مواقع رفتار خوبی دارند—وگرنه نمی‌توانستیم کاری انجام دهیم—اما باید بپذیریم که گاهی هر فرض زمانی ممکن است نقض شود. در چنین مواقعی، network delay، pause و clock error می‌توانند به‌طور نامحدود بزرگ شوند.

**Asynchronous model**

در این مدل algorithm اجازه ندارد هیچ فرض زمانی داشته باشد؛ در واقع حتی clock هم ندارد و نمی‌تواند از timeout استفاده کند. بعضی algorithmها را می‌توان برای asynchronous model طراحی کرد، اما این مدل بسیار محدودکننده است.

علاوه بر مسئله‌های زمانی، باید failureهای node را هم در نظر بگیریم. سه system model رایج برای nodeها عبارت‌اند از:

**Crash-stop faults**

در crash-stop model، algorithm مجاز است فرض کند node فقط به یک شکل fail می‌شود: crash می‌کند. یعنی node ممکن است در هر لحظه ناگهان از پاسخ‌گویی دست بکشد و برای همیشه از دسترس خارج شود؛ دیگر برنمی‌گردد.

**Crash-recovery faults**

فرض می‌کنیم nodeها ممکن است در هر لحظه crash کنند و پس از مدتی نامشخص دوباره پاسخ‌گو شوند. در crash-recovery model فرض می‌شود nodeها stable storage دارند—یعنی disk غیر فراری—که هنگام crash حفظ می‌شود، اما state داخل memory از بین می‌رود.

**Byzantine (arbitrary) faults**

Nodeها ممکن است هر رفتار دلخواهی داشته باشند؛ از جمله تلاش کنند nodeهای دیگر را فریب دهند، همان‌طور که در بخش قبل توضیح دادیم.

برای مدل کردن systemهای واقعی، معمولاً ترکیب partially synchronous model و crash-recovery fault مفیدترین است. اما distributed algorithmها چطور با این مدل کنار می‌آیند؟

#### Correctness of an algorithm

برای تعریف درست بودن یک algorithm، می‌توانیم propertyهای آن را مشخص کنیم. برای مثال، خروجی sorting algorithm این property را دارد که در هر دو عنصر متمایز از لیست خروجی، عنصر سمت چپ از عنصر سمت راست کوچک‌تر باشد. این صرفاً تعریف رسمی مرتب بودن یک لیست است.

به همین شکل، با نوشتن propertyهای مورد انتظار از distributed algorithm می‌توانیم تعریف کنیم که درست بودن آن یعنی چه. برای مثال، اگر برای یک lock، fencing token تولید می‌کنیم—به بخش «Fencing tokens» در صفحهٔ ۳۰۳ مراجعه کنید—ممکن است این propertyها را بخواهیم:

**Uniqueness**

هیچ دو requestای مقدار fencing token یکسان دریافت نکنند.

**Monotonic sequence**

اگر request `x`، token `tx` را دریافت کرده و request `y`، token `ty` را گرفته باشد و `x` پیش از آغاز `y` تمام شده باشد، آنگاه باید `tx < ty` باشد.

**Availability**

Nodeای که fencing token درخواست می‌کند و crash نمی‌کند، سرانجام response دریافت می‌کند.

Algorithm در یک system model زمانی درست است که در تمام وضعیت‌هایی که آن مدل ممکن می‌داند، همیشه propertyهایش را رعایت کند. اما اگر همهٔ nodeها crash کنند یا تمام network ناگهان delayهای بی‌نهایت داشته باشد، هیچ algorithmی نمی‌تواند کاری پیش ببرد. پس این تعریف چطور معنا پیدا می‌کند؟

#### Safety and liveness

برای روشن شدن موضوع، باید دو نوع property را از هم جدا کنیم: **safety property** و **liveness property**. در مثال بالا، uniqueness و monotonic sequence از نوع safety هستند، اما availability از نوع liveness است.

چطور این دو نوع را تشخیص دهیم؟ معمولاً liveness propertyها در تعریفشان واژهٔ «سرانجام» دارند. (بله، درست حدس زدید: eventual consistency هم liveness property است [89].) Safety را به‌طور غیررسمی «هیچ اتفاق بدی نمی‌افتد» و liveness را «سرانجام اتفاق خوبی می‌افتد» تعریف می‌کنند. اما نباید بیش از حد به این تعریف‌های غیررسمی تکیه کرد، چون خوب و بد مفاهیمی subjective هستند. تعریف دقیق safety و liveness ریاضی است [90]:

- اگر safety propertyای نقض شود، می‌توان به نقطهٔ مشخصی از زمان اشاره کرد که نقض در آن رخ داده است؛ مثلاً اگر uniqueness نقض شده باشد، می‌توان operation مشخصی را پیدا کرد که در آن fencing token تکراری بازگردانده شده است. پس از نقض safety property، دیگر نمی‌توان آن را برگرداند؛ آسیب رخ داده است.
- liveness برعکس است: ممکن است در لحظه‌ای برقرار نباشد؛ مثلاً nodeای request فرستاده اما هنوز response نگرفته باشد. بااین‌حال، همیشه این امید هست که در آینده برقرار شود؛ مثلاً response برسد.

تفکیک safety و liveness به ما کمک می‌کند با system modelهای دشوار کنار بیاییم. در distributed algorithmها معمول است که بخواهیم safety propertyها در تمام وضعیت‌های ممکن system model برقرار بمانند [88]. یعنی حتی اگر تمام nodeها crash کنند یا کل network از کار بیفتد، algorithm نباید result نادرستی برگرداند و safety propertyها باید حفظ شوند.

اما دربارهٔ liveness می‌توانیم شرط‌هایی بگذاریم. مثلاً بگوییم request فقط وقتی باید response بگیرد که اکثریت nodeها crash نکرده باشند و network نهایتاً از outage recover کند. تعریف partially synchronous model ایجاب می‌کند system بالاخره به وضعیت synchronous برگردد؛ یعنی هر دورهٔ قطع network مدت محدودی طول بکشد و بعد برطرف شود.

#### Mapping system models to the real world

Safety و liveness propertyها و system modelها برای استدلال دربارهٔ correctness distributed algorithm بسیار مفیدند. اما هنگام پیاده‌سازی algorithm در عمل، پیچیدگی واقعیت دوباره خودش را نشان می‌دهد و روشن می‌شود که system model انتزاعی ساده‌شده‌ای از واقعیت است.

برای مثال، algorithmهای crash-recovery عموماً فرض می‌کنند data در stable storage پس از crash باقی می‌ماند. اما اگر data روی disk corrupt شود یا به‌دلیل خطای hardware یا misconfiguration پاک شود چه؟ [91] اگر server bugی در firmware داشته باشد و پس از reboot، hard driveها را نشناسد، با اینکه driveها درست به server وصل‌اند، چه اتفاقی می‌افتد؟ [92]

Quorum algorithmها—به بخش «Quorums for reading and writing» در صفحهٔ ۱۷۹ مراجعه کنید—به این وابسته‌اند که node، dataای را که ادعا می‌کند ذخیره کرده به خاطر داشته باشد. اگر node دچار فراموشی شود و data ذخیره‌شدهٔ قبلی را از دست بدهد، شرط quorum نقض می‌شود و در نتیجه correctness algorithm هم از بین می‌رود. شاید به system model جدیدی نیاز باشد که فرض کند stable storage بیشتر اوقات از crash جان سالم به در می‌برد، اما گاهی ممکن است از دست برود. بااین‌حال، استدلال کردن دربارهٔ آن مدل دشوارتر می‌شود.

شرح نظری algorithm می‌تواند اعلام کند که فرض می‌شود بعضی اتفاق‌ها رخ نمی‌دهند؛ در systemهای non-Byzantine نیز باید دربارهٔ faultهایی که می‌توانند یا نمی‌توانند رخ دهند فرض‌هایی داشته باشیم. بااین‌حال، پیاده‌سازی واقعی شاید همچنان نیاز داشته باشد حالتی را مدیریت کند که اتفاقی ظاهراً ناممکن رخ داده است؛ حتی اگر راه مدیریت آن در نهایت اجرای `printf("Sucks to be you")` و `exit(666)` باشد و کار تمیز کردن خرابی به operator انسانی سپرده شود [93]. (شاید این همان تفاوت computer science و software engineering باشد.)

این حرف به‌معنای بی‌ارزش بودن system modelهای نظری و انتزاعی نیست؛ درست برعکس. آن‌ها برای فروکاستن پیچیدگی system واقعی به مجموعه‌ای قابل‌مدیریت از faultها که بتوان درباره‌شان استدلال کرد، بسیار مفیدند. به این ترتیب مسئله را می‌فهمیم و می‌توانیم روشمندانه حلش کنیم. با نشان دادن اینکه propertyهای algorithm در یک system model مشخص همواره برقرارند، می‌توان correctness آن را اثبات کرد.

اثبات correctness algorithm تضمین نمی‌کند که پیاده‌سازی آن در system واقعی همیشه درست رفتار کند. اما گام نخست بسیار خوبی است، چون تحلیل نظری می‌تواند مشکل‌هایی را در algorithm آشکار کند که شاید مدت‌ها در system واقعی پنهان بمانند و فقط وقتی فرض‌ها—مثلاً فرض‌های زمانی—در شرایط غیرعادی نقض شوند، به مشکل تبدیل شوند. تحلیل نظری و test تجربی هر دو به یک اندازه اهمیت دارند.

## Key Terms

- `Distributed Knowledge` — دانشی که هر node فقط از messageهای دریافتی و state محلی خود به دست می‌آورد؛ هیچ nodeای دید قطعی و کاملی از کل system ندارد.
- `Majority Quorum` — quorumی شامل بیش از نیمی از nodeها که برای یک تصمیم مشترک رأی می‌دهند.
- `Supermajority` — آستانهٔ رأیی بزرگ‌تر از دوسوم nodeها که برخی Byzantine fault-tolerant algorithmها برای ادامهٔ امن لازم دارند.
- `Byzantine Fault` — رفتاری که در آن node ممکن است messageهای دلخواه، نادرست یا فریب‌کارانه بفرستد و از protocol پیروی نکند.
- `Byzantine Fault Tolerance (BFT)` — توانایی system برای ادامهٔ عملکرد درست، حتی با وجود nodeهای خرابکار یا خطاهای arbitrary.
- `Malicious Node` — nodeای که عمداً برای فریب یا مختل کردن رفتار مورد انتظار system اقدام می‌کند.
- `Unreliable Node` — nodeای که ممکن است کند شود، پاسخ ندهد یا state قدیمی داشته باشد، اما در فرض این فصل عمداً دروغ نمی‌گوید.
- `System Model` — abstraction رسمی از timing و faultهایی که algorithm مجاز است برای طراحی و اثبات correctness خود فرض کند.
- `Timing Assumptions` — فرض‌هایی دربارهٔ upper bound تأخیر network، process pause و clock error که یک algorithm بر آن‌ها تکیه می‌کند.
- `Synchronous Model` — مدلی با upper bound مشخص برای network delay، process pause و clock error.
- `Partially Synchronous Model` — مدلی که معمولاً boundهای زمانی را رعایت می‌کند، اما گاهی ممکن است از آن‌ها فراتر رود.
- `Asynchronous Model` — مدلی که هیچ فرضی دربارهٔ timing ندارد و algorithm نمی‌تواند به clock یا timeout متکی باشد.
- `Crash-Stop Fault` — failureای که در آن node ناگهان متوقف می‌شود و دیگر هرگز برنمی‌گردد.
- `Crash-Recovery Fault` — failureای که node پس از crash ممکن است دوباره فعال شود؛ stable storage باقی می‌ماند، اما state درون memory از بین می‌رود.
- `Arbitrary Fault` — هر رفتار دلخواه node، از جمله ارسال message نادرست یا تلاش برای فریب سایر nodeها.
- `Byzantine Generals Problem` — صورت‌بندی consensus در محیطی با participantهایی که برخی از آن‌ها ممکن است messageهای جعلی یا فریب‌کارانه بفرستند.
- `Fencing Token` — عدد monotonic که با هر صدور lock یا lease افزایش می‌یابد و resource با بررسی آن writeهای stale را رد می‌کند.
- `Safety Property` — شرطی که باید همیشه برقرار بماند؛ نقض آن در یک لحظه مشخص رخ می‌دهد و پس از وقوع قابل‌برگشت نیست.
- `Liveness Property` — شرطی دربارهٔ پیشرفت system که ممکن است اکنون برقرار نباشد، اما انتظار می‌رود تحت فرض‌های مشخص سرانجام برقرار شود.
- `Stable Storage` — storage غیر فراری که انتظار می‌رود data آن پس از crash باقی بماند.
- `Truth and Uncertainty` — محدودیت دانسته‌های هر node دربارهٔ وضعیت system، چون مشاهدهٔ remote state به messageهای نامطمئن وابسته است.

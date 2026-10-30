# Chapter 8 — The Trouble with Distributed Systems

## Unreliable Clocks

Clock و time اهمیت زیادی دارند. Applicationها برای پاسخ به پرسش‌هایی از این دست به آن‌ها وابسته‌اند:

1. آیا timeout این request تمام شده است؟
2. صدک ۹۹ response time این service چقدر است؟
3. این service در پنج دقیقهٔ گذشته به‌طور متوسط چند query در ثانیه پردازش کرده است؟
4. کاربر چه مدت در سایت ما بوده است؟
5. این مقاله چه زمانی منتشر شده است؟
6. reminder email باید در چه تاریخ و ساعتی ارسال شود؟
7. این cache entry چه زمانی منقضی می‌شود؟
8. timestamp این error در log file چیست؟

مثال‌های ۱ تا ۴ مدت‌ها را اندازه می‌گیرند؛ مثلاً فاصلهٔ زمانی میان ارسال request و دریافت response. مثال‌های ۵ تا ۸ به نقطه‌ای از زمان اشاره می‌کنند؛ یعنی eventهایی که در تاریخ و ساعت مشخصی رخ می‌دهند.

در distributed system، time مسئله‌ای پیچیده است، چون ارتباط فوری نیست: message برای عبور از network و رسیدن از یک machine به machine دیگر زمان می‌برد. زمان دریافت message همیشه پس از زمان ارسال آن است، اما به‌دلیل variable delayهای network نمی‌دانیم این فاصله چقدر است. وقتی چند machine درگیرند، همین موضوع گاهی تشخیص ترتیب رخدادها را دشوار می‌کند.

علاوه بر این، هر machine در network clock خودش را دارد؛ یک قطعهٔ سخت‌افزاری واقعی که معمولاً از نوسان‌ساز بلوری quartz استفاده می‌کند. این دستگاه‌ها کاملاً دقیق نیستند، بنابراین هر machine برداشت خودش را از time دارد و clock آن ممکن است کمی تندتر یا کندتر از clock machineهای دیگر کار کند. تا حدی می‌توان clockها را synchronize کرد. رایج‌ترین روش **Network Time Protocol (NTP)** است که clock کامپیوتر را بر اساس time گزارش‌شده از گروهی از serverها تنظیم می‌کند [37]. خود این serverها time را از منبع دقیق‌تری، مانند GPS receiver، می‌گیرند.

### Monotonic Versus Time-of-Day Clocks

کامپیوترهای امروزی دست‌کم دو نوع clock دارند: **time-of-day clock** و **monotonic clock**. هر دو time را اندازه می‌گیرند، اما چون کاربردهای متفاوتی دارند، باید آن‌ها را از هم تفکیک کرد.

#### Time-of-day clocks

Time-of-day clock همان کاری را می‌کند که به‌طور شهودی از یک clock انتظار داریم: تاریخ و ساعت فعلی را طبق یک تقویم برمی‌گرداند؛ به این مقدار **wall-clock time** هم گفته می‌شود. برای مثال، `clock_gettime(CLOCK_REALTIME)` در Linux و `System.currentTimeMillis()` در Java تعداد ثانیه‌ها یا millisecondهای سپری‌شده از epoch را برمی‌گردانند: نیمه‌شب UTC در ۱ ژانویهٔ ۱۹۷۰ بر اساس تقویم Gregorian، بدون احتساب leap secondها. بعضی systemها تاریخ دیگری را به‌عنوان نقطهٔ مرجع به‌کار می‌برند.

Time-of-day clockها معمولاً با NTP synchronize می‌شوند؛ بنابراین timestamp روی یک machine—در حالت ایده‌آل—همان معنایی را دارد که timestamp روی machine دیگر. بااین‌حال، این clockها رفتارهای غیرعادی دیگری هم دارند که در بخش بعد می‌آیند. به‌خصوص اگر clock محلی خیلی جلوتر از NTP server باشد، ممکن است به‌اجبار reset شود و ناگهان به نقطه‌ای در گذشته برگردد. این جهش‌ها و همچنین نادیده گرفتن leap secondها باعث می‌شوند time-of-day clock برای اندازه‌گیری elapsed time مناسب نباشد [38].

با اینکه نام clock در `CLOCK_REALTIME` شامل واژهٔ real-time است، این نام ارتباطی با real-time operating system ندارد؛ به بخش «Response time guarantees» در همین فصل مراجعه کنید.

از نظر تاریخی، resolution این clockها نیز نسبتاً درشت بوده است؛ مثلاً در systemهای قدیمی‌تر Windows، مقدار clock هر بار ۱۰ ms جلو می‌رفت [39]. در systemهای جدید این مسئله کمتر مشکل‌ساز است.

#### Monotonic clocks

Monotonic clock برای اندازه‌گیری duration یا time interval مناسب است؛ مثلاً برای timeout یا response time یک service. `clock_gettime(CLOCK_MONOTONIC)` در Linux و `System.nanoTime()` در Java نمونه‌هایی از monotonic clock هستند. نام آن از این ویژگی می‌آید که تضمین شده همیشه رو به جلو حرکت کند؛ درحالی‌که time-of-day clock ممکن است به عقب بپرد.

می‌توانید مقدار monotonic clock را در یک لحظه بخوانید، کاری انجام دهید و بعدتر دوباره clock را بخوانید. اختلاف دو مقدار نشان می‌دهد میان دو خواندن چقدر time گذشته است. اما مقدار مطلق clock معنای مشخصی ندارد؛ ممکن است تعداد nanosecondهای سپری‌شده از زمان راه‌اندازی کامپیوتر باشد یا مقداری قراردادی از همین جنس. به‌ویژه، مقایسهٔ مقدار monotonic clock دو کامپیوتر بی‌معناست، چون مقدارها معنای مشترکی ندارند.

در serverی با چند CPU socket ممکن است برای هر CPU یک timer جدا وجود داشته باشد که لزوماً با timerهای CPUهای دیگر synchronize نیست. Operating system اختلاف‌ها را جبران می‌کند و می‌کوشد نمایی monotonic از clock در اختیار application threadها قرار دهد، حتی وقتی threadها روی CPUهای مختلف schedule می‌شوند. بااین‌حال، بهتر است این تضمین monotonic بودن را با کمی احتیاط بپذیریم [40].

اگر NTP تشخیص دهد quartz محلی کامپیوتر تندتر یا کندتر از server کار می‌کند، می‌تواند frequency حرکت monotonic clock را تنظیم کند؛ به این کار **slewing the clock** می‌گویند. NTP به‌طور پیش‌فرض اجازه می‌دهد rate clock حداکثر ۰٫۰۵٪ تند یا کند شود، اما نمی‌تواند monotonic clock را به جلو یا عقب بپراند. Resolution این clockها معمولاً خوب است و در بیشتر systemها می‌توانند intervalهایی در حد microsecond یا کمتر را اندازه بگیرند.

در distributed system، استفاده از monotonic clock برای سنجش elapsed time—مثلاً timeout—معمولاً مناسب است، چون به synchronization میان clock nodeهای مختلف وابسته نیست و نسبت به کمی خطای اندازه‌گیری حساس نیست.

### Clock Synchronization and Accuracy

Monotonic clockها به synchronization نیاز ندارند، اما برای مفید بودن، time-of-day clockها باید بر اساس NTP server یا منبع بیرونی دیگری تنظیم شوند. متأسفانه، روش‌های ما برای درست نشان دادن time به اندازه‌ای که انتظار داریم reliable یا دقیق نیستند؛ hardware clockها و NTP گاهی رفتار پیش‌بینی‌ناپذیری دارند. چند نمونه:

- Quartz clock کامپیوتر چندان دقیق نیست و **drift** می‌کند؛ یعنی تندتر یا کندتر از مقدار درست حرکت می‌کند. Clock drift به دمای machine هم وابسته است. Google برای serverهایش drift برابر ۲۰۰ ppm (parts per million) را فرض می‌کند [41]. این مقدار برای clockی که هر ۳۰ ثانیه با server دوباره synchronize می‌شود معادل ۶ ms drift است؛ اگر روزی یک‌بار synchronize شود، drift به ۱۷ ثانیه می‌رسد. این drift حتی وقتی همه‌چیز درست کار می‌کند، سقفی برای بهترین accuracy ممکن ایجاد می‌کند.
- اگر clock کامپیوتر بیش از حد با NTP server اختلاف داشته باشد، ممکن است از synchronize شدن خودداری کند یا clock محلی به‌اجبار reset شود [37]. Applicationهایی که time را پیش و پس از reset مشاهده می‌کنند، ممکن است ببینند time به عقب برگشته یا ناگهان به جلو جهیده است.
- اگر node به‌اشتباه از دسترسی به NTP serverها با firewall محروم شود، ممکن است این misconfiguration مدتی شناسایی نشود. شواهد تجربی نشان می‌دهد چنین اتفاقی در عمل رخ می‌دهد.
- کیفیت NTP synchronization نمی‌تواند از کیفیت network delay بهتر باشد؛ بنابراین در network شلوغی که packet delayهای variable دارد، accuracy محدود می‌شود. در آزمایشی نشان داده شد که هنگام synchronization از طریق internet، حداقل خطای ۳۵ ms دست‌یافتنی است [42]؛ اما spikeهای گاه‌به‌گاه در network delay خطا را به حدود یک ثانیه می‌رسانند. بسته به configuration، network delay زیاد ممکن است باعث شود NTP client به‌کلی از تلاش دست بکشد.
- بعضی NTP serverها time اشتباه یا misconfigureشده گزارش می‌کنند؛ گاهی با اختلاف چند ساعت [43, 44]. NTP clientها نسبتاً مقاوم‌اند، چون چند server را query می‌کنند و outlierها را نادیده می‌گیرند. بااین‌حال، سپردن correctness system به timeای که یک غریبه در internet گزارش کرده، تا حدی نگران‌کننده است.
- به‌خاطر leap second، ممکن است یک دقیقه ۵۹ یا ۶۱ ثانیه طول بکشد و فرض‌های timing در systemهایی را که برای leap second طراحی نشده‌اند برهم بزند [45]. اینکه leap secondها باعث crash بسیاری از systemهای بزرگ شده‌اند [38, 46] نشان می‌دهد فرض نادرست دربارهٔ clock چقدر راحت می‌تواند وارد system شود. یکی از راه‌های مناسب برخورد با leap second این است که NTP serverها «دروغ بگویند» و اصلاح leap second را طی یک روز به‌تدریج انجام دهند؛ این روش **smearing** نام دارد [47, 48]، هرچند رفتار واقعی NTP serverها در عمل متفاوت است [49].
- در virtual machineها، hardware clock مجازی‌سازی شده است و برای applicationهایی که به timekeeping دقیق نیاز دارند، چالش‌های بیشتری ایجاد می‌کند [50]. وقتی یک CPU core میان VMها share می‌شود، اجرای هر VM ممکن است برای ده‌ها millisecond متوقف شود تا VM دیگری اجرا شود. از دید application، این pause شبیه جهش ناگهانی clock به جلوست [26].
- اگر software را روی deviceهایی اجرا می‌کنید که کنترل کاملی بر آن‌ها ندارید—مثلاً mobile یا embedded deviceها—احتمالاً نمی‌توانید به hardware clock آن‌ها اعتماد کنید. بعضی userها عمداً تاریخ و ساعت نادرستی روی device تنظیم می‌کنند؛ مثلاً برای دور زدن محدودیت زمانی در gameها. در نتیجه clock ممکن است زمانی بسیار دور در گذشته یا آینده را نشان دهد.

اگر به‌اندازهٔ کافی برای clock accuracy منابع صرف کنید، می‌توان به accuracy بسیار خوبی رسید. برای مثال، پیش‌نویس مقررات اروپایی MiFID II برای مؤسسه‌های مالی از تمام high-frequency trading fundها می‌خواهد clockهایشان را در فاصلهٔ حداکثر ۱۰۰ microsecond از UTC synchronize کنند. هدف کمک به debug کردن ناهنجاری‌های بازار مانند «flash crash» و تشخیص دست‌کاری بازار است [51].

این دقت با GPS receiver، **Precision Time Protocol (PTP)** [52] و deployment و monitoring دقیق قابل دستیابی است. بااین‌حال، به تلاش و تخصص زیادی نیاز دارد و راه‌های بسیاری برای خراب شدن clock synchronization وجود دارد. اگر NTP daemon شما misconfigure باشد یا firewall ترافیک NTP را مسدود کند، خطای clock ناشی از drift می‌تواند به‌سرعت بزرگ شود.

### Relying on Synchronized Clocks

Clockها ساده و قابل‌استفاده به نظر می‌رسند، اما دام‌های فراوان و غیرمنتظره‌ای دارند: یک روز ممکن است دقیقاً ۸۶٬۴۰۰ ثانیه نباشد، time-of-day clock ممکن است به عقب برگردد و time روی یک node ممکن است با time روی node دیگر اختلاف زیادی داشته باشد.

پیش‌تر در این chapter دربارهٔ networkهایی صحبت کردیم که packetها را drop می‌کنند یا به‌شکلی دلخواه به تأخیر می‌اندازند. هرچند networkها بیشتر اوقات درست کار می‌کنند، software باید با این فرض طراحی شود که گاهی network fault رخ می‌دهد و باید بتواند به‌خوبی با آن کنار بیاید. دربارهٔ clockها هم همین‌طور است: با اینکه بیشتر اوقات درست کار می‌کنند، software مقاوم باید برای clock نادرست آماده باشد.

یکی از مشکل‌ها این است که clock نادرست به‌سادگی از چشم پنهان می‌ماند. اگر CPU یک machine خراب یا network آن misconfigure باشد، احتمالاً machine اصلاً کار نمی‌کند و مشکل سریع شناسایی و رفع می‌شود. اما اگر quartz clock آن خراب یا NTP client آن misconfigure باشد، بیشتر چیزها ظاهراً درست کار می‌کنند، درحالی‌که clock آرام‌آرام از زمان واقعی فاصله می‌گیرد. اگر بخشی از software به clockی با synchronization دقیق وابسته باشد، نتیجه بیشتر احتمال دارد data loss خاموش و ظریفی باشد تا crash آشکار [53, 54].

بنابراین، اگر از softwareای استفاده می‌کنید که به clockهای synchronizeشده نیاز دارد، باید offset clock میان تمام machineها را با دقت monitor کنید. هر nodeای که clock آن بیش از حد از بقیه فاصله گرفته، باید dead اعلام و از cluster خارج شود. این monitoring کمک می‌کند clock خراب را پیش از وارد کردن آسیب جدی تشخیص دهید.

#### Timestamp برای مرتب‌سازی eventها

یک مورد را در نظر بگیریم که اتکا به clock وسوسه‌انگیز، اما خطرناک است: مرتب‌سازی eventها در چند node. برای مثال، اگر دو client در یک distributed database write کنند، کدام‌یک زودتر رسیده است؟ کدام write جدیدتر است؟ Figure 8-3 استفادهٔ خطرناک از time-of-day clock را در databaseای با multi-leader replication نشان می‌دهد؛ مثال شبیه Figure 5-9 است. Client A روی node 1 مقدار `x = 1` را می‌نویسد؛ write به node 3 replicate می‌شود؛ client B مقدار `x` را روی node 3 افزایش می‌دهد و مقدار به `x = 2` می‌رسد؛ سپس هر دو write به node 2 replicate می‌شوند.

**Figure 8-3.** Write مربوط به client B از نظر causality پس از write مربوط به client A رخ داده، اما timestamp مربوط به B زودتر است.

در Figure 8-3، هنگام replicate شدن write به nodeهای دیگر، timestampی بر اساس time-of-day clock node مبدأ به آن الصاق می‌شود. در این مثال clock synchronization بسیار خوب است: skew میان node 1 و node 3 کمتر از ۳ ms است؛ دقتی که احتمالاً در عمل نمی‌توان انتظارش را داشت.

بااین‌حال، timestampها ترتیب eventها را درست نشان نمی‌دهند: timestamp مربوط به write `x = 1` برابر ۴۲٫۰۰۴ ثانیه است، اما timestamp مربوط به write `x = 2` برابر ۴۲٫۰۰۳ ثانیه است، با اینکه `x = 2` بی‌تردید بعدتر رخ داده است. وقتی node 2 این دو event را دریافت کند، اشتباهاً نتیجه می‌گیرد `x = 1` مقدار جدیدتر است و write مربوط به `x = 2` را کنار می‌گذارد. در عمل، increment مربوط به client B از دست می‌رود.

این راهبرد conflict resolution، **last write wins (LWW)** نام دارد و هم در multi-leader replication و هم در databaseهای leaderless مانند Cassandra [53] و Riak [54] به‌طور گسترده استفاده می‌شود؛ به بخش «Last write wins (discarding concurrent writes)» در صفحهٔ ۱۸۶ مراجعه کنید. بعضی implementationها timestamp را به‌جای server روی client تولید می‌کنند، اما مشکل بنیادی LWW تغییر نمی‌کند:

- ممکن است writeهای database به‌شکلی مرموز ناپدید شوند: nodeای که clock آن عقب است، تا زمانی که clock skew میان nodeها برطرف نشده، نمی‌تواند valueهایی را که nodeی با clock جلوتر نوشته overwrite کند [54, 55]. در نتیجه ممکن است مقدار دلخواهی data، بی‌صدا و بدون گزارش error به application، drop شود.
- LWW نمی‌تواند writeهایی را که پشت سر هم و با فاصلهٔ کم رخ داده‌اند—در Figure 8-3، increment کلاینت B قطعاً پس از write کلاینت A رخ می‌دهد—از writeهایی که واقعاً concurrent بوده‌اند و هیچ‌یک از نویسنده‌ها از دیگری خبر نداشته، تشخیص دهد. برای جلوگیری از نقض causality به mechanismهای دیگری مانند version vector نیاز است؛ به بخش «Detecting Concurrent Writes» در صفحهٔ ۱۸۴ مراجعه کنید.
- ممکن است دو node مستقل writeهایی با timestamp یکسان تولید کنند؛ به‌خصوص وقتی resolution clock فقط millisecond باشد. برای حل چنین conflictهایی به مقدار tie-breaker دیگری نیاز است، که می‌تواند یک عدد تصادفی بزرگ باشد؛ اما این راه هم ممکن است causality را نقض کند [53].

پس هرچند حل conflict با نگه داشتن value «جدیدتر» و کنار گذاشتن بقیه وسوسه‌انگیز است، باید توجه داشت که تعریف «جدیدتر» به time-of-day clock محلی وابسته است و آن clock ممکن است اشتباه باشد. حتی با clockهایی که با NTP بسیار دقیق synchronize شده‌اند، ممکن است packetی در timestamp برابر ۱۰۰ ms طبق clock فرستنده ارسال شود و در timestamp برابر ۹۹ ms طبق clock گیرنده برسد؛ در ظاهر، packet پیش از ارسال شدن رسیده است، که ناممکن است.

آیا می‌توان NTP synchronization را آن‌قدر دقیق کرد که چنین ترتیب نادرستی رخ ندهد؟ احتمالاً نه، چون accuracy مربوط به synchronization در NTP، علاوه بر منبع‌های خطای دیگر مانند quartz drift، خودش به round-trip time شبکه محدود است. برای مرتب‌سازی درست، منبع clock باید به‌مراتب دقیق‌تر از چیزی باشد که اندازه می‌گیرید؛ یعنی network delay.

**Logical clock**ها [56, 57] جایگزین امن‌تری برای مرتب‌سازی eventها هستند. آن‌ها به‌جای oscillator بلوری، بر counterهای افزایشی تکیه می‌کنند؛ به بخش «Detecting Concurrent Writes» در صفحهٔ ۱۸۴ مراجعه کنید. Logical clock زمان روز یا تعداد ثانیه‌های سپری‌شده را اندازه نمی‌گیرد، بلکه فقط ترتیب نسبی eventها را مشخص می‌کند؛ اینکه eventی پیش یا پس از event دیگر رخ داده است. در مقابل، time-of-day و monotonic clockها که elapsed time واقعی را اندازه می‌گیرند، **physical clock** هم نامیده می‌شوند. مرتب‌سازی را در بخش «Ordering Guarantees» در صفحهٔ ۳۳۹ دوباره بررسی می‌کنیم.

#### خواندن clock همراه با بازهٔ اطمینان

ممکن است بتوانید time-of-day clock یک machine را با resolution در حد microsecond یا حتی nanosecond بخوانید. اما اندازه‌گیری با resolution بسیار ریز به‌معنای دقت واقعی value تا همان precision نیست. در واقع، به احتمال زیاد چنین نیست: همان‌طور که گفتیم، drift یک quartz clock غیردقیق حتی با synchronize کردن هر دقیقه با NTP server محلی، به‌راحتی به چند millisecond می‌رسد. اگر NTP server روی public internet باشد، بهترین accuracy ممکن احتمالاً در حد ده‌ها millisecond است و هنگام network congestion ممکن است error از ۱۰۰ ms هم بیشتر شود [57].

بنابراین، درست نیست clock reading را یک نقطهٔ دقیق در زمان بدانیم؛ بهتر است آن را بازه‌ای از زمان‌ها در قالب **confidence interval** در نظر بگیریم. برای مثال، system ممکن است با اطمینان ۹۵٪ بداند time فعلی بین ۱۰٫۳ و ۱۰٫۵ ثانیه پس از دقیقه است، اما دقیق‌تر از این نمی‌داند [58]. اگر فقط time را با دقت `+/- 100 ms` بدانیم، رقم‌های microsecond در timestamp عملاً بی‌معنا هستند.

می‌توان bound مربوط به uncertainty را بر اساس time source محاسبه کرد. اگر GPS receiver یا atomic (caesium) clock مستقیماً به کامپیوتر وصل باشد، سازنده range خطای مورد انتظار را گزارش می‌کند. اگر time را از server بگیرید، uncertainty تقریباً از مجموع quartz drift مورد انتظار از آخرین synchronization، uncertainty خود NTP server و round-trip time شبکه تا server به دست می‌آید؛ با این فرض که به server اعتماد دارید.

متأسفانه بیشتر systemها این uncertainty را در اختیار application نمی‌گذارند. برای مثال، وقتی `clock_gettime()` را فراخوانی می‌کنید، مقدار بازگشتی error مورد انتظار timestamp را نمی‌گوید؛ بنابراین نمی‌دانید confidence interval آن پنج millisecond است یا پنج سال.

استثنای جالب، **TrueTime API** گوگل در Spanner است [41] که confidence interval مربوط به clock محلی را به‌صراحت گزارش می‌کند. با درخواست time فعلی، دو مقدار `[earliest, latest]` دریافت می‌کنید: زودترین و دیرترین timestamp ممکن. بر اساس محاسبات uncertainty، clock می‌داند time واقعی فعلی جایی در این interval قرار دارد. عرض interval، از جمله، به مدت زمانی بستگی دارد که از آخرین synchronization quartz clock محلی با منبع دقیق‌تر گذشته است.

#### استفاده از clockهای synchronizeشده برای global snapshot

در بخش «Snapshot Isolation and Repeatable Read» در صفحهٔ ۲۳۷، snapshot isolation را بررسی کردیم؛ قابلیتی بسیار مفید در databaseهایی که باید هم transactionهای کوچک و سریع read-write و هم transactionهای طولانی read-only—مثلاً برای backup یا analytics—را پشتیبانی کنند. این قابلیت اجازه می‌دهد transactionهای read-only، database را در state سازگاری در یک نقطهٔ زمانی مشخص ببینند، بی‌آنکه transactionهای read-write را lock کنند یا در کارشان اختلال ایجاد کنند.

رایج‌ترین پیاده‌سازی snapshot isolation به transaction IDای نیاز دارد که به‌طور monotonic افزایش یابد. اگر write پس از snapshot رخ داده باشد—یعنی transaction ID آن از ID snapshot بیشتر باشد—آن write برای snapshot transaction نامرئی است. در database تک‌گرهی، یک counter ساده برای تولید transaction ID کافی است.

اما وقتی database روی machineهای متعدد و احتمالاً چند datacenter توزیع شده باشد، تولید transaction ID سراسری و monotonic—در تمام partitionها—دشوار است، چون به coordination نیاز دارد. Transaction ID باید causality را بازتاب دهد: اگر transaction B مقداری را بخواند که transaction A نوشته، ID مربوط به B باید از ID مربوط به A بزرگ‌تر باشد؛ وگرنه snapshot consistent نخواهد بود. در systemی با transactionهای کوچک و سریع فراوان، تولید transaction ID به bottleneck تحمل‌ناپذیری تبدیل می‌شود.

> generatorهای توزیع‌شدهٔ sequence number، مانند Snowflake متعلق به Twitter، می‌توانند IDهای یکتای تقریباً monotonic را به‌شکلی scalable تولید کنند؛ مثلاً با اختصاص blockهایی از فضای ID به nodeهای مختلف. اما معمولاً نمی‌توانند ترتیبی سازگار با causality را تضمین کنند، چون بازهٔ زمانی تخصیص blockهای ID از بازهٔ زمانی read و writeهای database طولانی‌تر است. برای توضیح بیشتر به «Ordering Guarantees» در صفحهٔ ۳۳۹ مراجعه کنید.

آیا می‌توان از timestampهای time-of-day clockهای synchronizeشده به‌عنوان transaction ID استفاده کرد؟ اگر synchronization به‌اندازهٔ کافی دقیق بود، این timestampها ویژگی لازم را داشتند: transactionهای بعدی timestamp بزرگ‌تری می‌گرفتند. اما مشکل، uncertainty مربوط به clock accuracy است.

Spanner، snapshot isolation را در چند datacenter به این روش پیاده‌سازی می‌کند [59, 60]. این system از confidence interval مربوط به clock در TrueTime API استفاده می‌کند و بر این مشاهده تکیه دارد: اگر دو confidence interval، یکی برای A و دیگری برای B، به‌شکل `A = [Aearliest, Alatest]` و `B = [Bearliest, Blatest]` باشند و هم‌پوشانی نداشته باشند—یعنی `Aearliest < Alatest < Bearliest < Blatest`—آن‌گاه B قطعاً پس از A رخ داده است. فقط اگر intervalها هم‌پوشانی داشته باشند، ترتیب رخ دادن A و B نامشخص است.

برای اینکه timestampهای transaction causality را بازتاب دهند، Spanner پیش از commit کردن read-write transaction عمداً به‌اندازهٔ طول confidence interval صبر می‌کند. به این ترتیب، تضمین می‌کند هر transactionای که ممکن است data را بخواند، به‌اندازهٔ کافی دیرتر رخ می‌دهد تا confidence intervalهایشان overlap نکنند. برای کوتاه نگه داشتن زمان انتظار، Spanner باید uncertainty clock را تا جای ممکن کم کند؛ Google برای این منظور در هر datacenter یک GPS receiver یا atomic clock نصب می‌کند تا clockها با دقتی در حدود ۷ ms synchronize شوند [41].

استفاده از clock synchronization برای تعریف semantics مربوط به distributed transactionها موضوع پژوهش فعال است [57, 61, 62]. این ایده‌ها جالب‌اند، اما تا امروز خارج از Google در databaseهای رایج پیاده‌سازی نشده‌اند.

### Process Pauses

مثال دیگری از استفادهٔ خطرناک از clock در distributed system را در نظر بگیریم. فرض کنید database شما برای هر partition یک leader دارد و فقط leader مجاز به پذیرش write است. یک node از کجا می‌فهمد هنوز leader است—یعنی بقیه آن را dead اعلام نکرده‌اند—و می‌تواند با اطمینان write بپذیرد؟

یک راه این است که leader از nodeهای دیگر **lease** بگیرد؛ lease شبیه lockای با timeout است [63]. در هر لحظه فقط یک node می‌تواند lease را در اختیار داشته باشد؛ بنابراین وقتی node lease می‌گیرد، می‌داند تا زمان انقضای آن برای مدتی مشخص leader است. برای leader ماندن، node باید lease را به‌طور دوره‌ای پیش از انقضا تمدید کند. اگر node fail شود، تمدید lease را متوقف می‌کند و پس از انقضای آن، node دیگری می‌تواند takeover کند.

می‌توان حلقهٔ رسیدگی به request را چیزی شبیه این تصور کرد:

```java
while (true) {
    request = getIncomingRequest();

    // Ensure that the lease always has at least 10 seconds remaining
    if (lease.expiryTimeMillis - System.currentTimeMillis() < 10000) {
        lease = lease.renew();
    }

    if (lease.isValid()) {
        process(request);
    }
}
```

این code چه مشکلی دارد؟ نخست، به clockهای synchronizeشده متکی است: زمان انقضای lease را machine دیگری تعیین کرده است—مثلاً ممکن است expiry را current time به‌علاوهٔ ۳۰ ثانیه حساب کرده باشد—اما اینجا با system clock محلی مقایسه می‌شود. اگر clockها بیش از چند ثانیه با هم اختلاف داشته باشند، code رفتارهای عجیبی نشان می‌دهد.

حتی اگر protocol را طوری تغییر دهیم که فقط از monotonic clock محلی استفاده کند، مشکل دیگری باقی می‌ماند: code فرض می‌کند میان زمانی که clock را check می‌کند (`System.currentTimeMillis()`) و زمانی که request را پردازش می‌کند (`process(request)`)، زمان بسیار کمی می‌گذرد. معمولاً code سریع اجرا می‌شود و buffer ده‌ثانیه‌ای برای جلوگیری از منقضی شدن lease در میانهٔ پردازش کافی است.

اما اگر اجرای program به‌طور غیرمنتظره pause شود چه؟ فرض کنید thread نزدیک خط `lease.isValid()` برای ۱۵ ثانیه متوقف شود و بعد ادامه دهد. احتمالاً تا زمان پردازش request، lease منقضی شده و node دیگری رهبری را به‌دست گرفته است. بااین‌حال، هیچ نشانه‌ای به این thread نمی‌گوید که این‌قدر طولانی pause شده؛ بنابراین تا دور بعدی loop متوجه انقضای lease نمی‌شود. در آن زمان ممکن است با پردازش request کاری ناامن انجام داده باشد.

آیا فرض pause شدن thread برای این مدت طولانی غیرمنطقی است؟ متأسفانه نه. علت‌های مختلفی می‌توانند چنین وضعی ایجاد کنند:

- بسیاری از runtimeهای زبان برنامه‌نویسی، مانند Java Virtual Machine، garbage collector (GC) دارند که گاهی باید تمام threadهای در حال اجرا را متوقف کند. این **stop-the-world GC pause**ها گاهی چند دقیقه طول کشیده‌اند [64]. حتی garbage collectorهایی که «concurrent» نامیده می‌شوند، مانند CMS در HotSpot JVM، نمی‌توانند کاملاً موازی با application اجرا شوند و گاهی آن‌ها هم باید اجرای همه را متوقف کنند [65]. ممکن است با تغییر الگوی allocation یا تنظیم GC pauseها را کم کرد [66]، اما برای ارائهٔ guarantee مقاوم باید بدترین حالت را فرض کنیم.
- در محیط virtualized ممکن است یک virtual machine suspend شود—اجرای تمام processها متوقف و محتوای memory روی disk ذخیره شود—و بعداً resume شود؛ یعنی memory بازگردانده و اجرا از سر گرفته شود. این pause در هر نقطه‌ای از اجرای process ممکن است رخ دهد و مدت آن می‌تواند دلخواهانه طولانی باشد. گاهی از این قابلیت برای live migration یک VM از hostی به host دیگر، بدون reboot، استفاده می‌شود؛ در این حالت مدت pause به rate نوشتن processها در memory بستگی دارد [67].
- در deviceهای کاربر نهایی، مانند laptop، اجرا نیز ممکن است به‌طور دلخواه suspend و resume شود؛ مثلاً وقتی کاربر در laptop را می‌بندد.
- وقتی operating system به thread دیگری context switch می‌کند، یا hypervisor در زمان اجرای virtual machine به VM دیگری switch می‌کند، thread جاری ممکن است در هر نقطه‌ای از code pause شود. در virtual machine، زمان CPUای که VMهای دیگر مصرف می‌کنند **steal time** نام دارد. اگر machine زیر load سنگین باشد و queue بلندی از threadهای منتظر اجرا وجود داشته باشد، ممکن است مدتی طول بکشد تا thread متوقف‌شده دوباره اجرا شود.
- اگر application دسترسی synchronous به disk داشته باشد، thread ممکن است هنگام انتظار برای تکمیل disk I/O کند pause شود [68]. در بسیاری از زبان‌ها، disk access ممکن است به‌طور غیرمنتظره اتفاق بیفتد، حتی اگر code صریحاً به file اشاره نکند. برای مثال، Java classloader فایل‌های class را هنگام اولین استفاده به‌صورت lazy load می‌کند و این کار ممکن است در هر زمانی از اجرای program رخ دهد. I/O pause و GC pause حتی ممکن است با هم ترکیب شوند و delay بیشتری بسازند [69]. اگر disk در واقع network filesystem یا network block deviceای مانند EBS متعلق به Amazon باشد، latency مربوط به I/O تحت تأثیر variability شبکه نیز قرار می‌گیرد [29].
- اگر operating system اجازهٔ swap کردن به disk (paging) را داشته باشد، یک memory access ساده ممکن است page fault ایجاد کند و لازم شود pageای از disk به memory بار شود. thread تا پایان این I/O کند متوقف می‌ماند. اگر فشار memory زیاد باشد، ممکن است لازم شود page دیگری نیز به disk منتقل شود. در شرایط حاد، operating system بیشتر وقتش را صرف جابه‌جا کردن pageها به داخل و خارج memory می‌کند و کار مفید اندکی انجام می‌دهد؛ این وضعیت **thrashing** نام دارد. برای جلوگیری از آن، paging را در serverها اغلب غیرفعال می‌کنند؛ یعنی ترجیح می‌دهند برای آزاد کردن memory یک process را kill کنند تا اینکه خطر thrashing را بپذیرند.
- می‌توان یک Unix process را با ارسال signal `SIGSTOP` pause کرد؛ مثلاً با فشردن Ctrl-Z در shell. این signal بلافاصله process را از دریافت CPU cycle بیشتر بازمی‌دارد تا وقتی با `SIGCONT` resume شود؛ سپس process از همان‌جایی که متوقف شده بود ادامه می‌دهد. حتی اگر محیط شما معمولاً از `SIGSTOP` استفاده نکند، ممکن است یک operations engineer تصادفی آن را ارسال کند.

تمام این اتفاق‌ها می‌توانند thread در حال اجرا را در هر نقطه‌ای preempt کنند و مدتی بعد دوباره به اجرا درآورند، بی‌آنکه thread متوجه شود. این مسئله شبیه thread-safe کردن code چندنخی روی یک machine است: نمی‌توان دربارهٔ timing فرضی داشت، چون context switchهای دلخواه و اجرای موازی ممکن‌اند.

برای نوشتن code چندنخی thread-safe روی یک machine، ابزارهای خوبی داریم: mutex، semaphore، atomic counter، lock-free data structure، blocking queue و مانند آن. متأسفانه این ابزارها مستقیماً به distributed system منتقل نمی‌شوند، چون distributed system حافظهٔ shared ندارد و تنها راه ارتباط آن ارسال message از طریق networkی غیرقابل‌اعتماد است.

یک node در distributed system باید فرض کند اجرای آن در هر نقطه‌ای، حتی در میانهٔ یک function، ممکن است برای مدت قابل‌توجهی pause شود. در این فاصله، بقیهٔ system به کار خود ادامه می‌دهد و حتی ممکن است node متوقف‌شده را به‌دلیل پاسخ ندادن dead اعلام کند. سرانجام node ممکن است اجرای خود را از سر بگیرد، بی‌آنکه بداند خواب بوده است؛ تا وقتی که کمی بعد clock خود را بررسی کند.

#### Response time guarantees

همان‌طور که گفتیم، در بسیاری از زبان‌های برنامه‌نویسی و operating systemها، threadها و processها ممکن است برای مدت نامحدودی pause شوند. اگر به‌اندازهٔ کافی تلاش کنید، می‌توان این علت‌های pause را حذف کرد.

بعضی softwareها در محیطی اجرا می‌شوند که پاسخ ندادن در زمان مشخص می‌تواند آسیب جدی ایجاد کند: کامپیوترهایی که هواپیما، موشک، robot، خودرو و اشیای فیزیکی دیگر را کنترل می‌کنند باید به ورودی sensorهایشان سریع و قابل‌پیش‌بینی پاسخ دهند. در این systemها deadline مشخصی برای پاسخ software وجود دارد؛ اگر deadline رعایت نشود، ممکن است کل system fail شود. به این systemها **hard real-time system** گفته می‌شود.

> در embedded systemها، real-time یعنی system با دقت طراحی و test شده تا در تمام شرایط timing guaranteeهای مشخص را رعایت کند. این معنا با کاربرد مبهم‌تر اصطلاح real-time در وب متفاوت است؛ جایی که real-time به push کردن data از server به client و stream processing، بدون محدودیت سخت برای response time، اشاره می‌کند؛ به Chapter 11 مراجعه کنید.

برای مثال، اگر sensorهای onboard خودرو تشخیص دهند که تصادفی در حال رخ دادن است، نمی‌خواهید باز شدن airbag به‌دلیل GC pause نامناسب در system کنترل airbag به تأخیر بیفتد.

ارائهٔ real-time guarantee در یک system به پشتیبانی در تمام لایه‌های software stack نیاز دارد: باید از real-time operating system (RTOS) استفاده شود که processها را با allocation تضمین‌شدهٔ CPU time در بازه‌های مشخص schedule کند؛ library functionها باید worst-case execution time خود را مستند کنند؛ dynamic memory allocation ممکن است محدود یا کاملاً ممنوع شود—real-time garbage collector هم وجود دارد، اما application باید همچنان مراقب باشد کار بیش از حدی به GC نسپارد—و برای اطمینان از رعایت guaranteeها باید مقدار بسیار زیادی test و measurement انجام شود.

همهٔ این موارد کار اضافی زیادی می‌خواهند و دامنهٔ زبان‌های برنامه‌نویسی، libraryها و toolهای قابل‌استفاده را به‌شدت محدود می‌کنند، چون بیشتر آن‌ها real-time guarantee ندارند. به همین دلیل، توسعهٔ real-time system بسیار پرهزینه است و معمولاً در embedded deviceهای safety-critical به‌کار می‌رود. علاوه بر این، real-time با high-performance یکسان نیست؛ در واقع ممکن است throughput در real-time system کمتر باشد، چون اولویت اصلی پاسخ به‌موقع است؛ به بخش «Latency and Resource Utilization» در صفحهٔ ۲۸۶ نیز مراجعه کنید.

برای بیشتر data processing systemهای سمت server، real-time guarantee از نظر اقتصادی مقرون‌به‌صرفه یا مناسب نیست. در نتیجه، این systemها باید pauseها و ناپایداری clock را که از اجرا در محیط non-real-time ناشی می‌شوند تحمل کنند.

#### کم کردن اثر garbage collection

می‌توان اثر منفی process pauseها را بدون استفاده از guaranteeهای گران‌قیمت real-time scheduling کاهش داد. Runtimeهای زبان در زمان‌بندی GC تا حدی انعطاف دارند، چون می‌توانند rate ساخت object و مقدار memory آزاد باقی‌مانده را در گذر زمان پیگیری کنند.

ایده‌ای که در حال رواج است، برخورد با GC pause مانند outage برنامه‌ریزی‌شده و کوتاه یک node است: هنگام جمع‌آوری garbage در یک node، nodeهای دیگر requestهای clientها را می‌گیرند. اگر runtime بتواند به application هشدار دهد که node به‌زودی به GC pause نیاز دارد، application می‌تواند ارسال request جدید به آن node را متوقف کند، منتظر بماند requestهای در حال اجرا تمام شوند و سپس وقتی requestی در جریان نیست GC را انجام دهد. این روش GC pause را از دید client پنهان می‌کند و percentileهای بالای response time را کاهش می‌دهد [70, 71]. بعضی systemهای مالی حساس به latency از این رویکرد استفاده می‌کنند [72].

گونه‌ای از این ایده آن است که garbage collector فقط برای objectهای کوتاه‌عمر استفاده شود؛ جمع‌آوری آن‌ها سریع است. همچنین می‌توان processها را به‌صورت دوره‌ای، پیش از آنکه objectهای بلندعمر کافی انباشته شوند و full GC لازم شود، restart کرد [65, 73]. هر بار یک node restart می‌شود و پیش از restart، traffic از آن node دور می‌شود؛ مشابه rolling upgrade که در Chapter 4 دیدیم.

این اقدام‌ها نمی‌توانند GC pause را کاملاً حذف کنند، اما می‌توانند اثر آن را بر application به‌طور مفیدی کاهش دهند.

## Key Terms

- `Monotonic Clock` — clockی که فقط رو به جلو حرکت می‌کند و برای اندازه‌گیری durationهایی مانند timeout مناسب است.
- `Time-of-Day Clock` — clockی که تاریخ و ساعت فعلی را طبق یک تقویم گزارش می‌کند و ممکن است با اصلاح NTP به عقب بپرد.
- `Wall-Clock Time` — زمان تقویمی قابل‌مشاهده برای انسان که time-of-day clock ارائه می‌دهد.
- `Clock Drift` — جلو یا عقب افتادن تدریجی clock محلی نسبت به time مرجع، معمولاً بر اثر نادقیقی oscillator.
- `Clock Skew` — اختلاف مقدار time گزارش‌شده میان clockهای دو machine.
- `Clock Synchronization` — تنظیم clock machineها با منبع time مشترک برای کم کردن اختلاف آن‌ها.
- `Accuracy` — نزدیکی clock reading به time واقعی یا مرجع.
- `Precision` — ریزبودن resolution اندازه‌گیری؛ precision بالا به‌تنهایی accuracy را تضمین نمی‌کند.
- `Network Time Protocol (NTP)` — protocolی برای synchronize کردن clock کامپیوترها با serverهای time.
- `Timestamp` — مقدار time الصاق‌شده به event یا record؛ ممکن است به‌دلیل clock skew ترتیب واقعی eventها را نشان ندهد.
- `Event Ordering` — تعیین ترتیب رخداد eventها؛ timestamp فیزیکی به‌تنهایی ترتیب درست eventهای distributed را تضمین نمی‌کند.
- `Logical Clock` — counter یا mechanismی برای تعیین ترتیب نسبی eventها بدون اندازه‌گیری time تقویمی.
- `Causality` — رابطه‌ای که نشان می‌دهد یک event پیش‌نیاز یا علت event دیگری بوده است؛ ترتیب timestamp به‌تنهایی همیشه آن را حفظ نمی‌کند.
- `Last Write Wins (LWW)` — conflict resolutionی که write دارای timestamp ظاهراً جدیدتر را نگه می‌دارد؛ clock skew ممکن است باعث حذف write معتبر شود.
- `Confidence Interval` — بازه‌ای که time واقعی با سطح اطمینان مشخصی درون آن قرار دارد.
- `TrueTime API` — API گوگل در Spanner که time جاری را به‌صورت بازهٔ `[earliest, latest]` همراه uncertainty گزارش می‌کند.
- `Lease` — مجوزی زمان‌دار که تا پیش از expiry، حق انجام نقش یا دسترسی مشخصی را به یک node می‌دهد.
- `Process Pause` — توقف موقت اجرای process یا thread؛ بقیهٔ distributed system در این فاصله به کار ادامه می‌دهد.
- `Garbage Collection (GC)` — reclaim کردن memory اشغال‌شده توسط objectهایی که دیگر استفاده نمی‌شوند؛ بعضی GCها اجرای threadها را موقتاً متوقف می‌کنند.
- `Stop-the-World Pause` — GC pauseای که طی آن تمام threadهای application متوقف می‌شوند.
- `Hard Real-Time System` — systemی که باید در تمام شرایط deadlineهای زمانی مشخص را رعایت کند و برای این guarantee در کل software stack طراحی شده است.
- `Clock Uncertainty` — میزان خطای ممکن در clock reading که به‌صورت bound یا confidence interval بیان می‌شود.

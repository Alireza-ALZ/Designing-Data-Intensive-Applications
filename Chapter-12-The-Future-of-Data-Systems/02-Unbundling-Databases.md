# Chapter 12 — The Future of Data Systems

## Unbundling Databases

در انتزاعی‌ترین سطح، databaseها، Hadoop و operating systemها کارکردی یکسان دارند: data را ذخیره می‌کنند و امکان پردازش و query کردن آن را می‌دهند. database داده را در قالب recordهای یک data model ذخیره می‌کند (مثل row در table، document یا vertex در graph)، درحالی‌که filesystem یک operating system داده را در file نگه می‌دارد؛ اما در هسته، هر دو systemهای «مدیریت اطلاعات» هستند. همان‌طور که در Chapter 10 دیدیم، اکوسیستم Hadoop تا حدی شبیه نسخهٔ distributedای از Unix است. البته تفاوت‌های عملی زیادی وجود دارد: برای نمونه، بسیاری از filesystemها با directory دارای ۱۰ میلیون file کوچک خوب کنار نمی‌آیند، درحالی‌که databaseای با ۱۰ میلیون record کوچک کاملاً عادی است. بااین‌حال، بررسی شباهت‌ها و تفاوت‌های operating systemها و databaseها ارزشمند است.

Unix و relational databaseها مسئلهٔ مدیریت اطلاعات را با فلسفه‌هایی بسیار متفاوت حل کرده‌اند. هدف Unix ارائهٔ یک hardware abstraction منطقی، اما نسبتاً low-level، به programmerها بود؛ در مقابل، relational databaseها می‌خواستند به application programmerها abstractionای high-level بدهند که پیچیدگی‌های data structureهای روی disk، concurrency، crash recovery و مانند آن را پنهان کند. Unix، pipe و fileهایی ساخت که صرفاً دنباله‌ای از byteها هستند؛ databaseها هم SQL و transactionها را توسعه دادند.

کدام رویکرد بهتر است؟ به هدف شما بستگی دارد. Unix از این نظر «ساده‌تر» است که لایه‌ای نسبتاً نازک روی resourceهای سخت‌افزاری می‌کشد. relational database از این نظر «ساده‌تر» است که یک declarative query کوتاه می‌تواند از زیرساخت قدرتمندی مانند query optimization، index، روش‌های join، concurrency control و replication استفاده کند، بی‌آنکه نویسندهٔ query لازم باشد جزئیات پیاده‌سازی آن‌ها را بداند.

تنش میان این دو فلسفه چند دهه ادامه داشته است (Unix و relational model هر دو در اوایل دههٔ ۱۹۷۰ شکل گرفتند) و هنوز هم حل نشده است. برای مثال، من جنبش NoSQL را تلاشی برای به‌کارگیری رویکردی شبیه Unix ــ یعنی abstractionهای low-level ــ در حوزهٔ distributed OLTP data storage می‌دانم.

در این بخش می‌کوشم این دو فلسفه را با هم آشتی دهم تا بتوانیم بهترین ویژگی‌های هر دو را ترکیب کنیم.

### Composing Data Storage Technologies

در طول این کتاب دربارهٔ قابلیت‌های مختلف databaseها و نحوهٔ کار آن‌ها صحبت کردیم، از جمله:

- Secondary indexها که جست‌وجوی کارآمد recordها را بر اساس مقدار یک field ممکن می‌کنند.
- Materialized viewها که نوعی cache از پیش محاسبه‌شدهٔ نتیجهٔ query هستند.
- Replication logها که copyهای data روی nodeهای دیگر را به‌روز نگه می‌دارند.
- Full-text search indexها که جست‌وجوی keyword در متن را ممکن می‌کنند و در برخی relational databaseها نیز وجود دارند.

در Chapter 10 و 11 نیز مضمون‌های مشابهی مطرح شد: ساخت full-text search index، نگهداری materialized view و replicate کردن changeهای database به derived data systemها. به‌نظر می‌رسد قابلیت‌های تعبیه‌شده در databaseها با derived data systemهایی که افراد با batch و stream processor می‌سازند، شباهت‌هایی دارند.

#### ساختن یک index

وقتی در یک relational database دستور `CREATE INDEX` را اجرا می‌کنید، database باید یک consistent snapshot از table را scan کند، مقدار fieldهای موردنظر برای index را بیرون بکشد، آن‌ها را sort کند و index را بنویسد. سپس باید writeهایی را که از زمان گرفتن snapshot انجام شده‌اند پردازش کند (با این فرض که هنگام ساخت index، table را lock نکرده‌ایم و writeها ادامه داشته‌اند). پس از آن، database باید هر بار که transactionای در table می‌نویسد، index را نیز به‌روز نگه دارد.

این فرایند شباهت چشمگیری به راه‌اندازی یک follower replica جدید دارد و همچنین شبیه bootstrap کردن Change Data Capture در یک streaming system است. هر بار که `CREATE INDEX` اجرا می‌کنید، database در عمل dataset موجود را دوباره پردازش می‌کند و index را به‌عنوان viewای تازه از data موجود مشتق می‌سازد. data موجود ممکن است snapshotای از state باشد، نه log همهٔ changeهایی که تاکنون رخ داده‌اند؛ بااین‌حال، این دو مفهوم ارتباط نزدیکی دارند.

#### meta-database همه‌چیز

از این زاویه، dataflow در سراسر یک سازمان شبیه یک database عظیم به نظر می‌رسد. هرگاه یک batch، stream یا ETL process داده را از جایی و شکلی به جای دیگر و شکلی منتقل می‌کند، مانند database subsystemای عمل می‌کند که indexها یا materialized viewها را به‌روز نگه می‌دارد.

با این نگاه، batch و stream processorها پیاده‌سازی‌های مفصلی از triggerها، stored procedureها و routineهای نگهداری materialized view هستند. derived data systemهایی که نگهداری می‌کنند، مانند نوع‌های مختلف index عمل می‌کنند. برای مثال، یک relational database ممکن است B-tree index، hash index، spatial index و انواع دیگری از index داشته باشد. در معماری نوظهور derived data systemها، این قابلیت‌ها به‌جای آنکه ویژگی‌های یک محصول database یکپارچه باشند، توسط نرم‌افزارهای مختلفی فراهم می‌شوند که روی machineهای متفاوت اجرا و توسط teamهای جداگانه اداره می‌شوند.

این تحول در آینده به کجا می‌رسد؟ اگر بپذیریم که هیچ data model یا storage format واحدی برای همهٔ access patternها مناسب نیست، به‌گمانم دو راه وجود دارد تا ابزارهای متفاوت storage و processing را به‌شکل یک system منسجم کنار هم بگذاریم:

**Federated databaseها: یکپارچه‌کردن readها**

می‌توان یک query interface یکپارچه برای مجموعه‌ای متنوع از storage engineها و روش‌های processing فراهم کرد؛ رویکردی که federated database یا polystore نامیده می‌شود. برای مثال، قابلیت foreign data wrapper در PostgreSQL با این الگو سازگار است. applicationهایی که به data model یا query interface تخصصی نیاز دارند، همچنان می‌توانند مستقیماً به storage engine زیربنایی دسترسی پیدا کنند؛ درعین‌حال، کاربرانی که می‌خواهند dataهای پراکنده در جاهای مختلف را ترکیب کنند، این کار را از طریق interface فدره‌شده به‌آسانی انجام می‌دهند.

یک federated query interface از سنت relational پیروی می‌کند: یک system یکپارچه با query language سطح‌بالا و semantics روشن؛ اما پیاده‌سازی آن پیچیده است.

**Unbundled databaseها: یکپارچه‌کردن writeها**

Federation مسئلهٔ query فقط‌خواندنی روی چند system را حل می‌کند، اما پاسخ خوبی برای همگام‌سازی writeها میان آن‌ها ندارد. گفتیم که در یک database، ساخت index سازگار قابلیتی داخلی است. وقتی چند storage system را کنار هم می‌گذاریم، باید مطمئن شویم همهٔ data changeها، حتی هنگام بروز fault، به همهٔ مقصدهای درست می‌رسند. ساده‌تر کردن اتصال قابل‌اعتماد storage systemها ــ مثلاً با Change Data Capture و event log ــ مانند بازکردن قابلیت نگهداری index در database است، به‌گونه‌ای که writeها میان فناوری‌های متفاوت همگام شوند.

رویکرد unbundled از سنت Unix پیروی می‌کند: ابزارهای کوچک که هرکدام یک کار را خوب انجام می‌دهند، از طریق APIای low-level و یکنواخت (pipeها) با هم ارتباط می‌گیرند و با زبانی high-levelتر (shell) ترکیب می‌شوند.

#### عملی‌کردن unbundling

Federation و unbundling دو روی یک سکه‌اند: ساخت systemای قابل‌اعتماد، مقیاس‌پذیر و قابل‌نگهداری از componentهای متنوع. query فدره‌شدهٔ read-only مستلزم نگاشت یک data model به data model دیگر است؛ این کار به فکر نیاز دارد، اما در نهایت مسئله‌ای قابل‌مدیریت است. به‌نظر من، همگام نگه‌داشتن writeها در چند storage system مسئلهٔ مهندسی دشوارتری است، بنابراین روی آن تمرکز می‌کنم.

رویکرد سنتی همگام‌سازی writeها، استفاده از distributed transaction میان storage systemهای ناهمگون است؛ به‌نظر من این راه‌حل مناسبی نیست. transaction درون یک storage system یا stream processing system شدنی است؛ اما وقتی data از مرز فناوری‌های مختلف عبور می‌کند، asynchronous event log همراه با writeهای idempotent رویکردی بسیار robustتر و عملی‌تر است.

برای نمونه، بعضی stream processorها برای رسیدن به exactly-once semantics از distributed transaction استفاده می‌کنند و این روش می‌تواند خوب کار کند. اما وقتی یک transaction باید systemهایی را درگیر کند که گروه‌های مختلفی ساخته‌اند ــ مثلاً نوشتن stream processor در distributed key-value store یا search index ــ نبود protocol استاندارد transaction، integration را بسیار دشوار می‌کند. log مرتب‌شده‌ای از eventها با consumerهای idempotent، abstractionای ساده‌تر است و پیاده‌سازی آن میان systemهای ناهمگون عملی‌تر خواهد بود.

مزیت اصلی integration مبتنی بر log، coupling سست میان componentهاست که دو بُعد دارد:

1. **در سطح system:** asynchronous event streamها کل system را در برابر outage یا افت performance در componentهای منفرد مقاوم‌تر می‌کنند. اگر consumerای کند شود یا از کار بیفتد، event log می‌تواند پیام‌ها را buffer کند؛ در نتیجه producer و consumerهای دیگر بدون تأثیر به کار ادامه می‌دهند. consumer معیوب پس از رفع مشکل می‌تواند عقب‌ماندگی‌اش را جبران کند و هیچ dataای را از دست ندهد؛ fault نیز در همان component محدود می‌ماند. در مقابل، تعامل synchronous در distributed transactionها معمولاً fault محلی را به failureای گسترده تبدیل می‌کند.
2. **در سطح انسانی:** unbundling data systemها اجازه می‌دهد componentها و serviceهای نرم‌افزاری را teamهای مختلف مستقل از یکدیگر توسعه دهند، بهبود دهند و نگهداری کنند. تخصص‌گرایی باعث می‌شود هر team روی انجام درست یک کار تمرکز کند و با systemهای teamهای دیگر interfaceهای مشخصی داشته باشد. event log interfaceای به‌اندازهٔ کافی قدرتمند برای بیان consistency نسبتاً قوی (به‌واسطهٔ durability و ترتیب eventها) و درعین‌حال به‌اندازهٔ کافی عمومی برای تقریباً هر نوع data فراهم می‌کند.

#### systemهای unbundled در برابر systemهای یکپارچه

اگر unbundling واقعاً به رویکرد آینده تبدیل شود، databaseها را با شکل فعلی‌شان جایگزین نمی‌کند؛ همچنان به همان اندازه به آن‌ها نیاز خواهیم داشت. برای نگهداری state در stream processorها و نیز پاسخ‌گویی به queryهای خروجی batch و stream processorها، database لازم است. query engineهای تخصصی نیز برای workloadهای خاص مهم می‌مانند؛ برای مثال، query engineهای MPP data warehouse برای queryهای تحلیلی اکتشافی بهینه شده‌اند و این workload را بسیار خوب مدیریت می‌کنند.

اجرای چند زیرساخت متفاوت می‌تواند پیچیدگی‌ساز باشد: هر نرم‌افزار منحنی یادگیری، مسئله‌های configuration و رفتارهای عملیاتی خاص خود را دارد؛ پس بهتر است تا حد امکان componentهای متحرک کمتری deploy کنیم. یک محصول نرم‌افزاری یکپارچه ممکن است برای workloadهایی که به‌خاطرشان طراحی شده، performance بهتر و قابل‌پیش‌بینی‌تری از systemی داشته باشد که با application code چند ابزار را به هم وصل کرده است. همان‌طور که در Preface گفتم، ساختن برای scaleای که نیاز ندارید اتلاف تلاش است و ممکن است شما را در طراحی‌ای انعطاف‌ناپذیر گرفتار کند؛ در عمل، این نوعی premature optimization است.

هدف unbundling رقابت با databaseهای منفرد بر سر performance در workloadهای خاص نیست؛ هدف آن ترکیب چند database برای دستیابی به performance مناسب در دامنهٔ بسیار وسیع‌تری از workloadهاست، دامنه‌ای که با یک نرم‌افزار واحد ممکن نیست. موضوع، گستردگی قابلیت‌هاست نه عمق بهینه‌سازی؛ مشابه تنوع modelهای storage و processing که در «مقایسهٔ Hadoop با distributed databaseها» بررسی کردیم.

بنابراین، اگر فناوری واحدی همهٔ نیازهای شما را برآورده می‌کند، احتمالاً بهتر است همان محصول را به کار ببرید تا اینکه بخواهید آن را از componentهای low-level خودتان دوباره پیاده‌سازی کنید. مزایای unbundling و composition فقط وقتی مطرح می‌شوند که هیچ نرم‌افزار واحدی تمام نیازها را برآورده نکند.

#### چه چیزی کم است؟

ابزارهای ترکیب data systemها بهتر شده‌اند، اما به‌نظر من یک جزء اساسی هنوز وجود ندارد: معادل shell در دنیای unbundled databaseها؛ یعنی زبانی high-level برای ترکیب ساده و declarative سیستم‌های storage و processing.

برای نمونه، دوست دارم بتوانیم با قیاس از Unix pipeها، فقط بنویسیم `mysql | elasticsearch`. این کار معادل unbundled دستور `CREATE INDEX` خواهد بود: همهٔ documentهای database در MySQL را می‌گیرد و در یک Elasticsearch cluster index می‌کند. سپس پیوسته changeهای database را capture می‌کند و خودکار روی search index اعمال می‌کند، بی‌آنکه لازم باشد application code سفارشی بنویسیم. چنین integrationای باید تقریباً برای هر نوع storage یا indexing system ممکن باشد.

به همین ترتیب، بهتر است بتوان cacheها را آسان‌تر از پیش محاسبه و به‌روز کرد. Materialized view در اصل cacheای ازپیش‌محاسبه‌شده است؛ پس می‌توان تصور کرد که cache را با تعریف declarative materialized viewهایی برای queryهای پیچیده ــ از جمله queryهای recursive روی graph و application logic ــ ایجاد کنیم. پژوهش‌های اولیهٔ جالبی در این حوزه، مانند differential dataflow، در جریان است و امیدوارم این ایده‌ها به systemهای production راه پیدا کنند.

### Designing Applications Around Dataflow

رویکرد unbundling databaseها از طریق ترکیب storage و processing systemهای تخصصی با application code، با عنوان رویکرد «database inside-out» نیز شناخته می‌شود؛ نامی برگرفته از عنوان سخنرانی‌ای که در سال ۲۰۱۴ ارائه کردم. بااین‌حال، نامیدن آن به‌عنوان «معماری جدید» اغراق‌آمیز است. من آن را بیشتر یک design pattern و نقطهٔ شروعی برای گفت‌وگو می‌دانم؛ صرفاً برای آن نامی گذاشته‌ایم تا راحت‌تر درباره‌اش صحبت کنیم.

این ایده‌ها از من نیستند؛ گردآوری ایده‌های دیگران‌اند که فکر می‌کنم باید از آن‌ها بیاموزیم. به‌ویژه، با dataflow languageهایی مانند Oz و Juttle، زبان‌های functional reactive programming (FRP) مانند Elm و زبان‌های logic programming مانند Bloom هم‌پوشانی زیادی دارند. اصطلاح unbundling در این معنا را Jay Kreps پیشنهاد کرد.

حتی spreadsheetها قابلیت‌های dataflow programming دارند که از قابلیت‌های بیشتر زبان‌های برنامه‌نویسی رایج بسیار جلوتر است. در spreadsheet می‌توانید در یک cell فرمولی بگذارید (مثلاً جمع cellهای یک ستون دیگر) و هر بار که یکی از ورودی‌های فرمول تغییر کند، نتیجه خودکار دوباره محاسبه می‌شود. در سطح data system هم دقیقاً چنین چیزی می‌خواهیم: وقتی recordای در database تغییر کرد، index مربوط به آن خودکار به‌روز شود و cache viewها یا aggregationهای وابسته به آن نیز تازه شوند. نباید نگران جزئیات فنی refresh باشید؛ باید بتوانید به درست کار کردن آن اعتماد کنید.

به‌نظر من، بیشتر data systemها هنوز می‌توانند از قابلیت‌هایی بیاموزند که VisiCalc در سال ۱۹۷۹ داشت. تفاوت این است که data systemهای امروزی باید fault-tolerant و مقیاس‌پذیر باشند و data را به‌شکل پایدار ذخیره کنند. همچنین باید بتوانند در گذر زمان فناوری‌های متفاوتی را که گروه‌های مختلف ساخته‌اند یکپارچه کنند و libraryها و serviceهای موجود را دوباره به‌کار بگیرند؛ واقع‌بینانه نیست انتظار داشته باشیم همهٔ نرم‌افزارها با یک زبان، framework یا ابزار مشخص ساخته شوند.

در ادامه این ایده‌ها را گسترش می‌دهم و راه‌هایی برای ساخت application بر پایهٔ ایدهٔ unbundled databaseها و dataflow بررسی می‌کنم.

#### application code به‌عنوان تابع مشتق‌سازی

وقتی یک dataset از dataset دیگری مشتق می‌شود، نوعی transformation function روی آن اعمال شده است. برای نمونه:

- Secondary index نوعی derived dataset با transformation function ساده است: برای هر row یا document در base table، مقدار columnها یا fieldهای indexشده را برمی‌دارد و بر اساس آن مقادیر sort می‌کند (با فرض استفاده از B-tree یا SSTable index که بر اساس key مرتب‌اند).
- Full-text search index با اجرای چندین تابع natural language processing ساخته می‌شود؛ از جمله تشخیص زبان، segment کردن واژه‌ها، stemming یا lemmatization، اصلاح spelling و شناسایی synonymها؛ سپس data structureای مانند inverted index برای lookup کارآمد ساخته می‌شود.
- در machine learning می‌توان model را چیزی دانست که با اعمال تابع‌های feature extraction و تحلیل آماری به training data مشتق شده است. وقتی model روی input data جدید اجرا می‌شود، output آن از input و model (و در نتیجه به‌طور غیرمستقیم از training data) مشتق می‌شود.
- Cache اغلب aggregationای از data را نگه می‌دارد، در شکلی که قرار است در user interface (UI) نمایش داده شود. بنابراین پرکردن cache مستلزم دانستن fieldهایی است که UI به آن‌ها ارجاع می‌دهد؛ تغییر UI ممکن است تعریف نحوهٔ پرشدن cache و بازسازی آن را لازم کند.

تابع مشتق‌سازی برای secondary index آن‌قدر رایج است که در بسیاری از databaseها قابلیت اصلی محسوب می‌شود و با دستور سادهٔ `CREATE INDEX` آن را فراخوانی می‌کنید. ممکن است قابلیت‌های زبانی پایه برای زبان‌های رایج در full-text indexing یک database تعبیه شده باشد؛ اما قابلیت‌های پیشرفته‌تر اغلب به tuning خاص domain نیاز دارند. در machine learning نیز feature engineering به‌شدت وابسته به application است و معمولاً باید دانش دقیقی از تعامل کاربر و نحوهٔ deployment application در آن لحاظ شود.

وقتی تابع ساخت derived dataset از نوع متداول و استانداردی مثل ساخت secondary index نیست، برای مدیریت جنبه‌های وابسته به application به custom code نیاز است. همین custom code جایی است که بسیاری از databaseها به مشکل برمی‌خورند. هرچند relational databaseها معمولاً از trigger، stored procedure و user-defined function پشتیبانی می‌کنند که اجرای application code درون database را ممکن می‌سازند، این قابلیت‌ها در طراحی database تا حدی در حاشیه بوده‌اند.

#### جداسازی application code از state

در تئوری، databaseها می‌توانند مانند operating system محیط اجرای application code دلخواه باشند؛ اما در عمل برای این کار چندان مناسب از آب درنیامده‌اند. آن‌ها با نیازهای توسعهٔ application مدرن، مانند dependency و package management، version control، rolling upgrade، evolvability، monitoring، metricها، فراخوانی network serviceها و integration با systemهای بیرونی، سازگاری خوبی ندارند.

در مقابل، ابزارهای deployment و مدیریت cluster مانند Mesos، YARN، Docker، Kubernetes و ابزارهای دیگر مشخصاً برای اجرای application code طراحی شده‌اند. چون روی یک کار متمرکزند، آن را بسیار بهتر از databaseای انجام می‌دهند که اجرای user-defined function فقط یکی از قابلیت‌های متعددش است.

به‌نظرم منطقی است بخش‌هایی از system را به storage پایدار data اختصاص دهیم و بخش‌های دیگری را به اجرای application code؛ این دو می‌توانند با هم تعامل داشته باشند و درعین‌حال مستقل بمانند.

امروزه بیشتر web applicationها به شکل serviceهای stateless deploy می‌شوند؛ هر user request را می‌توان به هر application serverای فرستاد و server پس از فرستادن response، چیزی از آن request به خاطر نمی‌سپارد. این شیوهٔ deployment راحت است، چون می‌توان serverها را هر زمان اضافه یا حذف کرد؛ اما state باید جایی نگهداری شود، که معمولاً database است. روند غالب این بوده که application logic بدون state را از مدیریت state (databaseها) جدا نگه داریم: نه application logic را در database بگذاریم و نه persistent state را در application. (اهل functional programming گاهی شوخی می‌کنند که «ما به جدایی Church از state باور داریم».)

در این مدل رایج web application، database مثل shared variableای mutable عمل می‌کند که به‌شکل synchronous از طریق network قابل‌دسترسی است. application می‌تواند آن variable را بخواند و به‌روز کند و database هم مسئول پایدار کردن آن، فراهم‌کردن مقداری concurrency control و تحمل fault است.

بااین‌حال، در بیشتر زبان‌های برنامه‌نویسی نمی‌توانید در برابر تغییرهای یک mutable variable subscribe کنید؛ فقط می‌توانید آن را دوره‌ای بخوانید. برخلاف spreadsheet، خواننده‌های variable از تغییر مقدار آن باخبر نمی‌شوند. (می‌توانید چنین notificationهایی را خودتان پیاده‌سازی کنید؛ این الگو observer pattern نام دارد، اما در بیشتر زبان‌ها قابلیت داخلی نیست.)

اشارهٔ شوخی به Church، به ریاضی‌دان Alonzo Church برمی‌گردد که lambda calculus را ابداع کرد؛ شکلی اولیه از محاسبه که مبنای بیشتر زبان‌های functional programming است. lambda calculus state mutable ندارد؛ یعنی variableای در آن نیست که بتوان مقدارش را بازنویسی کرد.

databaseها همین رویکرد منفعلانه نسبت به data mutable را به ارث برده‌اند: اگر بخواهید بفهمید محتوای database تغییر کرده یا نه، اغلب تنها راهتان polling است؛ یعنی query را مرتب تکرار کنید. قابلیت subscribe شدن به changeها تازه در حال ظهور است.

#### Dataflow: تعامل میان changeهای state و application code

نگاه کردن به application از دید dataflow یعنی باید رابطهٔ application code و مدیریت state را از نو تعریف کنیم. به‌جای آنکه database را variableای منفعل بدانیم که application آن را دست‌کاری می‌کند، به تعامل و همکاری میان state، changeهای state و codeای توجه می‌کنیم که آن changeها را پردازش می‌کند. application code به changeهای state در یک بخش پاسخ می‌دهد و با این کار changeهای state را در بخش دیگری به جریان می‌اندازد.

این طرز فکر را در «Databaseها و streamها» دیدیم؛ جایی که log تغییرهای database را streamای از eventها در نظر گرفتیم که می‌توان در آن subscribe شد. systemهای message-passing مانند actorها نیز بر پاسخ‌دادن به eventها تکیه دارند. مدل tuple space در دههٔ ۱۹۸۰ هم محاسبات distributed را با processهایی بیان می‌کرد که تغییرهای state را مشاهده و به آن‌ها واکنش نشان می‌دادند.

همان‌طور که گفته شد، وقتی triggerای در اثر تغییر data اجرا می‌شود یا secondary index برای بازتاب تغییر table به‌روز می‌شود، اتفاق مشابهی درون database رخ می‌دهد. unbundling database یعنی این ایده را به ساخت derived datasetها در بیرون از primary database گسترش دهیم: cache، full-text search index، machine learning یا analytics system. برای این منظور می‌توان از stream processing و messaging systemها استفاده کرد.

نکتهٔ مهم این است که نگهداری derived data با اجرای asynchronous job یکسان نیست؛ messaging systemها به‌طور سنتی برای اجرای job طراحی شده‌اند، نه لزوماً برای این کار:

- هنگام نگهداری derived data، ترتیب changeهای state اغلب اهمیت دارد. اگر چند view از یک event log مشتق شوند، باید eventها را به ترتیب یکسان پردازش کنند تا سازگاری‌شان با یکدیگر حفظ شود. بسیاری از message brokerها هنگام تحویل دوبارهٔ پیام‌های تأییدنشده این ویژگی را ندارند. dual write نیز راه‌حل مناسبی نیست.
- Fault tolerance برای derived data حیاتی است: از دست رفتن حتی یک پیام باعث می‌شود derived dataset برای همیشه از source data ناهماهنگ شود. هم تحویل پیام و هم به‌روزرسانی derived state باید قابل‌اعتماد باشند. برای مثال، بسیاری از actor systemها به‌طور پیش‌فرض state actor و پیام‌ها را در memory نگه می‌دارند؛ پس با crash کردن machine میزبان actor، هر دو از دست می‌روند.

ترتیب پایدار پیام‌ها و پردازش fault-tolerant آن‌ها الزام‌های سخت‌گیرانه‌ای هستند؛ بااین‌حال، هزینه‌شان بسیار کمتر از distributed transaction است و از نظر عملیاتی نیز robustترند. stream processorهای مدرن می‌توانند این guaranteeهای ترتیب و reliability را در scale فراهم کنند و application code را به‌صورت stream operator اجرا کنند.

این application code می‌تواند هر نوع processing دلخواهی انجام دهد؛ چیزی که تابع‌های مشتق‌سازی داخلی databaseها عموماً فراهم نمی‌کنند. همان‌طور که toolهای Unix را با pipe به هم وصل می‌کنیم، می‌توان stream operatorها را برای ساخت systemهای بزرگ مبتنی بر dataflow ترکیب کرد. هر operator، streamهایی از changeهای state را به‌عنوان input می‌گیرد و streamهای دیگری از changeهای state را به‌عنوان output تولید می‌کند.

#### Stream processorها و serviceها

سبک رایج این روزهای توسعهٔ application، شکستن قابلیت‌ها به مجموعه‌ای از serviceهاست که از طریق network requestهای synchronous، مانند REST API، با هم ارتباط دارند. مزیت service-oriented architecture نسبت به application monolithic، عمدتاً مقیاس‌پذیری سازمانی ناشی از coupling سست است: teamهای مختلف می‌توانند روی serviceهای جداگانه کار کنند و هماهنگی میان آن‌ها کمتر شود؛ البته به‌شرط آنکه serviceها مستقل از هم deploy و update شوند.

ترکیب stream operatorها در dataflow system ویژگی‌های زیادی شبیه رویکرد microservice دارد؛ اما سازوکار ارتباط زیربنایی متفاوت است: به‌جای تعامل synchronous از نوع request/response، از message streamهای asynchronous و یک‌جهته استفاده می‌شود.

علاوه بر مزایایی مانند fault tolerance بهتر که در «Message-Passing Dataflow» گفتیم، dataflow systemها ممکن است performance بهتری هم داشته باشند. فرض کنید مشتری کالایی را می‌خرد که قیمتش با ارزی غیر از ارز پرداختی است. برای تبدیل ارز باید نرخ فعلی را بدانید. این عملیات را می‌توان به دو شکل پیاده‌سازی کرد:

1. در رویکرد microservice، code پردازش‌کنندهٔ خرید احتمالاً برای گرفتن نرخ جاری ارز موردنظر، از exchange-rate service یا database query می‌گیرد.
2. در رویکرد dataflow، code پردازش خرید از پیش در stream updateهای نرخ ارز subscribe می‌شود و هر بار که نرخ تغییر می‌کند، مقدار جاری را در database محلی ثبت می‌کند. هنگام پردازش خرید، کافی است database محلی را query کند.

رویکرد دوم، network request synchronous به service دیگری را با query از database محلی جایگزین کرده است؛ database حتی می‌تواند روی همان machine یا در همان process باشد. رویکرد dataflow نه‌تنها سریع‌تر است، بلکه در برابر failure service دیگر نیز مقاوم‌تر است. سریع‌ترین و قابل‌اعتمادترین network request، اصلاً network request نکردن است! به‌جای RPC، اکنون stream joinای میان eventهای خرید و eventهای update نرخ ارز داریم.

این join به زمان وابسته است: اگر eventهای خرید را بعداً دوباره پردازش کنیم، نرخ ارز تغییر کرده است. برای بازسازی output اصلی باید نرخ تاریخی ارز در زمان خرید را به دست آوریم. چه نرخ را از service query کنیم و چه در stream updateهای نرخ subscribe شویم، باید این وابستگی زمانی را مدیریت کنیم.

subscribe شدن به stream تغییرها، به‌جای query کردن state جاری در زمان نیاز، ما را به مدل محاسباتی شبیه spreadsheet نزدیک می‌کند: با تغییر یک داده، هر derived data وابسته به آن می‌تواند سریع به‌روز شود. هنوز پرسش‌های باز زیادی، مثلاً دربارهٔ joinهای وابسته به زمان، وجود دارد؛ اما به‌نظر من ساخت application بر پایهٔ ایده‌های dataflow مسیری بسیار امیدبخش است.

### Observing Derived State

در سطحی انتزاعی، dataflow systemهایی که در بخش قبل بررسی کردیم، فرایندی برای ساخت derived datasetهایی مانند search index، materialized view و predictive model و به‌روز نگه‌داشتن آن‌ها فراهم می‌کنند. این فرایند را **write path** بنامیم: هرگاه اطلاعاتی در system نوشته می‌شود، ممکن است از چند مرحلهٔ batch و stream processing عبور کند و در نهایت هر derived dataset به‌روزرسانی شود تا آن data را دربر بگیرد. Figure 12-1 نمونه‌ای از update کردن search index را نشان می‌دهد.

در search index، writeها (updateهای document) با readها (queryها) به هم می‌رسند.

اما اصلاً چرا derived dataset را می‌سازیم؟ به‌احتمال زیاد چون می‌خواهیم بعداً دوباره آن را query کنیم. این بخش را **read path** می‌نامیم: هنگام پاسخ به user request، از derived dataset می‌خوانیم، شاید نتیجه را کمی بیشتر پردازش می‌کنیم و response را برای user می‌سازیم.

write path و read path در کنار هم، تمام مسیر data را از زمان جمع‌آوری تا زمان مصرف (احتمالاً توسط انسان دیگری) دربر می‌گیرند. write path بخشی از مسیر است که از پیش محاسبه می‌شود؛ یعنی به‌محض ورود data و مستقل از اینکه کسی درخواست مشاهده‌اش را داده باشد، اجرا می‌شود. read path فقط زمانی طی می‌شود که کسی data را درخواست کند. اگر با functional programming languageها آشنا باشید، شاید متوجه شوید write path به eager evaluation و read path به lazy evaluation شباهت دارد.

derived dataset نقطهٔ تلاقی write path و read path است. این dataset میان میزان کاری که باید هنگام write انجام شود و میزان کاری که هنگام read لازم است، trade-off ایجاد می‌کند.

#### Materialized view و caching

Full-text search index نمونهٔ خوبی است: write path index را به‌روز می‌کند و read path برای keywordها در آن جست‌وجو می‌کند. هم read و هم write مقداری کار لازم دارند. write باید entryهای index را برای همهٔ واژه‌های موجود در document به‌روز کند. read باید هر واژهٔ query را جست‌وجو کند و منطق Boolean را به کار ببرد تا documentهایی را پیدا کند که همهٔ واژه‌های query (عملگر AND) یا synonym هر واژه (عملگر OR) را دارند.

اگر index نداشتیم، search query باید همهٔ documentها را scan می‌کرد (مثل `grep`) که با زیاد شدن تعداد documentها بسیار پرهزینه می‌شد. نبود index یعنی کار کمتر در write path (چون indexای برای update کردن نیست)، اما کار بسیار بیشتر در read path.

از سوی دیگر، می‌توان نتیجهٔ همهٔ queryهای ممکن را از پیش محاسبه کرد. در این صورت کار read path کمتر می‌شود: دیگر منطق Boolean لازم نیست و فقط نتیجهٔ query را پیدا و برمی‌گردانیم. اما write path بسیار پرهزینه خواهد شد: مجموعهٔ queryهای ممکن بی‌نهایت است و بنابراین پیش‌محاسبهٔ همهٔ نتایج، زمان و فضای ذخیره‌سازی نامتناهی می‌خواهد؛ این رویکرد چندان عملی نیست.

گزینهٔ دیگر، پیش‌محاسبهٔ نتیجهٔ مجموعهٔ ثابتی از queryهای رایج است تا بدون مراجعه به index سریع پاسخ داده شوند. queryهای غیررایج همچنان از index پاسخ می‌گیرند. معمولاً به این کار cache کردن queryهای رایج می‌گویند؛ اما می‌توان آن را materialized view هم دانست، چون با ظهور documentهای تازه‌ای که باید در نتیجهٔ یکی از queryهای رایج باشند، لازم است update شود.

از این مثال می‌بینیم index تنها مرز ممکن میان write path و read path نیست. cache کردن نتیجهٔ searchهای رایج ممکن است؛ برای تعداد کمی document حتی scan شبیه `grep` بدون index هم ممکن است. از این دید، نقش cache، index و materialized view ساده است: مرز میان read path و write path را جابه‌جا می‌کنند. با پیش‌محاسبهٔ نتیجه، کار بیشتری در write path انجام می‌دهیم تا زحمت read path کمتر شود.

در واقع، جابه‌جایی مرز کار میان write path و read path موضوع مثال Twitter در ابتدای کتاب بود؛ آنجا هم دیدیم ممکن است این مرز برای celebrityها و userهای عادی متفاوت ترسیم شود. پس از ۵۰۰ صفحه، دوباره به نقطهٔ آغاز برگشتیم!

#### clientهای stateful و قابل‌استفاده به‌صورت offline

ایدهٔ مرز میان write و read path را جالب می‌دانم، چون می‌توانیم دربارهٔ جابه‌جایی آن و پیامدهای عملی‌اش صحبت کنیم. بیایید این ایده را در زمینه‌ای دیگر بررسی کنیم.

محبوبیت عظیم web applicationها در دو دههٔ گذشته، به شکل‌گیری فرض‌هایی دربارهٔ توسعهٔ application انجامیده که به‌آسانی بدیهی به‌نظر می‌رسند. به‌ویژه، مدل client/server ــ که در آن clientها عمدتاً stateless هستند و serverها مرجع اصلی data به‌شمار می‌روند ــ آن‌قدر رایج است که تقریباً فراموش می‌کنیم روش دیگری هم وجود دارد. اما فناوری پیوسته تغییر می‌کند و گاهی مهم است که وضع موجود را زیر سؤال ببریم.

به‌طور سنتی، web browserها clientهای statelessی بوده‌اند که فقط با اتصال به اینترنت کار مفیدی انجام می‌دهند؛ تقریباً تنها کار offline این بود که در صفحه‌ای که قبلاً online بارگذاری کرده‌اید بالا و پایین بروید. بااین‌حال، web appهای JavaScript جدیدتر از نوع single-page قابلیت‌های stateful زیادی پیدا کرده‌اند؛ از جمله تعامل UI در سمت client و storage محلی پایدار در browser. mobile appها نیز می‌توانند state زیادی روی device نگه دارند و برای بیشتر تعامل‌های user نیازی به round-trip به server ندارند.

این قابلیت‌های جدید، علاقه به applicationهای offline-first را دوباره زنده کرده‌اند: application تا جای ممکن با database محلی روی همان device کار می‌کند و به اتصال اینترنت وابسته نیست؛ هرگاه اتصال شبکه برقرار باشد، در پس‌زمینه با serverهای remote همگام می‌شود. چون اتصال cellular اینترنت در mobile deviceها اغلب کند و غیرقابل‌اعتماد است، برای user مزیت بزرگی است که UI منتظر network requestهای synchronous نماند و application تا حد زیادی offline هم کار کند.

وقتی از فرض clientهای stateless متصل به database مرکزی فاصله می‌گیریم و به سمت نگهداری state روی deviceهای کاربر می‌رویم، فرصت‌های تازه‌ای پدید می‌آید. به‌ویژه می‌توان state روی device را cacheای از state روی server دانست. pixelهای روی صفحه materialized viewای از model objectها در client app هستند؛ model objectها نیز replica محلی state در remote datacenter به‌شمار می‌روند.

#### فرستادن changeهای state به clientها

در web page معمولی، اگر صفحه را در browser باز کنید و data بعداً روی server تغییر کند، browser تا زمانی که صفحه را دوباره load نکنید از آن تغییر باخبر نمی‌شود. browser فقط یک‌بار data را می‌خواند و فرض می‌کند ثابت می‌ماند؛ updateهای server را subscribe نمی‌کند. بنابراین state روی device cacheای stale است که جز با polling صریح به‌روز نمی‌شود. protocolهای subscription مبتنی بر HTTP مانند RSS در واقع شکل ابتدایی polling هستند.

protocolهای جدیدتر از الگوی پایهٔ request/response در HTTP فراتر رفته‌اند: server-sent eventها (API مربوط به `EventSource`) و WebSocketها channel ارتباطی‌ای فراهم می‌کنند که در آن web browser می‌تواند TCP connection بازی را با server حفظ کند و تا وقتی متصل است، server فعالانه messageها را به browser push کند. این قابلیت به server فرصت می‌دهد client کاربر نهایی را از تغییرهای state ذخیره‌شدهٔ محلی باخبر کند و stale بودن state سمت client را کاهش دهد.

در مدل write path و read path، push فعالانهٔ changeهای state تا deviceهای client یعنی write path را تا کاربر نهایی امتداد داده‌ایم. client هنگام initialization اولیه همچنان برای گرفتن state به read path نیاز دارد، اما پس از آن می‌تواند به stream تغییرهایی تکیه کند که server می‌فرستد. ایده‌های stream processing و messaging فقط محدود به اجرای درون datacenter نیستند؛ می‌توان آن‌ها را تا deviceهای کاربر نهایی گسترش داد.

deviceها بخشی از زمان offline هستند و در آن مدت نمی‌توانند notification تغییرهای state را از server دریافت کنند. اما این مسئله را قبلاً حل کرده‌ایم: consumer یک log-based message broker می‌تواند پس از failure یا قطع اتصال دوباره وصل شود و مطمئن شود پیام‌های رسیده در مدت قطعی را از دست نداده است. همین روش برای userهای منفرد نیز کار می‌کند؛ هر device subscriber کوچکی برای stream کوچکی از eventهاست.

#### End-to-end event streamها

ابزارهای جدید ساخت client و UIهای stateful، مانند Elm و مجموعه‌ابزار React، Flux و Redux شرکت Facebook، از پیش state داخلی client را با subscribe شدن به streamای از eventها مدیریت می‌کنند؛ eventهایی که ورودی user یا responseهای server را نشان می‌دهند و ساختاری شبیه event sourcing دارند.

گسترش این مدل برنامه‌نویسی برای آنکه server هم eventهای تغییر state را به pipeline رویدادهای سمت client push کند، کاملاً طبیعی است. در نتیجه، changeهای state می‌توانند در یک write path سرتاسری جریان پیدا کنند: از تعامل روی یک device که change را آغاز می‌کند، از میان event logها و چند derived data system و stream processor، تا UI فردی که state را روی device دیگری مشاهده می‌کند. این changeها می‌توانند با تأخیر نسبتاً کم ــ مثلاً کمتر از یک ثانیه از ابتدا تا انتها ــ منتشر شوند.

بعضی applicationها، مانند پیام‌رسانی فوری و بازی‌های online، همین حالا چنین معماری «real-time»ای دارند (منظور تعامل کم‌تأخیر است، نه «guaranteeهای response time»). پس چرا همهٔ applicationها را این‌طور نمی‌سازیم؟

چالش این است که فرض clientهای stateless و تعامل request/response عمیقاً در databaseها، libraryها، frameworkها و protocolهای ما ریشه دوانده است. datastoreهای زیادی از عملیات read و write پشتیبانی می‌کنند که در آن هر request یک response برمی‌گرداند؛ اما datastoreهای بسیار کمتری امکان subscribe به changeها را فراهم می‌کنند؛ یعنی requestای که در گذر زمان streamای از responseها برگرداند.

برای امتداد دادن write path تا کاربر نهایی، باید شیوهٔ ساخت بسیاری از این systemها را از پایه بازنگری کنیم: از تعامل request/response به سمت dataflow از نوع publish/subscribe برویم. به‌نظر من، مزیت UIهای پاسخ‌گوتر و پشتیبانی بهتر از offline ارزش این تلاش را دارد. اگر data system طراحی می‌کنید، امیدوارم گزینهٔ subscribe شدن به changeها را هم در نظر بگیرید، نه فقط query کردن state جاری را.

#### readها هم event هستند

گفتیم وقتی stream processor، derived data را در storeای (database، cache یا index) می‌نویسد و user requestها آن store را query می‌کنند، store مرز میان write path و read path است. store امکان read queryهای random-access را روی data فراهم می‌کند؛ queryهایی که در غیر این صورت مستلزم scan کردن کل event log بودند.

در بسیاری از موارد، storage data از streaming system جداست. اما stream processorها برای انجام aggregation و join نیز باید state نگه دارند. این state معمولاً درون stream processor پنهان است، هرچند بعضی frameworkها اجازه می‌دهند clientهای بیرونی آن را query کنند و خود stream processor به نوعی database ساده تبدیل شود.

می‌خواهم این ایده را گسترش دهم. تا اینجا writeها از event log عبور می‌کنند، درحالی‌که readها network requestهای گذرایی هستند که مستقیماً به nodeهای نگهدارندهٔ data مورد query می‌روند. این طراحی معقول است، اما تنها گزینه نیست. می‌توان read requestها را نیز به شکل streamای از eventها نمایش داد و read eventها و write eventها را هر دو از stream processor عبور داد؛ processor در پاسخ به read event، نتیجهٔ read را به output stream می‌فرستد.

وقتی هم write و هم read را به‌صورت event نمایش دهیم و برای پردازش به همان stream operator بفرستیم، در واقع stream-table joinای میان stream queryهای read و database انجام می‌دهیم. read event باید به partition databaseای فرستاده شود که data را نگه می‌دارد؛ درست مانند batch و stream processorهایی که هنگام join باید inputها را بر اساس key مشترک copartition کنند.

این تناظر میان پاسخ‌گویی به request و اجرای join بنیادی است. read request یک‌باره فقط از join operator عبور می‌کند و بلافاصله فراموش می‌شود؛ اما request از نوع subscribe، join پایداری است با eventهای گذشته و آینده در سوی دیگر join.

ثبت log از read eventها می‌تواند برای ردیابی dependencyهای علّی و data provenance در سراسر system هم مفید باشد: با آن می‌توان بازسازی کرد user پیش از تصمیم‌گیری چه چیزی دیده است. برای مثال، در فروشگاه online احتمال دارد تاریخ پیش‌بینی‌شدهٔ ارسال و وضعیت موجودی‌ای که به مشتری نشان داده می‌شود، بر خریدن یا نخریدن کالا اثر بگذارد. برای تحلیل این ارتباط، باید نتیجهٔ query کاربر دربارهٔ زمان ارسال و موجودی را ثبت کنیم.

نوشتن read eventها در storage پایدار، ردیابی dependencyهای علّی را بهتر می‌کند؛ اما هزینهٔ storage و I/O اضافی دارد. بهینه‌سازی چنین systemهایی برای کاهش سربار همچنان مسئله‌ای پژوهشی و حل‌نشده است. بااین‌حال، اگر همین حالا برای هدف‌های عملیاتی read requestها را به‌عنوان side effect پردازش request log می‌کنید، تبدیل log به منبع خود requestها تغییر چندان بزرگی نیست.

#### پردازش data چند-partitionای

برای queryهایی که فقط یک partition را لمس می‌کنند، فرستادن query از stream و جمع‌آوری stream پاسخ‌ها شاید زیاده‌روی باشد. اما این ایده امکان اجرای distributed queryهای پیچیده‌ای را فراهم می‌کند که باید data چند partition را ترکیب کنند؛ چنین queryهایی می‌توانند از زیرساخت routing پیام، partitioning و join که stream processorها از پیش فراهم کرده‌اند بهره ببرند.

قابلیت distributed RPC در Storm از این الگو پشتیبانی می‌کند. برای مثال، از آن برای محاسبهٔ تعداد افرادی استفاده شده که یک URL را در Twitter دیده‌اند؛ یعنی union مجموعهٔ followerهای هرکسی که آن URL را tweet کرده است. چون مجموعهٔ userهای Twitter partition شده است، این محاسبه به ترکیب نتیجه‌های partitionهای زیادی نیاز دارد.

نمونهٔ دیگری از این الگو در پیشگیری از fraud دیده می‌شود: برای سنجش احتمال fraud بودن یک purchase event می‌توان reputation score مربوط به IP address، email address، billing address، shipping address کاربر و موارد دیگر را بررسی کرد. هر یک از این reputation databaseها خود partition شده‌اند؛ بنابراین گردآوری scoreهای یک purchase event مستلزم زنجیره‌ای از joinها با datasetهایی است که بر اساس partitionهای متفاوت سازمان‌دهی شده‌اند.

گراف‌های داخلی اجرای query در MPP databaseها نیز ویژگی‌های مشابهی دارند. اگر لازم است چنین join چند-partitionای انجام دهید، احتمالاً استفاده از databaseای که این قابلیت را فراهم می‌کند ساده‌تر از پیاده‌سازی آن با stream processor است. بااین‌حال، در نظر گرفتن queryها به‌عنوان stream گزینه‌ای برای پیاده‌سازی applicationهای بسیار بزرگی فراهم می‌کند که به محدودیت راه‌حل‌های آماده و متداول می‌رسند.

## Key Terms

- `Unbundled Database` — رویکردی که قابلیت‌های یک database را میان storage و processing systemهای تخصصی تقسیم می‌کند و آن‌ها را با dataflow به هم پیوند می‌دهد.
- `Federated Database (Polystore)` — interfaceای یکپارچه برای query کردن و ترکیب data از چند storage engine یا system متفاوت.
- `Write Path` — مسیر پیش‌محاسبه‌شده‌ای که data هنگام ورود از آن عبور می‌کند تا derived datasetها به‌روز شوند.
- `Read Path` — مسیری که هنگام درخواست data طی می‌شود و از derived dataset برای ساخت response استفاده می‌کند.
- `Derived Dataset` — datasetی که با transformation روی source data ساخته می‌شود؛ مانند index، cache یا predictive model.
- `Stream Operator` — واحدی از stream processing که streamهای ورودی را پردازش و streamهای خروجی تولید می‌کند.
- `Offline-First Application` — applicationای که قابلیت‌های اصلی‌اش را با state و storage محلی حتی بدون اتصال شبکه ارائه می‌دهد و بعداً با server همگام می‌شود.
- `End-to-End Event Stream` — جریان eventهایی که changeهای state را از مبدأ تا consumer نهایی، از جمله clientهای کاربر، منتقل می‌کند.

# Chapter 11 — Stream Processing

## Processing Streams

تا اینجا دربارهٔ منشأ streamها صحبت کردیم: eventهای فعالیت user، sensorها و writeهای database؛ و دیدیم streamها چگونه منتقل می‌شوند: با direct messaging، message brokerها و event logها. حالا باید بررسی کنیم پس از دراختیارداشتن stream چه می‌توان کرد؛ یعنی آن را پردازش کرد. به‌طور کلی سه انتخاب داریم:

1. data داخل eventها را در database، cache، search index یا storage system مشابهی بنویسیم تا clientهای دیگر بتوانند queryاش کنند. همان‌طور که در Figure 11-5 دیدیم، این روش برای sync نگه‌داشتن database با تغییرهایی که در بخش‌های دیگر system رخ می‌دهند مناسب است؛ به‌ویژه وقتی stream consumer تنها clientای باشد که در database می‌نویسد. نوشتن در storage system معادل streamیِ بحث «The Output of Batch Workflows» است.
2. eventها را به‌نوعی به userها برسانیم؛ مثلاً email alert یا push notification بفرستیم یا eventها را در real-time dashboardای برای نمایش stream کنیم. در این حالت، مصرف‌کنندهٔ نهایی stream انسان است.
3. یک یا چند input stream را پردازش کنیم و یک یا چند output stream بسازیم. streamها ممکن است از pipelineای با چند processing stage بگذرند تا سرانجام به outputی از نوع اول یا دوم برسند.

در ادامهٔ chapter، انتخاب سوم را بررسی می‌کنیم: پردازش stream برای تولید streamهای مشتق‌شده. قطعه codeای که این کار را انجام می‌دهد **operator** یا **job** نام دارد. این مفهوم به Unix processها و MapReduce jobهای Chapter 10 نزدیک است و الگوی dataflow هم مشابه است: stream processor، input streamها را فقط‌خواندنی مصرف می‌کند و output را به‌صورت append-only در محل دیگری می‌نویسد.

الگوهای partitioning و parallelization در stream processorها بسیار شبیه MapReduce و dataflow engineهای Chapter 10 است؛ بنابراین آن‌ها را تکرار نمی‌کنیم. operationهای پایهٔ map، مثل transformation و filtering recordها، نیز به همان شکل کار می‌کنند.

تفاوت اساسی با batch job این است که stream هرگز تمام نمی‌شود. این تفاوت پیامدهای زیادی دارد: همان‌طور که ابتدای chapter گفتیم، sort کردن برای dataset نامحدود معنایی ندارد؛ بنابراین sort-merge join قابل‌استفاده نیست. سازوکارهای fault tolerance هم باید تغییر کنند: اگر batch jobی پس از چند دقیقه شکست بخورد، می‌توان task ناموفق را از ابتدا اجرا کرد؛ اما برای stream jobای که چند سال در حال اجرا بوده، شروع دوباره از ابتدا پس از crash شاید عملی نباشد.

### Uses of Stream Processing

stream processing مدت‌ها برای monitoring به کار رفته است؛ یعنی زمانی که سازمان می‌خواهد با رخ‌دادن اتفاق‌های خاصی alert دریافت کند. برای نمونه:

- fraud detection system باید تشخیص دهد الگوی استفاده از credit card ناگهان تغییر کرده است یا نه، و اگر احتمال سرقت کارت وجود داشت آن را مسدود کند.
- trading system باید تغییر قیمت‌ها در بازار مالی را بررسی کند و بر اساس ruleهای تعیین‌شده معامله انجام دهد.
- system تولیدی باید وضعیت machineهای کارخانه را پایش کند و در صورت malfunction سریعاً مشکل را پیدا کند.
- systemهای نظامی و اطلاعاتی باید فعالیت مهاجم احتمالی را دنبال کنند و در صورت مشاهدهٔ نشانه‌های حمله هشدار دهند.

این applicationها به pattern matching و correlation نسبتاً پیچیده‌ای نیاز دارند. بااین‌حال، در گذر زمان کاربردهای دیگری هم برای stream processing پیدا شده است. در ادامه چند مورد را مقایسه می‌کنیم.

#### Complex event processing

**Complex event processing (CEP)** رویکردی است که در دههٔ ۱۹۹۰ برای تحلیل event streamها توسعه یافت و به‌خصوص برای applicationهایی مناسب است که باید الگوهای مشخصی از eventها را پیدا کنند. همان‌طور که regular expression برای جست‌وجوی pattern مشخصی از characterها در string به کار می‌رود، در CEP می‌توان ruleهایی تعریف کرد تا patternهای مشخصی از eventها در stream پیدا شوند.

CEP systemها اغلب از declarative query language سطح‌بالایی مانند SQL یا interface گرافیکی برای توصیف patternهایی استفاده می‌کنند که باید شناسایی شوند. queryها به processing engine داده می‌شوند؛ engine، input streamها را مصرف می‌کند و درون خود state machineای برای تطبیق patternها نگه می‌دارد. وقتی match پیدا شد، engine یک **complex event** شامل جزئیات pattern شناسایی‌شده تولید می‌کند؛ نام CEP هم از همین‌جا می‌آید.

در این systemها رابطهٔ query و data نسبت به database معمولی وارونه است. معمولاً database data را به‌صورت پایدار ذخیره می‌کند و query را گذرا می‌داند: query که برسد، database data منطبق با آن را جست‌وجو می‌کند و پس از پایان، query را فراموش می‌کند. CEP engine این نقش‌ها را برعکس می‌کند: queryها را در بلندمدت نگه می‌دارد و eventهای input stream را پیوسته از مقابل آن‌ها می‌گذراند تا event pattern منطبق با query پیدا شود.

از implementationهای CEP می‌توان Esper، IBM InfoSphere Streams، Apama، TIBCO StreamBase و SQLstream را نام برد. distributed stream processorهایی مانند Samza نیز پشتیبانی SQL برای queryهای declarative روی stream را اضافه می‌کنند.

#### Stream analytics

کاربرد دیگر، اجرای analytics روی stream است. مرز میان CEP و stream analytics کاملاً روشن نیست، اما analytics معمولاً کمتر به پیدا کردن sequence مشخصی از eventها می‌پردازد و بیشتر بر aggregation و metricهای آماری روی شمار زیادی event متمرکز است؛ برای مثال:

- اندازه‌گیری نرخ یک نوع event، یعنی تعداد وقوع آن در هر بازهٔ زمانی؛
- محاسبهٔ rolling average یک value در دوره‌ای مشخص؛
- مقایسهٔ آمار فعلی با بازه‌های قبلی، مثلاً برای تشخیص trend یا alert دادن دربارهٔ metricهایی که در مقایسه با همین زمان در هفتهٔ گذشته غیرعادی بالا یا پایین‌اند.

این آمار معمولاً در بازه‌های زمانی ثابت محاسبه می‌شود. مثلاً شاید بخواهید میانگین queryهای یک service در هر ثانیه طی پنج دقیقهٔ اخیر و 99th percentile زمان پاسخ آن‌ها را در همان بازه بدانید. میانگین‌گیری چنددقیقه‌ای نوسان‌های بی‌اهمیت یک‌ثانیه‌ای را هموار می‌کند، اما همچنان تصویری به‌روز از تغییر الگوی traffic می‌دهد. بازهٔ زمانی‌ای که روی آن aggregation می‌کنید **window** نام دارد؛ در «Reasoning About Time» با جزئیات بیشتری به windowing می‌پردازیم.

stream analytics systemها گاهی از الگوریتم‌های probabilistic استفاده می‌کنند؛ مانند Bloom filter برای membership در set، HyperLogLog برای تخمین cardinality و الگوریتم‌های مختلف تخمین percentile. این الگوریتم‌ها نتیجهٔ تقریبی می‌دهند، اما در مقایسه با الگوریتم دقیق به memory بسیار کمتری در stream processor نیاز دارند. استفاده از الگوریتم تقریبی گاهی این تصور را ایجاد می‌کند که stream processing همیشه lossy و غیردقیق است؛ این تصور نادرست است. در خود stream processing هیچ چیز ذاتاً تقریبی نیست و الگوریتم‌های probabilistic صرفاً بهینه‌سازی‌اند.

بسیاری از distributed stream processing frameworkهای open source با درنظرگرفتن analytics طراحی شده‌اند؛ از جمله Apache Storm، Spark Streaming، Flink، Concord، Samza و Kafka Streams. سرویس‌های میزبانی‌شده نیز شامل Google Cloud Dataflow و Azure Stream Analytics هستند.

#### نگهداری materialized viewها

در «Databases and Streams» دیدیم stream تغییرهای database می‌تواند systemهای derived data، مانند cache، search index و data warehouse را با database مبدأ به‌روز نگه دارد. می‌توان این مثال‌ها را حالت‌های خاصی از نگهداری **materialized view** دانست: ساخت view جایگزینی از یک dataset تا بتوان آن را کارآمد query کرد و update کردن view هر زمان که data زیربنایی تغییر می‌کند.

در event sourcing نیز application state با apply کردن event log نگهداری می‌شود؛ در اینجا هم application state نوعی materialized view است. برخلاف stream analytics، معمولاً کافی نیست فقط eventهای یک window زمانی را در نظر بگیریم. ساخت materialized view ممکن است به همهٔ eventهای یک بازهٔ زمانی دلخواه نیاز داشته باشد، جز eventهای منسوخی که log compaction اجازهٔ حذفشان را می‌دهد. عملاً به windowای نیاز دارید که تا «ابتدای زمان» عقب می‌رود.

در اصل، هر stream processorای می‌تواند برای نگهداری materialized view استفاده شود؛ بااین‌حال، نیاز به نگهداری همیشگی eventها با فرض بعضی frameworkهای analyticsمحور که بیشتر با windowهای محدود کار می‌کنند سازگار نیست. Samza و Kafka Streams با تکیه بر پشتیبانی Kafka از log compaction از این نوع استفاده پشتیبانی می‌کنند.

#### جست‌وجو روی streamها

علاوه بر CEP که امکان جست‌وجوی patternهای متشکل از چند event را می‌دهد، گاهی لازم است eventهای منفرد را هم بر اساس معیارهای پیچیده، مانند full-text search query، پیدا کنیم.

برای مثال، سرویس‌های پایش رسانه feed مقاله‌های خبری و برنامه‌های رسانه‌ها را دریافت می‌کنند و دنبال خبرهایی می‌گردند که به شرکت، محصول یا موضوع موردعلاقه اشاره دارند. query جست‌وجو از قبل تعریف می‌شود و سپس stream خبرها پیوسته با آن تطبیق داده می‌شود. قابلیت مشابهی در بعضی وب‌سایت‌ها وجود دارد: user سایت املاک می‌تواند بخواهد با عرضه‌شدن ملک تازه‌ای که با معیارهای جست‌وجویش منطبق است notification بگیرد. قابلیت percolator در Elasticsearch یکی از گزینه‌ها برای پیاده‌سازی این نوع stream search است.

search engine معمولی ابتدا documentها را index می‌کند و بعد query را روی index اجرا می‌کند. اما در جست‌وجو روی stream، روند وارونه می‌شود: queryها ذخیره می‌شوند و documentها از مقابلشان می‌گذرند؛ شبیه CEP. در ساده‌ترین حالت، هر document را با هر query مقایسه می‌کنید؛ اما اگر queryهای زیادی وجود داشته باشد این کار کند می‌شود. برای بهینه‌سازی می‌توان علاوه بر documentها، خود queryها را هم index کرد تا مجموعهٔ queryهایی که احتمال match شدن دارند محدود شود.

#### Message passing و RPC

در «Message-Passing Dataflow»، message-passing systemها را به‌عنوان جایگزینی برای RPC بررسی کردیم؛ یعنی سازوکاری برای ارتباط serviceها، مثلاً در actor model. با وجود اینکه این systemها هم بر message و event متکی‌اند، معمولاً آن‌ها را stream processor نمی‌دانیم:

- actor frameworkها در درجهٔ اول سازوکاری برای مدیریت concurrency و اجرای distributed moduleهای در حال ارتباط‌اند؛ stream processing در درجهٔ اول تکنیکی برای مدیریت data است.
- ارتباط actorها اغلب گذرا و یک‌به‌یک است، درحالی‌که event logها durable و چندمشترکی هستند.
- actorها می‌توانند به شکل‌های دلخواه، از جمله الگوهای چرخه‌ای request/response، با هم ارتباط برقرار کنند؛ stream processorها معمولاً در pipelineهای بدون cycle سازمان‌دهی می‌شوند که هر stream خروجی job مشخصی است و از مجموعهٔ معینی از input streamها مشتق می‌شود.

البته میان systemهای شبیه RPC و stream processing هم ناحیهٔ مشترکی وجود دارد. برای مثال، Apache Storm قابلیتی به نام distributed RPC دارد که queryهای user را میان مجموعه‌ای از nodeهایی پخش می‌کند که هم‌زمان event stream هم پردازش می‌کنند. queryها با eventهای input stream در هم آمیخته می‌شوند و نتیجه‌ها را می‌توان aggregate کرد و به user برگرداند.

با actor frameworkها هم می‌توان stream پردازش کرد. اما بسیاری از این frameworkها هنگام crash تحویل message را تضمین نمی‌کنند؛ بنابراین processing تا وقتی منطق retry اضافی پیاده‌سازی نکنید fault-tolerant نیست.

### Reasoning About Time

stream processorها اغلب باید با زمان سروکار داشته باشند، به‌خصوص برای analytics که معمولاً از windowهای زمانی‌ای مانند «میانگین پنج دقیقهٔ اخیر» استفاده می‌کند. شاید معنای «پنج دقیقهٔ اخیر» روشن و بدون ابهام به نظر برسد، اما در واقع مفهوم ظریفی است.

در batch process، taskها مجموعهٔ بزرگی از eventهای تاریخی را سریع پردازش می‌کنند. اگر لازم باشد data بر اساس زمان دسته‌بندی شود، batch process باید timestamp داخل هر event را بخواند. نگاه‌کردن به system clock ماشین اجراکنندهٔ batch بی‌فایده است؛ زمان اجرای process ارتباطی با زمان وقوع واقعی eventها ندارد.

batch process ممکن است eventهای یک سال را در چند دقیقه بخواند؛ معمولاً بازهٔ موردعلاقه همان یک سال تاریخچه است، نه چند دقیقه‌ای که صرف پردازش شده. استفاده از timestampهای خود event، processing را deterministic هم می‌کند: اجرای دوبارهٔ process روی input یکسان باید همان نتیجه را بدهد.

در مقابل، بسیاری از stream processing frameworkها برای تعیین window از ساعت محلی machine پردازش‌کننده استفاده می‌کنند؛ این را **processing time** می‌نامیم. این روش ساده است و وقتی delay میان تولید و پردازش event ناچیز باشد قابل‌قبول است. اما اگر processing lag قابل‌توجه باشد ــ یعنی event دیرتر از زمان وقوعش پردازش شود ــ روش از کار می‌افتد.

#### Event time در برابر processing time

پردازش ممکن است به دلایل مختلف عقب بیفتد: queueing، network fault، contention در message broker یا processor به‌دلیل مشکل performance، restart شدن stream consumer یا replay کردن eventهای قدیمی هنگام recovery از fault یا پس از رفع bug در code.

علاوه بر این، delay پیام‌ها می‌تواند ترتیبشان را غیرقابل‌پیش‌بینی کند. فرض کنید user ابتدا requestی می‌فرستد که web server A آن را پردازش می‌کند و بعد request دومی می‌فرستد که به web server B می‌رسد. هر دو server eventی دربارهٔ request پردازش‌شده تولید می‌کنند، اما event مربوط به B زودتر از event مربوط به A به message broker می‌رسد. پس stream processor ابتدا event مربوط به B و بعد event مربوط به A را می‌بیند، با اینکه در واقع ترتیب وقوع برعکس بوده است.

برای درک موضوع می‌توان فیلم‌های Star Wars را در نظر گرفت: Episode IV در 1977، Episode V در 1980 و Episode VI در 1983 اکران شدند؛ سپس Episode I، II و III به‌ترتیب در 1999، 2002 و 2005 و Episode VII در 2015 اکران شد. اگر فیلم‌ها را به ترتیب زمان اکران ببینید، ترتیب پردازشتان با ترتیب روایت داستان یکی نیست. شمارهٔ episode مانند timestamp رویداد است و تاریخی که فیلم را دیده‌اید مانند processing time. انسان‌ها می‌توانند با این ناپیوستگی کنار بیایند، اما الگوریتم stream processing باید مشخصاً برای چنین مسئله‌های زمان و ترتیب نوشته شود.

اشتباه‌گرفتن event time با processing time به data نادرست منجر می‌شود. مثلاً فرض کنید stream processor نرخ request را، با شمردن تعداد request در هر ثانیه، اندازه می‌گیرد. اگر processor را redeploy کنید، ممکن است یک دقیقه خاموش بماند و پس از راه‌اندازی دوباره backlog eventها را پردازش کند. اگر نرخ را بر اساس processing time حساب کنید، هنگام پردازش backlog جهش غیرعادی request به نظر می‌رسد، درحالی‌که نرخ واقعی requestها ثابت بوده است (Figure 11-7).

#### از کجا بدانیم window آماده است؟

وقتی window را بر اساس event time تعریف می‌کنید، نمی‌توانید مطمئن باشید همهٔ eventهای آن window را دریافت کرده‌اید یا eventهای دیگری هم در راه‌اند.

مثلاً eventها را در windowهای یک‌دقیقه‌ای گروه‌بندی می‌کنید تا تعداد request در هر دقیقه را بشمارید. تعدادی event با timestamp مربوط به دقیقهٔ ۳۷ ساعت را شمرده‌اید و زمان گذشته است؛ حالا بیشتر eventهای ورودی مربوط به دقیقه‌های ۳۸ و ۳۹ هستند. چه زمانی اعلام می‌کنید window دقیقهٔ ۳۷ تمام شده و مقدار counter را منتشر می‌کنید؟

می‌توانید پس از مدتی نرسیدن event تازه timeout کنید و window را آماده اعلام کنید؛ اما هنوز ممکن است eventهایی در machine دیگری buffer شده باشند و به‌دلیل اختلال network دیر برسند. باید بتوانید این eventهای عقب‌مانده (straggler) را که پس از اعلام پایان window می‌رسند مدیریت کنید. به‌طور کلی دو انتخاب دارید:

1. eventهای عقب‌مانده را نادیده بگیرید، چون در شرایط عادی احتمالاً درصد کمی از eventها هستند. تعداد eventهای dropشده را به‌عنوان metric نگه دارید و اگر مقدار قابل‌توجهی data از دست رفت alert بدهید.
2. اصلاحیه منتشر کنید: value به‌روزشدهٔ window را با درنظرگرفتن eventهای دیررس بفرستید. ممکن است لازم باشد output قبلی را هم پس بگیرید.

در بعضی موارد می‌توان پیام ویژه‌ای فرستاد با این معنا که «از این به بعد هیچ پیامی با timestamp قدیمی‌تر از `t` نخواهد رسید»؛ consumer می‌تواند از آن برای فعال‌کردن windowها استفاده کند. اما اگر چند producer روی machineهای مختلف event بسازند و هرکدام حداقل timestamp آستانهٔ خود را داشته باشند، consumer باید وضعیت تک‌تک producerها را دنبال کند. در این حالت اضافه یا حذف‌کردن producerها پیچیده‌تر می‌شود.

#### ساعتِ کدام machine؟

اگر eventها در چند نقطهٔ system buffer شوند، نسبت‌دادن timestamp به آن‌ها دشوارتر هم می‌شود. مثلاً mobile appای را در نظر بگیرید که eventهای مربوط به metricهای استفاده را به server گزارش می‌کند. ممکن است از app وقتی device offline است استفاده شود؛ در این صورت eventها محلی buffer می‌شوند و پس از دسترس‌پذیرشدن دوبارهٔ اینترنت ــ شاید ساعت‌ها یا حتی روزها بعد ــ به server فرستاده می‌شوند. از دید consumerهای stream، این eventها stragglerهایی با delay بسیار زیاد به نظر می‌رسند.

در این وضعیت، timestamp event باید زمان تعامل user بر اساس ساعت محلی device باشد. اما ساعتی که user کنترل می‌کند قابل‌اعتماد نیست؛ ممکن است تصادفاً یا عمداً روی زمان اشتباه تنظیم شده باشد. زمان دریافت event در server، بر اساس ساعت server، احتمالاً دقیق‌تر است چون server تحت کنترل شماست، اما برای توصیف زمان تعامل user معنای کمتری دارد.

برای اصلاح ساعت نادرست device می‌توان سه timestamp ثبت کرد:

- زمان وقوع event بر اساس ساعت device؛
- زمان ارسال event به server بر اساس ساعت device؛
- زمان دریافت event در server بر اساس ساعت server.

با کم‌کردن timestamp دوم از سوم می‌توان اختلاف ساعت device و server را تخمین زد؛ با این فرض که delay شبکه در مقایسه با دقت زمانی موردنیاز ناچیز باشد. سپس می‌توان این اختلاف را روی timestamp خود event اعمال کرد و زمان واقعی وقوع آن را تخمین زد؛ با این فرض که اختلاف ساعت device از زمان وقوع event تا زمان ارسالش تغییر نکرده باشد.

این مسئله منحصر به stream processing نیست؛ batch processing هم دقیقاً با همین چالش‌های استدلال دربارهٔ زمان روبه‌رو است. فقط در stream processing گذر زمان ملموس‌تر است و مسئله بیشتر به چشم می‌آید.

#### انواع window

وقتی مشخص شد timestamp یک event را چگونه تعیین می‌کنید، گام بعدی تعریف windowهای زمانی است. سپس می‌توان از window برای aggregation، مثلاً شمارش eventها یا محاسبهٔ میانگین valueها، استفاده کرد. چند نوع window متداول وجود دارد:

**Tumbling window:** طول ثابتی دارد و هر event دقیقاً به یک window تعلق می‌گیرد. برای نمونه، در tumbling window یک‌دقیقه‌ای، eventهای timestampدار بین `10:03:00` و `10:03:59` در یک window و eventهای بین `10:04:00` و `10:04:59` در window بعدی قرار می‌گیرند. می‌توان timestamp هر event را به نزدیک‌ترین دقیقه رو به پایین گرد کرد تا window آن مشخص شود.

**Hopping window:** آن هم طول ثابتی دارد، اما windowها می‌توانند هم‌پوشانی داشته باشند تا نتیجه هموارتر شود. مثلاً window پنج‌دقیقه‌ای با hop یک‌دقیقه‌ای ابتدا eventهای `10:03:00` تا `10:07:59` را در بر می‌گیرد و window بعدی eventهای `10:04:00` تا `10:08:59` را. می‌توان ابتدا tumbling windowهای یک‌دقیقه‌ای را ساخت و سپس چند window مجاور را aggregate کرد.

**Sliding window:** همهٔ eventهایی را در بر می‌گیرد که فاصلهٔ زمانی‌شان از یکدیگر کمتر از بازه‌ای مشخص است. مثلاً sliding window پنج‌دقیقه‌ای eventهای `10:03:39` و `10:08:12` را شامل می‌شود، چون فاصله‌شان کمتر از پنج دقیقه است. tumbling و hopping windowهای پنج‌دقیقه‌ای این دو event را در یک window نمی‌گذاشتند، چون مرزهای ثابت دارند. sliding window را می‌توان با نگه‌داشتن bufferای از eventهای مرتب‌شده بر اساس زمان و حذف eventهای منقضی‌شده پیاده‌سازی کرد.

**Session window:** برخلاف بقیه، مدت ثابتی ندارد. eventهای یک user را که از نظر زمانی نزدیک به هم رخ می‌دهند گروه‌بندی می‌کند و وقتی user برای مدتی غیرفعال بماند، مثلاً ۳۰ دقیقه eventی نفرستد، window را می‌بندد. sessionization در website analytics نیاز رایجی است.

### Stream Joins

در Chapter 10 دیدیم batch jobها چگونه datasetها را بر اساس key join می‌کنند و join چه نقش مهمی در data pipeline دارد. stream processing، pipelineهای data را به پردازش تدریجی datasetهای نامحدود تعمیم می‌دهد؛ بنابراین برای streamها هم دقیقاً به join نیاز داریم.

اما چون eventهای تازه هر لحظه ممکن است به stream برسند، join در stream از batch job دشوارتر است. برای روشن‌شدن موضوع سه نوع join را از هم جدا می‌کنیم: stream-stream join، stream-table join و table-table join.

#### Stream-stream join (window join)

فرض کنید در وب‌سایتتان search دارید و می‌خواهید trendهای تازه در URLهای جست‌وجوشده را تشخیص دهید. هر بار که کسی queryای وارد می‌کند، eventای حاوی query و نتیجه‌های برگشتی ثبت می‌کنید. هر بار که روی یکی از نتیجه‌ها کلیک می‌شود، event دیگری برای click ثبت می‌کنید. برای محاسبهٔ click-through rate هر URL باید event مربوط به search و event مربوط به click را که session ID یکسان دارند کنار هم بگذارید. تحلیل‌های مشابهی در advertising systemها لازم است.

اگر user از search منصرف شود ممکن است هیچ clickای رخ ندهد. حتی اگر click رخ دهد، فاصلهٔ زمانی search و click بسیار متغیر است: اغلب چند ثانیه، اما شاید چند روز یا چند هفته؛ مثلاً user search کند، tab مرورگر را فراموش کند و بعدتر برگردد و روی نتیجه کلیک کند. به‌دلیل delay متغیر network، event مربوط به click حتی ممکن است پیش از event مربوط به search برسد. برای join می‌توان window مناسبی انتخاب کرد؛ مثلاً click را به searchی join کرد که حداکثر یک ساعت با آن فاصله دارد.

قرار دادن جزئیات search در event مربوط به click معادل join کردن دو event نیست؛ این کار فقط موردهایی را نشان می‌دهد که user روی نتیجه کلیک کرده است، نه searchهایی را که به هیچ clickای منجر نشده‌اند. برای اندازه‌گیری کیفیت search به click-through rate دقیق نیاز دارید و برای آن باید هم eventهای search و هم eventهای click را داشته باشید.

برای اجرای این join، stream processor باید state نگه دارد؛ مثلاً همهٔ eventهای یک ساعت اخیر را بر اساس session ID index کند. هر بار که search event یا click event می‌رسد، به index مربوطه افزوده می‌شود و processor در index دیگر هم بررسی می‌کند که آیا eventی با همان session ID رسیده است یا نه. اگر match پیدا شد، eventی تولید می‌شود که نشان می‌دهد کدام نتیجه کلیک شده است. اگر search event منقضی شود و click متناظری نرسیده باشد، eventی تولید می‌شود که نتیجه‌های کلیک‌نشده را مشخص می‌کند.

#### Stream-table join (غنی‌سازی stream)

در مثال تحلیل eventهای فعالیت user در Chapter 10، batch jobای دو dataset را join می‌کرد: مجموعه‌ای از activity eventها و databaseای از profileهای user. طبیعی است activity eventها را stream در نظر بگیریم و همان join را به‌طور پیوسته در stream processor انجام دهیم. input، streamای از activity eventهای دارای user ID است و output، streamای از همان eventهاست که اطلاعات profile به آن اضافه شده است. گاهی این کار را **enrichment** کردن activity event با اطلاعات database می‌نامند.

برای اجرای join، stream processor باید هر بار یک activity event را بردارد، user ID آن را در database جست‌وجو کند و اطلاعات profile را به event اضافه کند. می‌توان database راه‌دور را query کرد، اما چنین queryهایی احتمالاً کند هستند و خطر overload کردن database را دارند. راه دیگر این است که copyای از database را در stream processor بارگذاری کنیم تا query محلی و بدون network round-trip انجام شود. این روش به hash joinهای «Map-Side Joins» شبیه است: اگر database کوچک باشد copy محلی می‌تواند in-memory hash table باشد، وگرنه می‌توان از index روی disk محلی استفاده کرد.

تفاوت با batch job در این است که batch job از snapshot database در یک نقطهٔ زمانی به‌عنوان input استفاده می‌کند؛ اما stream processor درازمدت اجرا می‌شود و محتوای database احتمالاً تغییر می‌کند. بنابراین copy محلی database در stream processor باید به‌روز بماند. CDC این مسئله را حل می‌کند: stream processor می‌تواند هم به changelog مربوط به profile database و هم به activity event stream subscribe شود. با ساخته یا تغییرکردن profile، processor copy محلی خود را update می‌کند. در نتیجه join میان دو stream داریم: activity eventها و updateهای profile.

stream-table join در واقع به stream-stream join بسیار شبیه است. تفاوت اصلی این است که برای stream مربوط به changelog table، join از windowای استفاده می‌کند که تا «ابتدای زمان» عقب می‌رود ــ windowی بی‌نهایت از نظر مفهومی ــ و نسخه‌های جدیدتر record، نسخه‌های قدیمی را overwrite می‌کنند. برای input مربوط به stream ممکن است اصلاً windowای نگه نداریم.

#### Table-table join (نگهداری materialized view)

مثال Twitter timeline را در «Describing Load» به یاد بیاورید. گفتیم برای نمایش home timeline یک user، پیمایش همهٔ افرادی که دنبال می‌کند، پیدا کردن tweetهای اخیرشان و merge کردن آن‌ها بیش‌ازحد پرهزینه است.

در عوض، timeline cache می‌خواهیم: نوعی inbox برای هر user که tweetها هنگام ارسال‌شدن در آن نوشته شوند تا خواندن timeline فقط به یک lookup نیاز داشته باشد. materialize و نگهداری این cache به پردازش eventهای زیر نیاز دارد:

- وقتی userِ `u` tweet تازه‌ای می‌فرستد، tweet به timeline همهٔ userهایی اضافه می‌شود که `u` را دنبال می‌کنند.
- وقتی userای tweetی را حذف می‌کند، tweet از timeline همهٔ userها برداشته می‌شود.
- وقتی userِ `u1` شروع به دنبال‌کردن `u2` می‌کند، tweetهای اخیر `u2` به timelineِ `u1` اضافه می‌شوند.
- وقتی `u1`، `u2` را unfollow می‌کند، tweetهای `u2` از timelineِ `u1` حذف می‌شوند.

برای نگهداری این cache در stream processor، streamهایی از eventهای tweet ــ ارسال و حذف ــ و رابطه‌های follow ــ دنبال‌کردن و لغو دنبال‌کردن ــ لازم دارید. stream process باید databaseای از مجموعهٔ followerهای هر user نگه دارد تا هنگام رسیدن tweet تازه بداند کدام timelineها را update کند.

می‌توان به این stream process به چشم نگهدارندهٔ materialized view مربوط به queryای نگاه کرد که دو table یعنی `tweets` و `follows` را join می‌کند؛ چیزی شبیه این:

```sql
SELECT follows.follower_id AS timeline_id,
  array_agg(tweets.* ORDER BY tweets.timestamp DESC)
FROM tweets
JOIN follows ON follows.followee_id = tweets.sender_id
GROUP BY follows.follower_id
```

join کردن streamها مستقیماً متناظر با join کردن tableها در این query است. timelineها عملاً cache نتیجهٔ query هستند که با هر تغییر در tableهای زیربنایی update می‌شوند. اگر stream را مشتق table در نظر بگیریم و join را حاصل‌ضرب دو tableِ `u·v`، تغییرهای materialized join از قاعدهٔ حاصل‌ضرب پیروی می‌کنند:

```text
(u·v)′ = u′v + uv′
```

به بیان کلامی، هر تغییر در tweetها با followerهای فعلی join می‌شود و هر تغییر در followerها با tweetهای فعلی.

#### وابستگی joinها به زمان

سه نوع join مطرح‌شده ــ stream-stream، stream-table و table-table ــ نکته‌های مشترک زیادی دارند: stream processor باید بر اساس یکی از inputهای join، stateای نگه دارد ــ eventهای search و click، profileهای user یا فهرست followerها ــ و هنگام دریافت پیام از input دیگر، آن state را query کند.

ترتیب eventهایی که state را تغییر می‌دهند مهم است؛ اینکه اول follow انجام شود و بعد unfollow، با ترتیب برعکس یکسان نیست. در partitioned log ترتیب eventهای یک partition حفظ می‌شود، اما معمولاً میان streamها یا partitionهای مختلف تضمین ترتیب وجود ندارد.

پس اگر eventهای streamهای مختلف تقریباً هم‌زمان رخ دهند، به چه ترتیبی پردازش می‌شوند؟ در مثال stream-table join، اگر user profile خود را update کند، کدام activity eventها با profile قدیمی join می‌شوند ــ چون پیش از update پردازش شده‌اند ــ و کدام با profile تازه؟ به بیان دیگر، وقتی state در طول زمان تغییر می‌کند و با آن join می‌زنید، از state مربوط به کدام نقطهٔ زمانی استفاده می‌کنید؟

این وابستگی به زمان در جاهای زیادی پیش می‌آید. مثلاً هنگام فروش کالا باید نرخ مالیات درست را به invoice اعمال کنید؛ نرخ مالیات به کشور یا ایالت، نوع محصول و تاریخ فروش وابسته است، چون ممکن است در گذر زمان تغییر کند. هنگام join کردن فروش‌ها با table نرخ مالیات، احتمالاً باید از نرخ زمان فروش استفاده کنید؛ این نرخ ممکن است هنگام پردازش دوبارهٔ data تاریخی با نرخ فعلی فرق داشته باشد.

اگر ترتیب eventها میان streamها مشخص نباشد، join nondeterministic می‌شود. در نتیجه، ممکن است اجرای دوبارهٔ همان job روی همان input لزوماً همان نتیجه را ندهد؛ چون eventهای input streamها ممکن است در اجرای مجدد با ترتیب متفاوتی در هم آمیخته شوند.

در data warehouseها این مسئله **slowly changing dimension (SCD)** نام دارد و اغلب با شناسهٔ یکتایی برای هر نسخه از record حل می‌شود. مثلاً هر بار که نرخ مالیات تغییر می‌کند شناسهٔ تازه‌ای می‌گیرد و invoice شناسهٔ نرخ مالیاتِ زمان فروش را نگه می‌دارد. این کار join را deterministic می‌کند، اما log compaction دیگر ممکن نیست، چون باید همهٔ نسخه‌های recordهای table را نگه داشت.

### Fault Tolerance

در بخش پایانی این chapter بررسی می‌کنیم stream processorها چگونه fault را تحمل می‌کنند. در Chapter 10 دیدیم frameworkهای batch processing نسبتاً آسان از faultها recovery می‌کنند: اگر taskای در MapReduce job شکست بخورد، روی machine دیگری از نو اجرا می‌شود و output task ناموفق دور ریخته می‌شود. این retry شفاف ممکن است چون input fileها immutable هستند، هر task output خود را در file جداگانه‌ای در HDFS می‌نویسد و output فقط پس از موفقیت task قابل‌مشاهده می‌شود.

به‌خصوص، روش batch برای fault tolerance تضمین می‌کند output job همان باشد که انگار مشکلی رخ نداده است، حتی اگر در واقع بعضی taskها شکست خورده باشند. از دید output گویی هر input record دقیقاً یک‌بار پردازش شده است: هیچ recordی جا نیفتاده و هیچ recordی دوبار پردازش نشده است. ممکن است taskها واقعاً چند بار اجرا شده باشند، اما اثر قابل‌مشاهده در output مانند اجرای یک‌باره است. این اصل **exactly-once semantics** نام دارد، هرچند اصطلاح effectively-once توصیف دقیق‌تری است.

همین مسئلهٔ fault tolerance در stream processing هم وجود دارد، اما حل آن سرراست نیست: نمی‌توان تا پایان task صبر کرد و بعد output را قابل‌مشاهده کرد، چون stream بی‌نهایت است و پردازش آن هیچ‌وقت تمام نمی‌شود.

#### Microbatching و checkpointing

یک راه‌حل این است که stream را به blockهای کوچک تقسیم کنیم و هر block را مانند batch process کوچکی پردازش کنیم. این روش **microbatching** نام دارد و در Spark Streaming استفاده می‌شود. اندازهٔ batch معمولاً حدود یک ثانیه است؛ انتخابی که مصالحه‌ای برای performance است: batchهای کوچک‌تر سربار scheduling و coordination بیشتری دارند و batchهای بزرگ‌تر باعث می‌شوند نتیجهٔ stream processor دیرتر قابل‌مشاهده شود.

microbatching به‌طور ضمنی tumbling windowای برابر اندازهٔ batch ایجاد می‌کند؛ window بر اساس processing time است، نه timestampهای event. jobهایی که window بزرگ‌تری لازم دارند باید state را صریحاً از یک microbatch به بعدی منتقل کنند.

روش دیگری که Apache Flink به کار می‌برد، گرفتن rolling checkpoint از state و نوشتن دوره‌ای آن در storage پایدار است. اگر stream operator crash کند، می‌تواند از آخرین checkpoint راه‌اندازی شود و output تولیدشده از checkpoint تا زمان crash را دور بریزد. checkpointها با barrierهایی در message stream trigger می‌شوند؛ شبیه مرز microbatchها، اما بدون تحمیل اندازهٔ window مشخص.

در محدودهٔ خود stream processing framework، microbatching و checkpointing همان exactly-once semantics مربوط به batch processing را فراهم می‌کنند. اما به‌محض اینکه output از stream processor خارج شود ــ مثلاً در database نوشته شود، به message broker بیرونی فرستاده شود یا email ارسال شود ــ framework دیگر نمی‌تواند output مربوط به batch ناموفق را دور بریزد. در این حالت restart کردن task ناموفق باعث می‌شود side effect بیرونی دوباره رخ دهد؛ microbatching یا checkpointing به‌تنهایی برای جلوگیری از این مشکل کافی نیست.

#### بازنگری atomic commit

برای ایجاد ظاهر exactly-once processing هنگام رخ‌دادن fault باید تضمین کنیم همهٔ outputها و side effectهای پردازش event اگر و فقط اگر پردازش موفق شد اعمال شوند. این effectها شامل messageهایی است که به operatorهای downstream یا systemهای messaging بیرونی فرستاده می‌شوند ــ از جمله email و push notification ــ، writeهای database، تغییرهای operator state و acknowledgment پیام‌های input، از جمله جلو بردن consumer offset در message broker مبتنی بر log.

این کارها یا باید همگی atomic انجام شوند یا هیچ‌کدام؛ نباید از یکدیگر ناهماهنگ شوند. این همان مسئله‌ای است که در «Exactly-once message processing» در زمینهٔ distributed transaction و two-phase commit بررسی کردیم.

در Chapter 9 مشکل‌های implementation سنتی distributed transaction مانند XA را دیدیم. اما در محیط‌های محدودتر می‌توان قابلیت atomic commit را به‌شکل کارآمد پیاده‌سازی کرد. Google Cloud Dataflow و VoltDB از این رویکرد استفاده می‌کنند و برای افزودن قابلیت‌های مشابه به Apache Kafka هم برنامه‌هایی وجود داشته است. برخلاف XA، این implementationها نمی‌کوشند transaction را میان فناوری‌های ناهمگون فراهم کنند؛ بلکه تغییر state و messaging را درون stream processing framework مدیریت می‌کنند. با پردازش چند input message در یک transaction می‌توان سربار protocol تراکنش را سرشکن کرد.

#### Idempotence

هدف این است که output ناقص taskهای شکست‌خورده را کنار بگذاریم تا بتوان task را با اطمینان retry کرد و اثرش دوبار اعمال نشود. distributed transaction یکی از راه‌های رسیدن به این هدف است؛ راه دیگر تکیه بر **idempotence** است.

operation idempotent را می‌توان چند بار اجرا کرد و اثرش با یک‌بار اجرا تفاوتی ندارد. مثلاً تنظیم یک key در key-value store روی value ثابتی idempotent است؛ نوشتن دوباره همان value قبلی را overwrite می‌کند. اما increment کردن counter idempotent نیست، چون اجرای دوباره counter را دو بار افزایش می‌دهد.

حتی اگر operation ذاتاً idempotent نباشد، اغلب می‌توان با metadata اضافی آن را idempotent کرد. مثلاً هنگام مصرف message از Kafka، هر پیام offset پایدار و افزایشی دارد. هنگام نوشتن value در database بیرونی می‌توان offset پیام محرک آخرین write را همراه value ذخیره کرد. به این ترتیب می‌توان فهمید update قبلاً اعمال شده است یا نه و از اجرای دوبارهٔ آن جلوگیری کرد.

مدیریت state در Storm Trident بر ایده‌ای مشابه تکیه دارد. اتکا به idempotence چند فرض دارد: task ناموفق باید با replay همان پیام‌ها و به همان ترتیب restart شود؛ message broker مبتنی بر log این کار را انجام می‌دهد؛ processing باید deterministic باشد؛ و node دیگری نباید هم‌زمان همان value را update کند.

هنگام failover از یک processing node به node دیگر، شاید به fencing نیاز باشد تا nodeای که مرده فرض شده اما در واقع هنوز زنده است نتواند دخالت کند. با وجود همهٔ این شرط‌ها، operationهای idempotent راه مؤثری برای رسیدن به exactly-once semantics با سربار کم هستند.

#### بازسازی state پس از failure

هر stream processای که state لازم دارد ــ مثلاً aggregationهای windowed مانند counter، average و histogram، یا tableها و indexهای موردنیاز join ــ باید امکان recovery آن state پس از failure را فراهم کند.

یک انتخاب، نگهداری state در datastore راه‌دور و replicate کردن آن است؛ هرچند query کردن database راه‌دور برای تک‌تک messageها می‌تواند کند باشد. راه دیگر نگهداری state به‌صورت local در stream processor و replicate کردن دوره‌ای آن است. در این صورت، هنگام recovery از failure، task تازه می‌تواند state replicateشده را بخواند و بدون از‌دست‌دادن data پردازش را از سر بگیرد.

برای مثال، Flink به‌طور دوره‌ای از operator state snapshot می‌گیرد و آن را در storage پایداری مانند HDFS می‌نویسد. Samza و Kafka Streams تغییرهای state را با فرستادنشان به Kafka topic اختصاصی‌ای که log compaction دارد replicate می‌کنند؛ روشی مشابه CDC. VoltDB با پردازش تکراری هر input message روی چند node، state را replicate می‌کند.

در بعضی موارد اصلاً لازم نیست state را replicate کنیم، چون می‌توان آن را از input streamها بازسازی کرد. اگر state از aggregationهای window نسبتاً کوتاهی تشکیل شده باشد، شاید replay کردن eventهای همان window کافی و سریع باشد. اگر state، replicaی محلی از database باشد که با CDC نگهداری شده، می‌توان database را از change stream فشرده‌شده با log compaction هم بازسازی کرد.

بااین‌حال، همهٔ این trade-offها به ویژگی‌های performance زیرساخت وابسته‌اند. در بعضی systemها delay شبکه از latency دسترسی به disk کمتر است و پهنای باند network می‌تواند با پهنای باند disk قابل‌مقایسه باشد. برای همهٔ وضعیت‌ها trade-off ایدئالی وجود ندارد و مزیت state محلی در برابر راه‌دور نیز ممکن است با تغییر فناوری‌های storage و network عوض شود.

## Key Terms

- `Complex Event Processing (CEP)` — تشخیص patternهای چندمرحله‌ای در event stream و تولید event تازه هنگام یافتن match.
- `Stream Analytics` — محاسبهٔ aggregation و metricهای آماری روی حجم زیادی از eventهای جاری، معمولاً در windowهای زمانی.
- `Event Time` — زمان وقوع واقعی event که معمولاً از timestamp خود event به دست می‌آید.
- `Processing Time` — زمان پردازش event روی machine؛ ممکن است به‌دلیل backlog یا delay با event time تفاوت داشته باشد.
- `Window` — بازه یا گروه زمانی‌ای که eventها برای aggregation یا تحلیل در آن جمع می‌شوند.
- `Tumbling Window` — window ثابت و بدون هم‌پوشانی که هر event دقیقاً در یکی از آن‌ها قرار می‌گیرد.
- `Hopping Window` — window ثابت با گام حرکتی کوچک‌تر از طول window که باعث هم‌پوشانی windowها می‌شود.
- `Sliding Window` — windowی مبتنی بر فاصلهٔ زمانی eventها، نه مرزهای ثابت دوره‌ای.
- `Session Window` — گروه‌بندی eventهای نزدیک به هم برای یک user که با دوره‌ای از عدم فعالیت پایان می‌یابد.
- `Late-Arriving Event` — eventی که پس از آنکه processor window مربوط به زمان وقوعش را بسته یا نهایی کرده برسد.
- `Stream-Stream Join` — join دو stream از eventها با نگهداری state و معمولاً محدودکردن تطبیق به window زمانی.
- `Stream-Table Join` — join هر event از stream با نسخهٔ جاری یا محلیِ table، که اغلب با change stream به‌روز نگه داشته می‌شود.
- `Table-Table Join` — نگهداری پیوستهٔ نتیجهٔ join چند table به‌صورت materialized view در واکنش به تغییرهایشان.
- `Exactly-Once Semantics` — تضمینی که effect قابل‌مشاهدهٔ پردازش هر input یک‌بار اعمال شود، حتی اگر task در عمل retry شود.
- `Microbatching` — تقسیم stream به batchهای کوچک برای اجرای دوره‌ای و محدودکردن تأخیر، در برابر سربار scheduling و coordination.

# Chapter 11 — Stream Processing

## Transmitting Event Streams

در batch processing، input و output یک job معمولاً fileهایی هستند که شاید روی distributed filesystem قرار گرفته باشند. معادل این مدل در stream processing چیست؟

وقتی input یک file است ــ یعنی دنباله‌ای از byteها ــ نخستین گام پردازش معمولاً parse کردن آن به دنباله‌ای از recordهاست. در stream processing به record معمولاً **event** می‌گویند، اما از نظر ماهیت همان چیز است: objectی کوچک، مستقل و immutable که جزئیات اتفاقی را در زمان مشخصی ثبت می‌کند. event معمولاً timestampای دارد که زمان وقوع را بر اساس time-of-day clock نشان می‌دهد.

آن اتفاق می‌تواند کاری باشد که user انجام داده است، مثل دیدن یک page یا خرید محصول. ممکن است از یک machine آمده باشد، مانند اندازه‌گیری دوره‌ای temperature sensor یا metric مربوط به CPU utilization. در مثال «Batch Processing with Unix Tools»، هر خط log مربوط به web server یک event است.

event را می‌توان به‌شکل رشتهٔ متنی، JSON یا قالبی binary encode کرد. این encoding ذخیره‌کردن event را ممکن می‌کند؛ مثلاً با append کردن آن به file، درج آن در relational table یا نوشتن در document database. همچنین می‌توان event را از طریق network برای پردازش به node دیگری فرستاد.

در batch processing، file یک‌بار نوشته می‌شود و ممکن است چند job آن را بخوانند. در اصطلاح stream، event یک‌بار توسط **producer** ــ که publisher یا sender هم نامیده می‌شود ــ تولید می‌شود و سپس ممکن است چند **consumer** ــ که subscriber یا recipient هم گفته می‌شوند ــ آن را پردازش کنند. در filesystem، filename مجموعه‌ای از recordهای مرتبط را مشخص می‌کند؛ در سیستم stream، eventهای مرتبط معمولاً در یک **topic** یا stream گروه‌بندی می‌شوند.

در اصل، یک file یا database برای وصل‌کردن producerها به consumerها کافی است: producer هر event را در datastore می‌نویسد و هر consumer به‌صورت دوره‌ای datastore را poll می‌کند تا eventهایی را که از آخرین اجرا اضافه شده‌اند پیدا کند. در واقع batch processای که در پایان هر روز data همان روز را پردازش می‌کند همین کار را انجام می‌دهد.

اما برای حرکت به‌سمت پردازش مداوم با delay کم، اگر datastore برای چنین استفاده‌ای طراحی نشده باشد، polling پرهزینه می‌شود. هرچه بیشتر poll کنید، درصد بیشتری از requestها بدون event جدید برمی‌گردند و سربار افزایش می‌یابد. بهتر است به‌جای آن، هنگام رسیدن event جدید به consumerها notification داده شود.

databaseها به‌طور سنتی پشتیبانی چندان خوبی از این نوع notification نداشته‌اند. relational databaseها معمولاً triggerهایی دارند که می‌توانند به یک change واکنش نشان دهند، مثلاً درج یک row در table؛ اما قابلیتشان محدود است و در طراحی database هم چندان در مرکز توجه نبوده‌اند. به همین دلیل، ابزارهای تخصصی برای تحویل event notification ساخته شده‌اند.

### Messaging Systems

روش رایج برای آگاه‌کردن consumerها از eventهای تازه، استفاده از **messaging system** است: producer پیامی حاوی event می‌فرستد و آن پیام به consumerها push می‌شود. پیش‌تر در «Message-Passing Dataflow» اشاره‌ای به این سیستم‌ها داشتیم؛ اینجا جزئیات بیشتری را بررسی می‌کنیم.

یک channel ارتباطی مستقیم، مانند Unix pipe یا TCP connection میان producer و consumer، راه ساده‌ای برای ساخت messaging system است. اما بیشتر messaging systemها این مدل پایه را گسترش می‌دهند. Unix pipe و TCP دقیقاً یک sender را به یک recipient وصل می‌کنند، درحالی‌که messaging system اجازه می‌دهد چند producer node به یک topic پیام بفرستند و چند consumer node پیام‌های همان topic را دریافت کنند.

سیستم‌های مختلف در چارچوب publish/subscribe انتخاب‌های متنوعی دارند و یک روش واحد برای همهٔ کاربردها بهترین نیست. برای مقایسهٔ آن‌ها، پاسخ به دو پرسش به‌ویژه مفید است:

1. اگر producerها سریع‌تر از توان پردازش consumerها پیام بفرستند چه می‌شود؟ به‌طور کلی سه انتخاب وجود دارد: پیام‌ها drop شوند، در queue buffer شوند یا backpressure ــ که flow control هم نام دارد ــ اعمال شود؛ یعنی producer تا زمانی که ظرفیت خالی شود از فرستادن پیام بیشتر بازبماند. برای مثال، Unix pipe و TCP از backpressure استفاده می‌کنند: buffer کوچک و ثابتی دارند و اگر پر شود، sender تا زمانی که recipient dataای از آن خارج کند block می‌شود. اگر پیام‌ها در queue buffer شوند، باید دانست با بزرگ‌شدن queue چه اتفاقی می‌افتد. آیا وقتی دیگر در memory جا نشود system crash می‌کند یا پیام‌ها را روی disk می‌نویسد؟ اگر روی disk نوشته شوند، دسترسی به disk چه اثری بر performance messaging system دارد؟
2. اگر nodeها crash کنند یا موقتاً offline شوند، آیا پیامی از دست می‌رود؟ مانند databaseها، durability ممکن است به ترکیبی از نوشتن روی disk و replication نیاز داشته باشد که هر دو هزینه دارند. اگر بتوانید گاهی از دست‌رفتن پیام را بپذیرید، احتمالاً با همان hardware می‌توانید throughput بیشتر و latency کمتری بگیرید.

پذیرفتنی‌بودن از دست‌رفتن پیام به application بستگی دارد. برای نمونه، در sensor readingها و metricهایی که مرتب ارسال می‌شوند، گم‌شدن گهگاه یک data point شاید مهم نباشد، چون مقدار تازه‌ای کمی بعد می‌رسد. اما اگر پیام‌های زیادی drop شوند، ممکن است مدتی متوجه نادرست‌بودن metricها نشوید. در مقابل، برای شمارش eventها تحویل قابل‌اتکای پیام مهم‌تر است، چون هر پیام ازدست‌رفته شمارنده را نادرست می‌کند.

یکی از ویژگی‌های خوب batch processing systemهایی که در Chapter 10 دیدیم، تضمین reliability قوی آن‌هاست: taskهای شکست‌خورده خودکار retry می‌شوند و output ناقصشان دور ریخته می‌شود. در نتیجه، output همان است که انگار هیچ failureای رخ نداده؛ این موضوع programming model را ساده‌تر می‌کند. در ادامهٔ chapter بررسی می‌کنیم چطور می‌توان تضمین‌هایی مشابه را در stream processing فراهم کرد.

#### ارسال مستقیم پیام از producer به consumer

چند messaging system بدون node واسط، از ارتباط network مستقیم میان producer و consumer استفاده می‌کنند:

- UDP multicast در صنعت مالی برای streamهایی مانند feedهای بازار سهام کاربرد گسترده دارد؛ چون latency کم اهمیت زیادی دارد. خود UDP قابل‌اتکا نیست، اما protocolهای سطح application می‌توانند packetهای ازدست‌رفته را بازیابی کنند. برای retransmit کردن، producer باید packetهایی را که فرستاده در حافظه نگه دارد تا در صورت درخواست دوباره بفرستد.
- کتابخانه‌های messaging بدون broker، مانند ZeroMQ و nanomsg، رویکرد مشابهی دارند و publish/subscribe را روی TCP یا IP multicast پیاده‌سازی می‌کنند.
- StatsD و Brubeck از پیام‌رسانی UDP غیرقابل‌اتکا برای جمع‌آوری metric از machineهای network و پایش آن‌ها استفاده می‌کنند. در protocolِ StatsD، counter metricها فقط وقتی دقیق‌اند که همهٔ پیام‌ها دریافت شوند؛ پس با UDP، این metricها حداکثر تقریبی خواهند بود.
- اگر consumer سرویسی را روی network expose کند، producer می‌تواند برای push کردن پیام، مستقیماً HTTP request یا RPC به آن بفرستد. ایدهٔ webhooks همین است: callback URL یک service در service دیگری ثبت می‌شود و آن service هنگام رخ‌دادن event به آن URL request می‌فرستد.

این روش‌های direct messaging در موقعیت‌هایی که برایشان طراحی شده‌اند خوب عمل می‌کنند، اما معمولاً application code باید احتمال گم‌شدن پیام را در نظر بگیرد. faultهایی که تحمل می‌کنند محدود است: حتی اگر protocolها packet ازدست‌رفته در network را تشخیص دهند و دوباره بفرستند، عموماً فرض می‌کنند producer و consumer همیشه online هستند.

اگر consumer offline باشد، ممکن است پیام‌هایی را که در مدت دسترس‌ناپذیری‌اش فرستاده شده‌اند از دست بدهد. بعضی protocolها به producer اجازه می‌دهند تحویل ناموفق پیام را retry کند، اما اگر producer crash کند و buffer پیام‌هایی که باید دوباره فرستاده می‌شدند از بین برود، این روش ممکن است دیگر کار نکند.

#### Message brokerها

جایگزین رایج، فرستادن پیام از طریق **message broker** است که message queue هم نامیده می‌شود و در اصل نوعی database بهینه‌شده برای مدیریت stream پیام‌هاست. broker به‌شکل server اجرا می‌شود و producerها و consumerها به‌عنوان client به آن وصل می‌شوند. producer پیام را در broker می‌نویسد و consumer با خواندن پیام از broker آن را دریافت می‌کند.

متمرکزکردن data در broker تحمل clientهایی را که وصل، قطع یا crash می‌شوند آسان‌تر می‌کند و مسئولیت durability را به broker می‌سپارد. بعضی brokerها پیام‌ها را فقط در memory نگه می‌دارند؛ برخی دیگر، بسته به configuration، آن‌ها را روی disk هم می‌نویسند تا با crash شدن broker از دست نروند. در برابر consumerهای کند، معمولاً اجازه می‌دهند queue بدون حد مشخصی رشد کند، نه اینکه پیام‌ها را drop کنند یا backpressure اعمال کنند؛ هرچند این رفتار ممکن است به configuration وابسته باشد.

یکی از پیامدهای queue شدن این است که consumerها معمولاً asynchronous هستند: وقتی producer پیامی می‌فرستد، عموماً فقط منتظر می‌ماند broker تأیید کند پیام را buffer کرده است؛ منتظر نمی‌ماند consumer آن را پردازش کند. تحویل پیام به consumer در زمانی نامشخص در آینده انجام می‌شود؛ اغلب در کسری از ثانیه، اما اگر queue backlog داشته باشد گاهی بسیار دیرتر.

#### مقایسهٔ message brokerها با databaseها

بعضی message brokerها حتی می‌توانند با استفاده از XA یا JTA در two-phase commit شرکت کنند. این قابلیت آن‌ها را از بعضی جهات به database شبیه می‌کند، اما تفاوت‌های عملی مهمی همچنان وجود دارد:

- database معمولاً data را تا زمانی نگه می‌دارد که صریحاً حذف شود؛ بیشتر message brokerها پیام را پس از تحویل موفق به consumer خودکار حذف می‌کنند. چنین brokerهایی برای نگهداری بلندمدت data مناسب نیستند.
- چون پیام‌ها را سریع حذف می‌کنند، بیشتر brokerها فرض می‌کنند working set نسبتاً کوچک است؛ یعنی queueها کوتاه‌اند. اگر broker به‌دلیل کندی consumer مجبور شود پیام‌های زیادی را buffer کند ــ و شاید پیام‌هایی را که در memory جا نمی‌شوند روی disk بریزد ــ پردازش هر پیام طولانی‌تر می‌شود و throughput کلی ممکن است افت کند.
- databaseها اغلب secondary index و روش‌های متنوع جست‌وجوی data دارند؛ message brokerها معمولاً راهی برای subscribe شدن به زیرمجموعه‌ای از topicها بر اساس یک pattern فراهم می‌کنند. سازوکارها متفاوت‌اند، اما هر دو راهی هستند تا client بخش موردنیاز data را انتخاب کند.
- نتیجهٔ query در database معمولاً بر اساس snapshotی از data در یک لحظهٔ مشخص است. اگر client دیگری بعداً database را تغییر دهد و نتیجهٔ query عوض شود، client اول از stale شدن نتیجهٔ قبلی باخبر نمی‌شود؛ مگر اینکه query را تکرار کند یا تغییرها را poll کند. message broker، در مقابل، query دلخواه پشتیبانی نمی‌کند، اما با تغییر data ــ یعنی در دسترس قرارگرفتن پیام‌های تازه ــ به clientها notification می‌دهد.

این دیدگاه سنتی دربارهٔ message brokerها در standardهایی مانند JMS و AMQP بازتاب یافته و در نرم‌افزارهایی مانند RabbitMQ، ActiveMQ، HornetQ، Qpid، TIBCO Enterprise Message Service، IBM MQ، Azure Service Bus و Google Cloud Pub/Sub پیاده‌سازی شده است.

#### چند consumer

وقتی چند consumer پیام‌های یک topic را می‌خوانند، دو الگوی اصلی messaging به کار می‌رود:

**Load balancing:** هر پیام به یکی از consumerها تحویل داده می‌شود تا آن‌ها کار پردازش پیام‌های topic را میان خود تقسیم کنند. broker ممکن است پیام‌ها را به‌صورت دلخواه به consumerها اختصاص دهد. این الگو وقتی مفید است که پردازش پیام‌ها پرهزینه باشد و بخواهید با افزودن consumer، پردازش را parallel کنید. در AMQP می‌توان با مصرف چند client از یک queue load balancing را پیاده‌سازی کرد؛ در JMS این الگو shared subscription نام دارد.

**Fan-out:** هر پیام به همهٔ consumerها تحویل داده می‌شود. Fan-out به چند consumer مستقل اجازه می‌دهد به یک broadcast پیام گوش کنند، بی‌آنکه بر کار یکدیگر اثر بگذارند؛ معادل streamیِ چند batch job است که یک input file یکسان را می‌خوانند. در JMS این قابلیت با topic subscription و در AMQP با exchange binding فراهم می‌شود.

این دو الگو را می‌توان ترکیب کرد: مثلاً دو گروه جدا از consumerها می‌توانند هر دو به یک topic subscribe شوند؛ همهٔ پیام‌ها به هر دو گروه می‌رسند، اما درون هر گروه هر پیام را فقط یکی از nodeها دریافت می‌کند.

#### Acknowledgment و تحویل دوباره

consumerها ممکن است هر لحظه crash کنند. ممکن است broker پیامی را به consumer تحویل دهد، اما consumer آن را اصلاً پردازش نکند یا پیش از crash فقط بخشی از پردازش را انجام دهد. برای اینکه پیام از دست نرود، message brokerها از acknowledgment استفاده می‌کنند: client باید پس از پایان پردازش پیام صریحاً به broker اطلاع دهد تا broker بتواند آن را از queue حذف کند.

اگر connection به client بسته شود یا timeout شود و broker acknowledgmentی دریافت نکرده باشد، فرض می‌کند پیام پردازش نشده است و آن را دوباره به consumer دیگری تحویل می‌دهد. البته ممکن است پیام کاملاً پردازش شده باشد، اما acknowledgment در network گم شده باشد. رسیدگی به این حالت به atomic commit protocol نیاز دارد.

ترکیب redelivery با load balancing پیامد جالبی برای ترتیب پیام‌ها دارد. معمولاً consumerها پیام‌ها را به همان ترتیبی پردازش می‌کنند که producerها فرستاده‌اند. اما فرض کنید consumer 2 هنگام پردازش پیام `m3` crash کند و هم‌زمان consumer 1 در حال پردازش `m4` باشد. پیام `m3` که acknowledgment نشده، بعداً به consumer 1 تحویل داده می‌شود؛ در نتیجه consumer 1 پیام‌ها را به ترتیب `m4`، `m3`، `m5` پردازش می‌کند، نه به همان ترتیبی که producer 1 فرستاده بود.

حتی اگر broker تلاش کند ترتیب پیام‌ها را حفظ کند ــ چنان‌که standardهای JMS و AMQP هر دو می‌خواهند ــ ترکیب load balancing و redelivery ناگزیر به جابه‌جایی ترتیب می‌انجامد. برای پرهیز از این مسئله می‌توان برای هر consumer queue جداگانه‌ای داشت و از قابلیت load balancing استفاده نکرد. جابه‌جایی ترتیب وقتی پیام‌ها کاملاً مستقل باشند مشکل‌ساز نیست، اما اگر میانشان dependency علّی وجود داشته باشد اهمیت پیدا می‌کند؛ موضوعی که بعدتر در chapter به آن برمی‌گردیم.

### Partitioned Logs

فرستادن packet روی network یا request زدن به network service معمولاً عملیاتی گذراست و ردپای دائمی باقی نمی‌گذارد. البته می‌توان آن را به‌طور پایدار ثبت کرد، مثلاً با packet capture و logging، اما معمولاً این‌طور به آن نگاه نمی‌کنیم. حتی message brokerهایی که پیام‌ها را به‌طور durable روی disk می‌نویسند، پس از تحویل به consumer سریع حذفشان می‌کنند؛ چون بر اساس ذهنیت messaging گذرا ساخته شده‌اند.

database و filesystem رویکردی متضاد دارند: انتظار می‌رود هر چیزی که در database یا file نوشته می‌شود دائماً ثبت بماند، دست‌کم تا زمانی که کسی صریحاً آن را حذف کند.

این تفاوت در رویکرد بر نحوهٔ تولید derived data اثر زیادی دارد. یکی از ویژگی‌های اصلی batch processها این است که می‌توان آن‌ها را بارها اجرا کرد و مراحل پردازش را آزمود، بی‌آنکه input در معرض تغییر باشد؛ چون input فقط خواندنی است. در messaging به سبک AMQP/JMS چنین نیست: دریافت پیام عملیاتی مخرب است، چون acknowledgment باعث حذف پیام از broker می‌شود. بنابراین نمی‌توان همان consumer را دوباره اجرا کرد و انتظار داشت همان پیام‌ها را دریافت کند.

اگر consumer تازه‌ای به messaging system اضافه کنید، معمولاً فقط پیام‌هایی را می‌گیرد که پس از ثبت‌شدنش ارسال شده‌اند؛ پیام‌های قبلی حذف شده‌اند و دیگر قابل‌بازیابی نیستند. در مقابل، می‌توان هر زمان client تازه‌ای به fileها و databaseها اضافه کرد و dataهایی را که حتی مدت‌ها پیش نوشته شده‌اند خواند، به شرط آنکه application آن‌ها را overwrite یا حذف نکرده باشد.

چرا رویکردی ترکیبی نداشته باشیم که storage پایدار database را با قابلیت notification کم‌تأخیر messaging ترکیب کند؟ ایدهٔ **log-based message broker** همین است.

#### استفاده از log برای ذخیرهٔ پیام

log دنباله‌ای از recordها روی disk است که فقط به انتهایش append می‌شود. پیش‌تر در Chapter 3 دربارهٔ log در زمینهٔ log-structured storage engineها و write-ahead logها، و در Chapter 5 در زمینهٔ replication صحبت کردیم.

از همین ساختار می‌توان برای پیاده‌سازی message broker استفاده کرد: producer پیام را با append کردن به انتهای log می‌فرستد و consumer با خواندن ترتیبی log پیام را دریافت می‌کند. وقتی consumer به انتهای log برسد، منتظر notification می‌ماند تا پیام تازه‌ای append شود. ابزار Unix به نام `tail -f` که file را برای dataهای appendشده زیر نظر می‌گیرد، اساساً همین‌طور کار می‌کند.

برای رسیدن به throughput بیشتر از ظرفیت یک disk، log را می‌توان partition کرد. partitionهای متفاوت را می‌توان روی machineهای متفاوت قرار داد؛ در این صورت هر partition یک log جداگانه است که مستقل از بقیه خوانده و نوشته می‌شود. سپس می‌توان topic را مجموعه‌ای از partitionها تعریف کرد که همگی پیام‌هایی از یک نوع دارند.

درون هر partition، broker به هر پیام یک sequence number یا **offset** می‌دهد که به‌طور یکنواخت افزایش می‌یابد. چون partition فقط append می‌شود، پیام‌های داخل آن ترتیب کلی دارند؛ اما میان partitionهای مختلف هیچ تضمین ترتیبی وجود ندارد.

Apache Kafka، Amazon Kinesis Streams و DistributedLog متعلق به Twitter، message brokerهای مبتنی بر log هستند که به این شکل کار می‌کنند. Google Cloud Pub/Sub از نظر معماری مشابه است، اما به‌جای abstraction مبتنی بر log، APIای به سبک JMS ارائه می‌دهد. با اینکه این brokerها همهٔ پیام‌ها را روی disk می‌نویسند، با partition کردن میان چند machine می‌توانند به throughput چندمیلیون پیام در ثانیه و با replication پیام‌ها به fault tolerance دست یابند.

#### مقایسهٔ log با messaging سنتی

رویکرد log به‌سادگی از fan-out پشتیبانی می‌کند، چون چند consumer می‌توانند log را مستقل بخوانند و خواندن پیام آن را از log حذف نمی‌کند. برای load balancing میان گروهی از consumerها، broker به‌جای تخصیص پیام‌های جداگانه به clientها، می‌تواند کل partitionها را میان nodeهای consumer group تقسیم کند.

هر client همهٔ پیام‌های partitionهایی را که به آن اختصاص داده شده‌اند مصرف می‌کند. معمولاً consumer پیام‌های یک log partition را به‌ترتیب و به‌شکل سادهٔ تک‌threadی می‌خواند. این روش coarse-grained برای load balancing چند عیب دارد:

- تعداد nodeهایی که کار مصرف پیام‌های یک topic را تقسیم می‌کنند حداکثر برابر تعداد log partitionهای آن topic است؛ چون پیام‌های یک partition به یک node تحویل داده می‌شوند.
- اگر پردازش یک پیام کند باشد، پردازش پیام‌های بعدی همان partition را عقب می‌اندازد؛ نوعی head-of-line blocking.

بنابراین، وقتی پردازش پیام‌ها ممکن است پرهزینه باشد، می‌خواهید هر پیام را جداگانه parallel پردازش کنید و ترتیب پیام‌ها اهمیت زیادی ندارد، message broker به سبک JMS/AMQP مناسب‌تر است. اما در throughput بالای پیام، وقتی پردازش هر پیام سریع است و حفظ ترتیب اهمیت دارد، روش مبتنی بر log بسیار خوب عمل می‌کند.

#### Consumer offsetها

مصرف ترتیبی یک partition تشخیص پیام‌های پردازش‌شده را آسان می‌کند: همهٔ پیام‌هایی که offsetشان از offset فعلی consumer کمتر است پردازش شده‌اند و پیام‌هایی با offset بزرگ‌تر هنوز دیده نشده‌اند. بنابراین broker لازم نیست acknowledgment هر پیام را جداگانه track کند؛ کافی است offsetهای consumer را به‌صورت دوره‌ای ثبت کند. سربار bookkeeping کمتر و امکان batch و pipeline کردن، throughput سیستم‌های مبتنی بر log را افزایش می‌دهد.

این offset در واقع بسیار شبیه log sequence number در replication یک database تک‌leader است. در database replication، follower پس از قطع اتصال می‌تواند با استفاده از log sequence number دوباره به leader وصل شود و replication را بدون جاانداختن writeها ادامه دهد. اینجا هم اصل دقیقاً همین است: message broker مانند databaseای با نقش leader رفتار می‌کند و consumer مانند follower.

اگر consumer nodeای از کار بیفتد، node دیگری در consumer group partitionهای consumer شکست‌خورده را تحویل می‌گیرد و از آخرین offset ثبت‌شده به مصرف پیام‌ها ادامه می‌دهد. اگر consumer پیام‌های بعدی را پردازش کرده باشد اما هنوز offsetشان را ثبت نکرده باشد، پس از restart آن پیام‌ها دوباره پردازش می‌شوند. بعدتر در chapter راه‌های رسیدگی به این مسئله را بررسی می‌کنیم.

#### مصرف disk

اگر فقط به انتهای log اضافه کنید، سرانجام فضای disk تمام می‌شود. برای آزادکردن فضا، log را به segmentهایی تقسیم می‌کنند و هر از گاهی segmentهای قدیمی را حذف یا به archive storage منتقل می‌کنند.

اگر consumer کندتر از نرخ رسیدن پیام‌ها باشد و آن‌قدر عقب بیفتد که offsetش به segment حذف‌شده‌ای اشاره کند، بخشی از پیام‌ها را از دست می‌دهد. در عمل، log یک buffer با اندازهٔ محدود است که وقتی پر شود پیام‌های قدیمی را دور می‌ریزد؛ این ساختار circular buffer یا ring buffer هم نام دارد. اما چون buffer روی disk است، می‌تواند بسیار بزرگ باشد.

یک محاسبهٔ سرانگشتی: در زمان نگارش کتاب، ظرفیت یک hard drive بزرگ معمولی ۶ TB و throughput نوشتن ترتیبی آن ۱۵۰ MB/s بود. اگر با بیشترین سرعت ممکن پیام بنویسید، پرشدن drive حدود ۱۱ ساعت طول می‌کشد. پس disk می‌تواند پیام‌های حدود ۱۱ ساعت را buffer کند و پس از آن overwrite پیام‌های قدیمی را آغاز می‌کند. این نسبت حتی با استفاده از driveها و machineهای بیشتر نیز ثابت می‌ماند. در عمل، deploymentها به‌ندرت از تمام پهنای باند نوشتن disk استفاده می‌کنند؛ بنابراین log معمولاً می‌تواند buffer چند روز یا حتی چند هفته پیام را نگه دارد.

فارغ از مدت نگهداری پیام‌ها، throughput یک log کمابیش ثابت می‌ماند، چون هر پیام در هر حال روی disk نوشته می‌شود. این با messaging systemهایی فرق دارد که به‌طور پیش‌فرض پیام‌ها را در memory نگه می‌دارند و فقط با بزرگ‌شدن بیش‌ازحد queue آن‌ها را روی disk می‌نویسند: آن سیستم‌ها وقتی queue کوتاه است سریع‌اند و با شروع نوشتن روی disk کندتر می‌شوند؛ بنابراین throughputشان به مقدار history نگهداری‌شده وابسته است.

#### وقتی consumerها از producerها عقب می‌مانند

در «Messaging Systems» سه انتخاب را مطرح کردیم: اگر consumer نتواند با نرخ ارسال producerها هماهنگ شود، پیام‌ها را drop کنیم، buffer کنیم یا backpressure اعمال کنیم. در این دسته‌بندی، روش مبتنی بر log نوعی buffering با buffer بزرگ اما محدود است؛ محدودیتش را فضای disk موجود تعیین می‌کند.

اگر consumer آن‌قدر عقب بیفتد که پیام‌های موردنیازش قدیمی‌تر از data نگهداری‌شده روی disk باشند، دیگر نمی‌تواند آن پیام‌ها را بخواند. در نتیجه broker پیام‌هایی را که از ظرفیت buffer قدیمی‌تر شده‌اند عملاً حذف می‌کند. می‌توان میزان عقب‌ماندگی consumer نسبت به ابتدای log را پایش کرد و اگر زیاد شد alert داد. چون buffer بزرگ است، معمولاً operator انسانی فرصت دارد consumer کند را اصلاح کند تا پیش از ازدست‌رفتن پیام‌ها دوباره به‌روز شود.

حتی اگر consumer بیش‌ازحد عقب بیفتد و پیام‌هایی را از دست بدهد، فقط همان consumer آسیب می‌بیند و سرویس consumerهای دیگر مختل نمی‌شود. این مزیت عملیاتی مهمی است: می‌توان برای توسعه، test یا debugging یک log مربوط به production را آزمایشی مصرف کرد، بی‌آنکه نگرانی زیادی از اختلال در سرویس‌های production داشت. وقتی consumer خاموش یا crash می‌شود، دیگر resource مصرف نمی‌کند و فقط consumer offset آن باقی می‌ماند.

این رفتار با message brokerهای سنتی هم تفاوت دارد؛ در آن‌ها باید queueهای consumerهای خاموش‌شده را حذف کرد، وگرنه بی‌دلیل پیام جمع می‌کنند و memory را از consumerهای فعال می‌گیرند.

#### بازخوانی پیام‌های قدیمی

در message brokerهای سبک AMQP و JMS، پردازش و acknowledgment پیام عملیاتی مخرب است، چون پیام را از broker حذف می‌کند. در message broker مبتنی بر log، مصرف پیام بیشتر شبیه خواندن file است: عملیاتی فقط‌خواندنی است و log را تغییر نمی‌دهد.

تنها side effect پردازش، جدا از output خود consumer، جلو رفتن consumer offset است. اما offset تحت کنترل consumer است و در صورت نیاز به‌آسانی می‌توان آن را تغییر داد. مثلاً می‌توان یک نسخه از consumer را با offsetهای روز قبل اجرا کرد و output را در محل دیگری نوشت تا پیام‌های یک روز گذشته دوباره پردازش شوند. این کار را می‌توان هر چند بار خواست تکرار کرد و در هر بار code پردازش را تغییر داد.

از این نظر، messaging مبتنی بر log به batch processهای chapter قبل شبیه است: derived data با transformationای تکرارپذیر از input data جدا می‌ماند. این ویژگی امکان آزمایش بیشتر و recovery آسان‌تر از errorها و bugها را فراهم می‌کند و log-based messaging را به ابزاری مناسب برای یکپارچه‌کردن dataflowها درون یک سازمان تبدیل می‌کند.

## Key Terms

- `Event Stream` — دنباله‌ای از eventهای مستقل که به‌تدریج در طول زمان تولید و برای پردازش در دسترس قرار می‌گیرند.
- `Producer` — componentی که event یا message را تولید و به stream یا broker می‌فرستد.
- `Consumer` — componentی که پیام‌ها یا eventهای stream را دریافت و پردازش می‌کند.
- `Publish/Subscribe` — الگوی messaging که در آن producer پیام را برای topic منتشر می‌کند و consumerهای subscribeشده آن را دریافت می‌کنند.
- `Message Acknowledgment` — تأیید صریح consumer به broker پس از پردازش پیام، برای جلوگیری از حذف زودهنگام آن.
- `Message Redelivery` — تحویل دوبارهٔ پیام تأییدنشده، معمولاً پس از قطع connection یا timeout consumer.
- `Topic` — دسته یا stream نام‌گذاری‌شده‌ای که پیام‌های مرتبط را گروه‌بندی می‌کند.
- `Partitioned Log` — log append-only تقسیم‌شده به partitionهای مستقل برای افزایش throughput و توزیع بار؛ ترتیب فقط درون هر partition تضمین می‌شود.
- `Consumer Offset` — موقعیت پیام در partition که consumer تا آنجا پیش رفته یا پردازش کرده است.
- `Message Replay` — مصرف دوبارهٔ پیام‌های قدیمی با عقب‌بردن offset، بدون تغییر یا حذف خود log.
- `Append-Only Log` — دنبالهٔ پایدار recordها که write جدید فقط به انتهای آن اضافه می‌شود.

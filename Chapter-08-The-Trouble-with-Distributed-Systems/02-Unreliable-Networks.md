# Chapter 8 — The Trouble with Distributed Systems

## Unreliable Networks

همان‌طور که در مقدمهٔ بخش دوم کتاب گفتیم، distributed systemهایی که در این کتاب بررسی می‌کنیم، **shared-nothing** هستند؛ یعنی مجموعه‌ای از machineها که از طریق یک network به هم متصل شده‌اند. Network تنها راه ارتباط این machineهاست—فرض می‌کنیم هر machine memory و disk خودش را دارد و یک machine نمی‌تواند مستقیماً به memory یا disk machine دیگری دسترسی پیدا کند؛ مگر اینکه از طریق network، از یک service درخواست بفرستد.

Shared-nothing تنها روش ساخت system نیست، اما به چند دلیل به رویکرد غالب برای ساخت internet serviceها تبدیل شده است: نسبتاً ارزان است، چون به hardware ویژه‌ای نیاز ندارد؛ می‌تواند از cloud computing serviceهای commodity استفاده کند؛ و با ایجاد redundancy در چند datacenter جغرافیایی، می‌تواند reliability بالایی به دست آورد.

Internet و بیشتر networkهای داخلی datacenterها—که اغلب Ethernet هستند—**asynchronous packet network** محسوب می‌شوند. در این نوع network، یک node می‌تواند برای node دیگری message (یا packet) بفرستد، اما network هیچ تضمینی دربارهٔ زمان رسیدن آن یا حتی رسیدن آن به‌طور کلی ارائه نمی‌دهد. اگر requestی بفرستید و منتظر response باشید، ممکن است اتفاق‌های مختلفی رخ دهد؛ بعضی از آن‌ها در Figure 8-1 نشان داده شده‌اند:

1. ممکن است request گم شده باشد؛ مثلاً کسی کابل network را از جا کشیده باشد.
2. ممکن است request در یک queue منتظر مانده باشد و بعداً تحویل داده شود؛ مثلاً network یا دریافت‌کننده overload شده باشد.
3. ممکن است remote node از کار افتاده باشد؛ مثلاً crash کرده یا خاموش شده باشد.
4. ممکن است remote node موقتاً دیگر پاسخ ندهد؛ مثلاً یک garbage collection pause طولانی را تجربه کند، اما بعداً دوباره شروع به پاسخ‌گویی کند.
5. ممکن است remote node request شما را پردازش کرده باشد، اما response در network گم شده باشد؛ مثلاً یک network switch اشتباه configure شده باشد.
6. ممکن است remote node request شما را پردازش کرده باشد، اما response با تأخیر برسد و بعداً تحویل داده شود؛ مثلاً network یا machine خودتان overload شده باشد.

**Figure 8-1.** اگر requestی ارسال کنید و response نگیرید، نمی‌توان تشخیص داد که (a) request گم شده، (b) remote node از کار افتاده، یا (c) response گم شده است.

فرستنده حتی نمی‌تواند بفهمد packet تحویل داده شده است یا نه: تنها راه این است که گیرنده یک response message بفرستد؛ اما خود این response نیز ممکن است گم شود یا با تأخیر برسد. در یک asynchronous network، این حالت‌ها از یکدیگر قابل تشخیص نیستند: تنها اطلاعاتی که دارید این است که هنوز response را دریافت نکرده‌اید. اگر requestی برای node دیگری بفرستید و response دریافت نکنید، تشخیص علت آن غیرممکن است.

روش معمول برخورد با این مسئله، استفاده از **timeout** است: پس از گذشت مدت مشخصی، دیگر منتظر نمی‌مانید و فرض می‌کنید response قرار نیست برسد. بااین‌حال، هنگام رخ دادن timeout هنوز نمی‌دانید remote node request شما را دریافت کرده است یا نه. اگر request هنوز جایی در queue مانده باشد، ممکن است حتی پس از آنکه فرستنده از انتظار منصرف شده، به گیرنده تحویل داده شود.

### Network Faults in Practice

ما چند دهه است که computer network می‌سازیم؛ شاید انتظار داشته باشیم تا امروز راهی برای reliable کردن آن‌ها پیدا کرده باشیم. بااین‌حال، ظاهراً هنوز موفق نشده‌ایم.

مطالعه‌های نظام‌مند و شواهد تجربی فراوانی نشان می‌دهند که network problemها حتی در محیط‌های کنترل‌شده‌ای مانند datacenterی که یک شرکت اداره می‌کند، می‌توانند surprisingly common باشند [14]. در یک مطالعه روی datacenterی متوسط، حدود ۱۲ network fault در ماه ثبت شد؛ نیمی از آن‌ها یک machine را قطع می‌کردند و نیمی دیگر یک rack کامل را از دسترس خارج می‌کردند [15]. مطالعهٔ دیگری failure rate مربوط به componentهایی مانند top-of-rack switchها، aggregation switchها و load balancerها را اندازه‌گیری کرد [16]. نتیجه نشان داد اضافه کردن network equipmentهای redundant، آن‌قدر که انتظار دارید faultها را کم نمی‌کند، چون در برابر human error—برای مثال، misconfigure شدن switchها—محافظتی ندارد؛ درحالی‌که human error یکی از علت‌های اصلی outage است.

Public cloud serviceهایی مانند EC2 به داشتن transient network glitchهای مکرر مشهورند [14] و networkهای private datacenter که به‌خوبی مدیریت می‌شوند، ممکن است محیط باثبات‌تری باشند. بااین‌حال، هیچ‌کس از network problemها مصون نیست. برای مثال، یک مشکل هنگام software upgrade یک switch می‌تواند network topology را دوباره configure کند و در این مدت، packetهای network بیش از یک دقیقه با تأخیر روبه‌رو شوند [17]. حتی کوسه‌ها ممکن است کابل‌های زیر دریا را گاز بگیرند و به آن‌ها آسیب بزنند [18]. faultهای غیرمنتظرهٔ دیگری نیز وجود دارند؛ مثلاً network interfaceای که گاهی تمام inbound packetها را drop می‌کند، اما outbound packetها را با موفقیت می‌فرستد [19]. بنابراین، اینکه network link در یک جهت کار می‌کند، تضمین نمی‌کند که در جهت مخالف نیز کار کند.

#### Network partitions

وقتی به‌دلیل یک network fault، بخشی از network از بقیه جدا می‌شود، گاهی به آن **network partition** یا **netsplit** گفته می‌شود. در این کتاب معمولاً از اصطلاح عمومی‌تر network fault استفاده می‌کنیم تا با partition (shard)های یک storage system که در Chapter 6 بررسی شد، اشتباه نشود.

حتی اگر network faultها در محیط شما نادر باشند، همین واقعیت که امکان رخ دادن آن‌ها وجود دارد یعنی software شما باید بتواند با آن‌ها برخورد کند. هر زمان که ارتباطی از طریق network برقرار می‌شود، امکان failure آن وجود دارد و راهی برای دور زدن این واقعیت نیست.

اگر رفتار سیستم در برابر network faultها تعریف و test نشده باشد، ممکن است اتفاق‌های بسیار بدی رخ دهد: برای مثال، cluster ممکن است deadlock شود و حتی پس از recovery network، برای همیشه نتواند requestها را serve کند [20]؛ یا حتی ممکن است تمام data شما را حذف کند [21]. اگر software در موقعیتی پیش‌بینی‌نشده قرار بگیرد، ممکن است رفتارهای دلخواه و غیرقابل‌انتظاری نشان دهد.

برخورد با network fault الزاماً به‌معنای tolerate کردن آن نیست. اگر network شما معمولاً نسبتاً reliable است، یک رویکرد معتبر می‌تواند این باشد که هنگام بروز مشکل network، فقط error messageای به user نشان دهید. بااین‌حال، باید بدانید software شما در برابر network problemها چه واکنشی نشان می‌دهد و مطمئن شوید system می‌تواند از آن‌ها recover کند. حتی ممکن است منطقی باشد که عمداً network problem ایجاد کنید و response system را test کنید؛ این همان ایده‌ای است که پشت **Chaos Monkey** قرار دارد؛ برای توضیح بیشتر به بخش «Reliability» در صفحهٔ ۶ مراجعه کنید.

### Detecting Faults

بسیاری از systemها باید nodeهای faulty را به‌صورت خودکار detect کنند. برای مثال:

- یک load balancer باید ارسال request به nodeای را که از کار افتاده متوقف کند؛ یعنی آن node را از rotation خارج کند.
- در یک distributed database با single-leader replication، اگر leader fail شود، یکی از followerها باید به‌عنوان leader جدید promote شود؛ برای نمونه به بخش «Handling Node Outages» در صفحهٔ ۱۵۶ مراجعه کنید.

متأسفانه، uncertainty موجود در network تشخیص اینکه یک node در حال کار است یا نه را دشوار می‌کند. در بعضی شرایط خاص ممکن است feedbackی دریافت کنید که به‌طور صریح نشان دهد چیزی درست کار نمی‌کند:

- اگر بتوانید به machineای که node باید روی آن اجرا شود دسترسی پیدا کنید، اما هیچ processای روی destination port در حال listen کردن نباشد—مثلاً چون process crash کرده است—operating system با ارسال packetی از نوع RST یا FIN در پاسخ، TCP connection را به‌درستی می‌بندد یا آن را رد می‌کند. بااین‌حال، اگر node هنگام پردازش request شما crash کرده باشد، هیچ راهی ندارید بفهمید remote node چه مقدار از data را واقعاً پردازش کرده است [22].
- اگر node process crash کند—یا administrator آن را kill کند—اما operating system خود node همچنان در حال اجرا باشد، یک script می‌تواند nodeهای دیگر را از crash مطلع کند تا node دیگری بتواند سریع takeover کند و لازم نباشد منتظر پایان timeout بماند. برای مثال، HBase چنین کاری انجام می‌دهد [23].
- اگر به management interface مربوط به network switchهای datacenter دسترسی داشته باشید، می‌توانید آن‌ها را query کنید تا link failure را در سطح hardware تشخیص دهید؛ مثلاً بفهمید machine مقابل خاموش شده است. اگر از طریق internet متصل باشید، یا در shared datacenterی باشید که به خود switchها دسترسی ندارید، یا به‌دلیل network problem نتوانید به management interface برسید، این گزینه وجود ندارد.
- اگر router مطمئن باشد IP addressای که می‌خواهید به آن متصل شوید unreachable است، ممکن است با یک **ICMP Destination Unreachable** packet به شما پاسخ دهد. بااین‌حال، router نیز قابلیت جادویی برای failure detection ندارد و همان محدودیت‌های سایر participantهای network را دارد.

دریافت سریع feedback دربارهٔ از کار افتادن remote node مفید است، اما نمی‌توانید روی آن حساب کنید. حتی اگر TCP تأیید کند که packet تحویل داده شده است، ممکن است application پیش از پردازش آن crash کرده باشد. اگر می‌خواهید مطمئن شوید request موفق بوده، به response مثبت خود application نیاز دارید [24].

برعکس، اگر مشکلی رخ داده باشد، ممکن است در سطحی از stack یک error response دریافت کنید؛ اما به‌طور کلی باید فرض کنید که اصلاً responseای دریافت نخواهید کرد. می‌توانید چند بار retry کنید—TCP این کار را به‌صورت transparent انجام می‌دهد، اما ممکن است در application level نیز retry کنید—برای پایان timeout صبر کنید و در نهایت، اگر در طول timeout پاسخی دریافت نکردید، node را dead اعلام کنید.

### Timeouts and Unbounded Delays

اگر timeout تنها راه مطمئن برای تشخیص fault باشد، timeout باید چقدر طول بکشد؟ متأسفانه پاسخ ساده‌ای وجود ندارد.

timeout طولانی یعنی برای اعلام dead بودن یک node باید مدت زیادی صبر کنید؛ در این مدت userها ممکن است منتظر بمانند یا error message ببینند. timeout کوتاه fault را سریع‌تر detect می‌کند، اما خطر اینکه nodeای را که فقط دچار slowdown موقت شده dead اعلام کنید بیشتر است؛ مثلاً slowdown ناشی از load spike روی node یا network.

اعلام زودهنگام dead بودن node مشکل‌ساز است. اگر node واقعاً زنده باشد و در حال انجام کاری—مثلاً ارسال email—باشد و node دیگری takeover کند، ممکن است آن action دو بار انجام شود. این مسئله را در بخش «Knowledge, Truth, and Lies» در صفحهٔ ۳۰۰ و در Chapterهای ۹ و ۱۱ با جزئیات بیشتری بررسی می‌کنیم.

وقتی nodeای dead اعلام می‌شود، responsibilityهای آن باید به nodeهای دیگر منتقل شوند؛ این کار load اضافی روی nodeها و network قرار می‌دهد. اگر system از قبل با load بالا درگیر باشد، اعلام زودهنگام dead بودن nodeها می‌تواند مشکل را بدتر کند. به‌طور خاص، ممکن است node واقعاً dead نباشد و فقط به‌دلیل overload، کند پاسخ دهد؛ انتقال load آن به nodeهای دیگر می‌تواند به **cascading failure** منجر شود. در حالت شدید، همهٔ nodeها یکدیگر را dead اعلام می‌کنند و کل system از کار می‌افتد.

یک system فرضی را تصور کنید که network آن maximum delay مشخصی برای packetها تضمین می‌کند: هر packet یا حداکثر در مدت `d` تحویل داده می‌شود یا گم می‌شود، اما delivery آن هرگز بیشتر از `d` طول نمی‌کشد. همچنین فرض کنید بتوانید تضمین کنید که یک nodeی که fail نشده، همیشه request را حداکثر در مدت `r` پردازش می‌کند. در این صورت، می‌توانید تضمین کنید که هر request موفق حداکثر در مدت `2d + r` response دریافت می‌کند. اگر در این مدت response نگیرید، می‌دانید که یا network یا remote node درست کار نمی‌کند. در چنین شرایطی، `2d + r` timeout منطقی‌ای خواهد بود.

متأسفانه بیشتر systemهایی که با آن‌ها کار می‌کنیم هیچ‌کدام از این guaranteeها را ندارند: asynchronous networkها delay نامحدود دارند؛ یعنی تلاش می‌کنند packetها را هرچه سریع‌تر تحویل دهند، اما برای زمان رسیدن packet هیچ upper boundای وجود ندارد. علاوه بر این، بیشتر server implementationها نمی‌توانند تضمین کنند که requestها را در حداکثر زمان مشخصی پردازش می‌کنند؛ برای توضیح بیشتر به بخش «Response time guarantees» در صفحهٔ ۲۹۸ مراجعه کنید.

برای failure detection، سریع بودن system در بیشتر مواقع کافی نیست. اگر timeout شما کم باشد، تنها یک transient spike در round-trip time کافی است تا system را از تعادل خارج کند.

#### Network congestion and queueing

هنگام رانندگی، زمان سفر در road networkها اغلب به‌دلیل traffic congestion تغییر می‌کند. در computer networkها نیز بیشترین variability در packet delay معمولاً ناشی از queueing است [25]:

- اگر چند node مختلف هم‌زمان بخواهند packetهایی را برای یک مقصد بفرستند، network switch باید آن‌ها را queue کند و یکی‌یکی وارد network link مقصد کند؛ همان‌طور که در Figure 8-2 نشان داده شده است. در یک network link شلوغ، ممکن است packet مدتی منتظر بماند تا نوبت ارسالش برسد؛ به این حالت network congestion گفته می‌شود. اگر data ورودی آن‌قدر زیاد باشد که queue مربوط به switch پر شود، packet drop می‌شود و باید دوباره ارسال شود؛ حتی اگر network در اصل به‌درستی کار کند.
- وقتی packet به machine مقصد می‌رسد، اگر همهٔ CPU coreها مشغول باشند، operating system درخواست ورودی network را queue می‌کند تا application آمادهٔ پردازش آن شود. بسته به load machine، این انتظار می‌تواند مدت دلخواهی طول بکشد.
- در محیط‌های virtualized، operating system در حال اجرا اغلب برای ده‌ها millisecond متوقف می‌شود تا virtual machine دیگری از یک CPU core استفاده کند. در این مدت، VM نمی‌تواند dataای از network مصرف کند؛ بنابراین incoming data توسط virtual machine monitor به‌صورت queue یا buffer نگه داشته می‌شود [26] و variability مربوط به network delay بیشتر می‌شود.
- TCP **flow control** را انجام می‌دهد—که congestion avoidance یا backpressure نیز نامیده می‌شود—و در آن، یک node rate ارسال خودش را محدود می‌کند تا network link یا node دریافت‌کننده overload نشود [27]. در نتیجه، پیش از آنکه data حتی وارد network شود، در sender نیز queueing اضافی ایجاد می‌شود.

**Figure 8-2.** اگر چند machine برای یک مقصد network traffic بفرستند، queue مربوط به switch آن مقصد ممکن است پر شود. در این شکل، portهای 1، 2 و 4 همگی تلاش می‌کنند packetهایی را به port 3 بفرستند.

علاوه بر این، TCP وقتی packetی را lost در نظر می‌گیرد که در مدت timeout مشخصی—که از round-trip timeهای مشاهده‌شده محاسبه می‌شود—acknowledge نشده باشد. سپس packetهای گم‌شده را به‌صورت خودکار retransmit می‌کند. اگرچه application loss و retransmission packet را نمی‌بیند، delay حاصل را مشاهده می‌کند: ابتدا باید تا پایان timeout صبر کند و سپس منتظر acknowledge شدن packet retransmit‌شده بماند.

#### TCP Versus UDP

بعضی applicationهای latency-sensitive مانند videoconferencing و Voice over IP (VoIP)، به‌جای TCP از UDP استفاده می‌کنند. این انتخاب trade-offی میان reliability و variability در delay است: UDP چون flow control انجام نمی‌دهد و packetهای گم‌شده را retransmit نمی‌کند، بعضی علت‌های variable network delay را حذف می‌کند؛ هرچند همچنان در معرض switch queue و scheduling delay قرار دارد.

UDP در شرایطی انتخاب خوبی است که dataای که با تأخیر برسد دیگر ارزش نداشته باشد. برای مثال، در یک تماس VoIP احتمالاً پیش از زمانی که data یک packet گم‌شده باید از بلندگو پخش شود، زمان کافی برای retransmit آن وجود ندارد. در این حالت، retransmit کردن packet فایده‌ای ندارد؛ application باید جای خالی زمانی آن packet را با سکوت پر کند—که باعث interruption کوتاهی در صدا می‌شود—و در stream به جلو برود. retry در لایهٔ انسانی اتفاق می‌افتد: «لطفاً دوباره تکرار می‌کنید؟ صدا برای لحظه‌ای قطع شد.»

همهٔ این عوامل در variability مربوط به network delay نقش دارند. Queueing delay به‌خصوص وقتی system به maximum capacity خود نزدیک است، range بسیار گسترده‌ای دارد: systemی که capacity خالی زیادی دارد می‌تواند queueها را به‌سرعت خالی کند، اما در systemی با utilization بالا، queueهای طولانی می‌توانند خیلی سریع شکل بگیرند.

در public cloud و multi-tenant datacenter، resourceها میان customerهای متعدد shared هستند: network linkها و switchها، و حتی network interface و CPU هر machine—وقتی روی virtual machine اجرا می‌شوند—به‌صورت shared استفاده می‌شوند. Batch workloadهایی مانند MapReduce به‌سادگی می‌توانند network linkها را saturate کنند. چون کنترلی روی استفادهٔ customerهای دیگر از resourceهای shared ندارید و دیدی نسبت به آن ندارید، network delay می‌تواند بسیار variable باشد؛ اگر کسی در نزدیکی شما—یک **noisy neighbor**—resource زیادی مصرف کند [28, 29].

در چنین محیط‌هایی، تنها می‌توانید timeout را به‌صورت تجربی انتخاب کنید: توزیع network round-trip time را در بازه‌ای طولانی و روی machineهای متعدد اندازه‌گیری کنید تا variability مورد انتظار delay را مشخص کنید. سپس با در نظر گرفتن ویژگی‌های application، trade-off مناسب میان failure detection delay و خطر premature timeout را تعیین کنید.

حتی بهتر است به‌جای استفاده از timeoutهای ثابت و configureشده، systemها response time و variability آن—یا **jitter**—را به‌طور پیوسته اندازه‌گیری کنند و timeoutها را بر اساس توزیع response time مشاهده‌شده به‌صورت خودکار تنظیم کنند. این کار با **Phi Accrual failure detector** ممکن است؛ برای مثال، Akka و Cassandra از آن استفاده می‌کنند [30, 31]. TCP retransmission timeoutها نیز به روشی مشابه کار می‌کنند [27].

### Synchronous Versus Asynchronous Networks

اگر می‌توانستیم روی network حساب کنیم که packetها را با maximum delay ثابتی تحویل می‌دهد و packetها را drop نمی‌کند، distributed systemها بسیار ساده‌تر می‌شدند. چرا نمی‌توانیم این مسئله را در سطح hardware حل کنیم و network را reliable بسازیم تا software مجبور نباشد نگران آن باشد؟

برای پاسخ به این پرسش، مقایسهٔ networkهای datacenter با traditional fixed-line telephone network—غیرسلولی و غیر VoIP—جالب است. telephone network بسیار reliable است: delayed audio frame و dropped call بسیار نادرند. برای computer networkها نیز داشتن چنین reliability و predictabilityای جذاب نیست؟

وقتی از telephone network تماس می‌گیرید، یک **circuit** ایجاد می‌شود: مقدار ثابتی از bandwidth تضمین‌شده برای آن call، در تمام مسیر میان دو تماس‌گیرنده، allocate می‌شود. این circuit تا پایان call برقرار می‌ماند [32]. برای مثال، یک ISDN network با rate ثابت ۴۰۰۰ frame در ثانیه کار می‌کند. هنگام برقراری call، در هر frame—در هر دو جهت—۱۶ bit فضا به آن اختصاص داده می‌شود. بنابراین، در تمام مدت call، هر طرف تضمین دارد که بتواند دقیقاً هر ۲۵۰ microsecond، ۱۶ bit audio data بفرستد [33, 34].

این نوع network **synchronous** است: حتی وقتی data از چند router عبور می‌کند، دچار queueing نمی‌شود، چون ۱۶ bit مربوط به call از قبل در hop بعدی network رزرو شده است. چون queueing وجود ندارد، maximum end-to-end latency network ثابت است. به این وضعیت **bounded delay** می‌گوییم.

#### آیا نمی‌توان network delay را قابل‌پیش‌بینی کرد؟

توجه کنید که circuit در telephone network با TCP connection تفاوت زیادی دارد: circuit مقدار ثابتی bandwidth رزروشده است که تا زمان برقرار بودن circuit هیچ فرد دیگری نمی‌تواند از آن استفاده کند؛ درحالی‌که packetهای TCP connection به‌صورت opportunistic از هر مقدار bandwidth موجود استفاده می‌کنند. می‌توانید یک block data با اندازهٔ متغیر—مثلاً email یا web page—به TCP بدهید و TCP تلاش می‌کند آن را در کوتاه‌ترین زمان ممکن منتقل کند. وقتی TCP connection idle است، از bandwidth استفاده نمی‌کند؛ مگر احتمالاً برای یک keepalive packet گاه‌به‌گاه، اگر TCP keepalive فعال باشد.

اگر datacenter networkها و internet، circuit-switched بودند، هنگام برقراری circuit می‌شد maximum round-trip time تضمین‌شده‌ای تعیین کرد. اما چنین نیست: Ethernet و IP، protocolهای packet-switched هستند که به‌دلیل queueing از delayهای نامحدود رنج می‌برند. این protocolها مفهوم circuit را ندارند.

چرا datacenter networkها و internet از packet switching استفاده می‌کنند؟ چون برای trafficهای bursty بهینه شده‌اند. circuit برای audio یا video call مناسب است؛ call به تعداد نسبتاً ثابتی bit در ثانیه در تمام مدت نیاز دارد. در مقابل، هنگام درخواست web page، ارسال email یا انتقال file، bandwidth requirement مشخصی نداریم؛ فقط می‌خواهیم operation هرچه سریع‌تر کامل شود.

اگر می‌خواستید fileای را با circuit منتقل کنید، باید یک bandwidth allocation را حدس می‌زدید. اگر مقدار را کم حدس می‌زدید، transfer بی‌دلیل کند می‌شد و capacity network بدون استفاده می‌ماند. اگر مقدار را زیاد حدس می‌زدید، circuit برقرار نمی‌شد، چون network نمی‌تواند circuitی ایجاد کند که allocation آن را تضمین نمی‌کند. در نتیجه، استفاده از circuit برای bursty data transfer باعث اتلاف capacity network و کند شدن غیرضروری transfer می‌شود. در مقابل، TCP rate انتقال data را به‌صورت dynamic با capacity موجود network تطبیق می‌دهد.

تلاش‌هایی برای ساخت networkهای hybrid که هم circuit switching و هم packet switching را پشتیبانی کنند انجام شده است؛ از جمله ATM [32]. InfiniBand نیز شباهت‌هایی دارد [35]: این technology در link layer، end-to-end flow control را پیاده‌سازی می‌کند و در نتیجه نیاز به queueing در network را کاهش می‌دهد؛ هرچند همچنان ممکن است به‌دلیل link congestion دچار delay شود [36]. با استفادهٔ دقیق از **quality of service (QoS)**—اولویت‌بندی و scheduling packetها—و **admission control**—rate-limiting فرستنده‌ها—می‌توان circuit switching را روی packet network شبیه‌سازی کرد یا delayای با bound آماری ارائه داد [25, 32].

#### Latency and Resource Utilization

به‌طور کلی‌تر، می‌توان variable delay را پیامد dynamic resource partitioning دانست.

فرض کنید سیمی میان دو telephone switch دارید که می‌تواند هم‌زمان حداکثر ۱۰٬۰۰۰ call را منتقل کند. هر circuit که روی این سیم switch می‌شود، یکی از slotهای call را اشغال می‌کند. بنابراین می‌توان سیم را resourceی دانست که میان حداکثر ۱۰٬۰۰۰ user هم‌زمان shared می‌شود. این resource به‌صورت static تقسیم شده است: حتی اگر در این لحظه تنها call روی سیم متعلق به شما باشد و ۹۹۹۹ slot دیگر استفاده نشده باشند، circuit شما همان مقدار ثابت bandwidth را دارد که در زمان استفادهٔ کامل از سیم داشت.

در مقابل، internet bandwidth شبکه را به‌صورت dynamic share می‌کند. فرستنده‌ها برای رساندن packetهای خود روی سیم با یکدیگر رقابت و تلاش می‌کنند و network switchها از لحظه‌ای به لحظهٔ دیگر تصمیم می‌گیرند کدام packet ارسال شود؛ یعنی bandwidth چگونه allocate شود. این رویکرد عیب queueing را دارد، اما مزیتش maximum کردن utilization سیم است. هزینهٔ سیم ثابت است؛ بنابراین هرچه بهتر از آن استفاده کنید، هزینهٔ ارسال هر byte روی سیم کمتر می‌شود.

وضعیت مشابهی دربارهٔ CPUها وجود دارد: اگر هر CPU core را به‌صورت dynamic میان چند thread share کنید، یک thread گاهی باید در run queue مربوط به operating system منتظر بماند تا thread دیگری اجرا شود؛ بنابراین ممکن است thread برای مدت‌های متغیری pause شود. بااین‌حال، این روش hardware را بهتر از حالتی utilize می‌کند که تعداد staticای از CPU cycleها را به هر thread اختصاص دهید؛ به بخش «Response time guarantees» در صفحهٔ ۲۹۸ مراجعه کنید. استفادهٔ بهتر از hardware یکی از انگیزه‌های مهم استفاده از virtual machineها نیز هست.

در محیط‌های مشخص، latency guarantee قابل دستیابی است، اگر resourceها به‌صورت static partition شوند؛ مثلاً با hardware اختصاصی و bandwidth allocation انحصاری. اما این کار هزینهٔ کاهش utilization را دارد؛ به بیان دیگر، گران‌تر است. از سوی دیگر، multi-tenancy با dynamic resource partitioning utilization بهتری ایجاد می‌کند و ارزان‌تر است، اما عیب آن variable delay است.

Variable delay در network یک قانون طبیعت نیست؛ نتیجهٔ یک trade-off میان هزینه و فایده است.

بااین‌حال، چنین quality of serviceای در multi-tenant datacenterها و public cloudها، یا هنگام ارتباط از طریق internet، در حال حاضر فعال نیست. technologyای که اکنون deploy شده، اجازه نمی‌دهد دربارهٔ delay یا reliability network هیچ guaranteeای بدهیم. بنابراین باید فرض کنیم network congestion، queueing و unbounded delay رخ خواهند داد. در نتیجه، برای timeout هیچ مقدار «درستی» وجود ندارد و باید آن را به‌صورت تجربی تعیین کرد.

## Key Terms

- `Network Fault` — خطایی در مسیر یا componentهای network که باعث loss، delay، reorder یا عدم دسترسی packetها می‌شود.
- `Network Partition` — جدا شدن بخشی از network از بخش‌های دیگر؛ در این فصل برای جلوگیری از اشتباه با data partition، بیشتر از اصطلاح network fault استفاده می‌شود.
- `Failure Detection` — تشخیص اینکه یک node یا service واقعاً از کار افتاده است، نه اینکه فقط response آن با تأخیر برسد.
- `Timeout` — مدت زمانی که system برای دریافت response منتظر می‌ماند، پیش از آنکه failure یا عدم دسترسی را فرض کند.
- `Unbounded Delay` — وضعیتی که برای زمان رسیدن packet یا تکمیل request هیچ upper bound تضمین‌شده‌ای وجود ندارد.
- `Network Congestion` — رقابت packetها برای ظرفیت محدود network link که باعث queueing و افزایش delay می‌شود.
- `Packet Loss` — نرسیدن packet به مقصد، معمولاً به‌دلیل پر شدن queue، خرابی link یا fault در network.
- `Network Latency` — مدت زمان انتقال data یا تکمیل round trip میان دو endpoint.
- `Synchronous Network` — networkی که برای انتقال، bandwidth و maximum delay مشخصی را از پیش تضمین می‌کند.
- `Asynchronous Network` — networkی که دربارهٔ زمان تحویل packet یا تحویل قطعی آن guarantee زمانی نمی‌دهد.
- `Bounded Delay` — تضمین وجود یک maximum delay برای تحویل packet یا response.
- `Flow Control` — محدود کردن rate ارسال برای جلوگیری از overload شدن network link یا دریافت‌کننده.
- `Backpressure` — سیگنالی که upstream را وادار می‌کند سرعت تولید یا ارسال data را با ظرفیت downstream هماهنگ کند.
- `Cascading Failure` — زنجیره‌ای از failureها که وقتی overload یا failure یک component فشار بیشتری به componentهای دیگر منتقل می‌کند، گسترش می‌یابد.
- `Round-Trip Time (RTT)` — زمان رفت request یا packet به مقصد و برگشت response یا acknowledgement.
- `Phi Accrual Failure Detector` — failure detectorای که با اندازه‌گیری پیوستهٔ response time و jitter، احتمال faulty بودن node را برآورد می‌کند.
- `Circuit Switching` — تخصیص ثابت و رزروشدهٔ bandwidth در تمام مسیر ارتباط تا پایان session.
- `Packet Switching` — ارسال packetها با استفادهٔ dynamic و opportunistic از ظرفیت network موجود.
- `Quality of Service (QoS)` — اولویت‌بندی و scheduling traffic برای ارائهٔ guarantee یا رفتار بهتر به بعضی packetها.
- `Admission Control` — محدود کردن ورود traffic یا rate فرستنده‌ها برای جلوگیری از overload شدن system.
- `Resource Utilization` — میزان استفادهٔ مؤثر از CPU، network، storage یا resourceهای دیگر.
- `Noisy Neighbor` — tenant یا workload دیگری در یک محیط shared که با مصرف زیاد resource، performance و latency دیگران را ناپایدار می‌کند.

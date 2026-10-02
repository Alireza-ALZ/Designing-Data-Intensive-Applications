# Chapter 8 — The Trouble with Distributed Systems

## Faults and Partial Failures

وقتی برنامه‌ای را روی یک computer واحد می‌نویسید، معمولاً رفتار آن تا حد زیادی predictable است: یا کار می‌کند یا کار نمی‌کند. Software دارای bug ممکن است این تصور را ایجاد کند که computer گاهی «روز بدی دارد»؛ مشکلی که اغلب با reboot برطرف می‌شود. اما این وضعیت عمدتاً فقط پیامد software بد نوشته‌شده است.

هیچ دلیل fundamentalای وجود ندارد که software روی یک computer واحد flaky باشد: وقتی hardware درست کار می‌کند، یک operation یکسان همیشه result یکسانی تولید می‌کند؛ یعنی operation deterministic است. اگر hardware مشکل داشته باشد—برای مثال memory corruption یا loose connector—پیامد معمولاً failure کامل system است؛ مانند kernel panic، «blue screen of death» یا boot نشدن system. یک computer منفرد با software خوب معمولاً یا کاملاً functional است یا کاملاً broken، نه چیزی بین این دو.

این وضعیت انتخابی عمدی در design computerهاست: اگر fault داخلی رخ دهد، ترجیح می‌دهیم computer کاملاً crash کند تا اینکه result اشتباه برگرداند، چون برخورد با resultهای اشتباه دشوار و گیج‌کننده است. بنابراین computerها واقعیت فیزیکی مبهمی را که روی آن ساخته شده‌اند پنهان می‌کنند و یک system model ایده‌آل ارائه می‌دهند که با کمال ریاضی کار می‌کند. یک CPU instruction همیشه کار یکسانی انجام می‌دهد؛ اگر dataای در memory یا disk write کنید، آن data سالم می‌ماند و به‌صورت random corrupt نمی‌شود. این هدف، یعنی computation همیشه صحیح، به نخستین digital computerها برمی‌گردد [3].

وقتی softwareای را روی چند computer می‌نویسید که با network به هم متصل‌اند، situation از اساس متفاوت است. در distributed systemها دیگر در یک system model ایده‌آل کار نمی‌کنیم و چاره‌ای نداریم جز اینکه با واقعیت messy جهان فیزیکی روبه‌رو شویم. در جهان فیزیکی، دامنهٔ بسیار گسترده‌ای از چیزها ممکن است اشتباه پیش برود؛ همان‌طور که این خاطره نشان می‌دهد [4]:

> در تجربهٔ محدود من، با network partitionهای طولانی در یک datacenter، failure در PDU [power distribution unit]، failure در switch، power cycle تصادفی کل rackها، failure backbone کل datacenter، failure برق کل datacenter و حتی برخورد یک رانندهٔ hypoglycemic با Ford pickup خود به system مربوط به HVAC [heating, ventilation, and air conditioning] یک datacenter روبه‌رو شده‌ام. تازه من حتی آدم ops هم نیستم.
>
> —Coda Hale

در یک distributed system ممکن است بعضی بخش‌های system به شکلی unpredictable broken باشند، درحالی‌که بخش‌های دیگر به‌خوبی کار می‌کنند. به این وضعیت **partial failure** گفته می‌شود. دشواری کار در این است که partial failureها nondeterministic هستند: اگر کاری انجام دهید که چند node و network را درگیر کند، ممکن است گاهی موفق شود و گاهی به شکلی unpredictable fail شود. حتی ممکن است ندانید یک operation موفق شده است یا نه، چون زمانی که طول می‌کشد یک message از network عبور کند نیز nondeterministic است.

همین nondeterminism و امکان partial failure است که کار با distributed systemها را دشوار می‌کند [5].

### Cloud Computing and Supercomputing

برای ساختن large-scale computing systemها، طیفی از philosophyهای مختلف وجود دارد:

- در یک سوی این طیف، حوزهٔ **high-performance computing (HPC)** قرار دارد. Supercomputerهایی با هزاران CPU معمولاً برای taskهای علمی و computationally intensive استفاده می‌شوند؛ مانند weather forecasting یا molecular dynamics، یعنی شبیه‌سازی حرکت atomها و moleculeها.
- در سوی دیگر، cloud computing قرار دارد که تعریف کاملاً دقیقی ندارد [6]، اما معمولاً با multi-tenant datacenterها، computerهای commodity متصل از طریق IP network—اغلب Ethernet—، elastic یا on-demand resource allocation و metered billing شناخته می‌شود.
- Traditional enterprise datacenterها جایی میان این دو extreme قرار می‌گیرند.

این philosophyها approachهای بسیار متفاوتی برای handling fault دارند. در یک supercomputer، job معمولاً state مربوط به computation خود را هر از چند گاهی روی durable storage checkpoint می‌کند. اگر یک node fail شود، solution رایج این است که کل workload مربوط به cluster متوقف شود. پس از repair شدن node faulty، computation از آخرین checkpoint restart می‌شود [7, 8]. بنابراین supercomputer بیشتر شبیه یک computer single-node است تا یک distributed system: با partial failure این‌گونه برخورد می‌کند که اجازه می‌دهد failure به failure کامل escalate شود؛ اگر هر بخشی از system fail شد، بگذارید همه‌چیز crash کند، درست مانند kernel panic در یک machine منفرد.

در این book روی systemهایی تمرکز می‌کنیم که برای implement کردن internet serviceها استفاده می‌شوند و معمولاً تفاوت زیادی با supercomputerها دارند:

- بسیاری از applicationهای مرتبط با internet online هستند؛ یعنی باید بتوانند در هر لحظه با latency پایین به userها service بدهند. unavailable کردن service—برای مثال، متوقف کردن cluster برای repair—قابل‌قبول نیست. در مقابل، offline یا batch jobهایی مانند weather simulation را می‌توان با impact نسبتاً کمی متوقف و restart کرد.
- Supercomputerها معمولاً از hardware تخصصی ساخته می‌شوند که در آن هر node بسیار reliable است و nodeها از طریق shared memory و remote direct memory access (RDMA) با یکدیگر communication می‌کنند. در مقابل، nodeهای cloud serviceها از machineهای commodity ساخته می‌شوند. این machineها به‌دلیل economies of scale می‌توانند performance مشابهی با cost کمتر ارائه دهند، اما failure rate بالاتری نیز دارند.
- Networkهای datacenter بزرگ اغلب بر اساس IP و Ethernet ساخته می‌شوند و در topologyهای Clos قرار می‌گیرند تا **high bisection bandwidth** فراهم کنند [9]. Supercomputerها اغلب از network topologyهای تخصصی مانند multi-dimensional mesh و torus استفاده می‌کنند [10] که برای workloadهای HPC با communication patternهای شناخته‌شده performance بهتری دارند.
- هرچه system بزرگ‌تر شود، احتمال broken بودن یکی از componentهای آن بیشتر می‌شود. با گذشت زمان، componentهای broken repair می‌شوند و componentهای جدیدی broken می‌شوند؛ اما در systemی با هزاران node، منطقی است فرض کنیم همیشه چیزی broken است [7]. وقتی strategy مربوط به error handling صرفاً تسلیم شدن باشد، یک system بزرگ ممکن است بخش زیادی از زمان خود را به recovery از faultها اختصاص دهد، نه انجام work مفید [8].
- اگر system بتواند nodeهای failشده را تحمل کند و در مجموع به کار خود ادامه دهد، این feature برای operation و maintenance بسیار مفید است. برای مثال، می‌توانید یک **rolling upgrade** انجام دهید و هر بار فقط یک node را restart کنید، درحالی‌که service بدون interruption به userها service می‌دهد. در cloud environment، اگر یک virtual machine performance خوبی نداشته باشد، می‌توانید آن را kill کنید و یک virtual machine جدید request کنید؛ با این امید که machine جدید سریع‌تر باشد.
- در یک deployment جغرافیایی توزیع‌شده—که data را برای کاهش access latency در موقعیت جغرافیایی نزدیک userها نگه می‌دارد—communication احتمالاً از طریق internet انجام می‌شود که در مقایسه با local network کندتر و unreliableتر است. Supercomputerها عموماً فرض می‌کنند تمام nodeهای آن‌ها نزدیک به هم قرار دارند.

اگر می‌خواهیم distributed systemها کار کنند، باید possibility مربوط به partial failure را بپذیریم و mechanismهای Fault Tolerance را در software بسازیم. به بیان دیگر، باید systemی reliable را از componentهای unreliable بسازیم. (همان‌طور که در بخش «Reliability» در صفحهٔ ۶ بحث کردیم، reliability کامل وجود ندارد و باید محدودیت چیزهایی را که واقعاً می‌توانیم promise کنیم بشناسیم.)

حتی در systemهای کوچک که فقط چند node دارند نیز باید به partial failure فکر کنیم. در یک system کوچک احتمال زیادی وجود دارد که بیشتر componentها بیشتر اوقات درست کار کنند. بااین‌حال، دیر یا زود بخشی از system faulty خواهد شد و software باید somehow آن را handle کند. Fault handling باید بخشی از software design باشد و شما، به‌عنوان operator software، باید بدانید software در صورت رخ دادن fault چه رفتاری خواهد داشت.

خردمندانه نیست فرض کنیم faultها نادرند و صرفاً به بهترین حالت امیدوار باشیم. مهم است دامنهٔ گسترده‌ای از faultهای ممکن—حتی faultهای نسبتاً unlikely—را در نظر بگیریم و چنین situationهایی را به‌صورت artificial در testing environment ایجاد کنیم تا ببینیم چه اتفاقی رخ می‌دهد. در distributed systemها، suspicion، pessimism و paranoia نتیجه می‌دهند.

#### Building a Reliable System from Unreliable Components

ممکن است از خود بپرسید آیا این approach اصلاً معنا دارد یا نه؛ به‌صورت intuitive شاید به نظر برسد یک system فقط می‌تواند به‌اندازهٔ unreliableترین component خود reliable باشد؛ یعنی به‌اندازهٔ weakest link. اما چنین نیست: در واقع، ساختن systemی reliableتر از base زیرین unreliable، ایده‌ای قدیمی در computing است [11]. برای مثال:

- **Error-correcting code** اجازه می‌دهد digital data را با دقت از communication channelی عبور دهیم که گاهی بعضی bitها را اشتباه منتقل می‌کند؛ برای مثال، به‌دلیل radio interference در wireless network [12].
- `IP` یا Internet Protocol unreliable است: ممکن است packetها را drop، delay، duplicate یا reorder کند. `TCP` یا Transmission Control Protocol یک transport layer قابل‌اعتمادتر روی IP فراهم می‌کند؛ TCP تضمین می‌کند packetهای گم‌شده retransmit شوند، duplicateها حذف شوند و packetها به ترتیبی که ارسال شده‌اند دوباره assemble شوند.

اگرچه system می‌تواند از بخش‌های زیرین خود reliableتر باشد، همیشه limitی برای مقدار این بهبود وجود دارد. برای مثال، error-correcting code می‌تواند با تعداد کمی single-bit error برخورد کند، اما اگر signal با interference شدید پوشانده شود، limitی fundamental برای مقدار dataای که می‌توانید از communication channel عبور دهید وجود دارد [13]. TCP می‌تواند packet loss، duplication و reordering را از شما پنهان کند، اما نمی‌تواند delayهای network را به‌صورت جادویی حذف کند.

اگرچه system higher-level قابل‌اعتمادتر perfect نیست، همچنان مفید است، چون بخشی از faultهای tricky low-level را handle می‌کند و در نتیجه reasoning و handling faultهای باقی‌مانده معمولاً آسان‌تر می‌شود. در بخش «The End-to-End Argument» در صفحهٔ ۵۱۹ این موضوع را بیشتر بررسی خواهیم کرد.

## Key Terms

- `Distributed System` — مجموعه‌ای از componentها یا nodeهای مستقل که از طریق network با یکدیگر کار می‌کنند.
- `Fault` — انحراف یک component از behavior مورد انتظار یا specification خود.
- `Partial Failure` — وضعیتی که بخشی از distributed system fail شده، درحالی‌که بخش‌های دیگر همچنان کار می‌کنند.
- `Fault Tolerance` — توانایی system برای ادامهٔ کار با وجود failure برخی componentها.
- `Cloud Computing` — مدل ارائهٔ resourceهای computing در datacenterهای multi-tenant با allocation elastic و billing مبتنی بر مصرف.
- `Supercomputing` — استفاده از supercomputerهای بزرگ و تخصصی برای workloadهای computationally intensive مانند simulation علمی.
- `High-Performance Computing (HPC)` — حوزهٔ اجرای workloadهای علمی و محاسباتی سنگین روی systemهای بسیار قدرتمند.
- `Hardware Failure` — failure ناشی از component فیزیکی مانند memory، disk، connector یا machine.
- `Software Failure` — failure ناشی از bug، error یا behavior نادرست software.
- `Performance Degradation` — کاهش performance یا افزایش latency و failure rate بدون توقف کامل system.
- `Redundancy` — وجود component، node یا مسیر جایگزین برای ادامهٔ service هنگام failure.
- `Checkpoint` — ذخیرهٔ دوره‌ای state computation روی durable storage برای restart کردن پس از failure.
- `Commodity Hardware` — hardware عمومی و نسبتاً ارزان که با economies of scale تهیه می‌شود، اما ممکن است failure rate بالاتری داشته باشد.
- `Error-Correcting Code` — techniqueای برای تشخیص و اصلاح بعضی errorهای bit هنگام انتقال data.

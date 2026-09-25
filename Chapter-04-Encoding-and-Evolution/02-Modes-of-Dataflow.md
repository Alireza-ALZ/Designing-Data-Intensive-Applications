# Chapter 4 — Encoding and Evolution

## Modes of Dataflow

در ابتدای این chapter گفتیم هر زمان بخواهید dataای را به process دیگری بفرستید که با آن memory مشترک ندارید - برای مثال، هنگام ارسال data از طریق network یا نوشتن آن در file - باید data را به sequenceای از byteها encode کنید. سپس encodingهای مختلفی را که برای این کار وجود دارند بررسی کردیم.

همچنین دربارهٔ forward و backward compatibility صحبت کردیم که برای evolvability اهمیت دارند؛ یعنی با اجازه دادن به upgrade مستقل بخش‌های مختلف system و بی‌نیاز کردن ما از تغییر دادن همه‌چیز به‌صورت هم‌زمان، change را آسان می‌کنند. Compatibility رابطه‌ای میان processی است که data را encode می‌کند و processی که آن را decode می‌کند.

این ایده تا اینجا نسبتاً abstract است؛ data می‌تواند به روش‌های مختلفی از یک process به process دیگر flow کند. چه کسی data را encode می‌کند و چه کسی آن را decode می‌کند؟ در ادامهٔ chapter، رایج‌ترین روش‌های flow کردن data میان processها را بررسی می‌کنیم:

- از طریق databaseها (به بخش «Dataflow Through Databases» مراجعه کنید)
- از طریق service callها (به بخش «Dataflow Through Services: REST and RPC» مراجعه کنید)
- از طریق asynchronous message passing (به بخش «Message-Passing Dataflow» مراجعه کنید)

### Dataflow Through Databases

در یک database، processی که در database write می‌کند data را encode می‌کند و processی که آن را read می‌کند data را decode می‌کند. ممکن است فقط یک process به database دسترسی داشته باشد؛ در این صورت reader صرفاً version بعدی همان process است. در چنین حالتی می‌توانید ذخیره کردن چیزی در database را مانند ارسال یک message به future خود در نظر بگیرید.

در اینجا backward compatibility به‌وضوح ضروری است؛ در غیر این صورت future شما نمی‌تواند چیزی را که قبلاً write کرده‌اید decode کند.

به‌طور کلی رایج است که چند process به‌صورت هم‌زمان به یک database دسترسی داشته باشند. این processها ممکن است چند application یا service متفاوت باشند، یا فقط چند instance از یک service باشند که برای scalability یا Fault Tolerance به‌صورت parallel اجرا می‌شوند. در هر دو حالت، در محیطی که application در حال تغییر است، احتمالاً برخی processهای دسترسی‌دهنده به database code جدید و برخی code قدیمی را اجرا می‌کنند؛ برای مثال، چون version جدیدی در حال deployment به‌صورت rolling upgrade است و بعضی instanceها update شده‌اند، درحالی‌که برخی هنوز update نشده‌اند.

این موضوع یعنی ممکن است valueای در database با version جدید code write شود و بعد version قدیمی code که هنوز در حال اجراست آن را read کند. بنابراین برای databaseها معمولاً forward compatibility نیز لازم است.

اما یک مشکل دیگر هم وجود دارد. فرض کنید field جدیدی به schema یک record اضافه می‌کنید و code جدید value مربوط به آن field را در database write می‌کند. سپس version قدیمی code (که هنوز field جدید را نمی‌شناسد) record را read می‌کند، آن را update می‌کند و دوباره write می‌کند. در این وضعیت، رفتار مطلوب معمولاً این است که code قدیمی field جدید را دست‌نخورده نگه دارد، حتی اگر نتواند آن را تفسیر کند.

Encoding formatهایی که پیش‌تر بررسی کردیم از حفظ fieldهای ناشناخته پشتیبانی می‌کنند، اما گاهی باید در سطح application نیز دقت کنید؛ همان‌طور که در شکل ۴-۷ نشان داده شده است. برای مثال، اگر یک database value را در application به model object decode کنید و بعداً همان model objectها را دوباره encode کنید، ممکن است field ناشناخته در این فرآیند translation از بین برود. حل کردن این مشکل دشوار نیست؛ فقط باید از وجود آن آگاه باشید.

**شکل ۴-۷.** وقتی version قدیمی application dataای را که قبلاً توسط version جدید application write شده update می‌کند، اگر دقت نکنید ممکن است data از بین برود.

#### Valueهای متفاوتی که در زمان‌های متفاوت write شده‌اند

یک database معمولاً اجازه می‌دهد هر valueای در هر زمانی update شود. بنابراین درون یک database ممکن است valueهایی داشته باشید که پنج millisecond پیش write شده‌اند و valueهای دیگری که پنج سال پیش write شده‌اند.

وقتی version جدیدی از application خود را deploy می‌کنید (دست‌کم در یک server-side application)، می‌توانید version قدیمی را در عرض چند دقیقه به‌طور کامل با version جدید جایگزین کنید. این موضوع دربارهٔ محتوای database صدق نمی‌کند: data پنج‌ساله همچنان با encoding اصلی خود در database وجود خواهد داشت، مگر اینکه از آن زمان تاکنون صراحتاً آن را rewrite کرده باشید. این مشاهده گاهی با این عبارت خلاصه می‌شود که data از code بیشتر عمر می‌کند.

Rewrite کردن (migrate کردن) data به schema جدید قطعاً امکان‌پذیر است، اما انجام این کار روی dataset بزرگ پرهزینه است؛ بنابراین بیشتر databaseها تا حد امکان از آن اجتناب می‌کنند. بیشتر relational databaseها اجازه می‌دهند schema changeهای ساده، مانند اضافه کردن column جدید با default value برابر `null`، بدون rewrite کردن data موجود انجام شوند.* وقتی یک row قدیمی read می‌شود، database برای هر columnای که در data encodedشده روی disk وجود ندارد، `null` قرار می‌دهد. `Espresso`، document database شرکت LinkedIn، برای storage از Avro استفاده می‌کند و در نتیجه می‌تواند از ruleهای schema evolution در Avro بهره ببرد [23].

*پاورقی: به‌جز MySQL که اغلب تمام table را rewrite می‌کند، حتی اگر این کار از نظر strict لازم نباشد؛ همان‌طور که در بخش «Schema flexibility in the document model» در صفحهٔ ۳۹ گفته شد.*

بنابراین schema evolution باعث می‌شود کل database طوری به نظر برسد که گویی با یک schema واحد encode شده است، حتی اگر storage زیرین شامل recordهایی باشد که با versionهای تاریخی مختلف schema encode شده‌اند.

#### Archival storage

ممکن است هر از گاهی از database خود snapshot بگیرید؛ مثلاً برای backup یا load کردن data در یک data warehouse (به بخش «Data Warehousing» در صفحهٔ ۹۱ مراجعه کنید). در این حالت، data dump معمولاً با latest schema encode می‌شود، حتی اگر encoding اصلی در source database ترکیبی از schema versionهای متعلق به دوره‌های مختلف باشد. از آنجا که data را در هر صورت copy می‌کنید، بهتر است copy آن را به‌صورت consistent encode کنید.

چون data dump در یک مرحله write می‌شود و پس از آن immutable است، formatهایی مانند Avro object container fileها برای این کار fit خوبی هستند. این کار فرصت خوبی است تا data را در column-oriented formatای که برای analytics مناسب است، مانند `Parquet`، encode کنید (به بخش «Column Compression» در صفحهٔ ۹۷ مراجعه کنید).

در Chapter 10 دربارهٔ استفاده از data در archival storage بیشتر صحبت خواهیم کرد.

### Dataflow Through Services: REST and RPC

وقتی processها باید از طریق network با یکدیگر ارتباط برقرار کنند، چند روش متفاوت برای سازمان‌دهی این ارتباط وجود دارد. رایج‌ترین arrangement شامل دو role است: client و server. Serverها APIای را روی network expose می‌کنند و clientها می‌توانند به serverها connect شوند تا requestهایی برای آن API ارسال کنند. APIای که server expose می‌کند `service` نام دارد.

Web نیز به همین روش کار می‌کند: clientها (web browserها) به web serverها request می‌فرستند؛ با requestهای `GET` برای download کردن HTML، CSS، JavaScript، image و موارد دیگر و با requestهای `POST` برای submit کردن data به server. API از مجموعه‌ای standardized از protocolها و data formatها تشکیل شده است (`HTTP`، URLها، `SSL/TLS`، HTML و غیره). چون web browserها، web serverها و نویسندگان website عمدتاً بر سر این standardها توافق دارند، می‌توانید با هر web browserای به هر websiteای دسترسی پیدا کنید (دست‌کم در تئوری!).

Web browser تنها نوع client نیست. برای مثال، یک native app که روی mobile device یا desktop computer اجرا می‌شود نیز می‌تواند به server request network بفرستد. همچنین یک client-side JavaScript application که داخل web browser اجرا می‌شود می‌تواند با استفاده از `XMLHttpRequest` به یک HTTP client تبدیل شود؛ این technique به نام `Ajax` شناخته می‌شود [30]. در این حالت، response مربوط به server معمولاً HTMLای برای نمایش به انسان نیست، بلکه dataای در encodingای است که برای processing بیشتر توسط client-side application code convenient باشد (مانند JSON). هرچند ممکن است HTTP به‌عنوان transport protocol استفاده شود، APIای که روی آن implement می‌شود application-specific است و client و server باید دربارهٔ جزئیات آن API توافق داشته باشند.

علاوه بر این، خود یک server می‌تواند clientِ service دیگری باشد (برای مثال، یک web app server معمولاً clientِ یک database است). این approach اغلب برای decompose کردن یک application بزرگ به serviceهای کوچک‌تر بر اساس حوزهٔ functionality استفاده می‌شود؛ به‌طوری‌که وقتی یک service به functionality یا dataای از service دیگر نیاز دارد، برای آن service request می‌فرستد. این روش ساخت application در گذشته `service-oriented architecture (SOA)` نامیده می‌شد و در سال‌های اخیر با عنوان `microservices architecture` اصلاح و rebrand شده است [31, 32].

از بعضی جنبه‌ها، serviceها شبیه databaseها هستند: معمولاً به clientها اجازه می‌دهند data را submit و query کنند. بااین‌حال، databaseها queryهای arbitrary را با استفاده از query languageهایی که در Chapter 2 بررسی کردیم می‌پذیرند، درحالی‌که serviceها یک application-specific API expose می‌کنند که فقط input و outputهای از پیش تعیین‌شده توسط business logic (یعنی application code) آن service را می‌پذیرد [33]. این محدودیت degreeای از encapsulation فراهم می‌کند: serviceها می‌توانند محدودیت‌های fine-grained روی کارهایی که clientها می‌توانند یا نمی‌توانند انجام دهند اعمال کنند.

یکی از هدف‌های اصلی service-oriented یا microservices architecture این است که با independently deployable و evolvable کردن serviceها، تغییر و نگهداری application را آسان‌تر کند. برای مثال، هر service باید تحت مالکیت یک team باشد و آن team بتواند versionهای جدید service را به‌طور مکرر release کند، بدون اینکه لازم باشد با teamهای دیگر coordinate کند. به بیان دیگر، باید انتظار داشته باشیم versionهای قدیمی و جدید serverها و clientها هم‌زمان در حال اجرا باشند؛ بنابراین data encoding مورد استفادهٔ serverها و clientها باید در versionهای مختلف service API compatible باشد، دقیقاً همان موضوعی که در این chapter دربارهٔ آن صحبت می‌کنیم.

#### Web services

وقتی از HTTP به‌عنوان protocol زیرین برای صحبت با service استفاده می‌شود، به آن `web service` می‌گویند. این نام شاید کمی نادقیق باشد، چون web serviceها فقط در web استفاده نمی‌شوند و در contextهای مختلف کاربرد دارند. برای مثال:

1. یک client application که روی device کاربر اجرا می‌شود (برای مثال، native app روی mobile device یا JavaScript web appای که از Ajax استفاده می‌کند) و از طریق HTTP به service request می‌فرستد. این requestها معمولاً از طریق public internet ارسال می‌شوند.
2. یک service که به service دیگری request می‌فرستد و هر دو service متعلق به یک organization هستند؛ این serviceها اغلب در یک datacenter قرار دارند و این ارتباط بخشی از service-oriented یا microservices architecture است. (Softwareای که چنین use caseای را پشتیبانی می‌کند گاهی `middleware` نامیده می‌شود.)
3. یک service که به serviceای متعلق به organization دیگری request می‌فرستد، معمولاً از طریق internet. این روش برای data exchange میان backend systemهای organizationهای مختلف استفاده می‌شود. public APIهایی که online serviceها ارائه می‌کنند، مانند systemهای پردازش credit card یا `OAuth` برای دسترسی مشترک به user data، در این دسته قرار می‌گیرند.

دو approach محبوب برای web serviceها وجود دارد: REST و SOAP. این دو از نظر philosophy تقریباً در دو سوی کاملاً مخالف قرار دارند و اغلب موضوع debateهای داغ میان طرفداران خود هستند.†

`REST` protocol نیست، بلکه design philosophyای است که بر اصول HTTP بنا شده است [34, 35]. REST بر data formatهای ساده، استفاده از URL برای شناسایی resourceها و استفاده از featureهای HTTP برای cache control، authentication و content type negotiation تأکید می‌کند. REST، دست‌کم در context integration service میان organizationهای مختلف، در مقایسه با SOAP محبوب‌تر شده است [36] و اغلب با microservices مرتبط دانسته می‌شود [31]. APIای که بر اساس اصول REST طراحی شده باشد `RESTful` نام دارد.

در مقابل، `SOAP` protocolای مبتنی بر XML برای ارسال network API requestهاست.‡ هرچند SOAP بیشتر اوقات روی HTTP استفاده می‌شود، هدف آن مستقل بودن از HTTP است و از بیشتر featureهای HTTP استفاده نمی‌کند. در عوض، SOAP مجموعهٔ گسترده و پیچیده‌ای از standardهای مرتبط را دارد (که framework مربوط به web service و با نام `WS-*` شناخته می‌شود) و featureهای مختلفی اضافه می‌کند [37].

API مربوط به یک SOAP web service با استفاده از languageای مبتنی بر XML به نام `Web Services Description Language` یا `WSDL` توصیف می‌شود. WSDL امکان code generation را فراهم می‌کند تا client بتواند با استفاده از classها و method callهای local به یک remote service دسترسی پیدا کند؛ این callها توسط framework به XML message encode و دوباره decode می‌شوند. این قابلیت در statically typed programming languageها مفید است، اما در dynamically typed languageها کاربرد کمتری دارد (به بخش «Code generation and dynamically typed languages» در صفحهٔ ۱۲۷ مراجعه کنید).

از آنجا که WSDL برای human-readable بودن طراحی نشده است و SOAP messageها نیز اغلب برای ساختن دستی بیش از حد پیچیده‌اند، کاربران SOAP به‌شدت به tool support، code generation و IDEها متکی هستند [38]. برای کاربرانی که programming language مورد استفاده‌شان توسط SOAP vendorها پشتیبانی نمی‌شود، integration با SOAP serviceها دشوار است.

با اینکه SOAP و extensionهای مختلف آن ظاهراً standardized هستند، interoperability میان implementationهای vendorهای مختلف اغلب مشکل‌ساز می‌شود [39]. به همهٔ این دلایل، SOAP با وجود استفاده در enterpriseهای بزرگ، در بیشتر شرکت‌های کوچک‌تر محبوبیت خود را از دست داده است.

RESTful APIها معمولاً approachهای ساده‌تری را ترجیح می‌دهند و اغلب code generation و automated tooling کمتری لازم دارند. می‌توان از format تعریفی مانند `OpenAPI` که با نام `Swagger` نیز شناخته می‌شود [40] برای توصیف RESTful APIها و تولید documentation استفاده کرد.

*پاورقی: حتی درون هر camp نیز بحث‌های زیادی وجود دارد. برای مثال، HATEOAS (`hypermedia as the engine of application state`) اغلب باعث discussion می‌شود [35].*

*پاورقی: با وجود شباهت acronymها، SOAP الزام SOA نیست. SOAP یک technology مشخص است، درحالی‌که SOA رویکردی کلی برای ساخت systemهاست.*

#### مشکل‌های Remote Procedure Call (RPC)

Web serviceها فقط جدیدترین incarnation از زنجیرهٔ طولانی technologyهایی هستند که برای ارسال API request از طریق network ساخته شده‌اند؛ بسیاری از این technologyها hype زیادی دریافت کردند، اما مشکل‌های جدی دارند. `Enterprise JavaBeans (EJB)` و `Java Remote Method Invocation (RMI)` به Java محدود هستند. `Distributed Component Object Model (DCOM)` به platformهای Microsoft محدود است. `Common Object Request Broker Architecture (CORBA)` بیش از حد پیچیده است و backward یا forward compatibility فراهم نمی‌کند [41].

همهٔ این technologyها بر ایدهٔ `remote procedure call (RPC)` بنا شده‌اند که از دههٔ ۱۹۷۰ وجود داشته است [42]. مدل RPC تلاش می‌کند request به یک network service remote را شبیه call کردن یک function یا method در programming language شما و درون همان process نشان دهد؛ این abstraction `location transparency` نام دارد.

با اینکه RPC در نگاه اول convenient به نظر می‌رسد، این approach اساساً flawed است [43, 44]. یک network request تفاوت زیادی با local function call دارد:

- یک local function call قابل‌پیش‌بینی است و فقط بر اساس parameterهایی که تحت کنترل شما هستند موفق یا fail می‌شود. یک network request قابل‌پیش‌بینی نیست: request یا response ممکن است به‌دلیل network problem گم شود، یا machine remote کند یا unavailable باشد؛ این مشکل‌ها کاملاً خارج از کنترل شما هستند. Network problemها رایج‌اند، بنابراین باید آن‌ها را پیش‌بینی کنید؛ مثلاً با retry کردن request ناموفق.
- یک local function call یا result برمی‌گرداند، یا exception throw می‌کند، یا هرگز return نمی‌کند (چون وارد infinite loop می‌شود یا process crash می‌کند). یک network request نتیجهٔ ممکن دیگری نیز دارد: ممکن است به‌دلیل timeout بدون result return کند. در این حالت، به‌سادگی نمی‌دانید چه اتفاقی افتاده است: اگر responseای از remote service دریافت نکنید، هیچ راهی ندارید که بفهمید request به مقصد رسیده است یا نه. (در Chapter 8 این مسئله را با جزئیات بیشتری بررسی می‌کنیم.)
- اگر network request ناموفق را retry کنید، ممکن است requestها واقعاً به مقصد رسیده باشند و فقط responseها گم شده باشند. در این صورت، retry کردن باعث می‌شود action چند بار انجام شود، مگر اینکه mechanismای برای deduplication یا `idempotence` در protocol بسازید. Local function call چنین مشکلی ندارد. (در Chapter 11 دربارهٔ idempotence با جزئیات بیشتری صحبت می‌کنیم.)
- هر بار که یک local function را call می‌کنید، اجرای آن معمولاً تقریباً به همان مقدار زمان نیاز دارد. یک network request بسیار کندتر از function call است و latency آن نیز به‌شدت variable است: در شرایط خوب ممکن است در کمتر از یک millisecond کامل شود، اما وقتی network congested باشد یا remote service overload شده باشد، انجام دقیقاً همان کار ممکن است چندین second طول بکشد.
- هنگام call کردن یک local function می‌توانید referenceها (pointerها) به objectهای موجود در local memory را به‌صورت efficient به آن بدهید. اما هنگام ساختن network request، تمام این parameterها باید به sequenceای از byteها encode شوند تا از طریق network ارسال شوند. اگر parameterها primitiveهایی مانند number یا string باشند مشکلی وجود ندارد، اما با objectهای بزرگ این کار به‌سرعت مشکل‌ساز می‌شود.
- client و service ممکن است در programming languageهای متفاوتی implement شده باشند، بنابراین RPC framework باید datatypeها را از یک language به language دیگر translate کند. این کار می‌تواند ugly شود، چون همهٔ languageها typeهای یکسانی ندارند؛ برای مثال مشکل JavaScript با numberهای بزرگ‌تر از `2^53` را به یاد بیاورید (به بخش «JSON, XML, and Binary Variants» در صفحهٔ ۱۱۴ مراجعه کنید). این مشکل در یک process واحد که با یک language واحد نوشته شده است وجود ندارد.

تمام این عوامل نشان می‌دهند که تلاش برای شبیه کردن بیش از حد یک remote service به local object در programming language شما بی‌معناست، چون این دو اساساً چیزهای متفاوتی هستند. بخشی از جذابیت REST این است که تلاش نمی‌کند network protocol بودن خود را پنهان کند (هرچند این موضوع مانع ساختن RPC library روی REST توسط افراد نشده است).

#### جهت‌گیری‌های فعلی در RPC

با وجود تمام این مشکل‌ها، RPC در حال از بین رفتن نیست. frameworkهای RPC مختلفی روی encodingهایی که در این chapter بررسی کردیم ساخته شده‌اند: برای مثال، Thrift و Avro پشتیبانی RPC را به‌صورت built-in ارائه می‌کنند، `gRPC` یک implementation از RPC با استفاده از Protocol Buffers است، `Finagle` نیز از Thrift استفاده می‌کند و `Rest.li` از JSON روی HTTP استفاده می‌کند.

این نسل جدید frameworkهای RPC صریح‌تر از این واقعیت آگاه است که remote request با local function call تفاوت دارد. برای مثال، Finagle و Rest.li از `future`ها (یا `promise`ها) برای encapsulate کردن actionهای asynchronousای استفاده می‌کنند که ممکن است fail شوند. Futureها در موقعیت‌هایی که لازم است به چند service به‌صورت parallel request بفرستید و resultهای آن‌ها را با هم combine کنید نیز کار را ساده‌تر می‌کنند [45]. `gRPC` از streamها پشتیبانی می‌کند؛ در این حالت یک call فقط از یک request و یک response تشکیل نمی‌شود، بلکه در طول زمان مجموعه‌ای از requestها و responseها ردوبدل می‌شود [46].

برخی از این frameworkها `service discovery` را نیز ارائه می‌کنند؛ یعنی client می‌تواند بفهمد یک service مشخص در چه IP address و port numberای قابل دسترسی است. در بخش «Request Routing» در صفحهٔ ۲۱۴ دوباره به این موضوع برمی‌گردیم.

RPC protocolهای custom که از binary encoding format استفاده می‌کنند، می‌توانند performance بهتری از چیزی generic مانند JSON روی REST داشته باشند. بااین‌حال، RESTful API مزیت‌های مهم دیگری دارد: برای experimentation و debugging مناسب است (می‌توانید بدون code generation یا نصب software، با استفاده از web browser یا command-line tool `curl` به آن request بفرستید)، همهٔ programming languageها و platformهای اصلی از آن پشتیبانی می‌کنند و ecosystem بزرگی از toolها برای آن وجود دارد (server، cache، load balancer، proxy، firewall، monitoring، debugging tool، testing tool و غیره).

به این دلایل، REST به نظر می‌رسد style غالب برای public APIها باشد. تمرکز اصلی frameworkهای RPC روی requestهای میان serviceهایی است که متعلق به یک organization هستند و معمولاً در یک datacenter قرار دارند.

#### Data encoding و evolution برای RPC

برای evolvability مهم است که RPC clientها و serverها بتوانند مستقل از یکدیگر تغییر کنند و deploy شوند. در مقایسه با dataflow از طریق databaseها که در بخش قبل توضیح دادیم، در dataflow از طریق serviceها می‌توانیم یک assumption ساده‌کننده داشته باشیم: منطقی است فرض کنیم ابتدا تمام serverها update می‌شوند و بعد تمام clientها. بنابراین برای requestها فقط به backward compatibility و برای responseها به forward compatibility نیاز دارید.

خاصیت backward و forward compatibility در یک RPC scheme از encoding مورد استفادهٔ آن به ارث می‌رسد:

- Thrift، `gRPC` (بر پایهٔ Protocol Buffers) و `Avro RPC` را می‌توان بر اساس ruleهای compatibility مربوط به encoding format خود evolve کرد.
- در SOAP، requestها و responseها با XML schema مشخص می‌شوند. این schemaها قابل evolve هستند، اما مشکل‌های ظریفی دارند [47].
- RESTful APIها معمولاً برای responseها از JSON (بدون schema رسمی) و برای requestها از parameterهای JSON یا URI-encoded/form-encoded استفاده می‌کنند. اضافه کردن optional request parameterها و اضافه کردن fieldهای جدید به response objectها معمولاً changeهایی محسوب می‌شوند که compatibility را حفظ می‌کنند.

Service compatibility به این دلیل دشوارتر می‌شود که RPC اغلب برای ارتباط میان organizationهای مختلف استفاده می‌شود. در نتیجه provider یک service معمولاً کنترلی روی clientهای خود ندارد و نمی‌تواند آن‌ها را مجبور به upgrade کند. بنابراین compatibility باید برای مدت طولانی، شاید به‌صورت indefinite، حفظ شود. اگر changeای لازم باشد که compatibility را بشکند، provider service اغلب مجبور می‌شود چند version از service API را به‌صورت side by side نگهداری کند.

بر سر روش versioning کردن API توافق عمومی وجود ندارد؛ یعنی روش مشخصی وجود ندارد که client با آن اعلام کند می‌خواهد از کدام version API استفاده کند [48]. در RESTful APIها، approachهای رایج شامل قرار دادن version number در URL یا در HTTP `Accept` header است. برای serviceهایی که از API key برای شناسایی یک client مشخص استفاده می‌کنند، گزینهٔ دیگر این است که version درخواست‌شدهٔ client را روی server ذخیره کنیم و اجازه دهیم انتخاب این version از طریق administrative interface جداگانه update شود [49].

### Message-Passing Dataflow

تا اینجا روش‌های مختلف flow کردن data encodedشده از یک process به process دیگر را بررسی کردیم. دربارهٔ REST و RPC صحبت کردیم (که در آن یک process از طریق network به process دیگر request می‌فرستد و انتظار دارد response را در سریع‌ترین زمان ممکن دریافت کند) و دربارهٔ databaseها صحبت کردیم (که در آن یک process data encodedشده را write می‌کند و process دیگری مدتی بعد آن را دوباره read می‌کند).

در این بخش پایانی، نگاهی کوتاه به systemهای asynchronous message-passing می‌اندازیم که جایی میان RPC و database قرار دارند. این systemها از یک جهت شبیه RPC هستند: request مربوط به client (که معمولاً `message` نامیده می‌شود) با latency پایین به process دیگری تحویل داده می‌شود. از جهت دیگر شبیه database هستند: message از طریق connection مستقیم network ارسال نمی‌شود، بلکه از واسطه‌ای به نام `message broker` عبور می‌کند (که `message queue` یا `message-oriented middleware` نیز نامیده می‌شود) و این واسطه message را به‌صورت موقت ذخیره می‌کند.

استفاده از message broker در مقایسه با RPC مستقیم چند مزیت دارد:

- اگر recipient unavailable یا overloaded باشد، broker می‌تواند مانند buffer عمل کند و در نتیجه reliability سیستم را بهبود دهد.
- می‌تواند messageها را به‌صورت automatic برای processی که crash کرده است دوباره deliver کند و از lost شدن messageها جلوگیری کند.
- sender لازم نیست IP address و port number مربوط به recipient را بداند (این موضوع به‌خصوص در cloud deployment مفید است، جایی که virtual machineها اغلب ایجاد و حذف می‌شوند).
- اجازه می‌دهد یک message برای چند recipient ارسال شود.
- sender را از recipient به‌صورت logical decouple می‌کند (sender فقط messageها را publish می‌کند و اهمیتی نمی‌دهد چه کسی آن‌ها را consume می‌کند).

بااین‌حال، message-passing در مقایسه با RPC معمولاً یک‌طرفه است: sender عموماً انتظار ندارد در پاسخ به messageهای خود reply دریافت کند. یک process می‌تواند response ارسال کند، اما این کار معمولاً روی channel جداگانه انجام می‌شود. این الگوی ارتباط asynchronous است: sender منتظر deliver شدن message نمی‌ماند، بلکه آن را ارسال می‌کند و سپس دیگر دربارهٔ آن کاری انجام نمی‌دهد.

#### Message brokers

در گذشته، landscape مربوط به message brokerها در اختیار enterprise softwareهای تجاری شرکت‌هایی مانند `TIBCO`، `IBM WebSphere` و `webMethods` بود. در سال‌های اخیر، implementationهای open source مانند `RabbitMQ`، `ActiveMQ`، `HornetQ`، `NATS` و `Apache Kafka` محبوب شده‌اند. در Chapter 11 آن‌ها را با جزئیات بیشتری مقایسه خواهیم کرد.

جزئیات delivery semantics بسته به implementation و configuration متفاوت است، اما به‌طور کلی message brokerها به این شکل استفاده می‌شوند: یک process messageای را به queue یا topic نام‌گذاری‌شده ارسال می‌کند و broker تضمین می‌کند message به یک یا چند consumer از آن queue یا topic، یا subscriberهای آن، deliver شود. ممکن است producerها و consumerهای زیادی روی یک topic وجود داشته باشند.

یک topic فقط dataflow یک‌طرفه فراهم می‌کند. بااین‌حال، یک consumer می‌تواند خودش messageهایی را به topic دیگری publish کند (تا بتوان topicها را مانند زنجیره به هم متصل کرد؛ همان‌طور که در Chapter 11 خواهیم دید) یا message را به reply queueای بفرستد که sender message اصلی آن را consume می‌کند؛ در این حالت request/response dataflowای مشابه RPC ایجاد می‌شود.

Message brokerها معمولاً هیچ data model خاصی را enforce نمی‌کنند؛ یک message فقط sequenceای از byteها همراه با مقداری metadata است، بنابراین می‌توانید از هر encoding formatای استفاده کنید. اگر encoding شما backward و forward compatible باشد، بیشترین flexibility را برای تغییر مستقل publisherها و consumerها و deploy کردن آن‌ها در هر orderی خواهید داشت.

اگر consumer messageها را به topic دیگری republish کند، باید مراقب باشید fieldهای ناشناخته را حفظ کنید تا مشکل توضیح‌داده‌شده در context databaseها (شکل ۴-۷) رخ ندهد.

#### Distributed actor frameworks

`Actor model` یک programming model برای concurrency درون یک process واحد است. در این model به‌جای کار کردن مستقیم با threadها (و مشکل‌های مربوط به race condition، locking و deadlock)، logic درون actorها encapsulate می‌شود. هر actor معمولاً نمایندهٔ یک client یا entity است، ممکن است state محلی داشته باشد (که با هیچ actor دیگری share نمی‌شود) و با ارسال و دریافت asynchronous message با actorهای دیگر ارتباط برقرار می‌کند. Delivery message تضمین‌شده نیست: در برخی error scenarioها messageها گم خواهند شد. چون هر actor در هر لحظه فقط یک message را پردازش می‌کند، لازم نیست نگران threadها باشد و framework می‌تواند هر actor را به‌صورت مستقل schedule کند.

در distributed actor frameworkها از این programming model برای scale کردن application روی چند node استفاده می‌شود. بدون توجه به اینکه sender و recipient روی یک node یا nodeهای متفاوت قرار دارند، از همان message-passing mechanism استفاده می‌شود. اگر روی nodeهای متفاوت باشند، message به‌صورت transparent به byte sequence encode می‌شود، از طریق network ارسال می‌شود و در سمت دیگر decode می‌گردد.

Location transparency در actor model بهتر از RPC عمل می‌کند، چون actor model از ابتدا فرض می‌کند messageها ممکن است حتی درون یک process واحد نیز گم شوند. اگرچه latency روی network احتمالاً بیشتر از latency درون همان process است، هنگام استفاده از actor model mismatch بنیادی کمتری میان communication محلی و remote وجود دارد.

یک distributed actor framework در اصل message broker و actor programming model را در یک framework واحد integrate می‌کند. بااین‌حال، اگر بخواهید rolling upgrade برای application مبتنی بر actor انجام دهید، همچنان باید نگران forward و backward compatibility باشید؛ چون ممکن است message از nodeای که version جدید را اجرا می‌کند به nodeای که version قدیمی را اجرا می‌کند ارسال شود و برعکس.

سه distributed actor framework محبوب، message encoding را به شکل زیر مدیریت می‌کنند:

- `Akka` به‌صورت default از serialization built-in مربوط به Java استفاده می‌کند که forward یا backward compatibility فراهم نمی‌کند. بااین‌حال، می‌توانید آن را با چیزی مانند Protocol Buffers جایگزین کنید و در نتیجه امکان rolling upgrade را به دست آورید [50].
- `Orleans` به‌صورت default از custom data encoding formatای استفاده می‌کند که از rolling upgrade deployment پشتیبانی نمی‌کند. برای deploy کردن version جدید application، باید cluster جدیدی راه‌اندازی کنید، traffic را از cluster قدیمی به cluster جدید منتقل کنید و cluster قدیمی را خاموش کنید [51, 52]. مانند Akka، در Orleans نیز می‌توان از custom serialization plug-inها استفاده کرد.
- در `Erlang OTP`، تغییر دادن record schemaها به‌طرز شگفت‌آوری دشوار است (با وجود اینکه system featureهای زیادی برای high availability دارد). Rolling upgrade امکان‌پذیر است، اما باید با دقت برنامه‌ریزی شود [53]. datatype جدید و experimental به نام `maps` (ساختاری شبیه JSON که در Erlang R17 در سال ۲۰۱۴ معرفی شد) ممکن است در آینده این کار را آسان‌تر کند [54].

## Key Terms

- `Dataflow` — مسیر و شیوهٔ انتقال data میان processها، serviceها، databaseها یا nodeها.
- `Data Encoding` — تبدیل data به byte sequence برای انتقال یا storage میان componentهای مستقل.
- `Client` — componentی که به service request می‌فرستد یا از API آن استفاده می‌کند.
- `Server` — componentی که API یا service را expose می‌کند و requestهای client را پردازش می‌کند.
- `Request` — پیام یا فراخوانی‌ای که client برای دریافت data یا اجرای operation به server می‌فرستد.
- `Response` — نتیجه یا پیام برگشتی server در پاسخ به request.
- `API` — قرارداد ورودی و خروجی یک service یا component برای استفادهٔ clientها.
- `Service` — componentی با API مشخص که functionality یا data را به clientها ارائه می‌دهد.
- `REST` — design philosophy مبتنی بر اصول HTTP برای ساخت APIهای resource-oriented و قابل استفاده در network.
- `RESTful API` — APIای که بر اساس اصول REST طراحی شده است.
- `RPC (Remote Procedure Call)` — abstractionای که request به service remote را شبیه call کردن function یا method محلی نشان می‌دهد.
- `Web Service` — serviceای که API آن با استفاده از HTTP در دسترس قرار می‌گیرد.
- `Message` — واحد dataای که در message-passing system از sender به recipient ارسال می‌شود.
- `Message Broker` — واسطه‌ای که messageها را موقتاً نگه می‌دارد و به consumerها deliver می‌کند.
- `Message Queue` — queueای برای buffer، routing و تحویل message میان producerها و consumerها.
- `Message-Passing System` — systemی که componentها را با ارسال و دریافت message به هم متصل می‌کند.
- `Asynchronous Communication` — ارتباطی که sender بدون انتظار برای response یا delivery کامل، کار خود را ادامه می‌دهد.
- `Synchronous Communication` — ارتباطی که caller تا دریافت result یا response منتظر می‌ماند.
- `Service-Oriented Architecture (SOA)` — معماری‌ای که application را به serviceهای مستقل با interface مشخص تقسیم می‌کند.
- `Microservices Architecture` — رویکردی برای ساخت application از serviceهای کوچک، مستقل و independently deployable.
- `Service Discovery` — mechanism پیدا کردن location یک service، مانند IP address و port number آن.
- `Location Transparency` — پنهان کردن تفاوت local و remote بودن component از caller.
- `Idempotence` — خاصیتی که باعث می‌شود اجرای تکراری یک operation همان effect اجرای یک‌باره را داشته باشد.
- `API Versioning` — مدیریت versionهای مختلف contract یک API برای حفظ compatibility clientها و serverها.
- `Actor Model` — programming modelی که state و logic را در actorهای مستقل قرار می‌دهد و ارتباط را با message انجام می‌دهد.
- `Distributed Actor Framework` — frameworkی که actor model و message passing را برای اجرای application روی چند node فراهم می‌کند.
- `Delivery Semantics` — guaranteeهای broker دربارهٔ زمان، تعداد دفعات و مقصد تحویل message.

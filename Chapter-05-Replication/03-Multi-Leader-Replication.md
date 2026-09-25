# Chapter 5 — Replication

## Multi-Leader Replication

تا اینجا در این chapter فقط architectureهای replication با یک leader را بررسی کردیم. اگرچه این روش رایج است، alternativeهای جالبی نیز وجود دارند.

یکی از ضعف‌های اصلی leader-based replication این است که فقط یک leader وجود دارد و تمام writeها باید از آن عبور کنند.* اگر به هر دلیلی نتوانید به leader connect شوید، برای مثال به‌دلیل قطع شدن network میان شما و leader، نمی‌توانید در database write کنید.

گسترش طبیعی مدل leader-based replication این است که اجازه دهیم بیش از یک node write را بپذیرد. Replication همچنان به همان روش انجام می‌شود: هر nodeای که write را پردازش می‌کند باید data change را برای تمام nodeهای دیگر forward کند. این configuration را `multi-leader` می‌نامیم (که `master-master` یا `active/active replication` نیز نامیده می‌شود). در این setup، هر leader هم‌زمان به‌عنوان follower سایر leaderها نیز عمل می‌کند.

*پاورقی: اگر database partition شده باشد (به Chapter 6 مراجعه کنید)، هر partition یک leader دارد. ممکن است leaderهای partitionهای مختلف روی nodeهای متفاوتی قرار داشته باشند، اما هر partition در هر صورت باید یک leader node داشته باشد.*

### Use Cases for Multi-Leader Replication

استفاده از multi-leader setup درون یک datacenter به‌ندرت منطقی است، چون مزیت‌های آن معمولاً ارزش پیچیدگی اضافی را ندارند. بااین‌حال، موقعیت‌هایی وجود دارد که این configuration در آن‌ها reasonable است.

#### Multi-datacenter operation

فرض کنید databaseای دارید که replicaهای آن در چند datacenter قرار گرفته‌اند؛ شاید برای تحمل failure یک datacenter کامل یا برای نزدیک‌تر بودن به userها. در یک leader-based replication setup معمولی، leader باید در یکی از datacenterها قرار داشته باشد و تمام writeها باید از همان datacenter عبور کنند.

در multi-leader configuration می‌توانید در هر datacenter یک leader داشته باشید. شکل ۵-۶ نشان می‌دهد این architecture ممکن است چگونه باشد. درون هر datacenter از leader-follower replication معمولی استفاده می‌شود؛ میان datacenterها، leader هر datacenter changeهای خود را به leaderهای datacenterهای دیگر replicate می‌کند.

**شکل ۵-۶.** multi-leader replication در چند datacenter.

بیایید عملکرد configurationهای single-leader و multi-leader را در یک multi-datacenter deployment مقایسه کنیم:

**Performance**

در single-leader configuration، هر write باید از طریق internet به datacenterای برود که leader در آن قرار دارد. این کار می‌تواند latency قابل‌توجهی به writeها اضافه کند و حتی با هدف اصلی داشتن چند datacenter در تضاد باشد. در multi-leader configuration، هر write می‌تواند در datacenter محلی پردازش شود و سپس به‌صورت asynchronous به datacenterهای دیگر replicate شود. بنابراین delay مربوط به network میان datacenterها از دید user پنهان می‌ماند و performance ادراک‌شده می‌تواند بهتر باشد.

**Tolerance of datacenter outages**

در single-leader configuration، اگر datacenter دارای leader fail شود، failover می‌تواند followerای را در datacenter دیگری به leader تبدیل کند. در multi-leader configuration، هر datacenter می‌تواند مستقل از datacenterهای دیگر به کار خود ادامه دهد و وقتی datacenter failed دوباره online شود، replication catch up خواهد کرد.

**Tolerance of network problems**

ترافیک میان datacenterها معمولاً از public internet عبور می‌کند که ممکن است از network محلی درون یک datacenter unreliableتر باشد. single-leader configuration به مشکل‌های این link میان datacenterها بسیار حساس است، چون writeها به‌صورت synchronous روی این link انجام می‌شوند. multi-leader configuration با asynchronous replication معمولاً می‌تواند network problemها را بهتر تحمل کند: یک network interruption موقت مانع پردازش writeها نمی‌شود.

برخی databaseها به‌صورت default از multi-leader configuration پشتیبانی می‌کنند، اما این قابلیت اغلب با toolهای external نیز implement می‌شود؛ مانند `Tungsten Replicator` برای MySQL [26]، `BDR` برای PostgreSQL [27] و `GoldenGate` برای Oracle [19].

اگرچه multi-leader replication مزیت‌هایی دارد، یک downside بزرگ نیز دارد: ممکن است یک data به‌صورت concurrent در دو datacenter مختلف modify شود و این write conflictها باید resolve شوند (در شکل ۵-۶ با عنوان conflict resolution مشخص شده‌اند). این مسئله را در بخش «Handling Write Conflicts» در صفحهٔ ۱۷۱ بررسی خواهیم کرد.

از آنجا که multi-leader replication در بسیاری از databaseها featureای است که بعداً به system اضافه شده، اغلب با pitfallهای ظریف configuration و interactionهای غیرمنتظره با featureهای دیگر database همراه است. برای مثال، keyهای auto-incrementing، triggerها و integrity constraintها می‌توانند مشکل‌ساز باشند. به همین دلیل، multi-leader replication اغلب حوزه‌ای خطرناک تلقی می‌شود که باید در صورت امکان از آن اجتناب کرد [28].

#### Clientهایی با offline operation

موقعیت دیگری که multi-leader replication در آن مناسب است، applicationای است که باید هنگام disconnected بودن از internet نیز به کار خود ادامه دهد.

برای مثال، calendar appهای روی mobile phone، laptop و deviceهای دیگر خود را در نظر بگیرید. باید بتوانید در هر زمانی meetingهای خود را ببینید (read request انجام دهید) و meeting جدیدی وارد کنید (write request انجام دهید)، بدون توجه به اینکه device شما در آن لحظه connection اینترنت دارد یا نه. اگر هنگام offline بودن changeهایی ایجاد کنید، زمانی که device دوباره online شد باید این changeها با server و deviceهای دیگر sync شوند.

در این حالت، هر device یک local database دارد که به‌عنوان leader عمل می‌کند (write requestها را می‌پذیرد) و میان replicaهای calendar شما روی تمام deviceها، فرآیند asynchronous multi-leader replication (یعنی sync) وجود دارد. بسته به اینکه چه زمانی به internet دسترسی پیدا کنید، replication lag ممکن است چند hour یا حتی چند day باشد.

از دید architecture، این setup اساساً مشابه multi-leader replication میان datacenterهاست، با این تفاوت که به extreme رسیده است: هر device یک «datacenter» است و network connection میان آن‌ها به‌شدت unreliable است. history طولانی implementationهای خراب calendar sync نشان می‌دهد که درست پیاده‌سازی کردن multi-leader replication کار دشواری است.

toolهایی وجود دارند که هدفشان ساده‌تر کردن چنین multi-leader configurationای است. برای مثال، `CouchDB` برای این mode از operation طراحی شده است [29].

#### Collaborative editing

applicationهای real-time collaborative editing به چند نفر اجازه می‌دهند یک document را به‌صورت هم‌زمان edit کنند. برای مثال، `Etherpad` [30] و `Google Docs` [31] به چند نفر اجازه می‌دهند یک text document یا spreadsheet را به‌صورت concurrent edit کنند (algorithm آن به‌اختصار در بخش «Automatic Conflict Resolution» در صفحهٔ ۱۷۴ بررسی شده است).

معمولاً collaborative editing را یک replication problem در database در نظر نمی‌گیریم، اما این مسئله شباهت زیادی به use case مربوط به offline editing دارد. وقتی یک user document را edit می‌کند، changeها بلافاصله روی local replica او apply می‌شوند (state document در web browser یا client application او) و سپس به‌صورت asynchronous به server و هر user دیگری که همان document را edit می‌کند replicate می‌شوند.

اگر بخواهید تضمین کنید هیچ editing conflictای رخ نمی‌دهد، application باید پیش از edit کردن user روی document lock بگیرد. اگر user دیگری بخواهد همان document را edit کند، ابتدا باید صبر کند تا user اول changeهای خود را commit و lock را release کند. این collaboration model معادل single-leader replication با transaction روی leader است.

بااین‌حال، برای collaboration سریع‌تر ممکن است بخواهید unit مربوط به change را بسیار کوچک کنید (برای مثال، یک keystroke) و از lock کردن اجتناب کنید. این approach به چند user اجازه می‌دهد هم‌زمان edit کنند، اما تمام challengeهای multi-leader replication را نیز به همراه دارد؛ از جمله نیاز به conflict resolution [32].

### Handling Write Conflicts

بزرگ‌ترین مشکل multi-leader replication این است که write conflict ممکن است رخ دهد و در نتیجه به conflict resolution نیاز داریم.

برای مثال، wiki pageای را در نظر بگیرید که هم‌زمان توسط دو user edit می‌شود؛ همان‌طور که در شکل ۵-۷ نشان داده شده است. User 1 عنوان page را از A به B تغییر می‌دهد و user 2 هم‌زمان عنوان را از A به C تغییر می‌دهد. Change هر user با موفقیت روی leader محلی او apply می‌شود. اما وقتی changeها به‌صورت asynchronous replicate می‌شوند، conflict شناسایی می‌شود [33]. این مشکل در single-leader database رخ نمی‌دهد.

**شکل ۵-۷.** write conflict ناشی از update هم‌زمان یک record توسط دو leader.

#### Synchronous versus asynchronous conflict detection

در single-leader database، writer دوم یا block می‌شود و منتظر کامل شدن write اول می‌ماند، یا transaction مربوط به write دوم abort می‌شود و user مجبور می‌شود write را retry کند. در multi-leader setup، هر دو write موفق هستند و conflict فقط در زمانی بعد و به‌صورت asynchronous شناسایی می‌شود. در آن زمان ممکن است دیگر دیر شده باشد که از user بخواهیم conflict را resolve کند.

در principle می‌توانید conflict detection را synchronous کنید؛ یعنی پیش از اعلام موفقیت write به user منتظر بمانید تا write روی تمام replicaها replicate شود. بااین‌حال، با این کار مزیت اصلی multi-leader replication را از دست می‌دهید: اینکه هر replica بتواند مستقل از replicaهای دیگر write را بپذیرد. اگر conflict detection synchronous می‌خواهید، بهتر است از ابتدا single-leader replication استفاده کنید.

#### Conflict avoidance

ساده‌ترین strategy برای برخورد با conflictها اجتناب از آن‌هاست: اگر application بتواند تضمین کند تمام writeهای مربوط به یک record مشخص از یک leader واحد عبور می‌کنند، conflict رخ نخواهد داد. از آنجا که بسیاری از implementationهای multi-leader replication conflictها را به‌خوبی handle نمی‌کنند، اجتناب از conflict اغلب approach پیشنهادی است [34].

برای مثال، در applicationای که user می‌تواند data خودش را edit کند، می‌توانید تضمین کنید requestهای یک user مشخص همیشه به همان datacenter route شوند و برای read و write از leader همان datacenter استفاده کنند. Userهای مختلف می‌توانند datacenter «home» متفاوتی داشته باشند (که شاید بر اساس نزدیکی جغرافیایی به user انتخاب شده باشد)، اما از دید هر user، configuration اساساً single-leader است.

بااین‌حال، گاهی ممکن است بخواهید leader تعیین‌شده برای یک record را تغییر دهید؛ شاید یک datacenter fail شده باشد و لازم باشد traffic را به datacenter دیگری reroute کنید، یا user به location دیگری منتقل شده و حالا به datacenter متفاوتی نزدیک‌تر باشد. در این وضعیت conflict avoidance دیگر کار نمی‌کند و باید احتمال concurrent write روی leaderهای مختلف را در نظر بگیرید.

#### Converging toward a consistent state

یک single-leader database writeها را به‌صورت sequential order apply می‌کند: اگر چند update روی یک field انجام شود، آخرین write value نهایی field را تعیین می‌کند.

در multi-leader configuration order مشخصی برای writeها وجود ندارد، بنابراین روشن نیست value نهایی چه باید باشد. در شکل ۵-۷، روی leader 1 عنوان ابتدا به B و سپس به C update می‌شود؛ روی leader 2 ابتدا به C و سپس به B update می‌شود. هیچ‌کدام از این دو order از دیگری «صحیح‌تر» نیست.

اگر هر replica writeها را صرفاً به همان orderی که دیده است apply کند، database به state ناسازگاری می‌رسد: value نهایی در leader 1 برابر C و در leader 2 برابر B خواهد بود. این وضعیت قابل‌قبول نیست؛ هر replication scheme باید تضمین کند که data در نهایت در تمام replicaها یکسان باشد. بنابراین database باید conflict را به‌شکل convergent resolve کند؛ یعنی وقتی تمام changeها replicate شدند، تمام replicaها باید به یک value نهایی یکسان برسند.

راه‌های مختلفی برای رسیدن به convergent conflict resolution وجود دارد:

- به هر write یک ID یکتا بدهید (برای مثال timestamp، یک random number بزرگ، UUID یا hash مربوط به key و value)، writeای را که بالاترین ID را دارد winner انتخاب کنید و writeهای دیگر را دور بیندازید. اگر از timestamp استفاده شود، این technique `last write wins (LWW)` نام دارد. اگرچه این approach محبوب است، به‌شکل خطرناکی مستعد data loss است [35]. در پایان این chapter در بخش «Detecting Concurrent Writes» در صفحهٔ ۱۸۴، LWW را با جزئیات بیشتری بررسی خواهیم کرد.
- به هر replica یک ID یکتا بدهید و همیشه به writeهایی که از replica با number بالاتر منشأ گرفته‌اند، نسبت به writeهای replica با number پایین‌تر اولویت دهید. این approach نیز به معنای data loss است.
- valueها را به شکلی با هم merge کنید؛ برای مثال آن‌ها را به‌صورت alphabetic مرتب و سپس concatenate کنید. در شکل ۵-۷، عنوان mergeشده ممکن است چیزی شبیه `B/C` باشد.
- conflict را در data structureای صریح ثبت کنید که تمام information را حفظ کند و application code را طوری بنویسید که conflict را در زمان دیگری resolve کند؛ شاید با prompt کردن user.

#### Custom conflict resolution logic

از آنجا که مناسب‌ترین روش resolve کردن conflict ممکن است به application وابسته باشد، بیشتر multi-leader replication toolها اجازه می‌دهند conflict resolution logic را با application code بنویسید. این code ممکن است هنگام write یا هنگام read execute شود:

**On write**

به‌محض اینکه database system conflictی را در log مربوط به replicated changeها detect کند، conflict handler را call می‌کند. برای مثال، `Bucardo` اجازه می‌دهد برای این کار snippetای با Perl بنویسید. این handler معمولاً نمی‌تواند user را prompt کند؛ چون در background process اجرا می‌شود و باید سریع execute شود.

**On read**

وقتی conflict detect می‌شود، تمام writeهای conflicting ذخیره می‌شوند. دفعهٔ بعد که data read می‌شود، چند version از data به application برگردانده می‌شود. Application می‌تواند user را prompt کند یا conflict را به‌صورت automatic resolve کند و result را به database write کند. برای مثال، `CouchDB` به این روش کار می‌کند.

توجه کنید که conflict resolution معمولاً در سطح یک row یا document منفرد انجام می‌شود، نه در سطح یک transaction کامل [36]. بنابراین اگر transactionای داشته باشید که چند write متفاوت را به‌صورت atomic انجام می‌دهد (به Chapter 7 مراجعه کنید)، هر write همچنان برای conflict resolution به‌صورت جداگانه در نظر گرفته می‌شود.

#### Automatic Conflict Resolution

ruleهای conflict resolution می‌توانند به‌سرعت پیچیده شوند و custom code نیز مستعد error است. `Amazon` نمونه‌ای است که اغلب برای نشان دادن effectهای غیرمنتظرهٔ conflict resolution handler به آن اشاره می‌شود: برای مدتی، logic مربوط به conflict resolution در shopping cart، itemهای اضافه‌شده به cart را حفظ می‌کرد، اما itemهای حذف‌شده را حفظ نمی‌کرد. بنابراین گاهی itemهایی که customer قبلاً از cart حذف کرده بود دوباره در cart ظاهر می‌شدند [37].

در زمینهٔ resolve کردن automatic conflictهای ناشی از concurrent data modification، research جالبی انجام شده است. چند مسیر research مهم عبارت‌اند از:

- `Conflict-free replicated datatypes (CRDTs)` [32, 38] خانواده‌ای از data structureها برای set، map، ordered list، counter و موارد دیگر هستند که چند user می‌توانند آن‌ها را به‌صورت concurrent edit کنند و این structureها conflictها را به روش‌های sensible به‌صورت automatic resolve می‌کنند. برخی CRDTها در `Riak 2.0` implement شده‌اند [39, 40].
- `Mergeable persistent data structures` history را به‌صورت صریح track می‌کنند، مشابه system کنترل version `Git`، و از یک three-way merge function استفاده می‌کنند (درحالی‌که CRDTها از two-way merge استفاده می‌کنند).
- `Operational transformation` [42] الگوریتم conflict resolution پشت applicationهای collaborative editing مانند Etherpad [30] و Google Docs [31] است. این الگوریتم به‌طور خاص برای edit هم‌زمان یک ordered list از itemها، مانند list مربوط به characterهای یک text document، طراحی شده است.

implementation این algorithmها در databaseها هنوز جوان است، اما احتمالاً در آینده در data systemهای replicated بیشتری integrate خواهند شد. automatic conflict resolution می‌تواند data synchronization به روش multi-leader را برای applicationها بسیار ساده‌تر کند.

#### What is a conflict?

برخی نوع‌های conflict obvious هستند. در مثال شکل ۵-۷، دو write به‌صورت concurrent یک field یکسان در یک record را modify کرده‌اند و آن را به دو value متفاوت set کرده‌اند. تقریباً تردیدی وجود ندارد که این یک conflict است.

تشخیص برخی conflictهای دیگر ظریف‌تر است. برای مثال، یک booking system برای meeting room را در نظر بگیرید: این application track می‌کند کدام room در چه زمانی توسط کدام group از افراد book شده است. System باید تضمین کند هر room در هر لحظه فقط توسط یک group book شده باشد؛ یعنی برای یک room نباید bookingهای overlapشده وجود داشته باشد. در این حالت، اگر دو booking متفاوت برای یک room و در یک زمان ایجاد شوند، conflict ایجاد می‌شود. حتی اگر application پیش از اجازه دادن به user برای booking، availability را check کند، باز هم اگر دو booking روی دو leader متفاوت انجام شوند conflict ممکن است رخ دهد.

پاسخ سریع و آماده‌ای برای این مسئله وجود ندارد، اما در chapterهای بعدی مسیر رسیدن به درک خوبی از آن را دنبال خواهیم کرد. در Chapter 7 چند نمونهٔ دیگر از conflictها را می‌بینیم و در Chapter 12 دربارهٔ approachهای scalable برای detect و resolve کردن conflictها در replicated system صحبت می‌کنیم.

### Multi-Leader Replication Topologies

`Replication topology` مسیرهای communicationای را توصیف می‌کند که writeها از طریق آن‌ها از یک node به node دیگر propagate می‌شوند. اگر دو leader داشته باشید، مانند شکل ۵-۷، فقط یک topology محتمل وجود دارد: leader 1 باید تمام writeهای خود را برای leader 2 بفرستد و برعکس. با بیش از دو leader، topologyهای متفاوتی ممکن هستند. چند نمونه در شکل ۵-۸ نشان داده شده است.

**شکل ۵-۸.** سه topology نمونه که می‌توان multi-leader replication را با آن‌ها setup کرد.

عمومی‌ترین topology، `all-to-all` است (شکل ۵-۸ [c]) که در آن هر leader writeهای خود را برای تمام leaderهای دیگر می‌فرستد. بااین‌حال، topologyهای محدودتری نیز استفاده می‌شوند. برای مثال، MySQL به‌صورت default فقط از topology `circular` پشتیبانی می‌کند [34]؛ در این topology، هر node writeها را از یک node دریافت می‌کند و آن writeها را (به‌اضافهٔ writeهای خودش) برای یک node دیگر forward می‌کند. topology محبوب دیگر شکل `star` دارد:* یک root node تعیین‌شده writeها را برای تمام nodeهای دیگر forward می‌کند. topology star را می‌توان به یک tree تعمیم داد.

در topologyهای circular و star، ممکن است یک write پیش از رسیدن به تمام replicaها مجبور باشد از چند node عبور کند. بنابراین nodeها باید data changeهایی را که از nodeهای دیگر دریافت می‌کنند forward کنند. برای جلوگیری از infinite replication loop، به هر node یک identifier یکتا داده می‌شود و در replication log، هر write با identifier تمام nodeهایی که از آن‌ها عبور کرده tag می‌شود [43]. وقتی nodeای data changeای را دریافت می‌کند که identifier خودش را دارد، آن data change را ignore می‌کند، چون node می‌داند قبلاً آن را پردازش کرده است.

یک مشکل topologyهای circular و star این است که failure فقط یک node می‌تواند جریان replication messageها میان nodeهای دیگر را قطع کند و تا زمان fix شدن آن node، ارتباط آن‌ها را مختل کند. می‌توان topology را برای دور زدن node failed reconfigure کرد، اما در بیشتر deploymentها این reconfiguration باید به‌صورت manual انجام شود. Fault Tolerance در topology متراکم‌تری مانند all-to-all بهتر است، چون messageها می‌توانند از مسیرهای متفاوت عبور کنند و از single point of failure دور بمانند.

از سوی دیگر، topologyهای all-to-all نیز می‌توانند مشکل‌هایی داشته باشند. به‌خصوص ممکن است برخی network linkها از linkهای دیگر سریع‌تر باشند (برای مثال، به‌دلیل network congestion) و در نتیجه بعضی replication messageها از messageهای دیگر «سبقت بگیرند»؛ همان‌طور که در شکل ۵-۹ نشان داده شده است.

**شکل ۵-۹.** در multi-leader replication ممکن است writeها با order نادرست به برخی replicaها برسند.

در شکل ۵-۹، client A یک row را در tableای روی leader 1 insert می‌کند و client B همان row را روی leader 3 update می‌کند. بااین‌حال، ممکن است leader 2 writeها را با order متفاوتی دریافت کند: ابتدا update را دریافت کند (که از دید آن update مربوط به rowای است که در database وجود ندارد) و فقط بعد insert مربوطه را دریافت کند (که باید پیش از update رخ داده باشد).

این مسئله‌ای مربوط به causality است و شبیه مشکلی است که در بخش «Consistent Prefix Reads» در صفحهٔ ۱۶۵ دیدیم: update به insert قبلی وابسته است، بنابراین باید مطمئن شویم تمام nodeها ابتدا insert و سپس update را process می‌کنند. صرفاً attach کردن timestamp به هر write کافی نیست، چون نمی‌توان به اندازهٔ کافی synchronized بودن clockها برای order دادن درست این eventها در leader 2 اعتماد کرد (به Chapter 8 مراجعه کنید).

برای order دادن درست به این eventها می‌توان از techniqueای به نام `version vectors` استفاده کرد که بعداً در همین chapter (در بخش «Detecting Concurrent Writes» در صفحهٔ ۱۸۴) بررسی می‌شود. بااین‌حال، techniqueهای conflict detection در بسیاری از multi-leader replication systemها به‌خوبی implement نشده‌اند. برای مثال، در زمان نگارش این book، `PostgreSQL BDR` causal ordering مربوط به writeها را فراهم نمی‌کند [27] و `Tungsten Replicator` برای MySQL حتی تلاش نمی‌کند conflictها را detect کند [34].

اگر از systemی با multi-leader replication استفاده می‌کنید، باید از این مشکل‌ها آگاه باشید، documentation را با دقت بخوانید و database خود را به‌طور کامل test کنید تا مطمئن شوید guaranteeهایی را که تصور می‌کنید واقعاً ارائه می‌دهد.

*پاورقی: این موضوع را با star schema اشتباه نگیرید (به بخش «Stars and Snowflakes: Schemas for Analytics» در صفحهٔ ۹۳ مراجعه کنید)؛ star schema ساختار یک data model را توصیف می‌کند، نه communication topology میان nodeها.*

## Key Terms

- `Multi-Leader Replication` — replicationای که در آن بیش از یک node می‌تواند write را بپذیرد و leaderها changeهای خود را میان یکدیگر replicate می‌کنند.
- `Write Conflict` — وضعیتی که writeهای concurrent روی data یکسان valueهای متفاوتی ایجاد می‌کنند.
- `Conflict Detection` — شناسایی writeهایی که نمی‌توانند بدون resolve شدن هم‌زمان اعمال شوند.
- `Conflict Resolution` — انتخاب، merge یا نگهداری چند version برای رسیدن replicaها به state مشترک.
- `Conflict Avoidance` — routing کردن writeهای یک record به leader واحد برای جلوگیری از conflict.
- `Conflict-Free Replicated Data Type (CRDT)` — data structureای که editهای concurrent را با ruleهای داخلی merge و conflictها را automatic resolve می‌کند.
- `Last Write Wins (LWW)` — انتخاب write دارای بالاترین timestamp یا ID به‌عنوان winner و حذف writeهای دیگر.
- `Causal Ordering` — مرتب‌سازی writeها مطابق dependencyهای causal، به‌گونه‌ای که علت پیش از معلول process شود.
- `Causal Dependency` — وابستگی یک write یا event به write یا event قبلی.
- `Replication Topology` — مسیرهای communication که writeها از طریق آن‌ها میان leaderها propagate می‌شوند.
- `Circular Replication` — topologyای که هر node writeها را از یک node می‌گیرد و به node بعدی forward می‌کند.
- `Star Topology` — topologyای با یک root node که writeها را برای nodeهای دیگر forward می‌کند.
- `All-to-All Topology` — topologyای که هر leader writeهای خود را برای تمام leaderهای دیگر می‌فرستد.
- `Multi-Datacenter Replication` — replication میان replicaهایی که در datacenterهای مختلف قرار دارند.
- `Offline Operation` — ادامهٔ read و write application هنگام قطع بودن connection و sync کردن changeها پس از online شدن.
- `Collaborative Editing` — edit هم‌زمان یک document توسط چند user با replication asynchronous changeها.
- `Concurrent Write` — writeهایی که بدون order سراسری مشخص، تقریباً هم‌زمان روی replicaهای مختلف اجرا می‌شوند.
- `Replication Loop` — گردش بی‌نهایت یک data change در topology؛ با identifier و tracking مسیر باید از آن جلوگیری شود.
- `Version Vector` — metadataای برای track کردن version و causal order تغییرات میان replicaها.

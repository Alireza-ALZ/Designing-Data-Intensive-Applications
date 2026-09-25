# Chapter 5 — Replication

## Problems with Replication Lag

توانایی تحمل failure مربوط به nodeها تنها یکی از دلایل استفاده از replication است. همان‌طور که در مقدمهٔ Part II گفتیم، دلیل‌های دیگر شامل scalability (پردازش requestهای بیشتر از ظرفیت یک machine) و latency (قرار دادن replicaها از نظر جغرافیایی نزدیک‌تر به userها) هستند.

در leader-based replication، تمام writeها باید از یک node واحد عبور کنند، اما queryهای read-only می‌توانند به هر replicaای ارسال شوند. برای workloadهایی که عمدتاً از read تشکیل شده‌اند و فقط درصد کمی write دارند (الگوی رایجی در web)، یک option جذاب وجود دارد: followerهای زیادی ایجاد کنید و requestهای read را میان آن‌ها distribute کنید. این کار load را از leader برمی‌دارد و اجازه می‌دهد requestهای read از replicaهای نزدیک به user پاسخ داده شوند.

در این read-scaling architecture می‌توانید ظرفیت پاسخ‌گویی به requestهای read-only را صرفاً با اضافه کردن followerهای بیشتر افزایش دهید. بااین‌حال، این approach در عمل فقط با asynchronous replication کار می‌کند؛ اگر تلاش کنید تمام followerها را به‌صورت synchronous replicate کنید، failure یک node یا network outage کل system را برای write کردن unavailable می‌کند. هرچه nodeهای بیشتری داشته باشید، احتمال از کار افتادن یکی از آن‌ها بیشتر است، بنابراین configuration کاملاً synchronous بسیار unreliable خواهد بود.

متأسفانه اگر application از یک follower asynchronous read کند، ممکن است در صورتی که follower عقب مانده باشد، اطلاعات outdated ببیند. این موضوع به inconsistency ظاهری در database منجر می‌شود: اگر query یکسانی را هم‌زمان روی leader و follower اجرا کنید، ممکن است resultهای متفاوتی بگیرید، چون هنوز تمام writeها در follower منعکس نشده‌اند. این inconsistency فقط یک state موقت است؛ اگر write کردن به database را متوقف کنید و مدتی صبر کنید، followerها در نهایت catch up می‌کنند و با leader consistent می‌شوند. به همین دلیل، این effect `eventual consistency` نامیده می‌شود [22, 23].†

واژهٔ «eventually» عمداً مبهم است: به‌طور کلی هیچ limitای برای مقدار عقب‌ماندن replica وجود ندارد. در operation عادی، delay میان write شدن data روی leader و منعکس شدن آن روی follower - یعنی `replication lag` - ممکن است فقط کسری از یک second باشد و در عمل قابل‌مشاهده نباشد. بااین‌حال، اگر system نزدیک capacity خود کار کند یا network دچار مشکل باشد، lag می‌تواند به‌راحتی به چند second یا حتی چند minute افزایش یابد.

وقتی lag تا این اندازه زیاد شود، inconsistencyهایی که ایجاد می‌کند دیگر مسئله‌ای صرفاً تئوری نیستند، بلکه برای applicationها مشکل واقعی ایجاد می‌کنند. در این بخش سه نمونه از مشکل‌هایی را که احتمال دارد هنگام وجود replication lag رخ دهند برجسته می‌کنیم و چند approach برای حل آن‌ها ارائه می‌دهیم.

*پاورقی: اصطلاح eventual consistency را Douglas Terry و همکارانش ابداع کردند [24]، Werner Vogels آن را popular کرد [22] و این اصطلاح به شعار بسیاری از projectهای NoSQL تبدیل شد. بااین‌حال، فقط NoSQL databaseها eventual consistency ندارند؛ followerها در relational databaseای که از asynchronous replication استفاده می‌کند نیز همین ویژگی را دارند.*

### Reading Your Own Writes

بسیاری از applicationها به user اجازه می‌دهند dataای را submit کند و بعد همان data را که submit کرده مشاهده کند. این data ممکن است recordای در customer database، commentای در یک discussion thread یا چیزی مشابه باشد. وقتی data جدید submit می‌شود، باید به leader ارسال شود؛ اما وقتی user آن data را مشاهده می‌کند، می‌توان آن را از follower read کرد. این روش به‌خصوص زمانی مناسب است که data مرتب view می‌شود اما فقط گاهی write می‌شود.

در asynchronous replication مشکلی وجود دارد که در شکل ۵-۳ نشان داده شده است: اگر user کمی پس از write کردن data آن را view کند، ممکن است data جدید هنوز به replica نرسیده باشد. از دید user، چنین به نظر می‌رسد که data submit‌شده گم شده است؛ بنابراین طبیعتاً ناراضی خواهد شد.

**شکل ۵-۳.** user یک write انجام می‌دهد و سپس از replicaای stale read می‌کند. برای جلوگیری از این anomaly به read-after-write consistency نیاز داریم.

در این وضعیت به `read-after-write consistency` نیاز داریم که `read-your-writes consistency` نیز نامیده می‌شود [24]. این guarantee می‌گوید اگر user صفحه را reload کند، همیشه updateهایی را که خودش submit کرده خواهد دید. این guarantee دربارهٔ userهای دیگر هیچ قولی نمی‌دهد: updateهای userهای دیگر ممکن است تا مدتی بعد قابل مشاهده نباشند. بااین‌حال، به user اطمینان می‌دهد input خودش به‌درستی save شده است.

چگونه می‌توان read-after-write consistency را در systemی با leader-based replication implement کرد؟ چند technique ممکن وجود دارد. برای مثال:

- هنگام read کردن چیزی که ممکن است user آن را modify کرده باشد، data را از leader بخوانید؛ در غیر این صورت از follower read کنید. برای این کار باید راهی داشته باشید که بدون query کردن data بفهمید آیا ممکن است modify شده باشد یا نه. برای مثال، در یک social network معمولاً فقط owner پروفایل می‌تواند user profile را edit کند و دیگران نمی‌توانند. بنابراین یک rule ساده این است: همیشه profile خود user را از leader بخوانید و profile userهای دیگر را از follower.
- اگر بیشتر چیزهای application بالقوه قابل edit توسط user باشند، approach قبلی مؤثر نخواهد بود، چون بیشتر data باید از leader read شود و مزیت read scaling از بین می‌رود. در این صورت می‌توان از معیارهای دیگری برای تصمیم‌گیری دربارهٔ read کردن از leader استفاده کرد. برای مثال، زمان آخرین update را track کنید و تا یک minute پس از آخرین update، تمام readها را از leader انجام دهید. همچنین می‌توانید replication lag روی followerها را monitor کنید و query روی هر followerای را که بیش از یک minute از leader عقب است، متوقف کنید.
- client می‌تواند timestamp آخرین write خود را به خاطر بسپارد؛ سپس system می‌تواند تضمین کند replicaای که readهای آن user را پاسخ می‌دهد، updateها را دست‌کم تا آن timestamp منعکس کرده باشد. اگر replica به‌اندازهٔ کافی up to date نباشد، read می‌تواند توسط replica دیگری handle شود یا query تا catch up کردن replica منتظر بماند. این timestamp می‌تواند logical timestamp باشد (چیزی که order writeها را نشان می‌دهد، مانند `log sequence number`) یا system clock واقعی باشد؛ در حالت دوم، clock synchronization اهمیت زیادی پیدا می‌کند (به بخش «Unreliable Clocks» در صفحهٔ ۲۸۷ مراجعه کنید).
- اگر replicaهای شما در چند datacenter distributed باشند (برای نزدیک بودن جغرافیایی به userها یا برای availability)، پیچیدگی بیشتری ایجاد می‌شود. هر requestی که باید توسط leader پاسخ داده شود باید به datacenterای route شود که leader در آن قرار دارد.

پیچیدگی دیگری زمانی به وجود می‌آید که یک user از چند device به service شما دسترسی داشته باشد؛ برای مثال، از یک desktop web browser و یک mobile app. در این حالت ممکن است بخواهید `cross-device read-after-write consistency` فراهم کنید: اگر user اطلاعاتی را در یک device وارد کند و سپس آن را در device دیگری ببیند، باید اطلاعاتی را که همین چند لحظه پیش وارد کرده مشاهده کند.

در این حالت باید چند مسئلهٔ اضافی را در نظر بگیرید:

- approachهایی که به خاطر سپردن timestamp آخرین update user نیاز دارند دشوارتر می‌شوند، چون code در حال اجرا روی یک device نمی‌داند روی device دیگر چه updateهایی رخ داده است. این metadata باید centralized شود.
- اگر replicaهای شما در datacenterهای مختلف distributed باشند، هیچ guaranteeای وجود ندارد که connectionهای deviceهای مختلف به همان datacenter route شوند. (برای مثال، computer رومیزی user ممکن است از connection پهن‌باند خانه و mobile device او از cellular data network استفاده کند و routeهای network آن‌ها کاملاً متفاوت باشد.) اگر approach شما به read کردن از leader نیاز داشته باشد، ممکن است ابتدا لازم باشد requestهای تمام deviceهای یک user را به یک datacenter یکسان route کنید.

### Monotonic Reads

دومین نمونهٔ anomalyای که هنگام read کردن از followerهای asynchronous ممکن است رخ دهد این است که user چیزها را در حال حرکت به عقب در زمان ببیند.

این اتفاق زمانی رخ می‌دهد که user چند read را از replicaهای مختلف انجام دهد. برای مثال، شکل ۵-۴ user شمارهٔ ۲۳۴۵ را نشان می‌دهد که query یکسانی را دو بار اجرا می‌کند: بار اول روی followerای با lag کم و بار دوم روی followerای با lag بیشتر. (اگر user صفحهٔ web را refresh کند و هر request به server تصادفی route شود، این scenario کاملاً محتمل است.) query اول commentای را برمی‌گرداند که user شمارهٔ ۱۲۳۴ به‌تازگی اضافه کرده است، اما query دوم چیزی برنمی‌گرداند، چون follower عقب‌مانده هنوز آن write را دریافت نکرده است.

در واقع query دوم system را در نقطهٔ زمانی زودتری نسبت به query اول مشاهده می‌کند. اگر query اول چیزی برنگردانده بود، این موضوع چندان بد نبود، چون user ۲۳۴۵ احتمالاً نمی‌دانست user ۱۲۳۴ به‌تازگی commentی اضافه کرده است. اما برای user ۲۳۴۵ بسیار گیج‌کننده است که ابتدا comment user ۱۲۳۴ را ببیند و بعد شاهد ناپدید شدن آن باشد.

**شکل ۵-۴.** user ابتدا از replica تازه و سپس از replica stale read می‌کند. به نظر می‌رسد زمان به عقب برگشته است. برای جلوگیری از این anomaly به monotonic reads نیاز داریم.

`Monotonic reads` guarantee می‌کند این نوع anomaly رخ ندهد [23]. این guarantee از strong consistency ضعیف‌تر، اما از eventual consistency قوی‌تر است. هنگام read کردن data ممکن است value قدیمی ببینید؛ monotonic reads فقط به این معناست که اگر یک user چند read را به‌صورت متوالی انجام دهد، زمان را رو به عقب نخواهد دید؛ یعنی پس از خواندن data جدیدتر، data قدیمی‌تری read نمی‌کند.

یک راه برای دست‌یابی به monotonic reads این است که مطمئن شویم هر user همیشه readهای خود را از یک replica انجام می‌دهد (userهای مختلف می‌توانند از replicaهای متفاوتی read کنند). برای مثال، می‌توان replica را بر اساس hash مربوط به user ID انتخاب کرد، نه به‌صورت random. بااین‌حال، اگر آن replica fail شود، queryهای user باید به replica دیگری reroute شوند.

### Consistent Prefix Reads

سومین نمونه از anomalyهای replication lag به نقض causality مربوط است. گفت‌وگوی کوتاه زیر میان Mr. Poons و Mrs. Cake را تصور کنید:

**Mr. Poons**

تا چه مدت در آینده را می‌توانید ببینید، Mrs. Cake؟

**Mrs. Cake**

معمولاً حدود ده second، Mr. Poons.

میان این دو جمله یک `causal dependency` وجود دارد: Mrs. Cake سؤال Mr. Poons را شنیده و به آن پاسخ داده است.

حالا تصور کنید شخص سومی از طریق followerها به این گفت‌وگو گوش می‌دهد. جمله‌های Mrs. Cake از followerای با lag کم عبور می‌کنند، اما جمله‌های Mr. Poons replication lag بیشتری دارند (شکل ۵-۵). این observer گفت‌وگوی زیر را می‌شنود:

**Mrs. Cake**

معمولاً حدود ده second، Mr. Poons.

**Mr. Poons**

تا چه مدت در آینده را می‌توانید ببینید، Mrs. Cake؟

از دید observer چنین به نظر می‌رسد که Mrs. Cake پیش از آنکه Mr. Poons سؤال خود را بپرسد، به سؤال او پاسخ داده است. این توانایی‌های psychic impressive، اما بسیار گیج‌کننده‌اند [25].

**شکل ۵-۵.** اگر برخی partitionها با سرعت کمتری replicate شوند، observer ممکن است پاسخ را پیش از سؤال ببیند.

برای جلوگیری از این نوع anomaly به guarantee دیگری به نام `consistent prefix reads` نیاز داریم [23]. این guarantee می‌گوید اگر sequenceای از writeها با order مشخصی رخ دهد، هر کسی که آن writeها را read می‌کند، آن‌ها را با همان order خواهد دید.

این مسئله به‌خصوص در partitioned یا sharded databaseها مشکل‌ساز است؛ موضوعی که در Chapter 6 بررسی خواهیم کرد. اگر database همیشه writeها را با order یکسان apply کند، readها همیشه consistent prefix را می‌بینند و این anomaly نمی‌تواند رخ دهد. بااین‌حال، در بسیاری از distributed databaseها partitionهای مختلف به‌صورت مستقل کار می‌کنند، بنابراین order سراسری برای writeها وجود ندارد: وقتی user از database read می‌کند، ممکن است بعضی قسمت‌های database را در state قدیمی‌تر و بعضی قسمت‌ها را در state جدیدتر ببیند.

یک راه‌حل این است که مطمئن شویم هر writeای که از نظر causality با write دیگری مرتبط است در همان partition write می‌شود؛ اما در برخی applicationها این کار را نمی‌توان به‌صورت efficient انجام داد. algorithmهایی نیز وجود دارند که dependencyهای causal را به‌صورت صریح track می‌کنند؛ این موضوعی است که در بخش «The “happens-before” relationship and concurrency» در صفحهٔ ۱۸۶ دوباره به آن برمی‌گردیم.

### Solutions for Replication Lag

هنگام کار با systemی که eventual consistency دارد، ارزش دارد فکر کنید اگر replication lag به چند minute یا حتی چند hour افزایش پیدا کند، application چه رفتاری خواهد داشت. اگر پاسخ «هیچ مشکلی نیست» باشد، عالی است. اما اگر نتیجه، تجربهٔ بدی برای userها باشد، مهم است system را طوری طراحی کنید که guarantee قوی‌تری مانند read-after-write ارائه دهد. وانمود کردن به اینکه replication synchronous است، درحالی‌که واقعاً asynchronous است، recipeای برای مشکل‌های آینده است.

همان‌طور که پیش‌تر گفتیم، application می‌تواند guarantee قوی‌تری از database زیرین ارائه دهد؛ برای مثال، با انجام دادن typeهای خاصی از read روی leader. بااین‌حال، مدیریت این مسائل در application code پیچیده است و به‌راحتی ممکن است اشتباه انجام شود.

بهتر بود application developerها مجبور نباشند نگران مسئله‌های ظریف replication باشند و بتوانند به database خود اعتماد کنند که «کار درست را انجام می‌دهد». transactionها به همین دلیل وجود دارند: آن‌ها روشی هستند که database با استفاده از آن guaranteeهای قوی‌تری فراهم می‌کند تا application ساده‌تر شود.

Single-node transactionها مدت زیادی است که وجود دارند. بااین‌حال، هنگام حرکت به سمت databaseهای distributed (replicated و partitioned)، بسیاری از systemها آن‌ها را کنار گذاشته‌اند و ادعا کرده‌اند transactionها از نظر performance و availability بیش از حد expensive هستند و در یک system scalable، eventual consistency اجتناب‌ناپذیر است. این گفته تا حدی درست است، اما بیش از حد ساده‌سازی شده است و در ادامهٔ این book دیدگاه nuancedتری توسعه خواهیم داد. در Chapterهای ۷ و ۹ دوباره به transactionها برمی‌گردیم و در Part III چند mechanism جایگزین را بررسی می‌کنیم.

## Key Terms

- `Replication Lag` — فاصلهٔ زمانی یا مقداری میان state leader و state follower.
- `Eventual Consistency` — guaranteeای که می‌گوید replicaها پس از توقف تغییرات، در نهایت به state consistent می‌رسند؛ بدون تضمین زمان مشخص.
- `Read-After-Write Consistency` — guaranteeای که user پس از write می‌تواند update خود را در readهای بعدی ببیند.
- `Reading Your Own Writes` — الگوی consistencyای که user باید dataای را که خودش write کرده، در read بعدی مشاهده کند.
- `Monotonic Reads` — guaranteeای که مانع مشاهدهٔ data قدیمی‌تر پس از مشاهدهٔ data جدیدتر توسط یک user می‌شود.
- `Consistent Prefix Reads` — guaranteeای که order مشاهدهٔ writeهای causal را برای همهٔ readerها حفظ می‌کند.
- `Causality` — رابطهٔ علت و معلولی میان eventها یا writeهای system.
- `Causal Dependency` — وابستگی‌ای که نشان می‌دهد یک event بر اساس مشاهده یا رخ دادن event دیگری ایجاد شده است.
- `Causal Ordering` — مرتب‌سازی eventها به‌گونه‌ای که علت پیش از معلول مشاهده یا apply شود.
- `Session Guarantee` — مجموعه guaranteeهایی که consistency مشاهده‌شده در طول session یک user را حفظ می‌کنند.
- `Read Consistency` — guarantee مربوط به تازگی و order dataای که read از replica برمی‌گرداند.
- `Stale Read` — read کردن value قدیمی از replicaای که هنوز آخرین writeها را دریافت نکرده است.
- `Strong Consistency` — guarantee مشاهدهٔ stateی که با writeهای پذیرفته‌شده و order معتبر system سازگار است.
- `Consistent View` — viewای از data که state و order مشاهده‌شدهٔ آن برای reader سازگار باقی می‌ماند.
- `Read Scaling` — افزایش ظرفیت read با distribute کردن queryها میان چند follower یا replica.

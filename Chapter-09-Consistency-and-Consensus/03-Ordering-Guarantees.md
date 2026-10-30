# Chapter 9 — Consistency and Consensus

## Ordering Guarantees

پیش‌تر گفتیم یک register خطی‌پذیر طوری رفتار می‌کند که انگار فقط یک copy از data وجود دارد و هر operation در یک نقطهٔ زمانی به‌صورت اتمیک اثر می‌گذارد. از این تعریف نتیجه می‌شود که operationها در ترتیبی مشخص اجرا می‌شوند. در شکل 9-4، این ترتیب را با وصل‌کردن operationها بر اساس ترتیب ظاهری اجرای آن‌ها نشان دادیم.

ترتیب در بخش‌های مختلف کتاب تکرار شده است و به‌نظر می‌رسد ایده‌ای بنیادی باشد. چند موردی را که قبلاً درباره‌شان صحبت کردیم مرور کنیم:

- در فصل ۵ دیدیم هدف اصلی leader در single-leader replication تعیین ترتیب writeها در replication log است؛ یعنی ترتیبی که followerها writeها را apply می‌کنند. اگر single leader وجود نداشته باشد، operationهای concurrent ممکن است conflict ایجاد کنند.
- در فصل ۷ دیدیم serializability تضمین می‌کند transactionها طوری رفتار کنند که انگار در ترتیبی serial اجرا شده‌اند. این کار را می‌توان با اجرای واقعی transactionها به همان ترتیب یا با اجازه‌دادن به اجرای concurrent و جلوگیری از serialization conflictها از طریق lock یا abort انجام داد.
- استفاده از timestamp و clock در distributed systemها نیز تلاشی برای تحمیل نظم به دنیای بی‌نظم است؛ برای نمونه، برای تشخیص اینکه کدام‌یک از دو write دیرتر رخ داده است.

میان ordering، linearizability و consensus ارتباط‌های عمیقی وجود دارد. این موضوع کمی نظری‌تر و انتزاعی‌تر از بخش‌های دیگر کتاب است، اما به روشن‌شدن اینکه systemها چه کارهایی می‌توانند یا نمی‌توانند انجام دهند کمک می‌کند.

### Ordering and Causality

یکی از دلایل تکرار بحث ordering این است که به حفظ causality کمک می‌کند. در طول کتاب چند نمونه از اهمیت causality دیدیم:

- در مثال «Consistent Prefix Reads»، ناظر یک مکالمه ابتدا پاسخ یک سؤال و بعد خود سؤال را می‌دید. این ترتیب گیج‌کننده است، چون رابطهٔ علت و معلول را نقض می‌کند: اگر به سؤالی پاسخ داده شده، سؤال باید پیش‌تر مطرح شده باشد؛ چون پاسخ‌دهنده باید آن را دیده باشد. پس میان سؤال و پاسخ causal dependency وجود دارد.
- در مثال دیگری، replication میان سه leader را بررسی کردیم و دیدیم network delay ممکن است باعث شود بعضی writeها از writeهای دیگر «سبقت بگیرند». از دید یکی از replicaها، ممکن بود rowای update شود که هنوز وجود نداشت. causality در اینجا می‌گوید row باید پیش از update شدن ساخته شده باشد.
- برای دو operation به‌نام‌های A و B سه حالت ممکن است: A پیش از B رخ داده باشد، B پیش از A رخ داده باشد، یا آن دو concurrent باشند. رابطهٔ **happens-before** بیان دیگری از causality است: اگر A پیش از B رخ داده، ممکن است B از A اطلاع داشته، بر پایهٔ آن ساخته شده یا به آن وابسته بوده باشد. اگر concurrent باشند، هیچ پیوند علّی میانشان نیست؛ یعنی مطمئنیم هیچ‌کدام از دیگری خبر نداشته است.
- در snapshot isolation، transaction از یک consistent snapshot می‌خواند. «consistent» در اینجا یعنی سازگار با causality: اگر snapshot شامل پاسخ باشد، باید سؤال مربوط به آن را هم شامل شود. مشاهدهٔ کل database در یک نقطهٔ زمانی، با causality سازگار است: اثر تمام operationهایی که از نظر علّی پیش از آن نقطه رخ داده‌اند دیده می‌شود، اما اثر operationهای بعدی نه. read skew یعنی خواندن data در stateای که causality را نقض می‌کند.
- نمونه‌های write skew میان transactionها نیز causal dependency را نشان می‌دهند: Alice اجازه می‌یابد از حالت on-call خارج شود، چون transaction تصور می‌کند Bob هنوز on-call است، و برعکس. تصمیم خارج‌شدن از on-call به مشاهدهٔ اینکه چه کسی در حال حاضر on-call است وابسته است. Serializable Snapshot Isolation با track کردن causal dependencyهای میان transactionها write skew را تشخیص می‌دهد.
- در مثال Alice و Bob که فوتبال تماشا می‌کردند، Bob پس از شنیدن اعلام نتیجه از Alice، از server نتیجه‌ای قدیمی می‌گرفت. این نقض causality است: گفتهٔ Alice معلول اعلام نتیجه است، پس Bob نیز پس از شنیدن آن باید بتواند نتیجه را ببیند. همین الگو در وابستگی زمانی میان channelها، در مثال image resizing، تکرار شد.

Causality روی eventها ترتیب تحمیل می‌کند: علت پیش از معلول می‌آید؛ message پیش از دریافت‌شدن فرستاده می‌شود؛ و سؤال پیش از پاسخ مطرح می‌شود. همانند دنیای واقعی، یک رخداد به رخداد بعدی منتهی می‌شود: nodeای dataای را می‌خواند و در نتیجه چیزی می‌نویسد؛ node دیگری آن نوشته را می‌خواند و به‌نوبهٔ خود چیز دیگری می‌نویسد. زنجیره‌های operationهای وابسته به هم، causal order system را تعریف می‌کنند؛ یعنی اینکه چه چیزی پیش از چه چیزی رخ داده است.

اگر system ترتیب تحمیل‌شده از سوی causality را رعایت کند، می‌گوییم **causally consistent** است. برای نمونه، snapshot isolation causal consistency را فراهم می‌کند: اگر هنگام خواندن database، dataای را ببینید، باید بتوانید dataهایی را هم ببینید که از نظر علّی بر آن مقدم‌اند؛ مگر اینکه در این فاصله حذف شده باشند.

#### Causal order یک total order نیست

در **total order** هر دو عضو قابل‌مقایسه‌اند؛ برای هر جفت می‌توان گفت کدام بزرگ‌تر یا کوچک‌تر است. اعداد طبیعی total order دارند: مثلاً ۱۳ از ۵ بزرگ‌تر است. اما مجموعه‌های ریاضی total order ندارند. آیا `{a, b}` از `{b, c}` بزرگ‌تر است؟ نمی‌توان آن‌ها را مقایسه کرد، چون هیچ‌کدام زیرمجموعهٔ دیگری نیست. این دو مجموعه **incomparable** هستند و در نتیجه مجموعه‌ها **partial order** دارند: بعضی وقت‌ها می‌توان یکی را بزرگ‌تر دانست، اما گاهی دو عضو قابل‌مقایسه نیستند.

این تفاوت در consistency modelهای database هم دیده می‌شود:

- **Linearizability:** operationها total order دارند. اگر system طوری رفتار کند که انگار فقط یک copy از data وجود دارد و هر operation اتمیک است، برای هر دو operation می‌توان گفت کدام ابتدا رخ داده است.
- **Causality:** دو operation زمانی concurrent هستند که هیچ‌کدام پیش از دیگری رخ نداده باشد. دو event وقتی مرتب‌اند که رابطهٔ علّی داشته باشند؛ اگر concurrent باشند، قابل‌مقایسه نیستند. بنابراین causality یک partial order تعریف می‌کند، نه total order.

بر اساس این تعریف، در linearizable datastore هیچ operationای concurrent نیست: باید یک timeline واحد وجود داشته باشد که همهٔ operationها در آن total order داشته باشند. ممکن است چند request منتظر پردازش باشند، اما datastore هرکدام را در یک نقطهٔ زمانی، به‌صورت اتمیک و روی یک copy از data پردازش می‌کند. Concurrency به این معناست که timeline شاخه‌شاخه شود و دوباره به هم برسد؛ operationهای شاخه‌های مختلف با هم incomparable یا concurrent هستند. در فصل ۵ نیز نمونه‌ای دیدیم که به‌جای یک خط مستقیم، مجموعه‌ای از operationهای concurrent را نشان می‌داد و arrowها causal dependencyها، یعنی partial ordering، را مشخص می‌کردند.

اگر با distributed version controlهایی مانند Git آشنا باشید، تاریخچهٔ versionها شبیه graph causal dependencyهاست. اغلب commitها در یک خط پشت‌سرهم می‌آیند، اما گاهی چند نفر هم‌زمان روی project کار می‌کنند و branch ایجاد می‌شود؛ سپس commitهای concurrent با merge ترکیب می‌شوند.

#### Linearizability از causal consistency قوی‌تر است

رابطهٔ causal order و linearizability چیست؟ Linearizability causality را نتیجه می‌دهد: هر system خطی‌پذیری causality را درست حفظ می‌کند. به‌ویژه اگر system چند communication channel داشته باشد—مانند message queue و file storage service—linearizability causality را خودکار حفظ می‌کند و نیازی نیست componentها برای این کار timestampها را بین خود ردوبدل کنند.

همین حفظ خودکار causality باعث می‌شود فهم systemهای خطی‌پذیر ساده و استفاده از آن‌ها جذاب باشد. اما linearizability می‌تواند performance و availability را کاهش دهد، به‌ویژه وقتی network delay زیاد باشد؛ مثلاً در systemی که از نظر جغرافیایی توزیع شده است. به همین دلیل بعضی distributed data systemها linearizability را کنار گذاشته‌اند. این تصمیم performance را بهتر می‌کند، اما کار با system را دشوارتر می‌سازد.

راه میانه‌ای هم وجود دارد: linearizability تنها راه حفظ causality نیست. system می‌تواند بدون هزینهٔ performance ناشی از linearizability، causal consistency داشته باشد؛ در این حالت قضیهٔ CAP نیز صدق نمی‌کند. در واقع، causal consistency قوی‌ترین consistency modelی است که به‌دلیل network delay کند نمی‌شود و هنگام network failure نیز available می‌ماند.

در بسیاری از موارد، systemهایی که ظاهراً به linearizability نیاز دارند، در واقع فقط causal consistency می‌خواهند و می‌توان آن را کارآمدتر پیاده‌سازی کرد. بر همین مبنا، پژوهشگران databaseهای تازه‌ای را بررسی می‌کنند که causality را حفظ می‌کنند و از نظر performance و availability شبیه systemهای eventually consistent هستند. این پژوهش نسبتاً جدید است و هنوز به‌طور گسترده وارد systemهای production نشده و چالش‌هایی باقی مانده است؛ بااین‌حال، مسیر امیدوارکننده‌ای برای systemهای آینده است.

#### ثبت causal dependencyها

برای حفظ causality باید بدانیم کدام operation پیش از کدام operation دیگر رخ داده است. این یک partial order است: operationهای concurrent را می‌توان با هر ترتیبی پردازش کرد، اما اگر یکی پیش از دیگری رخ داده باشد، تمام replicaها باید آن‌ها را به همان ترتیب پردازش کنند. بنابراین replica هنگام پردازش operation باید مطمئن شود تمام operationهای دارای تقدم علّی قبلاً پردازش شده‌اند؛ اگر operation قبلی نرسیده باشد، operation بعدی باید منتظر بماند.

برای تشخیص causal dependencyها باید راهی برای توصیف «دانسته‌های» یک node داشته باشیم. اگر node پیش از صدور write مربوط به Y، value مربوط به X را دیده باشد، ممکن است X و Y رابطهٔ علّی داشته باشند. این بررسی شبیه پرسش‌هایی است که در تحقیق دربارهٔ تقلب می‌پرسند: آیا مدیرعامل هنگام تصمیم‌گیری Y از X خبر داشت؟

روش تشخیص اینکه کدام operation پیش از دیگری رخ داده، به روش‌های بخش «Detecting Concurrent Writes» شباهت دارد. آنجا causality را در leaderless datastore و برای تشخیص writeهای concurrent روی یک key بررسی کردیم تا lost update رخ ندهد. causal consistency دامنهٔ گسترده‌تری دارد: باید causal dependencyها را در کل database track کند، نه فقط برای یک key. می‌توان version vectorها را برای این کار تعمیم داد.

برای تعیین causal ordering، database باید بداند application کدام version از data را خوانده است. به همین دلیل در شکل 5-13، شمارهٔ version مربوط به operation قبلی هنگام write به database بازگردانده می‌شود. ایده‌ای مشابه در conflict detection مربوط به SSI دیده می‌شود: هنگام commit شدن transaction، database بررسی می‌کند آیا version dataای که transaction خوانده هنوز به‌روز است یا نه. برای این کار database track می‌کند هر transaction کدام data را خوانده است.

### Sequence Number Ordering

با اینکه causality از نظر نظری مهم است، track کردن تمام causal dependencyها در عمل می‌تواند غیرقابل‌اجرا باشد. در بسیاری از applicationها، clientها پیش از نوشتن، dataهای زیادی می‌خوانند و مشخص نیست write به همهٔ آن readهای قبلی وابسته است یا فقط به بعضی از آن‌ها. ثبت صریح تمام dataهای خوانده‌شده سربار زیادی ایجاد می‌کند.

راه بهتر استفاده از sequence number یا timestamp برای مرتب‌کردن eventهاست. timestamp لازم نیست از ساعت time-of-day یا physical clock گرفته شود؛ ساعت‌های فیزیکی مسئله‌های زیادی دارند. timestamp می‌تواند از **logical clock** بیاید: algorithmی برای تولید دنباله‌ای از عددها جهت شناسایی operationها که معمولاً از counterهایی استفاده می‌کند که با هر operation افزایش می‌یابند.

sequence numberها و timestampها فشرده‌اند و total order می‌سازند: هر operation شماره‌ای یکتا دارد و می‌توان با مقایسهٔ شماره‌ها تشخیص داد کدام بزرگ‌تر است و در نتیجه کدام operation دیرتر رخ داده است.

می‌توان sequence numberها را طوری ساخت که total order آن‌ها با causality سازگار باشد: اگر operation A از نظر علّی پیش از B رخ داده باشد، در total order نیز A باید قبل از B قرار گیرد و شمارهٔ پایین‌تری داشته باشد. operationهای concurrent را می‌توان با هر ترتیبی مرتب کرد. این total order تمام اطلاعات causality را حفظ می‌کند، اما ترتیبی فراتر از حد ضروری causality هم تحمیل می‌کند.

در databaseای با single-leader replication، replication log یک total order از writeها تعریف می‌کند که با causality سازگار است. leader برای هر operation یک counter را افزایش می‌دهد و sequence numberای monotonic به آن اختصاص می‌دهد. اگر followerها writeها را به همان ترتیب replication log apply کنند، state آن‌ها causal consistency دارد؛ حتی اگر از leader عقب باشند.

#### مولدهای sequence number ناسازگار با causality

اگر single leader نداشته باشیم—مثلاً database از نوع multi-leader یا leaderless باشد یا partition شده باشد—تولید sequence number دشوارتر است. چند روش متداول وجود دارد:

- هر node مجموعهٔ مستقل خود از sequence numberها را تولید کند. مثلاً یک node فقط عددهای فرد و node دیگر فقط عددهای زوج را بسازد. به‌طور کلی، می‌توان چند bit از نمایش دودویی sequence number را به شناسهٔ یکتای node اختصاص داد تا دو node شمارهٔ یکسان تولید نکنند.
- به هر operation یک timestamp از time-of-day clock (physical clock) الصاق شود. این timestampها sequential نیستند، اما اگر resolution کافی داشته باشند شاید برای ایجاد total order مناسب باشند. روش last-write-wins conflict resolution از این ایده استفاده می‌کند.
- blockهایی از sequence number از پیش تخصیص داده شوند. مثلاً node A بازهٔ ۱ تا ۱۰۰۰ و node B بازهٔ ۱۰۰۱ تا ۲۰۰۰ را بگیرد. هر node مستقل از دیگری از block خود شماره اختصاص می‌دهد و وقتی شماره‌ها رو به اتمام بود، block تازه‌ای می‌گیرد.

یک total order ناسازگار با causality به‌سادگی با تولید UUID تصادفی برای هر operation و مقایسهٔ لغوی آن‌ها ساخته می‌شود؛ اما چنین ترتیبی چیزی دربارهٔ اینکه کدام operation واقعاً زودتر رخ داده یا دو operation concurrent بوده‌اند نمی‌گوید.

هر سه روش از فرستادن تمام operationها به یک leader برای افزایش counter سریع‌تر و مقیاس‌پذیرترند. آن‌ها برای هر operation شماره‌ای یکتا و تقریباً افزایشی تولید می‌کنند، اما با causality سازگار نیستند؛ چون ترتیب operationها در nodeهای مختلف را درست منعکس نمی‌کنند:

- هر node ممکن است در هر ثانیه تعداد متفاوتی operation پردازش کند. پس counter عددهای زوج ممکن است از counter عددهای فرد عقب بیفتد یا برعکس؛ از روی یک operation با شمارهٔ فرد و دیگری با شمارهٔ زوج نمی‌توان فهمید کدام‌یک از نظر علّی زودتر رخ داده است.
- timestampهای physical clock تحت تأثیر clock skew هستند و ممکن است با causality ناسازگار شوند. ممکن است operationای که از نظر علّی دیرتر رخ داده timestamp پایین‌تری بگیرد.
- در روش block، ممکن است operationای شماره‌ای از بازهٔ ۱۰۰۱ تا ۲۰۰۰ بگیرد و operationای که از نظر علّی بعدتر رخ داده، شماره‌ای از بازهٔ ۱ تا ۱۰۰۰. این ترتیب نیز با causality ناسازگار است.

می‌توان timestampهای physical clock را با causality سازگار کرد. برای نمونه، Google Spanner clock skew مورد انتظار را برآورد می‌کند و پیش از commit شدن write تا پایان بازهٔ عدم‌قطعیت صبر می‌کند؛ به این ترتیب transactionای که از نظر علّی دیرتر است timestamp بزرگ‌تری می‌گیرد. بااین‌حال، بیشتر clockها معیار عدم‌قطعیت لازم را ارائه نمی‌کنند.

#### Lamport timestamp

با وجود اینکه سه روش بالا causality را حفظ نمی‌کنند، روش ساده‌ای برای تولید sequence number سازگار با causality وجود دارد: **Lamport timestamp** که Leslie Lamport در سال ۱۹۷۸ پیشنهاد کرد.

هر node شناسهٔ یکتا و counterای از تعداد operationهای پردازش‌شده دارد. Lamport timestamp یک زوج `(counter, node ID)` است. دو node ممکن است counter برابر داشته باشند، اما افزودن node ID timestamp را یکتا می‌کند.

Lamport timestamp ربطی به ساعت time-of-day فیزیکی ندارد، اما total order می‌سازد: timestampی که counter بزرگ‌تری دارد بزرگ‌تر است؛ اگر counterها مساوی باشند، node ID بزرگ‌تر ترتیب را تعیین می‌کند.

نکتهٔ کلیدی برای سازگاری با causality این است که هر node و هر client بزرگ‌ترین counterای را که تاکنون دیده track می‌کند و آن را در هر request می‌فرستد. اگر node، request یا responseای با counter بزرگ‌تر از counter فعلی‌اش دریافت کند، counter خود را فوراً تا آن مقدار بالا می‌برد.

برای نمونه، client A از node 2 مقدار counter برابر 5 می‌گیرد و سپس همین مقدار را برای node 1 می‌فرستد. counter مربوط به node 1 در آن لحظه فقط 1 است، اما به 5 افزایش می‌یابد؛ پس operation بعدی آن counter برابر 6 خواهد داشت. تا وقتی بیشترین counter همراه هر operation منتقل شود، هر causal dependency باعث افزایش timestamp می‌شود و ordering مربوط به Lamport timestamp با causality سازگار می‌ماند.

Lamport timestamp را گاهی با version vector اشتباه می‌گیرند. این دو شباهت‌هایی دارند اما هدفشان متفاوت است: version vector می‌تواند تشخیص دهد دو operation concurrent هستند یا یکی به‌صورت علّی به دیگری وابسته است؛ Lamport timestamp همیشه یک total order تحمیل می‌کند. از total order مربوط به Lamport timestamp نمی‌توان فهمید دو operation concurrent بوده‌اند یا رابطهٔ علّی داشته‌اند. مزیت Lamport timestamp نسبت به version vector، فشرده‌تر بودن آن است.

#### Timestamp ordering به‌تنهایی کافی نیست

با اینکه Lamport timestamp برای operationها total order سازگار با causality تعریف می‌کند، برای حل بسیاری از مسئله‌های رایج distributed system کافی نیست.

فرض کنید system باید تضمین کند username هر user را به‌صورت یکتا مشخص می‌کند. اگر دو user هم‌زمان بخواهند accountای با username یکسان بسازند، یکی باید موفق و دیگری ناموفق شود. در نگاه اول، total order operationها—مثلاً بر اساس Lamport timestamp—کافی به‌نظر می‌رسد: از میان دو account با username یکسان، timestamp پایین‌تر را برنده اعلام کنیم و دیگری را رد کنیم.

این روش برای تعیین برنده پس از وقوع اتفاق‌ها جواب می‌دهد: وقتی تمام درخواست‌های ساخت username را جمع‌آوری کردیم، timestampها را مقایسه می‌کنیم. اما برای nodeای که همین حالا request ساخت username را گرفته و باید فوراً موفق یا ناموفق بودنش را اعلام کند کافی نیست. node نمی‌داند آیا node دیگری هم‌زمان در حال ساخت account با همان username است یا آن node چه timestampی خواهد گرفت.

برای اطمینان از اینکه node دیگری با timestamp پایین‌تر مشغول claim کردن همان username نیست، باید از همهٔ nodeهای دیگر بپرسیم چه می‌کنند. اگر یکی از nodeها از کار افتاده باشد یا به‌دلیل مشکل network قابل‌دسترسی نباشد، کل system متوقف می‌شود؛ این system fault-tolerant نیست.

مسئله این است که total order operationها فقط پس از گردآوری همهٔ operationها مشخص می‌شود. اگر node دیگری operationهایی تولید کرده باشد که هنوز از آن‌ها خبر نداریم، نمی‌توانیم ترتیب نهایی را بسازیم؛ operationهای ناشناخته ممکن است در جایگاه‌های مختلفی از total order قرار گیرند.

پس برای پیاده‌سازی چیزی مانند uniqueness constraint برای username، داشتن total order کافی نیست؛ باید بدانیم آن order چه زمانی نهایی شده است. اگر operation ساخت usernameای دارید و مطمئن هستید هیچ node دیگری نمی‌تواند claim مربوط به همان username را در total order پیش از operation شما قرار دهد، آن‌وقت می‌توانید با اطمینان operation را موفق اعلام کنید. مفهوم دانستن اینکه total order چه زمانی نهایی شده، در **total order broadcast** بیان می‌شود.

### Total Order Broadcast

اگر program فقط روی یک CPU core اجرا شود، تعریف total order از operationها ساده است: همان ترتیبی که CPU آن‌ها را اجرا کرده است. اما در distributed system توافق تمام nodeها بر سر یک total order یکسان دشوار است. در بخش قبل دیدیم ordering مبتنی بر timestamp یا sequence number به‌اندازهٔ single-leader replication قدرت ندارد؛ مثلاً اگر برای اعمال uniqueness constraint از timestamp ordering استفاده کنید، تحمل fault ممکن نیست.

همان‌طور که گفتیم، single-leader replication با انتخاب یک node به‌عنوان leader و مرتب‌کردن همهٔ operationها روی یک CPU core در leader، total order را تعیین می‌کند. مسئله این است که اگر throughput بیشتر از ظرفیت یک leader باشد چگونه system را scale کنیم و اگر leader از کار افتاد چگونه failover انجام دهیم. در ادبیات distributed systemها، این مسئله **total order broadcast** یا **atomic broadcast** نامیده می‌شود.

#### دامنهٔ ordering guarantee

database partitionشده‌ای که برای هر partition یک single leader دارد، اغلب فقط ordering داخل همان partition را حفظ می‌کند؛ بنابراین نمی‌تواند guaranteeهایی مانند consistent snapshot یا foreign key reference را میان partitionها فراهم کند. total ordering در کل partitionها ممکن است، اما به coordination اضافی نیاز دارد.

Total order broadcast معمولاً protocolی برای تبادل message میان nodeها توصیف می‌شود. به‌طور غیررسمی، دو safety property باید همواره برقرار باشند:

- **Reliable delivery:** هیچ messageای گم نمی‌شود؛ اگر messageای به یک node تحویل داده شود، به همهٔ nodeها هم تحویل داده خواهد شد.
- **Totally ordered delivery:** messageها به تمام nodeها با ترتیب یکسان تحویل داده می‌شوند.

algorithm درست برای total order broadcast باید حتی هنگام fault در node یا network، reliability و ordering را حفظ کند. هنگام قطع network، طبیعتاً messageها تحویل نمی‌شوند؛ اما algorithm می‌تواند retry کند تا پس از برطرف‌شدن network fault، messageها برسند و همچنان با ترتیب درست تحویل شوند.

اصطلاح **atomic broadcast** نام سنتی این مفهوم است، اما ممکن است گمراه‌کننده باشد: ارتباطی با atomicity در ACID transactionها ندارد و فقط به‌طور غیرمستقیم با atomic operationها یا atomic registerها مرتبط است. **Total order multicast** نیز نام دیگری برای همین مفهوم است.

#### کاربردهای total order broadcast

سرویس‌های consensus مانند ZooKeeper و etcd در واقع total order broadcast پیاده‌سازی می‌کنند. این نکته از ارتباط قوی total order broadcast و consensus خبر می‌دهد.

Total order broadcast دقیقاً برای database replication مناسب است: اگر هر message نمایندهٔ یک write به database باشد و هر replica همان writeها را با ترتیب یکسان پردازش کند، replicaها با هم سازگار می‌مانند؛ البته ممکن است موقتاً replication lag داشته باشند. این اصل **state machine replication** نام دارد.

به‌همین‌ترتیب، total order broadcast می‌تواند برای پیاده‌سازی serializable transactionها به‌کار رود: اگر هر message نمایندهٔ transactionای deterministic باشد که به‌صورت stored procedure اجرا می‌شود و همهٔ nodeها messageها را با ترتیب یکسان پردازش کنند، partitionها و replicaهای database با هم سازگار می‌مانند.

ویژگی مهم total order broadcast این است که ترتیب message هنگام تحویل قطعی می‌شود. node مجاز نیست پس از تحویل messageهای بعدی، messageای را به‌صورت بازنگرانه در جایگاه قبلی ترتیب قرار دهد. این ویژگی total order broadcast را از timestamp ordering قوی‌تر می‌کند.

می‌توان total order broadcast را روشی برای ساخت log در نظر گرفت؛ مانند replication log، transaction log یا write-ahead log. تحویل یک message مانند append کردن آن به log است. چون همهٔ nodeها باید messageهای یکسان را با ترتیب یکسان تحویل بگیرند، همگی می‌توانند log را بخوانند و دنبالهٔ یکسانی از messageها را ببینند.

Total order broadcast برای پیاده‌سازی lock serviceای که fencing token می‌دهد نیز مفید است. هر درخواست گرفتن lock به‌صورت messageای به log append می‌شود و messageها بر اساس ترتیب log شماره‌گذاری می‌شوند. sequence number چون monotonic است می‌تواند fencing token باشد. در ZooKeeper این sequence number، `zxid` نام دارد.

#### پیاده‌سازی linearizable storage با total order broadcast

در system خطی‌پذیر، operationها total order دارند. آیا linearizability همان total order broadcast است؟ نه دقیقاً، هرچند ارتباط نزدیکی میان آن‌ها وجود دارد.

Total order broadcast asynchronous است: تضمین می‌شود messageها با ترتیب ثابت و به‌شکل reliable تحویل داده شوند، اما زمان تحویل تضمین نمی‌شود؛ ممکن است یک گیرنده از بقیه عقب بماند. در مقابل، linearizability recency guarantee است: read باید جدیدترین value نوشته‌شده را ببیند.

از نظر رسمی، مسئلهٔ یک linearizable read-write register ساده‌تر است: total order broadcast با consensus معادل است و در مدل asynchronous crash-stop راه‌حل deterministic ندارد، اما read-write register خطی‌پذیر را می‌توان در همان مدل پیاده‌سازی کرد. بااین‌حال، افزودن operation اتمیکی مانند compare-and-set یا increment-and-get، register را به مسئله‌ای معادل consensus تبدیل می‌کند.

بااین‌حال، می‌توان روی total order broadcast یک storage خطی‌پذیر ساخت. فرض کنید برای هر username register خطی‌پذیری با operation اتمیک compare-and-set دارید. مقدار اولیهٔ هر register، `null` است و یعنی username آزاد است. هنگام ساخت username، compare-and-set register آن را اجرا می‌کنید و فقط اگر مقدار قبلی `null` باشد، مقدار را به ID حساب user تغییر می‌دهید. اگر چند user هم‌زمان همان username را claim کنند، به‌دلیل linearizability فقط یکی از operationها موفق می‌شود.

می‌توان با استفاده از total order broadcast به‌عنوان append-only log، compare-and-set خطی‌پذیر را این‌طور پیاده‌سازی کرد:

1. Messageای به log append کنید که موقتاً username موردنظر را claim می‌کند.
2. log را بخوانید و منتظر بمانید تا message خودتان دوباره به شما تحویل داده شود.
3. messageهای claim‌کنندهٔ آن username را بررسی کنید. اگر اولین message مربوط به username موردنظر، message خودتان بود، موفق شده‌اید؛ می‌توانید claim را commit کنید—مثلاً با append کردن message دیگری به log—و موفقیت را به client اعلام کنید. اگر اولین message متعلق به user دیگری بود، operation را abort کنید.

اگر پس از enqueue شدن write فوراً آن را acknowledge کنید و منتظر تحویل message نمانید، رفتاری شبیه memory consistency model در CPUهای چند‌هسته‌ای x86 به‌دست می‌آید؛ این مدل نه linearizable است و نه sequentially consistent.

چون log entryها با ترتیب یکسان به همهٔ nodeها تحویل داده می‌شوند، در صورت چند write هم‌زمان، تمام nodeها توافق می‌کنند کدام زودتر بوده است. انتخاب اولین write متعارض به‌عنوان برنده و abort کردن بقیه باعث می‌شود همهٔ nodeها بر سر commit یا abort شدن write توافق داشته باشند. می‌توان از روشی مشابه برای ساخت multi-object transactionهای serializable روی log استفاده کرد.

این روش writeهای خطی‌پذیر را تضمین می‌کند، اما readهای خطی‌پذیر را نه: اگر از storeای بخوانید که به‌شکل asynchronous از log به‌روز می‌شود، ممکن است value قدیمی باشد. دقیق‌تر اینکه این روش **sequential consistency**—که timeline consistency هم نامیده می‌شود—را فراهم می‌کند؛ guaranteeای کمی ضعیف‌تر از linearizability. برای خطی‌پذیر کردن read چند راه وجود دارد:

- read را از مسیر log عبور دهید: messageای append کنید، log را بخوانید و وقتی message به شما تحویل شد، read واقعی را انجام دهید. جایگاه message در log نقطهٔ زمانی read را تعیین می‌کند. quorum read در etcd تا حدی به این روش عمل می‌کند.
- اگر بتوان جایگاه جدیدترین message در log را به‌شکل linearizable به‌دست آورد، آن جایگاه را query کنید، منتظر بمانید تمام entryهای تا آن نقطه به شما تحویل شوند و سپس read را انجام دهید. ایدهٔ operation `sync()` در ZooKeeper همین است.
- از replicaای بخوانید که writeها به‌صورت synchronous روی آن اعمال می‌شوند و بنابراین حتماً به‌روز است. از این روش در chain replication استفاده می‌شود.

#### پیاده‌سازی total order broadcast با linearizable storage

در بخش قبل دیدیم چطور می‌توان با total order broadcast یک compare-and-set خطی‌پذیر ساخت. جهت عکس هم ممکن است: فرض کنیم linearizable storage داریم و از آن total order broadcast بسازیم.

ساده‌ترین راه این است که register خطی‌پذیری از نوع integer داشته باشیم که operation اتمیک increment-and-get را فراهم کند. compare-and-set اتمیک نیز می‌تواند همین کار را انجام دهد.

برای هر messageای که باید از total order broadcast بگذرد، register خطی‌پذیر را increment-and-get می‌کنیم و مقدار برگشتی را به‌عنوان sequence number به message می‌چسبانیم. سپس message را برای همهٔ nodeها می‌فرستیم و messageهای گم‌شده را دوباره ارسال می‌کنیم. گیرنده‌ها messageها را بر اساس sequence number و به‌ترتیب تحویل می‌دهند.

برخلاف Lamport timestamp، عددهایی که از increment کردن register خطی‌پذیر می‌آیند gap ندارند. بنابراین اگر nodeای message شمارهٔ ۴ را تحویل گرفته و message شمارهٔ ۶ به آن برسد، می‌داند باید منتظر message شمارهٔ ۵ بماند. این در مورد Lamport timestampها صادق نیست و تفاوت کلیدی total order broadcast با timestamp ordering همین است.

ساخت integer خطی‌پذیری با operation اتمیک increment-and-get در حالت بدون fault ساده است؛ کافی است آن را در variableای روی یک node نگه داریم. دشواری، مدیریت قطع ارتباط network با آن node و بازیابی value پس از failure همان node است. اگر مولد sequence number خطی‌پذیر را با دقت بررسی کنید، در نهایت به consensus algorithm می‌رسید.

این اتفاق تصادفی نیست: می‌توان ثابت کرد register خطی‌پذیر با compare-and-set اتمیک یا increment-and-get و total order broadcast، هر دو معادل consensus هستند. اگر یکی از این مسئله‌ها را حل کنید، می‌توانید آن را به راه‌حلی برای دو مسئلهٔ دیگر تبدیل کنید. این نتیجه‌ای عمیق و در عین حال شگفت‌آور است؛ در ادامهٔ chapter مسئلهٔ consensus را مستقیم بررسی می‌کنیم.

## Key Terms

- `Ordering Guarantees` — تضمین‌هایی دربارهٔ ترتیب قابل‌مشاهدهٔ operationها یا تحویل messageها در یک data system یا distributed system.
- `Causality` — رابطه‌ای که مشخص می‌کند کدام eventها بر eventهای دیگر اثر گذاشته‌اند و در نتیجه باید پیش از آن‌ها دیده یا پردازش شوند.
- `Causal Consistency` — consistency modelی که ترتیب علت‌ومعلولی eventها را روی تمام replicaها حفظ می‌کند، اما برای eventهای concurrent ترتیب سراسری تحمیل نمی‌کند.
- `Total Order` — ترتیبی که هر دو event در آن قابل‌مقایسه‌اند و یکی پیش از دیگری قرار می‌گیرد.
- `Partial Order` — ترتیبی که فقط eventهای دارای رابطهٔ مشخص، مانند رابطهٔ علّی، را مرتب می‌کند؛ eventهای concurrent ممکن است incomparable باشند.
- `Sequence Number` — عدد یکتایی که برای مرتب‌کردن operationها یا messageها به آن‌ها اختصاص داده می‌شود؛ ویژگی‌های آن به روش تولیدش بستگی دارد.
- `Lamport Timestamp` — زوج `(counter, node ID)` که total order سازگار با causality می‌سازد، اما به‌تنهایی concurrency را از causal dependency تشخیص نمی‌دهد.
- `Total Order Broadcast` — protocolی که messageها را به‌شکل reliable و با ترتیب یکسان به تمام nodeها تحویل می‌دهد و ترتیب نهایی را تثبیت می‌کند.
- `Atomic Broadcast` — نام دیگر total order broadcast؛ اصطلاحی که به atomicity در ACID transactionها اشاره ندارد.
- `State Machine Replication` — روش replication که در آن همهٔ replicaها commandهای یکسان را با ترتیب یکسان اجرا می‌کنند تا state سازگار بماند.
- `Sequential Consistency` — guaranteeای که operationهای همهٔ clientها را در یک ترتیب کلی سازگار با ترتیب هر client قرار می‌دهد، اما الزاماً ترتیب real-time را مانند linearizability حفظ نمی‌کند.
- `Reliable Delivery` — propertyای در broadcast که تضمین می‌کند message تحویل‌شده به یک node، در نهایت به همهٔ nodeها تحویل شود.
- `Fencing Token` — tokenای با مقدار monotonic که ترتیب مالکیت lock یا درخواست‌ها را نشان می‌دهد و به resource کمک می‌کند درخواست‌های قدیمی را رد کند.

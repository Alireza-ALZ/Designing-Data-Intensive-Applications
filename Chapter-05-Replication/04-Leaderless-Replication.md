# Chapter 5 — Replication

## Leaderless Replication

روش‌های replicationای که تا اینجا در این chapter بررسی کردیم - یعنی single-leader و multi-leader replication - بر این ایده بنا شده‌اند که client یک write request را به یک node (leader) می‌فرستد و database system مسئول copy کردن آن write روی replicaهای دیگر است. Leader order پردازش writeها را تعیین می‌کند و followerها writeهای leader را با همان order apply می‌کنند.

برخی data storage systemها approach متفاوتی دارند: مفهوم leader را کنار می‌گذارند و به هر replica اجازه می‌دهند writeهای clientها را مستقیماً بپذیرد. برخی از نخستین data systemهای replicated، leaderless بودند [1, 44]، اما این ایده در دوران سلطهٔ relational databaseها تا حد زیادی فراموش شد. پس از آنکه Amazon از این architecture در system داخلی `Dynamo` استفاده کرد، این روش بار دیگر به architecture محبوب databaseها تبدیل شد [37].* `Riak`، `Cassandra` و `Voldemort` datastoreهای open sourceای هستند که modelهای leaderless replication آن‌ها از Dynamo الهام گرفته‌اند؛ بنابراین این نوع database با عنوان `Dynamo-style` نیز شناخته می‌شود.

در برخی implementationهای leaderless، client writeهای خود را مستقیماً برای چند replica می‌فرستد، درحالی‌که در برخی دیگر یک coordinator node این کار را از طرف client انجام می‌دهد. بااین‌حال، برخلاف leader database، این coordinator order خاصی برای writeها enforce نمی‌کند. همان‌طور که خواهیم دید، این تفاوت در design پیامدهای مهمی برای روش استفاده از database دارد.

*پاورقی: Dynamo برای userهای خارج از Amazon در دسترس نیست. به‌طور گیج‌کننده‌ای، AWS محصول database hostedای به نام `DynamoDB` ارائه می‌کند که architecture کاملاً متفاوتی دارد و بر single-leader replication بنا شده است.*

### Writing to the Database When a Node Is Down

فرض کنید databaseای با سه replica دارید و یکی از replicaها در حال حاضر unavailable است؛ شاید برای نصب system update در حال reboot باشد. در leader-based configuration، اگر بخواهید به پردازش writeها ادامه دهید، ممکن است لازم باشد failover انجام دهید (به بخش «Handling Node Outages» در صفحهٔ ۱۵۶ مراجعه کنید).

اما در leaderless configuration، failover وجود ندارد. شکل ۵-۱۰ اتفاقی را که رخ می‌دهد نشان می‌دهد: client (user شمارهٔ ۱۲۳۴) write را به‌صورت parallel برای هر سه replica می‌فرستد؛ دو replica available write را می‌پذیرند، اما replica unavailable آن را دریافت نمی‌کند. فرض کنید کافی است دو replica از سه replica write را acknowledge کنند؛ پس از اینکه user ۱۲۳۴ دو response از نوع `OK` دریافت کرد، write را successful در نظر می‌گیریم. Client به‌سادگی unavailable بودن یکی از replicaها و دریافت نکردن write توسط آن را نادیده می‌گیرد.

**شکل ۵-۱۰.** quorum write، quorum read و read repair پس از node outage.

حالا تصور کنید node unavailable دوباره online شود و clientها شروع کنند از آن read کردن. تمام writeهایی که هنگام down بودن node رخ داده‌اند، روی آن node وجود ندارند. بنابراین اگر از آن node read کنید، ممکن است valueهای stale (outdated) را به‌عنوان response دریافت کنید.

برای حل این مشکل، وقتی client از database read می‌کند، request خود را فقط برای یک replica نمی‌فرستد؛ requestهای read نیز به‌صورت parallel برای چند node ارسال می‌شوند. Client ممکن است از nodeهای مختلف responseهای متفاوتی دریافت کند؛ یعنی value up-to-date از یک node و value stale از node دیگر. برای تعیین اینکه کدام value جدیدتر است از version numberها استفاده می‌شود (به بخش «Detecting Concurrent Writes» در صفحهٔ ۱۸۴ مراجعه کنید).

#### Read repair و anti-entropy

Replication scheme باید تضمین کند که در نهایت تمام data روی تمام replicaها copy شود. پس از online شدن node unavailable، چگونه می‌توان آن را با writeهایی که از دست داده است catch up کرد؟

در Dynamo-style datastoreها معمولاً از دو mechanism استفاده می‌شود:

**Read repair**

وقتی client به‌صورت parallel از چند node read می‌کند، می‌تواند responseهای stale را detect کند. برای مثال، در شکل ۵-۱۰، user شمارهٔ ۲۳۴۵ از replica 3 یک value با version 6 و از replicaهای 1 و 2 یک value با version 7 دریافت می‌کند. Client می‌بیند value مربوط به replica 3 stale است و value جدیدتر را دوباره روی آن replica write می‌کند. این approach برای valueهایی که مرتب read می‌شوند به‌خوبی کار می‌کند.

**Anti-entropy process**

علاوه بر این، برخی datastoreها background processای دارند که دائماً به دنبال تفاوت در data میان replicaها می‌گردد و هر data missing را از یک replica به replica دیگر copy می‌کند. برخلاف replication log در leader-based replication، این anti-entropy process writeها را با order خاصی copy نمی‌کند و ممکن است پیش از copy شدن data، delay قابل‌توجهی وجود داشته باشد.

همهٔ systemها هر دو mechanism را implement نمی‌کنند؛ برای مثال، `Voldemort` در حال حاضر anti-entropy process ندارد. توجه کنید که بدون anti-entropy process، valueهایی که به‌ندرت read می‌شوند ممکن است روی برخی replicaها missing باشند و در نتیجه durability کمتری داشته باشند، چون read repair فقط زمانی انجام می‌شود که application آن value را read کند.

#### Quorumها برای read و write

در مثال شکل ۵-۱۰، write را successful در نظر گرفتیم، حتی اگر فقط روی دو replica از سه replica پردازش شده باشد. اگر فقط یک replica از سه replica write را قبول کند چه؟ تا چه حد می‌توانیم این روش را ادامه دهیم؟

اگر بدانیم هر write موفق حتماً روی دست‌کم دو replica از سه replica وجود دارد، یعنی حداکثر یک replica می‌تواند stale باشد. بنابراین اگر از دست‌کم دو replica read کنیم، مطمئن هستیم حداقل یکی از آن دو up to date است. اگر replica سوم down باشد یا کند response دهد، readها همچنان می‌توانند value up-to-date را برگردانند.

به‌طور کلی، اگر `n` replica داشته باشیم، هر write باید توسط `w` node تأیید شود تا successful در نظر گرفته شود و برای هر read باید دست‌کم `r` node را query کنیم. (در مثال ما، `n = 3`، `w = 2` و `r = 2` است.) تا زمانی که `w + r > n` باشد، انتظار داریم هنگام read کردن value up-to-date دریافت کنیم، چون دست‌کم یکی از `r` nodeهایی که read می‌کنیم باید value up-to-date را دیده باشد. Read و writeهایی که از این مقدارهای `r` و `w` پیروی می‌کنند `quorum read` و `quorum write` نامیده می‌شوند [44].‡ می‌توانید `r` و `w` را حداقل تعداد voteهای لازم برای معتبر بودن read یا write در نظر بگیرید.

*پاورقی: گاهی برای متمایز کردن این نوع quorum از sloppy quorum (که در بخش «Sloppy Quorums and Hinted Handoff» بررسی می‌شود) به آن `strict quorum` گفته می‌شود.*

در Dynamo-style databaseها، parameterهای `n`، `w` و `r` معمولاً قابل configuration هستند. انتخاب رایج این است که `n` عددی فرد باشد (معمولاً ۳ یا ۵) و `w = r = (n + 1) / 2` تنظیم شود (با round کردن به بالا). بااین‌حال، می‌توانید این numberها را متناسب با نیاز خود تغییر دهید. برای مثال، workloadای با write کم و read زیاد ممکن است از `w = n` و `r = 1` سود ببرد. این configuration readها را سریع‌تر می‌کند، اما عیب آن این است که failure فقط یک node باعث می‌شود تمام writeهای database fail شوند.

ممکن است در cluster بیش از `n` node وجود داشته باشد، اما هر value مشخص فقط روی `n` node ذخیره می‌شود. این کار اجازه می‌دهد dataset partition شود و datasetهایی بزرگ‌تر از ظرفیت یک node پشتیبانی شوند. در Chapter 6 دوباره به partitioning برمی‌گردیم.

شرط quorum یعنی `w + r > n` به system اجازه می‌دهد nodeهای unavailable را به شکل زیر تحمل کند:

- اگر `w < n` باشد، وقتی یک node unavailable است همچنان می‌توانیم writeها را پردازش کنیم.
- اگر `r < n` باشد، وقتی یک node unavailable است همچنان می‌توانیم readها را پردازش کنیم.
- با `n = 3`، `w = 2` و `r = 2` می‌توانیم یک node unavailable را تحمل کنیم.
- با `n = 5`، `w = 3` و `r = 3` می‌توانیم دو node unavailable را تحمل کنیم. این حالت در شکل ۵-۱۱ نشان داده شده است.
- معمولاً readها و writeها همیشه به‌صورت parallel برای هر `n` replica ارسال می‌شوند. Parameterهای `w` و `r` تعیین می‌کنند منتظر response چند node بمانیم؛ یعنی چند node از `n` node باید success را گزارش کنند تا read یا write را successful در نظر بگیریم.

**شکل ۵-۱۱.** اگر `w + r > n` باشد، دست‌کم یکی از `r` replicaای که read می‌کنید باید جدیدترین write موفق را دیده باشد.

اگر تعداد nodeهای available از `w` یا `r` موردنیاز کمتر باشد، write یا read با error برمی‌گردد. Node ممکن است به دلایل زیادی unavailable باشد: چون down شده (crash کرده یا خاموش است)، به‌دلیل error هنگام اجرای operation (مثلاً چون disk پر است و نمی‌تواند write کند)، به‌دلیل network interruption میان client و node یا هر دلیل دیگری. تنها چیزی که اهمیت دارد این است که node response موفق برگردانده یا نه؛ لازم نیست نوع مختلف fault را از هم متمایز کنیم.

### Limitations of Quorum Consistency

اگر `n` replica داشته باشید و `w` و `r` را طوری انتخاب کنید که `w + r > n` باشد، معمولاً می‌توانید انتظار داشته باشید هر read جدیدترین value نوشته‌شده برای یک key را برگرداند. دلیل آن این است که مجموعهٔ nodeهایی که روی آن‌ها write کرده‌اید با مجموعهٔ nodeهایی که از آن‌ها read می‌کنید overlap دارد. یعنی در میان nodeهایی که read می‌کنید باید دست‌کم یک node با جدیدترین value وجود داشته باشد؛ همان‌طور که در شکل ۵-۱۱ نشان داده شده است.

اغلب `r` و `w` به‌اندازهٔ majority (بیش از `n/2` node) انتخاب می‌شوند، چون این کار تضمین می‌کند `w + r > n` باشد و در عین حال تا `n/2` failure مربوط به nodeها را تحمل می‌کند. اما quorumها الزاماً majority نیستند؛ فقط overlap داشتن setهای node مورد استفاده در operation read و write، در دست‌کم یک node، اهمیت دارد. Assignmentهای دیگری برای quorum ممکن است و این موضوع انعطاف بیشتری در طراحی distributed algorithmها فراهم می‌کند [45].

همچنین می‌توانید `w` و `r` را کوچک‌تر انتخاب کنید، به‌طوری‌که `w + r ≤ n` باشد (یعنی شرط quorum برقرار نباشد). در این حالت read و write همچنان برای `n` node ارسال می‌شوند، اما برای موفق شدن operation به response موفق تعداد کمتری node نیاز است.

با `w` و `r` کوچک‌تر، احتمال read کردن valueهای stale بیشتر است، چون احتمال اینکه read شما node دارای جدیدترین value را شامل نشده باشد افزایش می‌یابد. در عوض، این configuration latency کمتر و availability بیشتری فراهم می‌کند: اگر network interruption رخ دهد و replicaهای زیادی unreachable شوند، احتمال بیشتری وجود دارد که بتوانید به پردازش read و write ادامه دهید. Database فقط زمانی برای write یا read unavailable می‌شود که تعداد replicaهای reachable به‌ترتیب از `w` یا `r` کمتر شود.

بااین‌حال، حتی وقتی `w + r > n` باشد نیز احتمال دارد در edge caseهایی valueهای stale برگردانده شوند. این موارد به implementation بستگی دارند، اما scenarioهای ممکن شامل این‌ها هستند:

- اگر از sloppy quorum استفاده شود (به بخش «Sloppy Quorums and Hinted Handoff» مراجعه کنید)، ممکن است `w` write روی nodeهایی متفاوت از nodeهای مربوط به `r` read قرار گرفته باشند؛ بنابراین دیگر overlap تضمین‌شده‌ای میان nodeهای `r` و nodeهای `w` وجود ندارد [46].
- اگر دو write به‌صورت concurrent رخ دهند، مشخص نیست کدام‌یک اول رخ داده است. در این حالت، تنها راه‌حل safe merge کردن writeهای concurrent است (به بخش «Handling Write Conflicts» در صفحهٔ ۱۷۱ مراجعه کنید). اگر winner بر اساس timestamp انتخاب شود (last write wins)، ممکن است به‌دلیل clock skew writeها از دست بروند [35]. در بخش «Detecting Concurrent Writes» در صفحهٔ ۱۸۴ دوباره به این موضوع برمی‌گردیم.
- اگر write هم‌زمان با read رخ دهد، ممکن است write فقط روی برخی replicaها منعکس شده باشد. در این حالت مشخص نیست read باید value قدیمی را برگرداند یا value جدید را.
- اگر write روی برخی replicaها موفق و روی برخی دیگر fail شود (برای مثال چون disk برخی nodeها پر است) و در مجموع روی کمتر از `w` replica موفق شده باشد، روی replicaهایی که write در آن‌ها موفق بوده rollback نمی‌شود. یعنی اگر writeای failed گزارش شود، readهای بعدی ممکن است value آن write را برگردانند یا برنگردانند [47].
- اگر nodeای که value جدید را نگه می‌دارد fail شود و data آن از replicaای که value قدیمی را دارد restore شود، تعداد replicaهایی که value جدید را نگه می‌دارند ممکن است از `w` کمتر شود و شرط quorum را بشکند.
- حتی اگر همه‌چیز به‌درستی کار کند، edge caseهایی وجود دارند که در آن‌ها به‌دلیل timing نامناسب، نتیجهٔ stale دریافت می‌کنید؛ همان‌طور که در بخش «Linearizability and quorums» در صفحهٔ ۳۳۴ خواهیم دید.

بنابراین با اینکه quorumها ظاهراً تضمین می‌کنند read جدیدترین value نوشته‌شده را برگرداند، در عمل مسئله به این سادگی نیست. Dynamo-style databaseها معمولاً برای use caseهایی optimize شده‌اند که می‌توانند eventual consistency را تحمل کنند. Parameterهای `w` و `r` اجازه می‌دهند احتمال read شدن value stale را تنظیم کنید، اما بهتر است آن‌ها را guaranteeهای مطلق در نظر نگیرید.

به‌خصوص معمولاً guaranteeهایی را که در بخش «Problems with Replication Lag» در صفحهٔ ۱۶۱ بررسی کردیم دریافت نمی‌کنید (reading your writes، monotonic reads یا consistent prefix reads)، بنابراین anomalyهای قبلی ممکن است در application رخ دهند. Guaranteeهای قوی‌تر معمولاً به transaction یا consensus نیاز دارند. در Chapter 7 و Chapter 9 دوباره به این موضوع‌ها برمی‌گردیم.

#### Monitoring staleness

از دید operational مهم است monitor کنید database شما resultهای up-to-date برمی‌گرداند یا نه. حتی اگر application بتواند stale read را تحمل کند، باید از health مربوط به replication آگاه باشید. اگر replication به‌طور قابل‌توجهی عقب بیفتد، باید به شما alert داده شود تا بتوانید علت را بررسی کنید (برای مثال، مشکل در network یا overloaded بودن یک node).

در leader-based replication، database معمولاً metricهایی برای replication lag expose می‌کند که می‌توانید آن‌ها را به monitoring system بدهید. این کار ممکن است، چون writeها روی leader و followerها با همان order apply می‌شوند و هر node یک position در replication log دارد (تعداد writeهایی که به‌صورت local apply کرده است). با subtract کردن position فعلی follower از position فعلی leader می‌توانید مقدار replication lag را اندازه‌گیری کنید.

بااین‌حال، در systemهای leaderless replication order ثابتی برای apply شدن writeها وجود ندارد و این موضوع monitoring را دشوارتر می‌کند. علاوه بر این، اگر database فقط از read repair استفاده کند (و anti-entropy نداشته باشد)، هیچ limitی برای قدیمی بودن یک value وجود ندارد؛ اگر valueای فقط به‌ندرت read شود، valueای که replica stale برمی‌گرداند می‌تواند بسیار قدیمی باشد.

دربارهٔ اندازه‌گیری staleness مربوط به replica در databaseهای leaderless replication و پیش‌بینی درصد مورد انتظار stale readها بر اساس parameterهای `n`، `w` و `r` researchهایی انجام شده است [48]. متأسفانه این کار هنوز به practice رایج تبدیل نشده، اما بهتر است measurement مربوط به staleness در مجموعهٔ استاندارد metricهای database قرار بگیرد. Eventual consistency عمداً guaranteeای مبهم است، اما برای operability مهم است بتوانیم «eventual» را quantify کنیم.

### Sloppy Quorums and Hinted Handoff

Databaseهایی که quorumهای مناسبی دارند می‌توانند failure مربوط به nodeهای منفرد را بدون نیاز به failover تحمل کنند. آن‌ها می‌توانند کند شدن nodeهای منفرد را نیز تحمل کنند، چون requestها لازم نیست منتظر response تمام `n` node بمانند؛ requestها پس از response دادن `w` یا `r` node می‌توانند برگردند. این ویژگی‌ها databaseهای leaderless replication را برای use caseهایی جذاب می‌کنند که به availability بالا و latency پایین نیاز دارند و می‌توانند stale readهای occasional را تحمل کنند.

بااین‌حال، quorumهایی که تا اینجا توضیح دادیم به‌اندازهٔ امکان خود fault-tolerant نیستند. یک network interruption می‌تواند به‌راحتی ارتباط client را با تعداد زیادی از database nodeها قطع کند. اگرچه آن nodeها زنده‌اند و clientهای دیگر ممکن است بتوانند به آن‌ها connect شوند، برای clientی که از database nodeها جدا شده است، آن nodeها عملاً مرده محسوب می‌شوند. در این وضعیت احتمال دارد کمتر از `w` یا `r` node reachable باقی بماند و client دیگر نتواند quorum تشکیل دهد.

در cluster بزرگی که تعداد nodeهای آن به‌طور قابل‌توجهی بیشتر از `n` است، احتمال دارد client هنگام network interruption بتواند به برخی database nodeها connect شود، اما نه به nodeهایی که برای تشکیل quorum مربوط به یک value مشخص لازم هستند. در این حالت، database designerها با یک trade-off مواجه می‌شوند:

- آیا بهتر است برای تمام requestهایی که نمی‌توانیم به quorumای از `w` یا `r` node برسیم error برگردانیم؟
- یا باید writeها را در هر صورت بپذیریم و آن‌ها را روی nodeهای reachableای write کنیم که جزو `n` nodeای نیستند که value معمولاً روی آن‌ها قرار دارد؟

گزینهٔ دوم `sloppy quorum` نام دارد [37]: read و write همچنان به `w` و `r` response موفق نیاز دارند، اما این responseها ممکن است از nodeهایی دریافت شوند که جزو `n` node «home» تعیین‌شده برای آن value نیستند. برای تشبیه، اگر خود را پشت در خانه‌تان قفل کرده باشید، می‌توانید در خانهٔ همسایه را بزنید و موقتاً از او بخواهید اجازه دهد روی مبلش بمانید.

وقتی network interruption برطرف شد، هر writeای که یک node موقتاً از طرف node دیگر پذیرفته است به nodeهای «home» مناسب ارسال می‌شود. این کار `hinted handoff` نام دارد. (وقتی کلید خانه‌تان را پیدا کردید، همسایه مؤدبانه از شما می‌خواهد از روی مبل بلند شوید و به خانهٔ خودتان بروید.)

Sloppy quorumها به‌خصوص برای افزایش write availability مفید هستند: تا زمانی که هر `w` nodeای available باشد، database می‌تواند writeها را بپذیرد. اما در این حالت، حتی وقتی `w + r > n` باشد نمی‌توانید مطمئن باشید جدیدترین value مربوط به یک key را read می‌کنید، چون ممکن است جدیدترین value به‌صورت موقت روی nodeهایی خارج از `n` node write شده باشد [47].

بنابراین sloppy quorum در معنای سنتی واقعاً quorum نیست. این روش فقط نوعی assurance برای durability است؛ یعنی data جایی روی `w` node ذخیره شده است. تا زمانی که hinted handoff کامل نشده باشد، هیچ guaranteeای وجود ندارد که read از `r` node آن data را ببیند.

Sloppy quorumها در تمام Dynamo implementationهای رایج optional هستند. در `Riak` به‌صورت default enabled و در `Cassandra` و `Voldemort` به‌صورت default disabled هستند [46, 49, 50].

#### Multi-datacenter operation

پیش‌تر cross-datacenter replication را به‌عنوان use case مربوط به multi-leader replication بررسی کردیم (به بخش «Multi-Leader Replication» در صفحهٔ ۱۶۸ مراجعه کنید). Leaderless replication نیز برای multi-datacenter operation مناسب است، چون برای تحمل concurrent writeهای متناقض، network interruption و latency spike طراحی شده است.

`Cassandra` و `Voldemort` پشتیبانی multi-datacenter خود را درون model معمول leaderless implement می‌کنند: تعداد replicaها یعنی `n` شامل nodeهای تمام datacenterهاست و در configuration می‌توانید مشخص کنید چند replica از `n` replica در هر datacenter قرار داشته باشد. هر write مربوط به client برای تمام replicaها ارسال می‌شود، صرف‌نظر از datacenter؛ اما client معمولاً فقط منتظر acknowledgment از quorumای از nodeهای datacenter محلی خود می‌ماند تا از delay و interruption روی link میان datacenterها تأثیر نگیرد. Writeهایی که برای datacenterهای دیگر latency بیشتری دارند اغلب به‌صورت asynchronous configure می‌شوند، هرچند configuration تا حدی انعطاف‌پذیر است [50, 51].

`Riak` تمام communication میان clientها و database nodeها را درون یک datacenter local نگه می‌دارد؛ بنابراین `n` تعداد replicaها درون یک datacenter را مشخص می‌کند. Cross-datacenter replication میان database clusterها به‌صورت asynchronous در background انجام می‌شود و شبیه multi-leader replication است [52].

### Detecting Concurrent Writes

Dynamo-style databaseها به چند client اجازه می‌دهند به‌صورت concurrent روی یک key write کنند؛ بنابراین حتی اگر از strict quorum استفاده شود، conflict رخ خواهد داد. این وضعیت شبیه multi-leader replication است (به بخش «Handling Write Conflicts» در صفحهٔ ۱۷۱ مراجعه کنید)، با این تفاوت که در Dynamo-style databaseها conflict ممکن است هنگام read repair یا hinted handoff نیز ایجاد شود.

مشکل این است که eventها به‌دلیل variable network delay و partial failure ممکن است با order متفاوتی به nodeهای مختلف برسند. برای مثال، شکل ۵-۱۲ دو client به نام A و B را نشان می‌دهد که به‌صورت هم‌زمان روی keyای به نام X در datastore سه‌nodeای write می‌کنند:

- Node 1 write مربوط به A را دریافت می‌کند، اما به‌دلیل یک outage موقت هرگز write مربوط به B را دریافت نمی‌کند.
- Node 2 ابتدا write مربوط به A و سپس write مربوط به B را دریافت می‌کند.
- Node 3 ابتدا write مربوط به B و سپس write مربوط به A را دریافت می‌کند.

**شکل ۵-۱۲.** writeهای concurrent در یک Dynamo-style datastore؛ order کاملاً مشخصی وجود ندارد.

اگر هر node هر بار که write requestای از client دریافت می‌کند value مربوط به key را overwrite کند، nodeها برای همیشه inconsistent می‌شوند؛ همان‌طور که در final get request شکل ۵-۱۲ دیده می‌شود: node 2 تصور می‌کند value نهایی X برابر B است، درحالی‌که nodeهای دیگر value را A می‌دانند.

برای رسیدن به eventual consistency، replicaها باید به سمت یک value یکسان converge کنند. چگونه این کار را انجام می‌دهند؟ ممکن است امیدوار باشیم replicated databaseها این کار را به‌صورت automatic handle کنند، اما متأسفانه بیشتر implementationها عملکرد خوبی ندارند. اگر بخواهید از data loss جلوگیری کنید، شما به‌عنوان application developer باید دربارهٔ internals مربوط به conflict handling در database خود اطلاعات زیادی داشته باشید.

در بخش «Handling Write Conflicts» به‌طور کوتاه به چند technique برای conflict resolution اشاره کردیم. پیش از پایان این chapter، بیایید این مسئله را کمی دقیق‌تر بررسی کنیم.

#### Last write wins (discarding concurrent writes)

یک approach برای رسیدن به convergence نهایی این است که فرض کنیم هر replica فقط باید جدیدترین value را نگه دارد و valueهای «قدیمی‌تر» را overwrite و discard کند. در این صورت، تا زمانی که راهی بدون ابهام برای تعیین اینکه کدام write «جدیدتر» است داشته باشیم و هر write در نهایت به تمام replicaها copy شود، replicaها در نهایت به یک value یکسان converge می‌کنند.

همان‌طور که quotation markهای اطراف واژهٔ «جدید» نشان می‌دهند، این ایده در واقع گمراه‌کننده است. در مثال شکل ۵-۱۲، هیچ‌کدام از clientها هنگام ارسال write request خود به database nodeها از دیگری خبر نداشتند؛ بنابراین مشخص نیست کدام write اول رخ داده است. در واقع، گفتن اینکه یکی از آن‌ها «اول» رخ داده چندان معنایی ندارد: می‌گوییم writeها concurrent هستند و order آن‌ها تعریف نشده است.

با اینکه writeها order طبیعی ندارند، می‌توانیم orderی arbitrary به آن‌ها تحمیل کنیم. برای مثال، به هر write یک timestamp attach کنیم، بزرگ‌ترین timestamp را «جدیدترین» در نظر بگیریم و writeهای دارای timestamp کوچک‌تر را discard کنیم. این conflict resolution algorithm که `last write wins (LWW)` نام دارد، تنها روش conflict resolution پشتیبانی‌شده در Cassandra [53] و featureای optional در Riak [35] است.

LWW به هدف eventual convergence می‌رسد، اما هزینهٔ آن durability است: اگر چند write concurrent روی یک key وجود داشته باشد، حتی اگر تمام آن writeها به client successful گزارش شده باشند (چون روی `w` replica write شده‌اند)، فقط یکی از writeها باقی می‌ماند و بقیه به‌صورت silent discard می‌شوند. علاوه بر این، LWW حتی ممکن است writeهایی را که concurrent نیستند نیز drop کند؛ این موضوع را در بخش «Timestamps for ordering events» در صفحهٔ ۲۹۱ بررسی خواهیم کرد.

در برخی situationها، مانند caching، شاید از دست رفتن write قابل‌قبول باشد. اما اگر data loss قابل‌قبول نیست، LWW انتخاب ضعیفی برای conflict resolution است. تنها روش safe برای استفاده از database با LWW این است که مطمئن شویم هر key فقط یک بار write می‌شود و پس از آن immutable در نظر گرفته می‌شود؛ در نتیجه updateهای concurrent روی همان key رخ نمی‌دهند. برای مثال، روش پیشنهادی استفاده از Cassandra این است که از `UUID` به‌عنوان key استفاده کنید تا هر write operation یک key یکتا داشته باشد [53].

#### The “happens-before” relationship و concurrency

چگونه تصمیم می‌گیریم دو operation concurrent هستند یا نه؟ برای ایجاد intuition، چند مثال را بررسی کنیم:

- در شکل ۵-۹، دو write concurrent نیستند: insert مربوط به A پیش از increment مربوط به B رخ داده است، چون valueای که B increment کرده همان valueای است که A insert کرده است. به بیان دیگر، operation مربوط به B بر operation مربوط به A بنا شده است؛ بنابراین operation B باید بعد از A رخ داده باشد. همچنین می‌گوییم B از نظر causal به A وابسته است.
- از سوی دیگر، دو write در شکل ۵-۱۲ concurrent هستند: وقتی هر client operation خود را شروع می‌کند، نمی‌داند client دیگری نیز در حال انجام operation روی همان key است. بنابراین هیچ causal dependency میان این operationها وجود ندارد.

Operation A پیش از operation B رخ داده است (`A happens before B`) اگر B از A خبر داشته باشد، به A وابسته باشد یا به هر شکلی بر A بنا شده باشد. اینکه یک operation پیش از operation دیگر رخ داده یا نه، اساس تعریف concurrency است. در واقع می‌توانیم بگوییم دو operation زمانی concurrent هستند که هیچ‌کدام پیش از دیگری رخ نداده باشد؛ یعنی هیچ‌کدام از دیگری خبر نداشته باشد [54].

بنابراین هر زمان دو operation به نام A و B داشته باشیم، سه امکان وجود دارد: یا A پیش از B رخ داده، یا B پیش از A رخ داده، یا A و B concurrent هستند. ما به algorithmی نیاز داریم که مشخص کند دو operation concurrent هستند یا نه. اگر یک operation پیش از operation دیگر رخ داده باشد، operation بعدی باید قبلی را overwrite کند؛ اما اگر operationها concurrent باشند، conflictی داریم که باید resolve شود.

#### Concurrency، time و relativity

ممکن است به نظر برسد دو operation زمانی باید concurrent نامیده شوند که «در یک زمان» رخ دهند، اما در واقع هم‌پوشانی literal آن‌ها در زمان مهم نیست. به‌دلیل مشکل clockها در distributed systemها، تشخیص اینکه دو چیز دقیقاً در یک زمان رخ داده‌اند بسیار دشوار است؛ این موضوع را در Chapter 8 با جزئیات بیشتری بررسی می‌کنیم.

برای تعریف concurrency، زمان دقیق اهمیت ندارد: دو operation را صرفاً زمانی concurrent می‌نامیم که از یکدیگر بی‌خبر باشند، بدون توجه به زمان فیزیکی رخ دادن آن‌ها. افراد گاهی میان این اصل و special theory of relativity در physics ارتباط برقرار می‌کنند [54]؛ این theory ایدهٔ آن را مطرح کرد که information نمی‌تواند سریع‌تر از speed of light حرکت کند. در نتیجه، دو event که در فاصله‌ای از هم رخ می‌دهند، اگر فاصلهٔ زمانی میان آن‌ها کمتر از زمان لازم برای طی کردن این فاصله توسط نور باشد، نمی‌توانند بر یکدیگر اثر بگذارند.

در computer systemها ممکن است دو operation concurrent باشند، حتی اگر speed of light از نظر تئوری اجازه داده باشد یکی بر دیگری اثر بگذارد. برای مثال، اگر network در آن زمان کند یا قطع باشد، دو operation می‌توانند با فاصلهٔ زمانی رخ دهند و همچنان concurrent باشند، چون network problem مانع شده است یکی از operationها از وجود دیگری باخبر شود.

#### Capturing the happens-before relationship

بیایید algorithmی را بررسی کنیم که تعیین می‌کند دو operation concurrent هستند یا یکی پیش از دیگری رخ داده است. برای ساده نگه داشتن موضوع، از databaseای شروع می‌کنیم که فقط یک replica دارد. پس از حل این مسئله روی یک replica، approach را به database leaderless با چند replica generalize می‌کنیم.

شکل ۵-۱۳ دو client را نشان می‌دهد که به‌صورت concurrent itemهایی را به یک shopping cart اضافه می‌کنند. (اگر این مثال بیش از حد ساده به نظر می‌رسد، به‌جای آن دو air traffic controller را تصور کنید که هم‌زمان aircraftهایی را به sector تحت نظارت خود اضافه می‌کنند.) در ابتدا cart خالی است. این دو client در مجموع پنج write در database انجام می‌دهند:

1. Client 1 شیر را به cart اضافه می‌کند. این اولین write برای آن key است، بنابراین server با موفقیت آن را ذخیره می‌کند و version 1 را به آن اختصاص می‌دهد. Server value را همراه با version number به client برمی‌گرداند.
2. Client 2 تخم‌مرغ را به cart اضافه می‌کند، بدون اینکه بداند client 1 هم‌زمان شیر را اضافه کرده است (client 2 تصور می‌کرد تخم‌مرغ تنها item موجود در cart است). Server version 2 را به این write اختصاص می‌دهد و تخم‌مرغ و شیر را به‌عنوان دو value جداگانه ذخیره می‌کند. سپس هر دو value را همراه با version number برابر 2 به client برمی‌گرداند.
3. Client 1، بدون اطلاع از write مربوط به client 2، می‌خواهد آرد را به cart اضافه کند؛ بنابراین تصور می‌کند محتوای فعلی cart باید `[milk, flour]` باشد. این value را همراه با version number برابر 1 که server قبلاً به client 1 داده بود ارسال می‌کند. Server از version number متوجه می‌شود write مربوط به `[milk, flour]` value قبلی یعنی `[milk]` را supersede می‌کند، اما با `[eggs]` concurrent است. بنابراین server version 3 را به `[milk, flour]` اختصاص می‌دهد، value مربوط به version 1 یعنی `[milk]` را overwrite می‌کند، اما value مربوط به version 2 یعنی `[eggs]` را نگه می‌دارد و هر دو value باقی‌مانده را به client برمی‌گرداند.
4. هم‌زمان، client 2 می‌خواهد ham را به cart اضافه کند و نمی‌داند client 1 به‌تازگی آرد را اضافه کرده است. Client 2 در response قبلی server دو value یعنی `[milk]` و `[eggs]` را دریافت کرده بود؛ بنابراین این valueها را merge می‌کند و ham را اضافه می‌کند تا value جدید `[eggs, milk, ham]` ساخته شود. این value را همراه با version number قبلی یعنی 2 به server می‌فرستد. Server تشخیص می‌دهد version 2 مقدار `[eggs]` را overwrite می‌کند اما با `[milk, flour]` concurrent است؛ بنابراین دو value باقی‌مانده عبارت‌اند از `[milk, flour]` با version 3 و `[eggs, milk, ham]` با version 4.
5. در نهایت، client 1 می‌خواهد bacon را اضافه کند. این client پیش‌تر `[milk, flour]` و `[eggs]` را از server در version 3 دریافت کرده بود؛ بنابراین این valueها را merge می‌کند، bacon را اضافه می‌کند و value نهایی `[milk, flour, eggs, bacon]` را همراه با version number برابر 3 برای server می‌فرستد. این write مقدار `[milk, flour]` را overwrite می‌کند (توجه کنید `[eggs]` در مرحلهٔ قبل overwrite شده بود)، اما با `[eggs, milk, ham]` concurrent است؛ بنابراین server این دو value concurrent را نگه می‌دارد.

**شکل ۵-۱۳.** ثبت causal dependency میان دو client که به‌صورت concurrent یک shopping cart را edit می‌کنند.

Dataflow میان operationهای شکل ۵-۱۳ به‌صورت graph در شکل ۵-۱۴ نشان داده شده است. arrowها مشخص می‌کنند کدام operation پیش از operation دیگر رخ داده است؛ یعنی operation بعدی از operation قبلی خبر داشته یا به آن وابسته بوده است. در این مثال، clientها هیچ‌گاه کاملاً با data روی server up to date نیستند، چون همیشه operation دیگری به‌صورت concurrent در حال انجام است. اما versionهای قدیمی value در نهایت overwrite می‌شوند و هیچ writeای از دست نمی‌رود.

**شکل ۵-۱۴.** graph مربوط به causal dependencyها در شکل ۵-۱۳.

توجه کنید server می‌تواند با نگاه کردن به version numberها تشخیص دهد دو operation concurrent هستند یا نه؛ لازم نیست خود value را تفسیر کند (بنابراین value می‌تواند هر data structureای باشد). این algorithm به شکل زیر کار می‌کند:

- Server برای هر key یک version number نگه می‌دارد، هر بار که آن key write می‌شود version number را increment می‌کند و version number جدید را همراه با value نوشته‌شده ذخیره می‌کند.
- وقتی client یک key را read می‌کند، server تمام valueهایی را که overwrite نشده‌اند، به‌همراه جدیدترین version number برمی‌گرداند. Client باید پیش از write کردن، key را read کند.
- وقتی client یک key را write می‌کند، باید version number مربوط به read قبلی را include کند و تمام valueهایی را که در read قبلی دریافت کرده با هم merge کند. Response مربوط به write request می‌تواند مانند read باشد و تمام valueهای فعلی را برگرداند؛ این کار اجازه می‌دهد چند write مانند مثال shopping cart به‌صورت زنجیره‌ای انجام شوند.
- وقتی server writeای را با version number مشخص دریافت می‌کند، می‌تواند تمام valueهایی را که version number آن‌ها برابر یا کمتر از آن number است overwrite کند (چون می‌داند این valueها در value جدید merge شده‌اند)، اما باید تمام valueهایی را که version number بالاتری دارند نگه دارد (چون آن valueها با write ورودی concurrent هستند).

وقتی یک write version number مربوط به read قبلی را شامل می‌شود، این موضوع state قبلی‌ای را مشخص می‌کند که write بر اساس آن ساخته شده است. اگر writeای بدون include کردن version number انجام دهید، آن write با تمام writeهای دیگر concurrent است؛ بنابراین هیچ valueای را overwrite نمی‌کند و فقط در readهای بعدی به‌عنوان یکی از valueها برگردانده می‌شود.

#### Merging concurrently written values

این algorithm تضمین می‌کند dataای به‌صورت silent drop نشود، اما متأسفانه به کار اضافی از سوی clientها نیاز دارد: اگر چند operation به‌صورت concurrent رخ دهند، clientها باید بعداً با merge کردن valueهای concurrent، وضعیت را clean up کنند. `Riak` به این valueهای concurrent، `siblings` می‌گوید.

Merge کردن siblingها اساساً همان مسئلهٔ conflict resolution در multi-leader replication است که پیش‌تر بررسی کردیم (به بخش «Handling Write Conflicts» در صفحهٔ ۱۷۱ مراجعه کنید). یک approach ساده این است که بر اساس version number یا timestamp یکی از valueها را انتخاب کنیم (last write wins)، اما این کار به از دست رفتن data منجر می‌شود. بنابراین ممکن است لازم باشد در application code روش هوشمندانه‌تری به کار ببرید.

در مثال shopping cart، یک approach reasonable برای merge کردن siblingها این است که union آن‌ها را بگیریم. در شکل ۵-۱۴، دو sibling نهایی `[milk, flour, eggs, bacon]` و `[eggs, milk, ham]` هستند؛ توجه کنید milk و eggs در هر دو وجود دارند، هرچند هرکدام فقط یک بار write شده‌اند. value mergeشده می‌تواند چیزی مانند `[milk, flour, eggs, bacon, ham]` باشد، بدون duplicate.

بااین‌حال، اگر بخواهید userها بتوانند itemها را از cart نیز remove کنند و فقط item جدید اضافه نکنند، گرفتن union از siblingها ممکن است result درستی ندهد: اگر دو sibling cart را merge کنید و itemی فقط در یکی از آن‌ها remove شده باشد، آن item در union siblingها دوباره ظاهر می‌شود [37]. برای جلوگیری از این مشکل، هنگام remove کردن item نباید آن را صرفاً از database delete کنید؛ در عوض system باید markerای با version number مناسب نگه دارد تا هنگام merge کردن siblingها نشان دهد item حذف شده است. این deletion marker `tombstone` نام دارد. (پیش‌تر tombstoneها را در context مربوط به log compaction در بخش «Hash Indexes» در صفحهٔ ۷۲ دیدیم.)

از آنجا که merge کردن siblingها در application code پیچیده و مستعد error است، تلاش‌هایی برای طراحی data structureهایی وجود دارد که این merge را به‌صورت automatic انجام دهند؛ همان‌طور که در بخش «Automatic Conflict Resolution» در صفحهٔ ۱۷۴ بحث کردیم. برای مثال، datatype support در Riak از خانواده‌ای از data structureها به نام `CRDT` استفاده می‌کند [38, 39, 55] که می‌توانند siblingها را به‌شکل sensible، از جمله با حفظ deletionها، به‌صورت automatic merge کنند.

#### Version vectors

مثال شکل ۵-۱۳ فقط از یک replica استفاده می‌کرد. وقتی چند replica داشته باشیم اما leader نداشته باشیم، algorithm چگونه تغییر می‌کند؟

شکل ۵-۱۳ از یک version number واحد برای ثبت dependency میان operationها استفاده می‌کند، اما وقتی چند replica به‌صورت concurrent write را می‌پذیرند این کافی نیست. در عوض، به version numberای برای هر replica و همچنین هر key نیاز داریم.

هر replica هنگام پردازش یک write version number خودش را increment می‌کند و همچنین version numberهایی را که از هر replica دیگر دیده است track می‌کند. این information مشخص می‌کند کدام valueها باید overwrite و کدام valueها باید به‌عنوان sibling نگه داشته شوند. مجموعهٔ version numberهای تمام replicaها `version vector` نام دارد [56].

چند variant از این ایده استفاده می‌شود، اما احتمالاً جالب‌ترین آن‌ها `dotted version vector` است [57] که در `Riak 2.0` استفاده می‌شود [58, 59]. وارد جزئیات نمی‌شویم، اما نحوهٔ کار آن بسیار شبیه چیزی است که در مثال cart دیدیم.

مانند version numberهای شکل ۵-۱۳، version vectorها هنگام read شدن value از database، از replicaهای database به client ارسال می‌شوند و هنگام write شدن دوبارهٔ value باید به database برگردانده شوند. (`Riak` version vector را به‌صورت stringای encode می‌کند که آن را `causal context` می‌نامد.) Version vector به database اجازه می‌دهد میان overwrite و concurrent write تفاوت بگذارد.

همانند مثال single-replica، application ممکن است لازم باشد siblingها را merge کند. ساختار version vector تضمین می‌کند read کردن از یک replica و سپس write کردن روی replica دیگری safe باشد. انجام این کار ممکن است باعث ایجاد sibling شود، اما تا زمانی که siblingها به‌درستی merge شوند، dataای از دست نمی‌رود.

#### Version vectors و vector clocks

گاهی version vector را `vector clock` نیز می‌نامند، هرچند این دو کاملاً یکی نیستند. تفاوت ظریف است؛ برای جزئیات به referenceها مراجعه کنید [57, 60, 61]. به‌طور خلاصه، هنگام مقایسهٔ state مربوط به replicaها، version vector data structure درست برای استفاده است.

## Key Terms

- `Leaderless Replication` — replicationای که در آن هیچ leader واحدی وجود ندارد و هر replica می‌تواند مستقیماً write client را بپذیرد.
- `Quorum` — حداقل تعداد responseهای موفق لازم برای معتبر شدن read یا write در میان replicaها.
- `Read Quorum` — readای که برای معتبر شدن به response حداقل `r` replica نیاز دارد.
- `Write Quorum` — writeای که برای موفق شدن باید توسط حداقل `w` replica acknowledge شود.
- `Read Repair` — اصلاح replica stale هنگام read کردن یک value از چند replica و مشاهدهٔ version جدیدتر در یکی از آن‌ها.
- `Anti-Entropy` — background processای که تفاوت replicaها را پیدا و data missing را میان آن‌ها copy می‌کند.
- `Hinted Handoff` — انتقال write موقتاً ذخیره‌شده روی node جایگزین به node home پس از بازگشت network یا node اصلی.
- `Sloppy Quorum` — quorumای که responseها را از nodeهای reachable خارج از nodeهای home نیز می‌پذیرد.
- `Strict Quorum` — quorumی که فقط nodeهای home تعیین‌شده برای value را در نظر می‌گیرد.
- `Coordinator Node` — nodeای که request client را برای چند replica ارسال و responseها را جمع‌آوری می‌کند، بدون اینکه order writeها را تعیین کند.
- `Dynamo-Style Database` — database leaderless الهام‌گرفته از معماری Dynamo، مانند Riak، Cassandra و Voldemort.
- `Data Loss` — از بین رفتن write یا valueای که system آن را پذیرفته یا قبلاً ذخیره کرده است.
- `Vector Clock` — metadataای برای مقایسهٔ causal order و state replicaها؛ با version vector مرتبط اما دقیقاً یکسان نیست.
- `Happens-Before Relationship` — رابطه‌ای که نشان می‌دهد یک operation از operation دیگر خبر داشته یا بر آن بنا شده است.
- `Tombstone` — markerای که حذف شدن value را هنگام merge کردن versionها ثبت می‌کند.
- `Sibling` — یکی از چند value concurrent که تا زمان conflict resolution هم‌زمان نگهداری می‌شوند.
- `Sibling Values` — versionهای concurrent یک key که هنوز merge یا resolve نشده‌اند.
- `Causal Context` — representation مربوط به dependencyهای causal که Riak برای version vector استفاده می‌کند.
- `Replica Staleness` — میزان قدیمی بودن value موجود در یک replica نسبت به جدیدترین state.
- `Availability` — توانایی system برای پاسخ‌گویی حتی هنگام failure یا unavailable بودن بخشی از nodeها.

# Chapter 6 — Partitioning

## Partitioning and Secondary Indexes

partitioning schemeهایی که تا اینجا بررسی کردیم، بر یک key-value data model تکیه دارند. اگر recordها همیشه فقط از طریق primary key دسترسی‌پذیر باشند، می‌توانیم partition را از روی آن key تعیین کنیم و read و write requestها را به partition مسئول همان key route کنیم.

وقتی secondary indexها وارد ماجرا می‌شوند، situation پیچیده‌تر می‌شود (همچنین به بخش «Other Indexing Structures» در صفحهٔ ۸۵ مراجعه کنید). یک secondary index معمولاً یک record را به‌طور unique شناسایی نمی‌کند؛ بلکه روشی برای جست‌وجوی occurrenceهای یک value خاص است: پیدا کردن تمام actionهای user شمارهٔ ۱۲۳، پیدا کردن تمام articleهایی که word `hogwash` را دارند، پیدا کردن تمام carهایی که color آن‌ها `red` است، و موارد مشابه.

Secondary indexها نان و کرهٔ relational databaseها هستند و در document databaseها نیز بسیار رایج‌اند. بسیاری از key-value storeها، مانند HBase و Voldemort، به‌دلیل پیچیدگی implementation اضافی از secondary indexها اجتناب کرده‌اند؛ اما بعضی دیگر، مانند Riak، به‌دلیل usefulness آن‌ها برای data modeling شروع به اضافه کردن secondary index کرده‌اند. در نهایت، secondary indexها دلیل اصلی وجود search serverهایی مانند Solr و Elasticsearch هستند.

مشکل secondary indexها این است که به‌سادگی با partitionها map نمی‌شوند. برای partition کردن databaseای که secondary index دارد، دو approach اصلی وجود دارد: **document-based partitioning** و **term-based partitioning**.

### Partitioning Secondary Indexes by Document

برای مثال، تصور کنید یک website برای فروش carهای دست‌دوم اداره می‌کنید (شکل ۶-۴). هر listing یک ID منحصربه‌فرد دارد—آن را document ID می‌نامیم—و database را بر اساس document ID partition می‌کنید؛ برای مثال، IDهای ۰ تا ۴۹۹ در partition ۰، IDهای ۵۰۰ تا ۹۹۹ در partition ۱، و به همین ترتیب.

می‌خواهید به userها اجازه دهید carها را search کنند و بر اساس color و make فیلتر کنند؛ بنابراین به secondary indexهایی روی color و make نیاز دارید. (در document database، این موارد field هستند و در relational database، column.) اگر index را declare کرده باشید، database می‌تواند indexing را به‌صورت automatic انجام دهد.ii برای مثال، هر زمان یک car قرمز به database اضافه شود، partition مربوط به database به‌صورت automatic document ID آن car را به فهرست document IDهای مربوط به index entry `color:red` اضافه می‌کند.

*پاورقی:* اگر database شما فقط از key-value model پشتیبانی می‌کند، ممکن است وسوسه شوید secondary index را در application code و با ایجاد mapping از valueها به document IDها پیاده‌سازی کنید. اگر این مسیر را انتخاب می‌کنید، باید بسیار دقت کنید که indexها با data زیرین consistent بمانند. race conditionها و write failureهای intermittent—که در آن بعضی changeها ذخیره شده‌اند و بعضی دیگر ذخیره نشده‌اند—به‌سادگی می‌توانند باعث out of sync شدن data شوند؛ به بخش «The need for multi-object transactions» در صفحهٔ ۲۳۱ مراجعه کنید.

**شکل ۶-۴.** partition کردن secondary indexها بر اساس document.

در این approach مربوط به indexing، هر partition کاملاً مستقل است: هر partition secondary indexهای خودش را نگه می‌دارد و این indexها فقط documentهای همان partition را پوشش می‌دهند. این partition کاری به data ذخیره‌شده در partitionهای دیگر ندارد. هر زمان لازم باشد در database write انجام دهید—برای add، remove یا update کردن یک document—فقط باید با partitionای کار کنید که document ID موردنظر را در خود دارد. به همین دلیل، document-partitioned index را **local index** نیز می‌نامند؛ در مقابل، **global index** در بخش بعدی توضیح داده می‌شود.

بااین‌حال، read کردن از یک document-partitioned index به دقت نیاز دارد: مگر اینکه کار خاصی با document IDها انجام داده باشید، دلیلی وجود ندارد که تمام carهایی با color یا make مشخص در یک partition قرار گرفته باشند. در شکل ۶-۴، carهای قرمز هم در partition ۰ و هم در partition ۱ وجود دارند. بنابراین اگر بخواهید carهای قرمز را search کنید، باید query را برای تمام partitionها بفرستید و تمام resultهای برگشتی را با هم combine کنید.

این approach برای query کردن یک partitioned database گاهی **scatter/gather** نامیده می‌شود و می‌تواند read queryهای مربوط به secondary index را بسیار پرهزینه کند. حتی اگر partitionها را به‌صورت parallel query کنید، scatter/gather مستعد **tail latency amplification** است (به بخش «Percentiles in Practice» در صفحهٔ ۱۶ مراجعه کنید). بااین‌حال، این approach به‌طور گسترده استفاده می‌شود: MongoDB، Riak [15]، Cassandra [16]، Elasticsearch [17]، SolrCloud [18] و VoltDB [19] همگی از document-partitioned secondary indexها استفاده می‌کنند. بیشتر database vendorها توصیه می‌کنند partitioning scheme خود را طوری طراحی کنید که queryهای secondary index از یک partition واحد پاسخ داده شوند؛ اما این کار همیشه ممکن نیست، به‌خصوص وقتی در یک query از چند secondary index استفاده می‌کنید، مانند filter کردن carها هم‌زمان بر اساس color و make.

### Partitioning Secondary Indexes by Term

به‌جای اینکه هر partition secondary index خودش را داشته باشد (local index)، می‌توانیم یک global index بسازیم که data تمام partitionها را پوشش دهد. بااین‌حال، نمی‌توانیم این index را فقط روی یک node ذخیره کنیم، چون احتمالاً به bottleneck تبدیل می‌شود و هدف partitioning را از بین می‌برد. یک global index نیز باید partition شود، اما می‌توان آن را با روشی متفاوت از primary key index partition کرد.

شکل ۶-۵ نشان می‌دهد این کار چگونه می‌تواند انجام شود: carهای قرمز از تمام partitionها زیر `color:red` در index قرار می‌گیرند، اما خود index طوری partition می‌شود که colorهایی که با حروف a تا r شروع می‌شوند در partition ۰ و colorهایی که با حروف s تا z شروع می‌شوند در partition ۱ قرار بگیرند. index مربوط به make car نیز به‌صورت مشابه partition می‌شود؛ در اینجا partition boundary میان f و h قرار دارد.

این نوع index را **term-partitioned** می‌نامیم، چون term مورد جست‌وجو partition مربوط به index را تعیین می‌کند. برای مثال، در اینجا یک term می‌تواند `color:red` باشد. نام term از full-text indexها می‌آید؛ full-text index نوع خاصی از secondary index است که termهای آن تمام wordهایی هستند که در یک document ظاهر می‌شوند.

مانند قبل، می‌توانیم index را بر اساس خود term یا بر اساس hash مربوط به term partition کنیم. partition کردن بر اساس خود term می‌تواند برای range scanها مفید باشد؛ برای مثال، وقتی با یک property عددی مانند asking price مربوط به car کار می‌کنیم. در مقابل، partition کردن بر اساس hash term باعث توزیع یکنواخت‌تر load می‌شود.

مزیت global index—یعنی term-partitioned index—در مقایسه با document-partitioned index این است که readها را efficientتر می‌کند: به‌جای scatter/gather روی تمام partitionها، client فقط لازم است request خود را به partitionای بفرستد که term موردنظر در آن قرار دارد. اما عیب global index این است که writeها کندتر و پیچیده‌تر می‌شوند، چون یک write روی یک document ممکن است اکنون چند partition از index را تحت تأثیر قرار دهد؛ هر term موجود در document ممکن است در partition متفاوتی و روی node متفاوتی قرار داشته باشد.

در یک دنیای ایده‌آل، index همیشه up to date است و هر documentای که در database نوشته می‌شود، بلافاصله در index نیز منعکس می‌گردد. اما در term-partitioned index، این کار به یک distributed transaction میان تمام partitionهایی نیاز دارد که write روی آن‌ها اثر گذاشته است؛ چنین transactionای در همهٔ databaseها پشتیبانی نمی‌شود (به Chapter 7 و Chapter 9 مراجعه کنید).

در عمل، update کردن global secondary indexها اغلب asynchronous است؛ یعنی اگر کمی بعد از یک write، index را read کنید، ممکن است changeای که همین الآن ایجاد کرده‌اید هنوز در index منعکس نشده باشد. برای مثال، Amazon DynamoDB اعلام می‌کند که global secondary indexهای آن در شرایط عادی در کسری از ثانیه update می‌شوند، اما در صورت بروز fault در infrastructure ممکن است propagation delay طولانی‌تری داشته باشند [20].

نمونه‌های دیگر استفاده از global term-partitioned index عبارت‌اند از search feature مربوط به Riak [21] و data warehouse مربوط به Oracle که به شما اجازه می‌دهد میان local indexing و global indexing انتخاب کنید [22]. در Chapter 12 دوباره به موضوع implementation مربوط به term-partitioned secondary indexها برمی‌گردیم.

## Key Terms

- `Secondary Index` — index اضافی، جدا از primary key index، برای search بر اساس fieldها یا columnهای دیگر.
- `Local Index` — secondary indexای که هر partition به‌صورت مستقل برای documentهای خودش نگه می‌دارد.
- `Global Index` — indexای که data تمام partitionها را پوشش می‌دهد و خودش نیز معمولاً partition شده است.
- `Document Partitioning` — partition کردن secondary indexها بر اساس documentای که index entry به آن تعلق دارد.
- `Term Partitioning` — partition کردن global index بر اساس term یا value مورد جست‌وجو.
- `Index Entry` — entryای در index که یک value یا term را به document IDها یا recordهای matching مرتبط می‌کند.
- `Query Routing` — تعیین partition یا node مناسب برای ارسال یک read یا write request.
- `Scatter/Gather` — ارسال یک query به تمام partitionها و combine کردن resultهای برگشتی.
- `Cross-Partition Query` — queryای که برای پاسخ دادن به data چند partition نیاز دارد.
- `Tail Latency Amplification` — افزایش اثر latencyهای انتهایی وقتی یک query به چند partition یا service وابسته است.
- `Inverted Index` — indexای که term یا word را به documentهایی که آن را شامل می‌شوند map می‌کند.
- `Distributed Transaction` — transactionای که operationهای آن روی چند partition یا node اجرا می‌شود و به هماهنگی میان آن‌ها نیاز دارد.
- `Asynchronous Index Update` — update شدن index با فاصله‌ای پس از write اصلی، به‌طوری‌که index ممکن است موقتاً stale باشد.

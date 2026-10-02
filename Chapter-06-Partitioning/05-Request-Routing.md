# Chapter 6 — Partitioning

## Request Routing

اکنون dataset خود را میان چند node که روی چند machine اجرا می‌شوند partition کرده‌ایم. اما هنوز یک سؤال باز وجود دارد: وقتی client می‌خواهد requestی ارسال کند، از کجا می‌داند باید به کدام node متصل شود؟ با rebalanced شدن partitionها، assignment مربوط به partitionها به nodeها تغییر می‌کند. باید componentی وجود داشته باشد که از این تغییرات مطلع بماند تا بتواند به این سؤال پاسخ دهد: اگر بخواهم key مربوط به `foo` را read یا write کنم، باید به کدام IP address و port number متصل شوم؟

این یک نمونه از مسئلهٔ عمومی‌تری به نام **service discovery** است که فقط به databaseها محدود نمی‌شود. هر softwareای که از طریق network قابل دسترسی باشد با این مسئله روبه‌روست، به‌خصوص اگر هدف آن High Availability باشد و در configurationای redundant روی چند machine اجرا شود. شرکت‌های زیادی ابزارهای service discovery داخلی خود را نوشته‌اند و بسیاری از این ابزارها بعداً به‌صورت open source منتشر شده‌اند [30].

در سطح کلی، چند approach متفاوت برای این مسئله وجود دارد (شکل ۶-۷):

1. به clientها اجازه دهید با هر nodeای contact بگیرند؛ برای مثال، از طریق یک `round-robin load balancer`. اگر آن node به‌طور اتفاقی مالک partition مربوط به request باشد، می‌تواند request را مستقیماً handle کند؛ در غیر این صورت، request را به node مناسب forward می‌کند، reply را دریافت می‌کند و reply را برای client می‌فرستد.
2. ابتدا تمام requestهای clientها را به یک **routing tier** بفرستید. routing tier nodeای را که باید هر request را handle کند تعیین کرده و request را به آن node forward می‌کند. routing tier خودش هیچ requestی را handle نمی‌کند؛ فقط مانند یک **partition-aware load balancer** عمل می‌کند.
3. از clientها بخواهید از partitioning و assignment مربوط به partitionها به nodeها آگاه باشند. در این حالت، client می‌تواند بدون intermediary مستقیماً به node مناسب متصل شود.

در تمام این حالت‌ها، مسئلهٔ اصلی این است: componentی که تصمیم routing را می‌گیرد—خواه یکی از nodeها باشد، خواه routing tier یا client—چگونه از تغییرات assignment مربوط به partitionها به nodeها مطلع می‌شود؟

این مسئله دشوار است، چون مهم است تمام participantها با یکدیگر توافق داشته باشند؛ در غیر این صورت requestها به nodeهای اشتباه ارسال می‌شوند و به‌درستی handle نخواهند شد. برای دستیابی به consensus در یک distributed system protocolهایی وجود دارد، اما پیاده‌سازی صحیح آن‌ها دشوار است (به Chapter 9 مراجعه کنید).

بسیاری از distributed data systemها برای track کردن این cluster metadata به coordination service جداگانه‌ای مانند `ZooKeeper` وابسته‌اند؛ شکل ۶-۸ این arrangement را نشان می‌دهد. هر node خود را در `ZooKeeper` register می‌کند و `ZooKeeper` authoritative mapping مربوط به partitionها به nodeها را نگه می‌دارد. actorهای دیگر، مانند routing tier یا clientای که از partitioning آگاه است، می‌توانند در `ZooKeeper` مشترک این information شوند. هر زمان ownership یک partition تغییر کند یا nodeای اضافه یا حذف شود، `ZooKeeper` routing tier را notify می‌کند تا routing information خود را up to date نگه دارد.

**شکل ۶-۷.** سه روش متفاوت برای route کردن request به node مناسب.

**شکل ۶-۸.** استفاده از `ZooKeeper` برای track کردن assignment مربوط به partitionها به nodeها.

برای مثال، `Espresso` متعلق به LinkedIn برای cluster management از `Helix` [31] استفاده می‌کند—که خود به `ZooKeeper` متکی است—و routing tierای مانند آنچه در شکل ۶-۸ نشان داده شده implement می‌کند. `HBase`، `SolrCloud` و `Kafka` نیز برای track کردن partition assignment از `ZooKeeper` استفاده می‌کنند. `MongoDB` معماری مشابهی دارد، اما به implementation مربوط به config server خود و daemonهای `mongos` به‌عنوان routing tier متکی است.

`Cassandra` و `Riak` approach متفاوتی دارند: آن‌ها از یک **gossip protocol** میان nodeها استفاده می‌کنند تا هر تغییر در cluster state را منتشر کنند. Requestها می‌توانند به هر nodeای ارسال شوند و آن node request را برای partition موردنظر به node مناسب forward می‌کند (approach ۱ در شکل ۶-۷). این model complexity بیشتری را به database nodeها منتقل می‌کند، اما وابستگی به coordination service خارجی مانند `ZooKeeper` را از بین می‌برد.

`Couchbase` به‌صورت automatic rebalancing انجام نمی‌دهد و همین موضوع design آن را ساده‌تر می‌کند. این system معمولاً با routing tierای به نام `moxi` configure می‌شود؛ `moxi` تغییرات routing را از cluster nodeها دریافت می‌کند [32].

وقتی از routing tier استفاده می‌کنید یا requestها را به یک node random می‌فرستید، clientها همچنان باید IP addressهایی را که می‌خواهند به آن‌ها متصل شوند پیدا کنند. این addressها به‌اندازهٔ assignment مربوط به partitionها به nodeها سریع تغییر نمی‌کنند؛ بنابراین معمولاً استفاده از `DNS` برای این کار کافی است.

### Parallel Query Execution

تا اینجا روی queryهای بسیار ساده‌ای تمرکز کردیم که یک key واحد را read یا write می‌کنند؛ البته queryهای scatter/gather مربوط به document-partitioned secondary indexها نیز وجود دارند. بیشتر distributed NoSQL datastoreها تقریباً همین سطح از access را پشتیبانی می‌کنند.

بااین‌حال، relational database productهای **massively parallel processing (MPP)** که اغلب برای analytics استفاده می‌شوند، از نظر نوع queryهایی که پشتیبانی می‌کنند بسیار sophisticatedترند. یک data warehouse query معمولاً چندین operation از نوع join، filtering، grouping و aggregation دارد. `MPP query optimizer` این query پیچیده را به تعدادی execution stage و partition تقسیم می‌کند که بسیاری از آن‌ها می‌توانند به‌صورت parallel روی nodeهای مختلف database cluster اجرا شوند. queryهایی که بخش‌های بزرگی از dataset را scan می‌کنند، به‌طور خاص از چنین اجرای parallelای سود می‌برند.

اجرای سریع و parallel مربوط به data warehouse queryها موضوعی تخصصی است و با توجه به اهمیت تجاری analytics، توجه زیادی از صنعت دریافت می‌کند. در Chapter 10 برخی تکنیک‌های parallel query execution را بررسی خواهیم کرد. برای overview دقیق‌تر از تکنیک‌های استفاده‌شده در parallel databaseها، به referenceهای [1, 33] مراجعه کنید.

## Key Terms

- `Request Routing` — تعیین node یا partition مناسب برای دریافت و پردازش یک request.
- `Routing Tier` — لایه‌ای که requestها را دریافت می‌کند، node مسئول را تعیین می‌کند و request را به آن forward می‌کند.
- `Partition-Aware Load Balancer` — load balancerای که از assignment مربوط به partitionها آگاه است و request را به node مناسب می‌فرستد.
- `Service Discovery` — mechanism پیدا کردن location و endpoint مربوط به service یا node در یک network.
- `Cluster Metadata` — information مربوط به state cluster، ownership partitionها و mapping میان partitionها و nodeها.
- `Coordination Service` — service مستقلی مانند `ZooKeeper` برای نگه‌داری metadata authoritative و اطلاع‌رسانی تغییرات cluster.
- `Authoritative Mapping` — mapping مرجع و قابل‌اعتماد میان partitionها و nodeهای owner آن‌ها.
- `Gossip Protocol` — protocol توزیع‌شده‌ای که nodeها با exchange کردن state، تغییرات cluster را میان یکدیگر منتشر می‌کنند.
- `Parallel Query Execution` — اجرای هم‌زمان بخش‌های یک query روی چند node یا partition.
- `Massively Parallel Processing (MPP)` — معماری اجرای query که workload را میان تعداد زیادی node توزیع و parallelize می‌کند.
- `Execution Stage` — یکی از مرحله‌های برنامهٔ اجرای query که می‌تواند مستقل یا parallel با stageهای دیگر اجرا شود.
- `Distributed Query` — queryای که برای پردازش به چند node یا partition وابسته است.
- `Fan-Out` — فرستادن یک request یا query به چند partition یا node به‌صورت parallel.

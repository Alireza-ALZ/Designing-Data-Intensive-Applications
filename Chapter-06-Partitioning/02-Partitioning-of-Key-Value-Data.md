# Chapter 6 — Partitioning

## Partitioning of Key-Value Data

فرض کنید مقدار زیادی data دارید و می‌خواهید آن را partition کنید. چگونه تصمیم می‌گیرید هر record روی کدام node ذخیره شود؟

هدف ما از partitioning این است که data و query load را به‌صورت یکنواخت میان nodeها توزیع کنیم. اگر هر node سهم منصفانه‌ای داشته باشد، در تئوری ۱۰ node باید بتوانند—فعلاً با نادیده گرفتن replication—ده برابر یک node منفرد data و ده برابر throughput مربوط به read و write را مدیریت کنند.

اگر partitioning منصفانه نباشد و بعضی partitionها data یا query بیشتری از بقیه داشته باشند، می‌گوییم partitioning دچار **skew** شده است. وجود skew باعث می‌شود partitioning کارایی بسیار کمتری داشته باشد. در حالت extreme ممکن است تمام load روی یک partition قرار بگیرد؛ در این صورت ۹ node از ۱۰ node idle هستند و bottleneck شما همان یک node شلوغ خواهد بود. به partitionای که load نامتناسباً زیادی دارد **hot spot** گفته می‌شود.

ساده‌ترین راه برای جلوگیری از hot spot این است که recordها را به‌صورت random به nodeها اختصاص دهیم. این کار data را تقریباً به‌طور یکنواخت میان nodeها توزیع می‌کند، اما یک عیب بزرگ دارد: وقتی می‌خواهید item خاصی را read کنید، راهی ندارید که بدانید آن item روی کدام node قرار دارد؛ بنابراین باید همهٔ nodeها را به‌صورت parallel query کنید.

می‌توانیم بهتر از این عمل کنیم. فعلاً فرض کنید یک key-value data model ساده دارید که در آن همیشه با استفاده از primary key به یک record دسترسی پیدا می‌کنید. برای مثال، در یک encyclopedia کاغذی قدیمی، entry را با title آن پیدا می‌کنید؛ چون تمام entryها بر اساس title و به‌صورت alphabetically sorted قرار گرفته‌اند، می‌توانید entry موردنظر را سریع پیدا کنید.

### Partitioning by Key Range

یکی از روش‌های partitioning این است که یک range پیوسته از keyها—از minimum تا maximum مشخص—را به هر partition اختصاص دهیم؛ درست مانند volumeهای یک encyclopedia کاغذی (شکل ۶-۲). اگر boundaryهای میان rangeها را بدانید، به‌راحتی می‌توانید مشخص کنید یک key مشخص در کدام partition قرار دارد. اگر بدانید هر partition به کدام node اختصاص داده شده است، می‌توانید request خود را مستقیماً به node مناسب بفرستید؛ یا در مثال encyclopedia، کتاب درست را از قفسه بردارید.

**شکل ۶-۲.** یک encyclopedia چاپی که بر اساس key range partition شده است.

rangeهای key الزاماً فاصله‌های یکسانی ندارند، چون ممکن است data شما به‌صورت یکنواخت توزیع نشده باشد. برای مثال، در شکل ۶-۲، volume 1 شامل wordهایی است که با A و B شروع می‌شوند، اما volume 12 شامل wordهایی است که با T، U، V، X، Y و Z شروع می‌شوند. اگر صرفاً برای هر دو حرف از alphabet یک volume داشته باشیم، بعضی volumeها بسیار بزرگ‌تر از بقیه خواهند شد. برای توزیع یکنواخت data، partition boundaryها باید خود را با data تطبیق دهند.

partition boundaryها ممکن است به‌صورت دستی توسط administrator انتخاب شوند، یا database بتواند آن‌ها را به‌صورت automatic انتخاب کند (در بخش «Rebalancing Partitions» در صفحهٔ ۲۰۹، انتخاب partition boundaryها را با جزئیات بیشتری بررسی می‌کنیم). این partitioning strategy در Bigtable، معادل open source آن یعنی HBase [2, 3]، و همچنین RethinkDB و MongoDB پیش از version 2.4 استفاده شده است [4].

درون هر partition می‌توانیم keyها را به‌صورت sorted نگه داریم (به بخش «SSTables and LSM-Trees» در صفحهٔ ۷۶ مراجعه کنید). این کار دو مزیت دارد: range scanها آسان می‌شوند و می‌توانیم key را مانند یک concatenated index به‌کار ببریم تا چند record مرتبط را در یک query دریافت کنیم (به بخش «Multi-column indexes» در صفحهٔ ۸۷ مراجعه کنید). برای مثال، applicationای را در نظر بگیرید که data را از شبکه‌ای از sensorها ذخیره می‌کند و key آن timestamp اندازه‌گیری است (year-month-day-hour-minute-second). در این حالت range scanها بسیار مفیدند، چون اجازه می‌دهند مثلاً تمام readingهای یک ماه مشخص را به‌سادگی دریافت کنید.

بااین‌حال، عیب key-range partitioning این است که بعضی access patternها می‌توانند به hot spot منجر شوند. اگر key یک timestamp باشد، partitionها با rangeهای زمانی متناظر خواهند بود؛ برای مثال، یک partition برای هر روز. متأسفانه چون data مربوط به sensorها را هم‌زمان با رخ دادن measurementها در database می‌نویسیم، تمام writeها به یک partition واحد—partition مربوط به امروز—فرستاده می‌شوند. در نتیجه ممکن است این partition از writeها overloaded شود، درحالی‌که partitionهای دیگر idle هستند [5].

برای جلوگیری از این مشکل در database مربوط به sensorها، باید چیزی غیر از timestamp را به‌عنوان اولین عنصر key استفاده کنید. برای مثال، می‌توانید نام sensor را به ابتدای هر timestamp اضافه کنید تا partitioning ابتدا بر اساس sensor name و سپس بر اساس time انجام شود. اگر sensorهای زیادی هم‌زمان active باشند، write load به‌صورت یکنواخت‌تری میان partitionها توزیع خواهد شد. اما اکنون، وقتی می‌خواهید valueهای چند sensor را در یک range زمانی دریافت کنید، باید برای هر sensor name یک range query جداگانه اجرا کنید.

### Partitioning by Hash of Key

به‌دلیل همین خطر skew و hot spot، بسیاری از distributed datastoreها برای تعیین partition مربوط به یک key از **hash function** استفاده می‌کنند.

یک hash function خوب، data دارای skew را می‌گیرد و آن را به‌صورت یکنواخت توزیع می‌کند. فرض کنید یک hash function سی‌ودوبیتی دارید که یک string را دریافت می‌کند. هر بار که string جدیدی به آن بدهید، عددی ظاهراً random بین `0` و `2^32 - 1` برمی‌گرداند. حتی اگر stringهای ورودی بسیار شبیه هم باشند، hash آن‌ها در سراسر این range عددی به‌صورت یکنواخت توزیع می‌شود.

برای partitioning لازم نیست hash function از نظر cryptographic قوی باشد: برای مثال، Cassandra و MongoDB از MD5 و Voldemort از تابع Fowler–Noll–Vo استفاده می‌کنند. بسیاری از programming languageها hash functionهای ساده‌ای دارند که به‌صورت built-in ارائه شده‌اند (چون در hash tableها استفاده می‌شوند)، اما این functionها ممکن است برای partitioning مناسب نباشند. برای مثال، در `Java`، مقدار `Object.hashCode()` و در `Ruby`، مقدار `Object#hash` برای یک key یکسان می‌تواند در processهای مختلف متفاوت باشد [6].

پس از انتخاب یک hash function مناسب برای keyها، می‌توانید به‌جای rangeای از keyها، یک range از hashها را به هر partition اختصاص دهید. هر keyای که hash آن در range یک partition قرار بگیرد، در همان partition ذخیره خواهد شد. شکل ۶-۳ این روش را نشان می‌دهد.

**شکل ۶-۳.** partitioning بر اساس hash مربوط به key.

این روش در توزیع منصفانهٔ keyها میان partitionها عملکرد خوبی دارد. partition boundaryها می‌توانند با فاصله‌های یکسان انتخاب شوند یا به‌صورت pseudo-random انتخاب شوند؛ در حالت دوم، این روش گاهی **consistent hashing** نامیده می‌شود.

#### Consistent Hashing

Consistent hashing، آن‌طور که Karger و همکاران تعریف کرده‌اند [7]، روشی برای توزیع یکنواخت load در یک system در مقیاس اینترنت، مانند یک content delivery network (CDN)، است.

این روش از partition boundaryهای random استفاده می‌کند تا نیاز به control مرکزی یا distributed consensus را از بین ببرد. توجه کنید که واژهٔ *consistent* در اینجا هیچ ارتباطی با replica consistency (به Chapter 5 مراجعه کنید) یا ACID consistency (به Chapter 7 مراجعه کنید) ندارد؛ این واژه در اینجا یک approach مشخص برای rebalancing را توصیف می‌کند.

همان‌طور که در بخش «Rebalancing Partitions» در صفحهٔ ۲۰۹ خواهیم دید، این approach مشخص در واقع برای databaseها عملکرد چندان خوبی ندارد [8] و به همین دلیل در عمل به‌ندرت استفاده می‌شود. (مستندات بعضی databaseها هنوز به consistent hashing اشاره می‌کنند، اما این اشاره‌ها اغلب دقیق نیستند.) از آنجا که این terminology گیج‌کننده است، بهتر است از عبارت consistent hashing اجتناب کنیم و صرفاً آن را **hash partitioning** بنامیم.

متأسفانه با استفاده از hash مربوط به key برای partitioning، یکی از ویژگی‌های خوب key-range partitioning را از دست می‌دهیم: امکان اجرای efficient range query. keyهایی که پیش‌تر کنار یکدیگر بودند، اکنون در تمام partitionها پراکنده می‌شوند و sort order آن‌ها از بین می‌رود. در MongoDB، اگر hash-based sharding mode را فعال کرده باشید، هر range query باید برای تمام partitionها ارسال شود [4]. Riak [9]، Couchbase [10] و Voldemort از range query روی primary key پشتیبانی نمی‌کنند.

Cassandra میان این دو partitioning strategy یک compromise ایجاد می‌کند [11, 12, 13]. یک table در Cassandra می‌تواند primary keyای compound داشته باشد که از چند column تشکیل شده است. برای تعیین partition، فقط بخش اول key hash می‌شود؛ اما columnهای دیگر به‌عنوان concatenated index برای sort کردن data در SSTableهای Cassandra استفاده می‌شوند. بنابراین query نمی‌تواند درون بخش اول یک compound key به دنبال rangeای از valueها بگردد، اما اگر برای بخش اول یک value ثابت مشخص کند، می‌تواند روی columnهای دیگر key یک range scan کارآمد انجام دهد.

رویکرد concatenated index یک data model ظریف برای one-to-many relationshipها فراهم می‌کند. برای مثال، در یک social media site، یک user ممکن است updateهای زیادی منتشر کند. اگر primary key مربوط به updateها را به‌صورت `(user_id, update_timestamp)` انتخاب کنیم، می‌توانیم تمام updateهای یک user مشخص را در یک time interval، به‌صورت مرتب‌شده بر اساس timestamp، به‌طور کارآمد دریافت کنیم. userهای مختلف ممکن است در partitionهای متفاوت ذخیره شوند، اما updateهای هر user درون یک partition و به‌ترتیب timestamp ذخیره می‌شوند.

### Skewed Workloads and Relieving Hot Spots

همان‌طور که گفتیم، hash کردن یک key برای تعیین partition آن می‌تواند به کاهش hot spotها کمک کند. اما این روش نمی‌تواند به‌طور کامل از ایجاد hot spot جلوگیری کند: در حالت extreme، اگر تمام readها و writeها برای یک key یکسان باشند، باز هم تمام requestها به همان partition واحد route می‌شوند.

این نوع workload شاید unusual باشد، اما ناشناخته نیست. برای مثال، در یک social media site، یک user مشهور که میلیون‌ها follower دارد ممکن است هنگام انجام کاری خاص موج بزرگی از activity ایجاد کند [14]. این event می‌تواند حجم بزرگی از write را روی یک key یکسان ایجاد کند؛ برای مثال key ممکن است user ID آن celebrity یا ID مربوط به actionای باشد که userها دربارهٔ آن comment می‌گذارند. Hash کردن key کمکی نمی‌کند، چون hash دو ID یکسان نیز یکسان است.

امروزه بیشتر data systemها نمی‌توانند چنین workload به‌شدت skewشده‌ای را به‌صورت automatic جبران کنند؛ بنابراین مسئولیت کاهش skew بر عهدهٔ application است. برای مثال، اگر بدانیم یک key مشخص بسیار hot است، یک تکنیک ساده این است که یک random number را به ابتدا یا انتهای key اضافه کنیم. حتی یک random number اعشاری دودرقمی می‌تواند writeهای مربوط به آن key را به‌طور یکنواخت میان ۱۰۰ key متفاوت تقسیم کند و اجازه دهد این keyها در partitionهای مختلف توزیع شوند.

بااین‌حال، پس از تقسیم writeها میان keyهای متفاوت، readها باید کار بیشتری انجام دهند، چون لازم است data را از هر ۱۰۰ key بخوانند و آن‌ها را با هم combine کنند. این تکنیک به bookkeeping اضافی نیز نیاز دارد: اضافه کردن random number فقط برای تعداد کمی از hot keyها منطقی است؛ برای اکثریت بزرگی از keyها که write throughput پایینی دارند، این کار overhead غیرضروری ایجاد می‌کند. بنابراین باید راهی هم برای track کردن keyهایی داشته باشید که split شده‌اند.

شاید در آینده data systemها بتوانند workloadهای skewشده را به‌صورت automatic detect و جبران کنند؛ اما در حال حاضر باید trade-offهای مربوط به application خود را با دقت بررسی کنید.

## Key Terms

- `Key-Value Data Model` — data model ساده‌ای که recordها را با یک key، معمولاً primary key، پیدا می‌کند.
- `Key Range` — بازه‌ای پیوسته از keyها که به یک partition اختصاص داده می‌شود.
- `Range Partitioning` — partition کردن data بر اساس rangeهای پیوستهٔ key.
- `Hashing` — تبدیل key به مقدار hash برای تعیین partition و توزیع یکنواخت‌تر load.
- `Hash Function` — functionی که key را به مقدار hash تبدیل می‌کند.
- `Hash Partitioning` — partition کردن data بر اساس range مقدار hash key، نه range خود key.
- `Skew` — توزیع نابرابر data یا query load میان partitionها.
- `Hot Spot` — partition یا keyای که به‌دلیل load نامتناسباً زیاد به bottleneck تبدیل می‌شود.
- `Load Distribution` — پخش کردن data و workload میان nodeها و partitionها برای جلوگیری از تمرکز بار.
- `Consistent Hashing` — روشی مبتنی بر hash برای انتخاب partition boundaryها و توزیع load؛ در این chapter اصطلاح `Hash Partitioning` برای جلوگیری از ابهام ترجیح داده می‌شود.
- `Compound Primary Key` — primary keyای متشکل از چند column که می‌تواند برای partitioning و sort کردن data استفاده شود.
- `Concatenated Index` — indexای که از چند بخش key تشکیل می‌شود و برای دسترسی مرتب به recordهای مرتبط به‌کار می‌رود.
- `Range Scan` — خواندن کارآمد تمام recordهایی که key آن‌ها در یک range مشخص قرار دارد.
- `Hot Key` — keyای که حجم بسیار زیادی از read یا write را دریافت می‌کند و می‌تواند یک partition را overloaded کند.

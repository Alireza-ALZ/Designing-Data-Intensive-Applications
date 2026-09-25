# Chapter 3 — Storage and Retrieval

## Column-Oriented Storage

اگر در fact tableهای خود trillions ردیف و petabytes داده داشته باشید، ذخیره‌سازی و query کردن efficient آن‌ها به مسئله‌ای challenging تبدیل می‌شود. Dimension tableها معمولاً بسیار کوچک‌ترند (millions ردیف)، بنابراین در این بخش عمدتاً روی storage مربوط به factها تمرکز می‌کنیم.

Fact tableها اغلب بیش از ۱۰۰ column دارند، اما یک query معمولی data warehouse در هر نوبت فقط به ۴ یا ۵ مورد از آن‌ها دسترسی پیدا می‌کند (`SELECT *` در analytics به‌ندرت لازم است) [51]. Query مثال ۳-۱ را در نظر بگیرید: این query تعداد زیادی row را access می‌کند (هر occurrence از خرید fruit یا candy در طول سال تقویمی ۲۰۱۳)، اما فقط به سه column از tableِ `fact_sales` نیاز دارد: `date_key`، `product_sk` و `quantity`. Query تمام columnهای دیگر را نادیده می‌گیرد.

```sql
SELECT
  dim_date.weekday, dim_product.category,
  SUM(fact_sales.quantity) AS quantity_sold
FROM fact_sales
  JOIN dim_date    ON fact_sales.date_key = dim_date.date_key
  JOIN dim_product ON fact_sales.product_sk = dim_product.product_sk
WHERE
  dim_date.year = 2013 AND
  dim_product.category IN ('Fresh fruit', 'Candy')
GROUP BY
  dim_date.weekday, dim_product.category;
```

چگونه می‌توان این query را به‌شکل efficient اجرا کرد؟

در بیشتر OLTP databaseها، storage به‌شکل row-oriented چیده می‌شود: تمام valueهای یک row در کنار هم ذخیره می‌شوند. Document databaseها نیز شبیه همین هستند: معمولاً یک document کامل به‌صورت یک sequence پیوسته از byteها ذخیره می‌شود. این موضوع را می‌توان در مثال CSV شکل ۳-۱ مشاهده کرد.

برای پردازش queryای مانند مثال ۳-۱، ممکن است روی `fact_sales.date_key` و/یا `fact_sales.product_sk` index داشته باشید تا storage engine بداند تمام saleهای مربوط به یک date یا product مشخص را کجا پیدا کند. اما یک row-oriented storage engine همچنان باید تمام آن rowها را (که هرکدام بیش از ۱۰۰ attribute دارند) از disk در memory load کند، آن‌ها را parse کند و مواردی را که شرایط لازم را ندارند filter کند. این کار می‌تواند زمان زیادی ببرد.

ایدهٔ column-oriented storage ساده است: به‌جای اینکه تمام valueهای یک row را کنار هم ذخیره کنیم، تمام valueهای هر column را کنار هم ذخیره می‌کنیم. اگر هر column در file جداگانه‌ای ذخیره شود، query فقط لازم است columnهایی را بخواند و parse کند که در همان query استفاده شده‌اند؛ در نتیجه مقدار زیادی از کار حذف می‌شود. این اصل در شکل ۳-۱۰ نشان داده شده است.

فهم column storage در یک relational data model ساده‌تر است، اما این روش به data غیررابطه‌ای نیز به همان اندازه اعمال می‌شود. برای مثال، `Parquet` [57] یک columnar storage format است که از document data model پشتیبانی می‌کند و بر پایهٔ `Dremel` گوگل [54] ساخته شده است.

**شکل ۳-۱۰.** ذخیره‌سازی relational data بر اساس column، به‌جای row.

طرح column-oriented storage به این موضوع متکی است که file مربوط به هر column، rowها را با همان order نگهداری کند. بنابراین اگر لازم باشد یک row کامل را دوباره بسازید، می‌توانید entry شمارهٔ ۲۳ را از هرکدام از fileهای column بردارید و آن‌ها را کنار هم بگذارید تا row شمارهٔ ۲۳ table ساخته شود.

### Column Compression

علاوه بر اینکه فقط columnهای لازم برای query را از disk load می‌کنیم، می‌توانیم با compress کردن data نیاز به disk throughput را نیز کاهش دهیم. خوشبختانه column-oriented storage اغلب برای compression بسیار مناسب است.

به sequenceهای valueها برای هر column در شکل ۳-۱۰ نگاه کنید: این sequenceها اغلب بسیار repetitive هستند که نشانهٔ خوبی برای compression است. بسته به data موجود در column می‌توان از techniqueهای compression متفاوتی استفاده کرد. یکی از techniqueهایی که در data warehouseها به‌خصوص effective است، bitmap encoding است که در شکل ۳-۱۱ نشان داده شده است.

**شکل ۳-۱۱.** storage فشرده و bitmap-indexed برای یک column.

اغلب تعداد valueهای distinct در یک column، در مقایسه با تعداد rowها کم است (برای مثال، یک retailer ممکن است billions تراکنش sale داشته باشد، اما فقط ۱۰۰٬۰۰۰ product distinct داشته باشد). اکنون می‌توان columnی با `n` value distinct را به `n` bitmap جداگانه تبدیل کرد: برای هر value distinct یک bitmap، با یک bit برای هر row. اگر row آن value را داشته باشد، bit برابر ۱ و در غیر این صورت برابر ۰ است.

اگر `n` بسیار کوچک باشد (برای مثال، یک column مربوط به country ممکن است تقریباً ۲۰۰ value distinct داشته باشد)، این bitmapها را می‌توان با یک bit به‌ازای هر row ذخیره کرد. اما اگر `n` بزرگ‌تر باشد، در بیشتر bitmapها صفرهای زیادی وجود خواهد داشت (می‌گوییم bitmapها sparse هستند). در این حالت می‌توان bitmapها را علاوه بر آن با run-length encoding فشرده کرد؛ همان‌طور که در پایین شکل ۳-۱۱ نشان داده شده است. این کار می‌تواند encoding یک column را به‌شکل قابل‌توجهی compact کند.

Bitmap indexهایی از این نوع برای queryهایی که در data warehouse رایج‌اند بسیار مناسب هستند. برای مثال:

```text
WHERE product_sk IN (30, 68, 69):
    سه bitmap مربوط به product_sk = 30، product_sk = 68 و product_sk = 69 را load کنید و
    bitwise OR آن‌ها را محاسبه کنید؛ این کار را می‌توان بسیار efficient انجام داد.
```

```text
WHERE product_sk = 31 AND store_sk = 3:
    bitmapهای product_sk = 31 و store_sk = 3 را load کنید و bitwise AND آن‌ها را محاسبه
    کنید. این کار به این دلیل ممکن است که columnها rowها را با همان order نگهداری می‌کنند؛
    بنابراین kth bit در bitmap یک column، به همان rowای مربوط است که kth bit در bitmap
    column دیگر به آن مربوط است.
```

برای typeهای مختلف data، schemeهای compression دیگری نیز وجود دارد، اما در اینجا وارد جزئیات آن‌ها نمی‌شویم؛ برای overview به [58] مراجعه کنید.

#### Column-oriented storage و column families

`Cassandra` و `HBase` مفهومی به نام column family دارند که آن را از `Bigtable` [9] به ارث برده‌اند. بااین‌حال، column-oriented نامیدن آن‌ها بسیار گمراه‌کننده است: این systemها درون هر column family، تمام columnهای یک row را همراه با row key در کنار هم ذخیره می‌کنند و از column compression استفاده نمی‌کنند. بنابراین مدل `Bigtable` همچنان عمدتاً row-oriented است.

#### Memory bandwidth و vectorized processing

برای data warehouse queryهایی که باید میلیون‌ها row را scan کنند، یک bottleneck بزرگ bandwidth لازم برای انتقال data از disk به memory است. اما این تنها bottleneck نیست. توسعه‌دهندگان analytical databaseها همچنین نگران استفادهٔ efficient از bandwidth بین main memory و CPU cache، جلوگیری از branch mispredictionها و bubbleها در pipeline پردازش instructionهای CPU، و استفاده از instructionهای single-instruction-multi-data (SIMD) در CPUهای مدرن هستند [59, 60].

علاوه بر کاهش حجم dataای که باید از disk load شود، layoutهای column-oriented storage برای استفادهٔ efficient از CPU cycleها نیز مناسب‌اند. برای مثال، query engine می‌تواند یک chunk از column data فشرده را که به‌راحتی در L1 cache پردازنده جا می‌شود بردارد و آن را در یک loop فشرده iterate کند (یعنی بدون function call). CPU می‌تواند چنین loopی را بسیار سریع‌تر از codeای اجرا کند که برای هر record پردازش‌شده به function callها و conditionهای زیادی نیاز دارد. Column compression باعث می‌شود rowهای بیشتری از یک column در همان مقدار L1 cache جا بگیرند. Operatorهایی مانند bitwise AND و OR که پیش‌تر توضیح داده شدند، می‌توانند مستقیماً روی چنین chunkهایی از column data فشرده کار کنند. این technique، vectorized processing نام دارد [58, 49].

### Sort Order in Column Storage

در یک column store، order ذخیره شدن rowها لزوماً اهمیتی ندارد. ساده‌ترین روش این است که rowها را به همان orderی ذخیره کنیم که insert شده‌اند؛ چون در این صورت insert کردن یک row جدید فقط به append کردن آن به هرکدام از fileهای column نیاز دارد. بااین‌حال، می‌توانیم مانند کاری که پیش‌تر با SSTableها انجام دادیم، یک order را اعمال کنیم و از آن به‌عنوان indexing mechanism استفاده کنیم.

توجه کنید که sort کردن مستقل هر column منطقی نیست، چون در آن صورت دیگر نمی‌دانیم کدام itemها در columnهای مختلف به یک row تعلق دارند. ما فقط به این دلیل می‌توانیم یک row را بازسازی کنیم که می‌دانیم item شمارهٔ `k` در یک column به همان rowای تعلق دارد که item شمارهٔ `k` در column دیگر به آن تعلق دارد.

در عوض، data باید در سطح یک row کامل sort شود، حتی اگر به‌صورت column ذخیره شده باشد. administrator database می‌تواند با توجه به queryهای رایج مشخص کند table بر اساس کدام columnها sort شود. برای مثال، اگر queryها اغلب محدوده‌های date، مانند ماه گذشته، را هدف بگیرند، شاید منطقی باشد `date_key` را اولین sort key قرار دهیم. در این صورت query optimizer می‌تواند فقط rowهای ماه گذشته را scan کند که بسیار سریع‌تر از scan کردن تمام rowهاست.

یک column دوم می‌تواند order مربوط به rowهایی را تعیین کند که در column اول value یکسانی دارند. برای مثال، اگر `date_key` اولین sort key در شکل ۳-۱۰ باشد، شاید منطقی باشد `product_sk` را دومین sort key قرار دهیم تا تمام saleهای یک product در یک روز در storage کنار هم group شوند. این کار به queryهایی کمک می‌کند که لازم است saleها را بر اساس product، در یک date range مشخص، group یا filter کنند.

یکی دیگر از مزیت‌های sorted order این است که می‌تواند به compression columnها کمک کند. اگر primary sort column valueهای distinct زیادی نداشته باشد، پس از sort کردن sequenceهای طولانی‌ای ایجاد می‌شود که در آن‌ها یک value بارها پشت سر هم تکرار شده است. یک run-length encoding ساده، مانند آنچه برای bitmapهای شکل ۳-۱۱ استفاده کردیم، می‌تواند آن column را حتی در صورتی که table billions row داشته باشد به چند kilobyte فشرده کند.

اثر compression روی اولین sort key بیشترین مقدار را دارد. دومین و سومین sort keyها بیشتر درهم‌ریخته‌اند و در نتیجه runهای طولانی از valueهای تکراری ندارند. columnهایی که در اولویت sort پایین‌تر قرار دارند، اساساً به‌شکل random ظاهر می‌شوند و احتمالاً به همان خوبی compress نمی‌شوند. بااین‌حال، sorted بودن چند column اول در مجموع همچنان سودمند است.

#### چند sort order متفاوت

ایدهٔ هوشمندانه‌ای برای گسترش این روش ابتدا در `C-Store` معرفی و بعد در data warehouse تجاری `Vertica` به کار گرفته شد [61, 62]. Queryهای مختلف از sort orderهای مختلف سود می‌برند؛ پس چرا همان data را در چند روش مختلف sort‌شده ذخیره نکنیم؟ Data در هر صورت باید روی چند machine replicate شود تا در صورت failure یک machine، data را از دست ندهیم. بنابراین می‌توانیم از data redundant برای نگهداری نسخه‌هایی با sort orderهای متفاوت استفاده کنیم تا هنگام پردازش query، نسخه‌ای را انتخاب کنیم که با pattern همان query بیشترین تناسب را دارد.

داشتن چند sort order در یک column-oriented store تا حدی شبیه داشتن چند secondary index در یک row-oriented store است. اما تفاوت بزرگ این است که row-oriented store هر row را در یک location نگه می‌دارد (در heap file یا clustered index) و secondary indexها فقط pointerهایی به rowهای matching دارند. در column store معمولاً pointerای به data در location دیگر وجود ندارد و فقط columnهایی وجود دارند که valueها را نگه می‌دارند.

### Writing to Column-Oriented Storage

این optimizationها در data warehouseها منطقی هستند، چون بخش عمدهٔ load از queryهای بزرگ و read-only تشکیل می‌شود که analystها اجرا می‌کنند. Column-oriented storage، compression و sorting همگی به سریع‌تر شدن این read queryها کمک می‌کنند. بااین‌حال، نقطه‌ضعف آن‌ها این است که write کردن را دشوارتر می‌کنند.

رویکرد update-in-place، مانند رویکردی که B-treeها استفاده می‌کنند، با columnهای فشرده ممکن نیست. اگر بخواهید یک row را در میانهٔ tableای sorted insert کنید، به احتمال زیاد باید تمام fileهای column را دوباره بنویسید. از آنجا که rowها با position خود درون column شناسایی می‌شوند، insertion باید تمام columnها را به‌شکل consistent update کند.

خوشبختانه پیش‌تر در همین chapter راه‌حل خوبی برای این مسئله دیدیم: LSM-treeها. تمام writeها ابتدا به یک store در memory می‌روند؛ در آنجا به یک structure sorted اضافه می‌شوند و برای write شدن روی disk آماده می‌گردند. مهم نیست store در memory row-oriented باشد یا column-oriented. وقتی writeهای کافی جمع شد، با column fileهای روی disk merge می‌شوند و به‌صورت bulk در fileهای جدید write می‌گردند. `Vertica` نیز اساساً همین کار را انجام می‌دهد [62].

Queryها باید هم data مربوط به columnهای روی disk و هم writeهای جدید در memory را بررسی کنند و این دو را با هم combine کنند. بااین‌حال، query optimizer این تفاوت را از user پنهان می‌کند. از دید analyst، dataای که با insert، update یا delete تغییر کرده است، بلافاصله در queryهای بعدی منعکس می‌شود.

### Aggregation: Data Cubes and Materialized Views

هر data warehouse الزاماً column store نیست: databaseهای سنتی row-oriented و چند architecture دیگر نیز استفاده می‌شوند. بااین‌حال، columnar storage می‌تواند برای ad hoc analytical queryها به‌طور قابل‌توجهی سریع‌تر باشد و به همین دلیل به‌سرعت محبوبیت پیدا می‌کند [51, 63].

یکی دیگر از جنبه‌های data warehouseها که ارزش اشارهٔ کوتاه دارد، materialized aggregateهاست. همان‌طور که پیش‌تر گفتیم، data warehouse queryها اغلب شامل یک aggregate function مانند `COUNT`، `SUM`، `AVG`، `MIN` یا `MAX` در SQL هستند. اگر queryهای مختلف زیادی از aggregateهای یکسان استفاده کنند، پردازش data خام از ابتدا در هر بار می‌تواند wasteful باشد. چرا بعضی countها یا sumهایی را که queryها بیشتر از همه استفاده می‌کنند cache نکنیم؟

یکی از روش‌های ساختن چنین cacheای، materialized view است. در relational data model، materialized view اغلب مانند یک view استاندارد (virtual) تعریف می‌شود: objectای شبیه table که محتوای آن result یک query است. تفاوت این است که materialized view یک copy واقعی از result query است که روی disk write می‌شود، درحالی‌که virtual view فقط shortcutای برای نوشتن queryهاست. وقتی از یک virtual view read می‌کنید، SQL engine آن را در لحظه به query زیرین view expand می‌کند و سپس query گسترش‌یافته را پردازش می‌کند.

وقتی data زیرین تغییر می‌کند، materialized view باید update شود، چون یک copy denormalized از data است. database می‌تواند این کار را به‌صورت automatic انجام دهد، اما چنین updateهایی writeها را پرهزینه‌تر می‌کنند؛ به همین دلیل materialized viewها معمولاً در OLTP databaseها استفاده نمی‌شوند. در data warehouseهای read-heavy ممکن است استفاده از آن‌ها منطقی‌تر باشد (اینکه واقعاً read performance را بهتر کنند یا نه، به case مشخص بستگی دارد).

یک case خاص و رایج از materialized view، data cube یا OLAP cube نام دارد [64]. Data cube شبکه‌ای از aggregateهاست که بر اساس dimensionهای مختلف group شده‌اند. شکل ۳-۱۲ نمونه‌ای از آن را نشان می‌دهد.

**شکل ۳-۱۲.** دو dimension از یک data cube که data را با محاسبهٔ sum aggregate می‌کند.

فعلاً تصور کنید هر fact فقط به دو dimension table foreign key دارد؛ در شکل ۳-۱۲ این دو dimension عبارت‌اند از date و product. اکنون می‌توانید یک table دوبعدی رسم کنید که dateها در امتداد یک محور و productها در امتداد محور دیگر قرار گرفته‌اند. هر cell شامل aggregate (برای مثال `SUM`) یک attribute (برای مثال `net_price`) از تمام factهایی است که آن ترکیب date-product را دارند. سپس می‌توانید همین aggregate را روی هر row یا column اعمال کنید و summaryای به دست آورید که با یک dimension کمتر خلاصه شده است: saleهای هر product، بدون توجه به date، یا saleهای هر date، بدون توجه به product.

در حالت کلی، factها اغلب بیش از دو dimension دارند. در شکل ۳-۹ پنج dimension وجود دارد: date، product، store، promotion و customer. تصور کردن اینکه یک hypercube پنج‌بعدی چه شکلی دارد بسیار دشوارتر است، اما اصل همان است: هر cell شامل saleهای یک ترکیب مشخص از date-product-store-promotion-customer است. سپس می‌توان این valueها را به‌طور مکرر بر اساس هرکدام از dimensionها summary کرد.

مزیت یک materialized data cube این است که بعضی queryها بسیار سریع می‌شوند، چون عملاً از پیش compute شده‌اند. برای مثال، اگر بخواهید total sale به‌ازای هر store در روز گذشته را بدانید، فقط لازم است totalهای dimension مناسب را بخوانید و نیازی به scan کردن millions row ندارید.

نقطه‌ضعف این است که data cube انعطاف‌پذیری query کردن raw data را ندارد. برای مثال، راهی برای محاسبهٔ این موضوع وجود ندارد که چه proportionای از saleها از itemهایی با قیمت بیشتر از ۱۰۰ دلار به دست آمده است، چون price یکی از dimensionها نیست.

بنابراین بیشتر data warehouseها تلاش می‌کنند تا حد ممکن raw data را نگه دارند و از aggregateهایی مانند data cube فقط به‌عنوان boost عملکرد برای queryهای مشخص استفاده کنند.

## Key Terms

- `Column-Oriented Storage` — روشی که valueهای هر column را کنار هم ذخیره می‌کند، نه valueهای هر row را.
- `Column Store` — storage engine یا databaseای که layout اصلی آن column-oriented است.
- `Column Compression` — فشرده‌سازی مستقل columnها برای کاهش disk I/O و افزایش کارایی queryهای تحلیلی.
- `Bitmap Encoding` — نمایش valueهای column با bitmapهای جداگانه و یک bit برای هر row.
- `Run-Length Encoding` — compressionای که sequenceهای تکراری را به value و طول run تبدیل می‌کند.
- `Columnar Format` — format ذخیره‌سازی‌ای که data را به‌صورت columnar سازمان‌دهی می‌کند؛ مانند `Parquet`.
- `Vectorized Processing` — پردازش chunkهای data به‌صورت batch با استفادهٔ efficient از CPU cache و SIMD.
- `Sort Order` — ترتیب از پیش تعیین‌شدهٔ rowها در storage که می‌تواند به indexing، filtering و compression کمک کند.
- `Column Family` — groupingای از columnها در systemهایی مانند Cassandra و HBase که برخلاف column store واقعی، معمولاً rowها را درون family کنار هم ذخیره می‌کند.
- `Data Cube` — ساختار چندبعدی از aggregateهای از پیش محاسبه‌شده بر اساس dimensionهای مختلف.
- `Materialized View` — copy ذخیره‌شده روی disk از result یک query که برای کاهش هزینهٔ محاسبهٔ دوباره استفاده می‌شود.
- `Aggregation` — ترکیب مجموعه‌ای از rowها برای تولید valueهایی مانند count، sum، average، minimum یا maximum.

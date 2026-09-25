# Chapter 3 — Storage and Retrieval

## Data Structures That Power Your Database

ساده‌ترین database دنیا را در نظر بگیرید؛ databaseای که با دو Bash function پیاده‌سازی شده است:

~~~bash
#!/bin/bash

db_set () {
    echo "$1,$2" >> database
}

db_get () {
    grep "^$1," database | sed -e "s/^$1,//" | tail -n 1
}
~~~

این دو function یک key-value store پیاده‌سازی می‌کنند. می‌توانید db_set key value را call کنید تا key و value در database ذخیره شوند. key و value تقریباً می‌توانند هر چیزی باشند؛ برای مثال، value می‌تواند یک JSON document باشد. سپس می‌توانید db_get key را call کنید تا جدیدترین value مرتبط با آن key پیدا و برگردانده شود.

و این روش واقعاً کار می‌کند:

~~~bash
$ db_set 123456 '{"name":"London","attractions":["Big Ben","London Eye"]}'

$ db_set 42 '{"name":"San Francisco","attractions":["Golden Gate Bridge"]}'

$ db_get 42
{"name":"San Francisco","attractions":["Golden Gate Bridge"]}
~~~

فرمت storage بسیار ساده است: یک text file که در هر line آن یک key-value pair با comma از هم جدا شده است؛ تقریباً شبیه CSV، با صرف‌نظر از مسئلهٔ escaping. هر بار که db_set call می‌شود، داده به انتهای file append می‌شود. بنابراین اگر یک key را چند بار update کنید، versionهای قدیمی value overwrite نمی‌شوند و برای پیدا کردن جدیدترین value باید آخرین occurrence آن key را در file پیدا کنید؛ به همین دلیل در db_get از tail -n 1 استفاده شده است:

~~~bash
$ db_set 42 '{"name":"San Francisco","attractions":["Exploratorium"]}'

$ db_get 42
{"name":"San Francisco","attractions":["Exploratorium"]}

$ cat database
123456,{"name":"London","attractions":["Big Ben","London Eye"]}
42,{"name":"San Francisco","attractions":["Golden Gate Bridge"]}
42,{"name":"San Francisco","attractions":["Exploratorium"]}
~~~

functionِ db_set با وجود سادگی، performance نسبتاً خوبی دارد، چون append کردن به file معمولاً بسیار efficient است. بسیاری از databaseها نیز مانند db_set در داخل خود از یک log استفاده می‌کنند؛ یعنی data fileای append-only. Databaseهای واقعی باید با مسائل بیشتری مانند concurrency control، آزاد کردن disk space برای جلوگیری از رشد بی‌نهایت log و مدیریت errorها و recordهای ناقص یا partially written نیز کنار بیایند، اما principle پایه همان است. Logها بسیار مفیدند و در ادامهٔ این book چندین بار با آن‌ها روبه‌رو خواهیم شد.

در این book، واژهٔ log به معنای عمومی‌تری استفاده می‌شود: sequenceای append-only از recordها. log الزاماً human-readable نیست و ممکن است binary باشد و فقط برای خواندن توسط programهای دیگر طراحی شده باشد.

در مقابل، functionِ db_get اگر تعداد recordهای database زیاد باشد performance بسیار بدی دارد. هر بار که بخواهید keyای را lookup کنید، db_get باید کل database file را از ابتدا تا انتها scan کند و occurrenceهای آن key را پیدا کند. از نظر algorithmic، هزینهٔ lookup برابر با O(n) است: اگر تعداد recordهای database یعنی n را دو برابر کنید، lookup نیز دو برابر طول می‌کشد. این مطلوب نیست.

برای پیدا کردن efficient مقدار مربوط به یک key مشخص در database، به data structure متفاوتی نیاز داریم: یک index. در این chapter مجموعه‌ای از indexing structureها را بررسی و با هم مقایسه می‌کنیم. ایدهٔ کلی این است که metadata اضافی‌ای در کنار data اصلی نگه داریم که مانند یک signpost عمل کند و به پیدا کردن دادهٔ موردنظر کمک کند. اگر بخواهید یک data set را از چند روش مختلف search کنید، ممکن است به چند index روی بخش‌های مختلف data نیاز داشته باشید.

Index یک structure اضافی است که از data اصلی derive می‌شود. بسیاری از databaseها اجازه می‌دهند indexها را اضافه یا حذف کنید، بدون اینکه محتوای database تغییر کند؛ index فقط performance queryها را تغییر می‌دهد. نگهداری structureهای اضافی overhead ایجاد می‌کند، به‌خصوص هنگام write. از نظر write، شکست دادن performance append ساده به file دشوار است، چون این ساده‌ترین write operation ممکن است. هر نوع index معمولاً writeها را کند می‌کند، چون هر بار که data نوشته می‌شود، index نیز باید update شود.

این موضوع یک trade-off مهم در storage systemهاست: indexهای درست‌انتخاب‌شده read queryها را سریع می‌کنند، اما هر index writeها را کندتر می‌کند. به همین دلیل databaseها معمولاً به‌صورت پیش‌فرض روی همه‌چیز index نمی‌سازند و از شما، یعنی application developer یا database administrator، می‌خواهند indexها را به‌صورت دستی و بر اساس شناختی که از query patternهای معمول application دارید انتخاب کنید. به این ترتیب می‌توانید indexهایی را انتخاب کنید که بیشترین benefit را برای application دارند، بدون اینکه overhead غیرضروری ایجاد شود.

### Hash Indexes

از indexهای مربوط به key-value data شروع کنیم. این تنها نوع dataای نیست که می‌توان index کرد، اما بسیار رایج است و building block مفیدی برای indexهای پیچیده‌تر محسوب می‌شود.

Key-value storeها بسیار شبیه type دیکشنری در بیشتر programming languageها هستند؛ typeای که معمولاً با یک hash map یا hash table پیاده‌سازی می‌شود. hash mapها در بسیاری از کتاب‌های algorithm توضیح داده شده‌اند [1, 2]، بنابراین در اینجا وارد جزئیات نحوهٔ کار آن‌ها نمی‌شویم. حالا که برای data structureهای in-memory از hash map استفاده می‌کنیم، چرا از آن‌ها برای index کردن data روی disk استفاده نکنیم؟

فرض کنید data storage ما، مانند مثال قبل، فقط شامل append کردن به یک file باشد. در این صورت ساده‌ترین indexing strategy این است که یک hash map در memory نگه داریم که هر key را به یک byte offset در data file map کند؛ یعنی locationای که value در آن قرار دارد، همان‌طور که در شکل ۳-۱ نشان داده شده است. هر زمان که یک key-value pair جدید به file append می‌کنید، hash map را نیز update می‌کنید تا offset دادهٔ تازه‌نوشته‌شده را نشان دهد. این روش هم برای insert کردن keyهای جدید و هم برای update کردن keyهای موجود کار می‌کند. هنگام lookup یک value، از hash map برای پیدا کردن offset در data file استفاده می‌کنید، به آن location seek می‌کنید و value را می‌خوانید.

*شکل ۳-۱. ذخیرهٔ logای از key-value pairها در فرمت شبیه CSV، با indexای از نوع in-memory hash map.*

این روش شاید بیش از حد ساده به نظر برسد، اما رویکردی عملی است. در واقع، Bitcask، یعنی default storage engine در Riak، اساساً همین کار را انجام می‌دهد [3]. Bitcask read و writeهای high-performance ارائه می‌دهد، به شرطی که تمام keyها در RAM موجود جا شوند، چون hash map به‌طور کامل در memory نگهداری می‌شود. valueها می‌توانند فضایی بیشتر از memory موجود مصرف کنند، چون با فقط یک disk seek از disk load می‌شوند. اگر بخش موردنظر data file از قبل در filesystem cache باشد، read اصلاً به disk I/O نیاز نخواهد داشت.

Storage engineای مانند Bitcask برای شرایطی مناسب است که value مربوط به هر key مرتب update می‌شود. برای مثال، key می‌تواند URL یک cat video و value می‌تواند تعداد دفعات play شدن آن باشد؛ مقداری که هر بار user روی دکمهٔ play کلیک می‌کند increment می‌شود. در این workload، writeهای زیادی وجود دارد، اما تعداد keyهای distinct چندان زیاد نیست: writeهای زیادی برای هر key انجام می‌شود، ولی نگه داشتن همهٔ keyها در memory امکان‌پذیر است.

تا اینجا فقط به file append کرده‌ایم؛ پس چگونه از تمام شدن disk space جلوگیری کنیم؟ راه‌حل خوب این است که log را به segmentهایی با اندازهٔ مشخص تقسیم کنیم: وقتی یک segment file به اندازهٔ معین رسید، آن را close کنیم و writeهای بعدی را در segment file جدید انجام دهیم. سپس می‌توانیم روی این segmentها compaction انجام دهیم، همان‌طور که در شکل ۳-۲ نشان داده شده است. Compaction یعنی حذف keyهای duplicate در log و نگه داشتن فقط جدیدترین update برای هر key.

*شکل ۳-۲. Compaction یک key-value update log، با نگه داشتن جدیدترین value برای هر key.*

از طرف دیگر، چون compaction معمولاً segmentها را بسیار کوچک‌تر می‌کند (به شرطی که هر key به‌طور متوسط چند بار در یک segment overwrite شده باشد)، می‌توانیم چند segment را هم‌زمان با compaction با هم merge کنیم؛ همان‌طور که در شکل ۳-۳ نشان داده شده است. Segmentها پس از write شدن دیگر modify نمی‌شوند، بنابراین segment mergeشده در file جدیدی نوشته می‌شود. Merging و compaction segmentهای frozen می‌تواند در یک background thread انجام شود و در همین زمان، با استفاده از segment fileهای قدیمی، به serve کردن read و write requestها ادامه دهیم. پس از کامل شدن merge، read requestها را به segment جدید و mergeشده منتقل می‌کنیم و سپس می‌توانیم segment fileهای قدیمی را به‌سادگی delete کنیم.

*شکل ۳-۳. انجام هم‌زمان compaction و segment merging.*

اکنون هر segment hash table in-memory خودش را دارد که keyها را به file offsetها map می‌کند. برای پیدا کردن value مربوط به یک key، ابتدا hash map جدیدترین segment را بررسی می‌کنیم؛ اگر key در آن نبود، hash map دومین segment جدید را بررسی می‌کنیم و همین‌طور ادامه می‌دهیم. فرآیند merge تعداد segmentها را کم نگه می‌دارد، بنابراین lookupها مجبور نیستند hash mapهای زیادی را بررسی کنند.

برای اینکه این ایدهٔ ساده در عمل کار کند، جزئیات زیادی اهمیت دارد. به‌طور خلاصه، چند مسئلهٔ مهم در یک implementation واقعی عبارت‌اند از:

**File format**

CSV بهترین format برای log نیست. استفاده از binary format سریع‌تر و ساده‌تر است؛ در این format ابتدا طول یک string بر حسب byte encode می‌شود و سپس خود string خام، بدون نیاز به escaping، قرار می‌گیرد.

**Deleting records**

اگر بخواهید یک key و value مرتبط با آن را delete کنید، باید یک deletion record ویژه (که گاهی tombstone نامیده می‌شود) به data file append کنید. هنگام merge شدن log segmentها، tombstone به فرآیند merge می‌گوید تمام valueهای قبلی مربوط به key حذف‌شده را دور بیندازد.

**Crash recovery**

اگر database restart شود، hash mapهای in-memory از دست می‌روند. در اصل می‌توانید hash map هر segment را با خواندن کل segment file از ابتدا تا انتها restore کنید و در حین خواندن، offset جدیدترین value مربوط به هر key را ثبت کنید. اما اگر segment fileها بزرگ باشند، این کار ممکن است زمان زیادی ببرد و restart کردن server را دردناک کند. Bitcask با ذخیره کردن snapshot hash map هر segment روی disk، recovery را سریع‌تر می‌کند؛ این snapshot می‌تواند سریع‌تر در memory load شود.

**Partially written records**

Database ممکن است هر زمانی crash کند، حتی در میانهٔ append کردن یک record به log. Fileهای Bitcask شامل checksum هستند و به کمک آن‌ها می‌توان بخش‌های corrupt شدهٔ log را شناسایی و نادیده گرفت.

**Concurrency control**

چون writeها به log با ترتیبی کاملاً sequential append می‌شوند، یک انتخاب رایج در implementation این است که فقط یک writer thread وجود داشته باشد. Data file segmentها append-only و از جنبه‌های دیگر immutable هستند، بنابراین چندین thread می‌توانند هم‌زمان آن‌ها را read کنند.

در نگاه اول، append-only log wasteful به نظر می‌رسد: چرا file را in-place update نکنیم و value قدیمی را با value جدید overwrite نکنیم؟ اما design مربوط به append-only به چند دلیل مناسب است:

- Append کردن و segment merging، operationهای sequential write هستند و معمولاً بسیار سریع‌تر از random writeها انجام می‌شوند؛ به‌خصوص روی hard driveهای مغناطیسی و spinning-disk. تا حدی، sequential writeها روی solid-state drive یا SSD مبتنی بر flash نیز ترجیح داده می‌شوند [4]. این موضوع را در بخش «Comparing B-Trees and LSM-Trees» در صفحهٔ ۸۳ با جزئیات بیشتری بررسی می‌کنیم.
- اگر segment fileها append-only یا immutable باشند، concurrency و crash recovery بسیار ساده‌تر می‌شوند. برای مثال، لازم نیست نگران حالتی باشید که crash در زمانی رخ دهد که value در حال overwrite شدن است و file در نهایت شامل ترکیبی از بخشی از value قدیمی و بخشی از value جدید باشد.
- Merge کردن segmentهای قدیمی از مشکل fragmented شدن data fileها در طول زمان جلوگیری می‌کند.

بااین‌حال، hash table index محدودیت‌هایی نیز دارد:

- Hash table باید در memory جا شود؛ بنابراین اگر تعداد keyها بسیار زیاد باشد، با مشکل روبه‌رو می‌شوید. در اصل می‌توان hash map را روی disk نگه داشت، اما متأسفانه performant کردن hash map روی disk دشوار است. این کار به random access I/O زیادی نیاز دارد، وقتی hash map پر می‌شود grow کردن آن پرهزینه است و hash collisionها نیز به logic پیچیده و ظریف نیاز دارند [5].
- Range queryها efficient نیستند. برای مثال، نمی‌توانید به‌سادگی تمام keyهای بین kitty00000 و kitty99999 را scan کنید؛ مجبورید هر key را جداگانه در hash mapها lookup کنید.

در بخش بعدی indexing structureای را بررسی می‌کنیم که این محدودیت‌ها را ندارد.

### SSTables and LSM-Trees

در شکل ۳-۳، هر log-structured storage segment دنباله‌ای از key-value pairهاست. این pairها به ترتیبی قرار دارند که write شده‌اند و valueهایی که دیرتر در log آمده‌اند بر valueهای مربوط به همان key در بخش‌های قدیمی‌تر log اولویت دارند. به‌جز این، ترتیب key-value pairها در file اهمیتی ندارد.

اکنون می‌توانیم تغییر ساده‌ای در format segment fileها ایجاد کنیم: لازم است sequence مربوط به key-value pairها بر اساس key مرتب شده باشد. در نگاه اول، این requirement توانایی ما برای استفاده از sequential write را از بین می‌برد، اما کمی بعد به این موضوع برمی‌گردیم.

این format را Sorted String Table یا به‌اختصار SSTable می‌نامیم. همچنین لازم است هر key در هر merged segment file فقط یک بار ظاهر شود؛ فرآیند compaction از قبل این شرط را تضمین می‌کند. SSTableها نسبت به log segmentهایی که hash index دارند چند مزیت مهم دارند:

1. Merge کردن segmentها ساده و efficient است، حتی اگر fileها از memory موجود بزرگ‌تر باشند. این approach شبیه algorithm مربوط به mergesort است و در شکل ۳-۴ نشان داده شده است: خواندن input fileها را کنار هم شروع می‌کنید، اولین key در هر file را می‌بینید، key کوچک‌تر را بر اساس sort order در output file copy می‌کنید و این کار را تکرار می‌کنید. نتیجه، segment file جدیدی است که آن هم بر اساس key مرتب شده است.

   *شکل ۳-۴. Merge کردن چند SSTable segment و نگه داشتن جدیدترین value برای هر key.*

   اگر یک key در چند input segment ظاهر شود چه؟ به یاد داشته باشید که هر segment شامل تمام valueهایی است که در یک بازهٔ زمانی در database write شده‌اند. یعنی تمام valueهای موجود در یک input segment باید از تمام valueهای segment دیگر جدیدتر باشند؛ البته اگر همیشه segmentهای مجاور را merge کنیم. وقتی چند segment شامل یک key یکسان هستند، می‌توانیم value مربوط به جدیدترین segment را نگه داریم و valueهای segmentهای قدیمی‌تر را discard کنیم.

2. برای پیدا کردن یک key مشخص در file، دیگر لازم نیست index تمام keyها را در memory نگه دارید. شکل ۳-۵ را در نظر بگیرید: فرض کنید به دنبال keyِ handiwork هستید، اما offset دقیق آن را در segment file نمی‌دانید. بااین‌حال offset مربوط به keyهای handbag و handsome را می‌دانید و چون file مرتب شده است، می‌دانید handiwork باید بین این دو قرار داشته باشد. بنابراین می‌توانید به offset مربوط به handbag بروید و از آنجا scan کنید تا handiwork را پیدا کنید (یا اگر key در file وجود نداشت، به این نتیجه برسید).

   *شکل ۳-۵. یک SSTable با index in-memory.*

   همچنان به indexای در memory نیاز دارید تا offset بعضی keyها را مشخص کند، اما این index می‌تواند sparse باشد: یک key برای هر چند kilobyte از segment file کافی است، چون scan کردن چند kilobyte بسیار سریع انجام می‌شود.

3. چون read requestها برای range موردنظر در هر صورت باید چند key-value pair را scan کنند، می‌توان این recordها را در یک block گروه‌بندی و پیش از write روی disk compress کرد؛ ناحیهٔ shaded در شکل ۳-۵ این موضوع را نشان می‌دهد. هر entry در sparse in-memory index به ابتدای یک compressed block اشاره می‌کند. Compression علاوه بر صرفه‌جویی در disk space، مصرف I/O bandwidth را نیز کاهش می‌دهد.

اگر همهٔ keyها و valueها اندازهٔ ثابت داشتند، می‌توانستید روی segment file از binary search استفاده کنید و به in-memory index نیازی نداشته باشید. اما در عمل key و value معمولاً variable-length هستند و اگر index نداشته باشید، تشخیص اینکه یک record کجا تمام می‌شود و record بعدی از کجا شروع می‌شود دشوار است.

#### Constructing and maintaining SSTables

همه‌چیز خوب به نظر می‌رسد، اما در ابتدا چگونه data را بر اساس key sort کنیم؟ writeهای ورودی ما می‌توانند با هر ترتیبی اتفاق بیفتند.

نگهداری یک sorted structure روی disk ممکن است؛ در بخش «B-Trees» این روش را بررسی می‌کنیم، اما نگهداری آن در memory بسیار ساده‌تر است. data structureهای tree شناخته‌شدهٔ زیادی وجود دارند که می‌توان از آن‌ها استفاده کرد؛ مانند red-black tree یا AVL tree [2]. با این data structureها می‌توانید keyها را با هر ترتیبی insert کنید و آن‌ها را به‌صورت sorted read کنید.

حالا storage engine خود را به شکل زیر طراحی می‌کنیم:

- وقتی writeای وارد می‌شود، آن را به یک balanced tree data structure در memory اضافه می‌کنیم؛ برای مثال، یک red-black tree. این tree in-memory گاهی memtable نامیده می‌شود.
- وقتی اندازهٔ memtable از threshold مشخصی، معمولاً چند megabyte، بیشتر شد، آن را به‌صورت یک SSTable file روی disk write می‌کنیم. این کار efficient است، چون tree از قبل key-value pairها را بر اساس key مرتب نگه داشته است. SSTable file جدید به جدیدترین segment database تبدیل می‌شود. در زمانی که SSTable روی disk write می‌شود، writeها می‌توانند به یک memtable instance جدید ادامه پیدا کنند.
- برای serve کردن read request، ابتدا key را در memtable جست‌وجو می‌کنیم، سپس در جدیدترین on-disk segment، بعد در segment قدیمی‌تر بعدی و به همین ترتیب.
- هر از گاهی یک فرآیند merge و compaction را در background اجرا می‌کنیم تا segment fileها با هم ترکیب شوند و valueهای overwriteشده یا deleteشده discard شوند.

این scheme بسیار خوب کار می‌کند و فقط یک مشکل دارد: اگر database crash کند، جدیدترین writeها که در memtable هستند اما هنوز روی disk write نشده‌اند از دست می‌روند. برای جلوگیری از این مشکل، می‌توانیم یک log جداگانه روی disk نگه داریم که هر write بلافاصله در آن append شود؛ درست مانند بخش قبل. این log sorted نیست، اما مهم نیست، چون تنها هدف آن restore کردن memtable پس از crash است. هر بار که memtable به‌صورت یک SSTable روی disk write می‌شود، log مربوط به آن را می‌توان discard کرد.

#### Making an LSM-tree out of SSTables

الگوریتمی که توضیح دادیم اساساً همان چیزی است که در LevelDB [6] و RocksDB [7] استفاده می‌شود؛ این‌ها libraryهای key-value storage engine هستند که برای embed شدن در applicationهای دیگر طراحی شده‌اند. از جمله کاربردهای LevelDB این است که در Riak به‌عنوان alternativeای برای Bitcask استفاده شود. Storage engineهای مشابهی در Cassandra و HBase [8] نیز به کار می‌روند؛ هر دو از paper مربوط به Bigtable گوگل الهام گرفته‌اند [9]، paperای که اصطلاح‌های SSTable و memtable را معرفی کرد.

این indexing structure نخستین بار توسط Patrick O’Neil و همکارانش با نام Log-Structured Merge-Tree یا به‌اختصار LSM-Tree توصیف شد [10] و بر کارهای قبلی دربارهٔ log-structured filesystemها بنا شده بود [11]. Storage engineهایی که بر اصل merge و compact کردن sorted fileها متکی هستند، اغلب LSM storage engine نامیده می‌شوند.

Lucene، یعنی indexing engine مربوط به full-text search که در Elasticsearch و Solr استفاده می‌شود، روش مشابهی را برای ذخیرهٔ term dictionary خود به کار می‌گیرد [12, 13]. Full-text index بسیار پیچیده‌تر از key-value index است، اما بر ایدهٔ مشابهی بنا شده است: با دریافت یک word در search query، تمام documentهایی (web page، product description و غیره) را پیدا کن که آن word را ذکر کرده‌اند. این کار با key-value structureای پیاده‌سازی می‌شود که key آن یک word یا term و value آن فهرست ID تمام documentهایی است که آن word را در خود دارند؛ این فهرست postings list نامیده می‌شود. در Lucene، mapping میان term و postings list در sorted fileهایی شبیه SSTable نگه داشته می‌شود و در صورت نیاز در background merge می‌شوند [14].

#### Performance optimizations

مانند همیشه، برای performant کردن storage engine در عمل جزئیات زیادی اهمیت دارد. برای مثال، algorithm مربوط به LSM-tree ممکن است هنگام lookup کردن keyهایی که در database وجود ندارند کند باشد: باید ابتدا memtable و سپس تمام segmentها را تا قدیمی‌ترین segment بررسی کنید و شاید برای هرکدام از disk read انجام دهید، تا مطمئن شوید key وجود ندارد.

برای optimize کردن این نوع access، storage engineها اغلب از Bloom filterهای اضافی استفاده می‌کنند [15]. Bloom filter یک data structure کم‌مصرف برای approximate کردن محتوای یک set است. Bloom filter می‌تواند بگوید یک key در database وجود ندارد و در نتیجه بسیاری از disk readهای غیرضروری برای keyهای nonexistent را save کند.

برای تعیین order و timing مربوط به compact و merge کردن SSTableها strategyهای مختلفی وجود دارد. رایج‌ترین optionها size-tiered compaction و leveled compaction هستند. LevelDB و RocksDB از leveled compaction استفاده می‌کنند؛ به همین دلیل نام LevelDB را دارند. HBase از size-tiered compaction و Cassandra از هر دو پشتیبانی می‌کند [16].

در size-tiered compaction، SSTableهای جدیدتر و کوچک‌تر به‌صورت متوالی با SSTableهای قدیمی‌تر و بزرگ‌تر merge می‌شوند. در leveled compaction، key range به SSTableهای کوچک‌تر تقسیم می‌شود و data قدیمی به levelهای جداگانه منتقل می‌شود. این روش اجازه می‌دهد compaction incrementalتر انجام شود و disk space کمتری مصرف کند.

با وجود ظرافت‌های متعدد، ایدهٔ پایهٔ LSM-tree، یعنی نگه داشتن cascadeای از SSTableها که در background با هم merge می‌شوند، ساده و effective است. حتی وقتی dataset بسیار بزرگ‌تر از memory موجود باشد نیز این روش به‌خوبی کار می‌کند. چون data به‌ترتیب sorted ذخیره شده است، می‌توانید range queryها را به‌صورت efficient اجرا کنید؛ یعنی تمام keyهای بالاتر از minimum مشخص و پایین‌تر از maximum مشخص را scan کنید. همچنین چون disk writeها sequential هستند، LSM-tree می‌تواند write throughput بسیار بالایی ارائه دهد.

### B-Trees

Indexهای log-structured که تاکنون بررسی کردیم در حال پذیرفته‌تر شدن هستند، اما رایج‌ترین نوع index نیستند. پرکاربردترین indexing structure کاملاً متفاوت است: B-tree.

B-tree در سال ۱۹۷۰ معرفی شد [17] و کمتر از ده سال بعد «ubiquitous» نام گرفت [18]. این structure آزمون زمان را به‌خوبی پشت سر گذاشته است. B-tree همچنان implementation استاندارد index در تقریباً تمام relational databaseهاست و بسیاری از nonrelational databaseها نیز از آن استفاده می‌کنند.

B-tree مانند SSTableها key-value pairها را بر اساس key مرتب نگه می‌دارد و به این ترتیب key-value lookup و range query را efficient می‌کند. اما شباهت در همین‌جا تمام می‌شود؛ design philosophy مربوط به B-tree کاملاً متفاوت است.

Indexهای log-structured که پیش‌تر دیدیم، database را به segmentهایی با اندازهٔ variable تقسیم می‌کنند که معمولاً چند megabyte یا بیشتر هستند و هر segment را همیشه به‌صورت sequential write می‌کنند. در مقابل، B-tree database را به blockها یا pageهایی با اندازهٔ ثابت تقسیم می‌کند؛ این pageها traditionally اندازهٔ ۴ KB دارند، هرچند گاهی بزرگ‌تر هستند، و در هر لحظه یک page را read یا write می‌کنند. این design با hardware زیرین سازگاری بیشتری دارد، چون diskها نیز به blockهایی با اندازهٔ ثابت تقسیم شده‌اند.

هر page را می‌توان با یک address یا location شناسایی کرد و به این ترتیب یک page می‌تواند به page دیگر reference دهد؛ چیزی شبیه pointer، اما روی disk به‌جای memory. با استفاده از این page referenceها می‌توان treeای از pageها ساخت، همان‌طور که در شکل ۳-۶ نشان داده شده است.

*شکل ۳-۶. پیدا کردن یک key با استفاده از B-tree index.*

یکی از pageها به‌عنوان root مربوط به B-tree تعیین می‌شود. هر زمان بخواهید keyای را در index lookup کنید، از این page شروع می‌کنید. Root page شامل چند key و reference به child pageهاست. هر child مسئول یک range پیوسته از keyهاست و keyهای میان referenceها نشان می‌دهند مرز میان این rangeها کجاست.

در مثال شکل ۳-۶، به دنبال keyِ 251 هستیم؛ بنابراین می‌دانیم باید page referenceای را دنبال کنیم که بین مرزهای 200 و 300 قرار دارد. این کار ما را به page مشابهی می‌رساند که range مربوط به 200 تا 300 را به subrangeهای بیشتری تقسیم می‌کند.

در نهایت به pageای می‌رسیم که شامل keyهای منفرد است؛ این page، leaf page نامیده می‌شود. leaf page یا value هر key را به‌صورت inline در خود نگه می‌دارد یا شامل referenceهایی به pageهایی است که valueها در آن‌ها قرار دارند.

تعداد referenceهای یک page به child pageها branching factor نامیده می‌شود. برای مثال، branching factor در شکل ۳-۶ برابر با ۶ است. در عمل، branching factor به فضای لازم برای ذخیرهٔ page referenceها و range boundaryها بستگی دارد، اما معمولاً چندصد است.

اگر بخواهید value مربوط به key موجودی را update کنید، leaf page دارای آن key را search می‌کنید، value را در همان page تغییر می‌دهید و page را دوباره روی disk می‌نویسید؛ هر reference به آن page همچنان معتبر باقی می‌ماند. اگر بخواهید key جدیدی اضافه کنید، باید pageای را پیدا کنید که range آن key جدید را دربرمی‌گیرد و key را به آن page اضافه کنید. اگر فضای خالی کافی در page وجود نداشته باشد، page به دو page نیمه‌پر split می‌شود و parent page برای ثبت subdivision جدید rangeهای key update می‌شود؛ شکل ۳-۷ این فرایند را نشان می‌دهد.

*شکل ۳-۷. رشد یک B-tree با split کردن page.*

این algorithm تضمین می‌کند که tree balanced باقی بماند: یک B-tree با n key همیشه depthای برابر با O(log n) دارد. بیشتر databaseها در B-treeای با سه یا چهار level جا می‌شوند، بنابراین برای پیدا کردن page موردنظر لازم نیست page referenceهای زیادی را دنبال کنید. یک tree چهارسطحی با pageهای ۴ KB و branching factor برابر با 500 می‌تواند تا 256 TB را ذخیره کند.

وارد کردن key جدید به B-tree نسبتاً قابل‌فهم است، اما delete کردن key در حالی که tree را balanced نگه داریم، پیچیده‌تر است [2].

#### Making B-trees reliable

عملیات write پایه در B-tree، overwrite کردن یک page روی disk با data جدید است. فرض بر این است که overwrite location مربوط به page را تغییر نمی‌دهد؛ یعنی تمام referenceها به آن page پس از overwrite همچنان معتبر می‌مانند. این موضوع تضاد شدیدی با indexهای log-structured مانند LSM-tree دارد که فقط به fileها append می‌کنند و در نهایت fileهای obsolete را delete می‌کنند، اما هیچ fileای را in-place modify نمی‌کنند.

می‌توانید overwrite کردن page روی disk را یک operation واقعی hardware در نظر بگیرید. روی magnetic hard drive، این کار شامل حرکت دادن disk head به location مناسب، صبر کردن تا position درست روی platter چرخان به محل موردنظر برسد و سپس overwrite کردن sector مناسب با data جدید است. روی SSD، ماجرا کمی پیچیده‌تر است، چون SSD باید blockهای نسبتاً بزرگی از storage chip را در هر بار erase و rewrite کند [19].

از طرف دیگر، بعضی operationها به overwrite کردن چند page مختلف نیاز دارند. برای مثال، اگر در نتیجهٔ insert یک page بیش از حد پر شود و آن را split کنید، باید دو page حاصل از split را write کنید و همچنین parent page را برای update کردن referenceهای دو child page overwrite کنید. این operation خطرناک است، چون اگر database پس از write شدن فقط بخشی از pageها crash کند، index corrupt می‌شود؛ برای مثال ممکن است pageای orphan ایجاد شود که child هیچ parentی نباشد.

برای resilient کردن database در برابر crash، معمولاً implementationهای B-tree یک data structure اضافی روی disk دارند: write-ahead log یا WAL که redo log نیز نامیده می‌شود. این log یک file append-only است که هر modification مربوط به B-tree باید پیش از اعمال شدن روی pageهای خود tree در آن write شود. وقتی database پس از crash دوباره بالا می‌آید، از این log برای restore کردن B-tree به یک state consistent استفاده می‌شود [5, 20].

پیچیدگی اضافی update کردن in-place pageها این است که اگر چند thread هم‌زمان به B-tree دسترسی داشته باشند، به concurrency control دقیق نیاز داریم؛ در غیر این صورت ممکن است یک thread tree را در stateای inconsistent ببیند. این کار معمولاً با محافظت از data structureهای tree توسط latchها انجام می‌شود؛ latchها lockهای سبکی هستند. رویکردهای log-structured از این نظر ساده‌ترند، چون تمام mergeها را در background و بدون تداخل با queryهای ورودی انجام می‌دهند و هر از گاهی segmentهای قدیمی را به‌صورت atomic با segmentهای جدید جایگزین می‌کنند.

#### B-tree optimizations

از آنجا که B-treeها مدت زیادی است وجود دارند، طبیعی است که در طول سال‌ها optimizationهای زیادی برای آن‌ها توسعه داده شده باشد. چند مورد از آن‌ها:

- به‌جای overwrite کردن pageها و نگهداری WAL برای crash recovery، بعضی databaseها مانند LMDB از schemeای به نام copy-on-write استفاده می‌کنند [21]. page تغییرکرده در location دیگری write می‌شود و version جدیدی از parent pageهای tree ساخته می‌شود که به location جدید اشاره می‌کند. این approach برای concurrency control نیز مفید است و در بخش «Snapshot Isolation and Repeatable Read» در صفحهٔ ۲۳۷ آن را بررسی خواهیم کرد.
- می‌توان با ذخیره نکردن کل key و کوتاه کردن آن، در pageها فضا save کرد. به‌خصوص در pageهای داخلی tree، keyها فقط باید اطلاعات کافی برای عمل کردن به‌عنوان boundary میان key rangeها را داشته باشند. قرار دادن keyهای بیشتر در یک page باعث افزایش branching factor می‌شود و در نتیجه tree levelهای کمتری خواهد داشت.
- به‌طور کلی pageها می‌توانند در هر locationای روی disk قرار بگیرند و هیچ الزامی وجود ندارد که pageهای مربوط به key rangeهای نزدیک، روی disk نیز نزدیک هم باشند. اگر query لازم باشد بخش بزرگی از key range را به‌ترتیب sorted scan کند، layout page-by-page می‌تواند inefficient باشد، چون برای هر pageای که read می‌شود ممکن است به disk seek نیاز باشد. به همین دلیل بسیاری از B-tree implementationها تلاش می‌کنند tree را طوری layout کنند که leaf pageها روی disk به‌ترتیب sequential قرار بگیرند. بااین‌حال، حفظ این order هنگام رشد tree دشوار است. در مقابل، LSM-treeها هنگام merge، segmentهای بزرگ storage را یک‌جا rewrite می‌کنند و بنابراین حفظ نزدیک بودن keyهای sequential به یکدیگر روی disk برای آن‌ها آسان‌تر است.
- pointerهای اضافی به tree اضافه شده‌اند. برای مثال، هر leaf page ممکن است referenceهایی به sibling page سمت چپ و راست خود داشته باشد؛ این کار اجازه می‌دهد keyها را بدون برگشتن به parent pageها به‌ترتیب scan کنیم.
- variantهایی از B-tree مانند fractal treeها بعضی ایده‌های log-structured را برای کاهش disk seek به عاریت گرفته‌اند؛ این treeها ارتباطی با fractalهای ریاضی ندارند [22].

این variant گاهی B+ tree نامیده می‌شود، هرچند این optimization آن‌قدر رایج است که اغلب آن را از سایر variantهای B-tree متمایز نمی‌کنند.

### Comparing B-Trees and LSM-Trees

با اینکه implementationهای B-tree معمولاً matureتر از implementationهای LSM-tree هستند، LSM-treeها به‌دلیل ویژگی‌های performance خود جالب‌اند. به‌عنوان یک rule of thumb، LSM-treeها معمولاً برای write سریع‌تر و B-treeها برای read سریع‌تر در نظر گرفته می‌شوند [23]. Readها معمولاً در LSM-tree کندترند، چون باید چند data structure و SSTable متفاوت را در stageهای مختلف compaction بررسی کنند.

بااین‌حال، benchmarkها اغلب inconclusive هستند و به جزئیات workload حساس‌اند. برای رسیدن به comparison معتبر باید systemها را با workload مشخص خود test کنید. در این بخش چند نکته را بررسی می‌کنیم که هنگام اندازه‌گیری performance storage engine ارزش توجه دارند.

#### Advantages of LSM-trees

یک B-tree index باید هر قطعهٔ data را دست‌کم دو بار write کند: یک بار در write-ahead log و یک بار در خود tree page؛ و شاید هنگام split شدن pageها دوباره نیز write شود. همچنین write کردن کل page overhead دارد، حتی اگر فقط چند byte در آن page تغییر کرده باشد. بعضی storage engineها حتی برای جلوگیری از باقی ماندن pageای partially updated در صورت power failure، همان page را دو بار overwrite می‌کنند [24, 25].

Indexهای log-structured نیز به‌دلیل compaction و merge مکرر SSTableها، data را چندین بار rewrite می‌کنند. این اثر، یعنی یک write به database که در طول lifetime database به چند write روی disk تبدیل می‌شود، write amplification نام دارد. Write amplification روی SSDها اهمیت ویژه‌ای دارد، چون SSD فقط می‌تواند blockها را تعداد محدودی overwrite کند و پس از آن blockها wear out می‌شوند.

در applicationهای write-heavy، bottleneck performance ممکن است نرخ write کردن database روی disk باشد. در این حالت، write amplification هزینهٔ performance مستقیمی دارد: هرچه storage engine data بیشتری روی disk write کند، با disk bandwidth موجود writeهای کمتری در ثانیه می‌تواند handle کند.

علاوه بر این، LSM-treeها معمولاً می‌توانند write throughput بالاتری را sustain کنند؛ بخشی از این تفاوت به write amplification کمتر مربوط است (هرچند این موضوع به configuration و workload storage engine بستگی دارد) و بخشی به این دلیل است که LSM-treeها fileهای compact و بزرگ SSTable را sequential write می‌کنند، نه اینکه مجبور باشند چندین page در tree را overwrite کنند [26]. این تفاوت روی magnetic hard driveها اهمیت بیشتری دارد، چون sequential write بسیار سریع‌تر از random write است.

LSM-treeها بهتر compress می‌شوند و بنابراین اغلب fileهای کوچک‌تری روی disk نسبت به B-tree ایجاد می‌کنند. B-tree storage engineها به‌دلیل fragmentation بخشی از disk space را بلااستفاده می‌گذارند: وقتی page split می‌شود یا یک row در page موجود جا نمی‌شود، بخشی از فضای page بدون استفاده باقی می‌ماند. چون LSM-treeها page-oriented نیستند و SSTableها را به‌صورت دوره‌ای rewrite می‌کنند تا fragmentation را حذف کنند، storage overhead کمتری دارند؛ به‌خصوص وقتی از leveled compaction استفاده شود [27].

در بسیاری از SSDها، firmware در داخل خود از algorithmای log-structured استفاده می‌کند تا random writeها را به sequential write روی storage chipهای زیرین تبدیل کند؛ بنابراین اثر الگوی write مربوط به storage engine کمتر می‌شود [19]. بااین‌حال، write amplification کمتر و fragmentation پایین‌تر همچنان روی SSD مزیت محسوب می‌شود، چون نمایش compactتر data اجازه می‌دهد در I/O bandwidth موجود read و write requestهای بیشتری انجام شود.

#### Downsides of LSM-trees

یکی از downsideهای log-structured storage این است که فرآیند compaction گاهی با performance read و writeهای جاری تداخل می‌کند. Storage engineها تلاش می‌کنند compaction را به‌صورت incremental و بدون اثرگذاری بر accessهای concurrent انجام دهند، اما diskها resourceهای محدودی دارند و به‌سادگی ممکن است requestی مجبور شود تا پایان یک compaction پرهزینه منتظر بماند. اثر این موضوع بر throughput و average response time معمولاً کم است، اما در percentileهای بالاتر (به بخش «Describing Performance» در صفحهٔ ۱۳ مراجعه کنید)، response time queryهای مربوط به log-structured storage engine گاهی بسیار زیاد می‌شود و B-treeها می‌توانند predictableتر باشند [28].

مشکل دیگر compaction در write throughput بالا رخ می‌دهد: disk bandwidth محدود باید میان initial write (log کردن و flush کردن memtable روی disk) و compaction threadهایی که در background اجرا می‌شوند تقسیم شود. هنگام write کردن در database خالی، تمام disk bandwidth می‌تواند برای initial write استفاده شود، اما هرچه database بزرگ‌تر می‌شود، compaction به disk bandwidth بیشتری نیاز دارد.

اگر write throughput بالا باشد و compaction به‌دقت configuration نشده باشد، ممکن است compaction نتواند با نرخ incoming writeها همگام شود. در این حالت، تعداد segmentهای mergeنشده روی disk دائماً افزایش می‌یابد تا زمانی که disk space تمام شود. Readها نیز کندتر می‌شوند، چون باید fileهای segment بیشتری را بررسی کنند. معمولاً storage engineهای مبتنی بر SSTable نرخ incoming writeها را throttle نمی‌کنند، حتی وقتی compaction نمی‌تواند همگام شود؛ بنابراین برای تشخیص این وضعیت به monitoring صریح نیاز دارید [29, 30].

یکی از مزیت‌های B-tree این است که هر key دقیقاً در یک محل از index وجود دارد، در حالی که یک log-structured storage engine ممکن است چند copy از همان key را در segmentهای مختلف داشته باشد. این ویژگی B-tree را برای databaseهایی که می‌خواهند strong transactional semantics ارائه دهند جذاب می‌کند: در بسیاری از relational databaseها، transaction isolation با lock کردن rangeهای key پیاده‌سازی می‌شود و در B-tree index می‌توان این lockها را مستقیماً به tree متصل کرد [5]. در فصل ۷ این نکته را با جزئیات بیشتری بررسی می‌کنیم.

B-treeها عمیقاً در architecture databaseها ریشه دارند و برای workloadهای مختلف performance خوب و consistentی ارائه می‌دهند؛ بنابراین بعید است به این زودی‌ها کنار گذاشته شوند. در datastoreهای جدید، indexهای log-structured روزبه‌روز محبوب‌تر می‌شوند. برای تعیین اینکه کدام نوع storage engine برای use case شما بهتر است، rule سریع و ساده‌ای وجود ندارد؛ بنابراین ارزش دارد performance را به‌صورت empirical test کنید.

### Other Indexing Structures

تا اینجا فقط key-value indexها را بررسی کردیم؛ این indexها در relational model شبیه primary key index هستند. Primary key یک row را در relational table، یک document را در document database یا یک vertex را در graph database به‌صورت unique شناسایی می‌کند. Recordهای دیگر database می‌توانند با استفاده از primary key یا ID به آن row، document یا vertex reference دهند و index برای resolve کردن این referenceها استفاده می‌شود.

داشتن secondary index نیز بسیار رایج است. در relational databaseها می‌توانید با commandِ CREATE INDEX چند secondary index روی یک table ایجاد کنید و این indexها اغلب برای اجرای efficient joinها حیاتی هستند. برای مثال، در شکل ۲-۱ فصل ۲، احتمالاً روی columnهای user_id یک secondary index می‌داشتید تا بتوانید تمام rowهای متعلق به یک user را در هر table پیدا کنید.

Secondary index را می‌توان به‌سادگی از یک key-value index ساخت. تفاوت اصلی این است که keyها unique نیستند؛ یعنی ممکن است rowها، documentها یا vertexهای زیادی key یکسانی داشته باشند. این مشکل به دو روش حل می‌شود: یا value موجود در index را به فهرستی از row identifierهای match‌شده تبدیل کنیم، مانند postings list در full-text index، یا با append کردن row identifier به key، هر key را unique کنیم. در هر دو حالت می‌توان از B-tree و log-structured index به‌عنوان secondary index استفاده کرد.

#### Storing values within the index

Key در index همان چیزی است که query برای آن search می‌کند، اما value می‌تواند یکی از دو چیز باشد: خود row، document یا vertex موردنظر، یا referenceای به rowای که در جای دیگری ذخیره شده است. در حالت دوم، محلی که rowها در آن ذخیره می‌شوند heap file نام دارد. Heap file داده را در order خاصی نگه نمی‌دارد؛ ممکن است append-only باشد یا rowهای deleteشده را track کند تا بعداً با data جدید overwrite شوند.

رویکرد heap file رایج است، چون وقتی چند secondary index وجود دارد از duplicate شدن data جلوگیری می‌کند: هر index فقط به locationای در heap file reference می‌دهد و data واقعی فقط در یک محل نگهداری می‌شود.

هنگام update کردن value بدون تغییر key، رویکرد heap file می‌تواند بسیار efficient باشد: اگر value جدید از value قدیمی بزرگ‌تر نباشد، record را می‌توان in-place overwrite کرد. اگر value جدید بزرگ‌تر باشد، وضعیت پیچیده‌تر می‌شود، چون احتمالاً باید record به location جدیدی در heap منتقل شود که فضای کافی دارد. در این حالت یا باید تمام indexها را update کرد تا به location جدید record در heap اشاره کنند، یا در location قدیمی heap یک forwarding pointer باقی گذاشت [5].

در بعضی شرایط، hop اضافی از index به heap file برای readها performance penalty زیادی دارد. در این حالت ممکن است بهتر باشد rowِ indexed را مستقیماً داخل index ذخیره کنیم. این structure clustered index نامیده می‌شود. برای مثال، در storage engineِ InnoDB در MySQL، primary key یک table همیشه clustered index است و secondary indexها به primary key reference می‌دهند، نه به locationای در heap file [31]. در SQL Server می‌توانید برای هر table یک clustered index مشخص کنید [32].

سازشی میان clustered index که تمام row data را داخل index نگه می‌دارد و nonclustered index که فقط reference به data را ذخیره می‌کند، covering index یا index with included columns نام دارد. این index بخشی از columnهای table را داخل خود ذخیره می‌کند [33]. در نتیجه بعضی queryها می‌توانند فقط با استفاده از index پاسخ داده شوند؛ در این حالت گفته می‌شود index آن query را cover می‌کند [32].

مانند هر نوع data duplication، clustered و covering indexها readها را سریع‌تر می‌کنند، اما storage اضافی نیاز دارند و می‌توانند writeها را پرهزینه‌تر کنند. Databaseها همچنین برای enforce کردن transactional guaranteeها باید تلاش بیشتری انجام دهند، چون application نباید inconsistencyهای ناشی از duplication را مشاهده کند.

#### Multi-column indexes

Indexهایی که تا اینجا بررسی کردیم فقط یک key را به یک value map می‌کردند. این روش برای query هم‌زمان چند column از یک table (یا چند field از یک document) کافی نیست.

رایج‌ترین نوع multi-column index، concatenated index نام دارد. این index چند field را با append کردن یک column به column دیگر در یک key ترکیب می‌کند؛ تعریف index مشخص می‌کند fieldها با چه ترتیبی concatenate شوند. این روش شبیه phone bookهای کاغذی قدیمی است که از (lastname, firstname) به phone number index می‌سازند. به‌دلیل sort order، index می‌تواند تمام افراد با lastname مشخص یا تمام افرادی را پیدا کند که ترکیب lastname و firstname مشخصی دارند. اما اگر بخواهید تمام افراد دارای firstname مشخص را پیدا کنید، این index بی‌فایده است.

Multi-dimensional index روش عمومی‌تری برای query هم‌زمان چند column است و برای geospatial data اهمیت ویژه‌ای دارد. برای مثال، یک web site جست‌وجوی restaurant ممکن است databaseای داشته باشد که latitude و longitude هر restaurant را ذخیره می‌کند. وقتی user restaurantها را روی map می‌بیند، web site باید تمام restaurantهای درون ناحیهٔ مستطیلی map را که user در حال مشاهدهٔ آن است پیدا کند. این کار به two-dimensional range queryای مانند query زیر نیاز دارد:

~~~sql
SELECT * FROM restaurants WHERE latitude > 51.4946 AND latitude < 51.5079
                            AND longitude > -0.1162 AND longitude < -0.1004;
~~~

یک B-tree یا LSM-tree استاندارد نمی‌تواند چنین queryای را efficient پاسخ دهد: می‌تواند تمام restaurantهای یک range از latitude را پیدا کند (اما با هر longitudeای)، یا تمام restaurantهای یک range از longitude را پیدا کند (اما در هر نقطه‌ای میان قطب شمال و جنوب)، ولی نمی‌تواند هر دو شرط را هم‌زمان اعمال کند.

یک option این است که location دوبعدی را با استفاده از space-filling curve به یک number منفرد تبدیل کنیم و سپس از B-tree معمولی استفاده کنیم [34]. روش رایج‌تر استفاده از spatial indexهای تخصصی مانند R-tree است. برای مثال، PostGIS از طریق Generalized Search Tree indexing facility در PostgreSQL، geospatial indexها را به‌صورت R-tree پیاده‌سازی می‌کند [35]. در اینجا فرصت توضیح جزئیات R-tree را نداریم، اما literature فراوانی دربارهٔ آن وجود دارد.

ایدهٔ جالب این است که multi-dimensional index فقط برای locationهای جغرافیایی نیست. برای مثال، در یک ecommerce site می‌توانید index سه‌بعدی‌ای روی dimensionهای (red, green, blue) بسازید تا productها را در یک range مشخص از رنگ‌ها search کنید. یا در databaseای از weather observationها می‌توانید index دوبعدی‌ای روی (date, temperature) داشته باشید تا تمام observationهای سال ۲۰۱۳ را که temperature آن‌ها بین ۲۵ و ۳۰ درجهٔ Celsius است efficient پیدا کنید.

با index یک‌بعدی مجبورید یا تمام recordهای سال ۲۰۱۳ را scan کنید (بدون توجه به temperature) و سپس آن‌ها را بر اساس temperature filter کنید، یا برعکس. یک 2D index می‌تواند هم‌زمان بر اساس timestamp و temperature محدوده را کوچک کند. این technique در HyperDex استفاده می‌شود [36].

#### Full-text search and fuzzy indexes

تمام indexهایی که تا اینجا بررسی کردیم فرض می‌کنند data دقیق است و اجازه می‌دهند valueهای دقیق key یا rangeای از valueهای key را بر اساس sort order query کنید. این indexها اجازه نمی‌دهند keyهای مشابه، مانند wordهای غلط‌املاء، را search کنید. چنین fuzzy queryهایی به techniqueهای متفاوتی نیاز دارند.

برای مثال، full-text search engineها معمولاً اجازه می‌دهند search برای یک word به synonymهای آن word نیز گسترش پیدا کند، variationهای grammatical wordها نادیده گرفته شوند، occurrenceهای wordهای نزدیک به هم در یک document پیدا شوند و featureهای دیگری نیز ارائه شود که به linguistic analysis متن وابسته‌اند. برای کنار آمدن با typo در document یا query، Lucene می‌تواند text را برای wordهایی در یک edit distance مشخص search کند؛ edit distance برابر با ۱ یعنی یک letter اضافه، حذف یا جایگزین شده است [37].

همان‌طور که در بخش «Making an LSM-tree out of SSTables» در صفحهٔ ۷۸ گفتیم، Lucene برای term dictionary خود از structureای شبیه SSTable استفاده می‌کند. این structure به یک index کوچک در memory نیاز دارد که به queryها بگوید برای پیدا کردن key باید در کدام offset از sorted file نگاه کنند. در LevelDB، این in-memory index مجموعه‌ای sparse از بعضی keyهاست، اما در Lucene، in-memory index یک finite state automaton روی characterهای keyهاست که به trie شباهت دارد [38]. این automaton را می‌توان به Levenshtein automaton تبدیل کرد؛ automatonی که search efficient برای wordهای دارای edit distance مشخص را پشتیبانی می‌کند [39].

تکنیک‌های دیگر fuzzy search در جهت document classification و machine learning حرکت می‌کنند. برای جزئیات بیشتر به textbookهای information retrieval مراجعه کنید [برای مثال، ۴۰].

#### Keeping everything in memory

Data structureهایی که تا اینجا در این chapter بررسی کردیم، همگی پاسخی به محدودیت‌های disk بوده‌اند. در مقایسه با main memory، کار کردن با disk دشوار است. هم در magnetic disk و هم در SSD، اگر بخواهید read و write performance خوبی داشته باشید، data روی disk باید با دقت layout شود. بااین‌حال این دشواری را تحمل می‌کنیم، چون disk دو مزیت مهم دارد: durable است و با خاموش شدن power محتوای آن از بین نمی‌رود؛ همچنین cost آن به ازای هر gigabyte از RAM کمتر است.

با ارزان‌تر شدن RAM، استدلال مربوط به cost-per-gigabyte ضعیف‌تر می‌شود. بسیاری از datasetها آن‌قدر بزرگ نیستند که نتوان آن‌ها را کاملاً در memory نگه داشت؛ حتی ممکن است data در چند machine distributed شود. این موضوع به توسعهٔ in-memory databaseها منجر شده است.

بعضی in-memory key-value storeها مانند Memcached فقط برای caching طراحی شده‌اند؛ در این storeها قابل‌قبول است که با restart شدن machine data از دست برود. اما in-memory databaseهای دیگری با هدف durability طراحی شده‌اند. Durability می‌تواند با hardware ویژه مانند battery-powered RAM، با write کردن log تغییرات روی disk، با write کردن snapshotهای دوره‌ای روی disk یا با replicate کردن state in-memory روی machineهای دیگر به‌دست آید.

وقتی یک in-memory database restart می‌شود، باید state خود را از disk یا از طریق network از یک replica reload کند؛ مگر اینکه از hardware ویژه استفاده شده باشد. با وجود write کردن روی disk، این همچنان یک in-memory database محسوب می‌شود، چون disk فقط به‌عنوان append-only log برای durability استفاده می‌شود و readها کاملاً از memory serve می‌شوند. Write کردن روی disk مزیت operational نیز دارد: fileهای روی disk را می‌توان به‌سادگی backup گرفت، inspect کرد و با utilityهای خارجی analyze کرد.

محصولاتی مانند VoltDB، MemSQL و Oracle TimesTen، in-memory databaseهایی با relational model هستند و vendorهای آن‌ها ادعا می‌کنند که با حذف overheadهای مربوط به مدیریت on-disk data structureها می‌توانند performance را به‌طور چشمگیری بهبود دهند [41, 42]. RAMCloud یک open source in-memory key-value store با durability است که برای data در memory و data روی disk از رویکرد log-structured استفاده می‌کند [43]. Redis و Couchbase با write کردن asynchronous data روی disk، durability ضعیفی ارائه می‌دهند.

برخلاف انتظار، مزیت performance مربوط به in-memory databaseها به این دلیل نیست که نیازی به read کردن از disk ندارند. حتی یک storage engine مبتنی بر disk نیز اگر memory کافی داشته باشید ممکن است هرگز مجبور به read از disk نشود، چون operating system blockهای disk را که اخیراً استفاده شده‌اند در memory cache می‌کند. In-memory databaseها می‌توانند سریع‌تر باشند، چون از overhead encode کردن data structureهای in-memory در formتی که قابل write شدن روی disk باشد اجتناب می‌کنند [44].

علاوه بر performance، حوزهٔ جالب دیگر برای in-memory databaseها ارائهٔ data modelهایی است که implementation آن‌ها با disk-based index دشوار است. برای مثال، Redis interfaceای شبیه database برای data structureهای مختلف مانند priority queue و set ارائه می‌دهد. چون تمام data را در memory نگه می‌دارد، implementation آن نسبتاً ساده است.

پژوهش‌های جدید نشان می‌دهند که architecture یک in-memory database را می‌توان برای پشتیبانی از datasetهایی بزرگ‌تر از memory موجود گسترش داد، بدون اینکه overheadهای architecture disk-centric برگردند [45]. این رویکرد که anti-caching نامیده می‌شود، وقتی memory کافی نیست کم‌استفاده‌ترین data را از memory به disk evict می‌کند و در آینده، هنگام access شدن دوبارهٔ data، آن را به memory load می‌کند.

این روش شبیه کاری است که operating system با virtual memory و swap file انجام می‌دهد، اما database می‌تواند memory را efficientتر از OS مدیریت کند، چون با granularity مربوط به recordهای منفرد کار می‌کند، نه pageهای کامل memory. بااین‌حال، در این approach نیز indexها باید به‌طور کامل در memory جا شوند؛ مانند مثال Bitcask در ابتدای chapter.

اگر تکنولوژی‌های non-volatile memory (NVM) به‌طور گسترده‌تری adopted شوند، احتمالاً تغییرات بیشتری در design مربوط به storage engine لازم خواهد بود [46]. در حال حاضر این حوزه‌ای جدید برای research است، اما ارزش دارد در آینده آن را دنبال کنیم.

## Key Terms

- `Storage Engine` — موتور داخلی ذخیره و بازیابی data؛ data structureها و algorithmهایی را برای write، read و سازمان‌دهی data روی storage فراهم می‌کند.
- `Index` — structure اضافی مشتق‌شده از data برای پیدا کردن سریع‌تر آن؛ read queryها را سریع می‌کند، اما معمولاً write و storage overhead ایجاد می‌کند.
- `Hash Index` — indexای که key را به offset یا value map می‌کند؛ برای key-value lookup سریع مناسب است، اما range query و dataset بسیار بزرگ را به‌خوبی پشتیبانی نمی‌کند.
- `Hash Table` — data structure مبتنی بر hash برای map کردن key به value؛ lookup معمولاً سریع است، اما به memory و مدیریت collision نیاز دارد.
- `SSTable` — فایل sorted از key-value pairها که هر key در آن یک بار ظاهر می‌شود؛ پایهٔ storage engineهای log-structured و مناسب merge، compression و range query است.
- `LSM-Tree` — indexing structure مبتنی بر merge و compaction فایل‌های sorted؛ write throughput بالا و range query efficient فراهم می‌کند، اما compaction و read از چند segment هزینه دارد.
- `Log-Structured Storage` — storage designای که writeها را به‌صورت append-only log انجام می‌دهد؛ sequential write و crash recovery را ساده می‌کند و معمولاً به compaction نیاز دارد.
- `Memtable` — balanced tree in-memory برای نگهداری writeهای پیش از تبدیل شدن به SSTable؛ writeهای جدید را مرتب نگه می‌دارد و بعداً به‌صورت sorted segment روی disk flush می‌شود.
- `Compaction` — حذف versionها و recordهای obsolete یا duplicate از segmentها؛ disk space را آزاد و تعداد segmentهای موردنیاز برای read را کم می‌کند.
- `Segment` — بخشی مستقل از log یا storage file؛ merge، compaction و مدیریت background را روی حجم‌های قابل‌کنترل ممکن می‌کند.
- `Bloom Filter` — data structure کم‌مصرف برای تشخیص احتمالی وجود key در set؛ disk read غیرضروری برای keyهای nonexistent را کاهش می‌دهد و false positive ممکن است داشته باشد.
- `B-Tree` — tree index متوازن با pageهای ثابت و keyهای sorted؛ index استاندارد بسیاری از databaseها برای lookup و range query است.
- `B-Tree Node` — page یا گره‌ای در B-tree که key و reference به childها را نگه می‌دارد؛ range keyها را تقسیم می‌کند و traversal از root تا leaf را ممکن می‌سازد.
- `Page` — block با اندازهٔ ثابت در storage؛ واحد read و write در B-tree و بسیاری از page-oriented storage engineهاست.
- `Branching Factor` — تعداد child referenceهای قابل نگهداری در یک B-tree page؛ هرچه بیشتر باشد، depth tree و تعداد page access لازم کمتر می‌شود.
- `Write-Ahead Log` — logای که modification پیش از اعمال روی data اصلی در آن write می‌شود؛ برای recovery پس از crash و بازگرداندن index به state consistent استفاده می‌شود.
- `Sequential Write` — write کردن data در sequence پیوسته؛ معمولاً از random write سریع‌تر است، به‌خصوص روی diskهای مغناطیسی.
- `Random Access` — دسترسی به locationهای پراکنده بدون ترتیب sequential؛ روی disk می‌تواند پرهزینه باشد و performance storage engine را کاهش دهد.
- `Range Query` — query برای تمام keyها یا recordهای داخل یک محدوده؛ به data sorted و index مناسب برای اجرای efficient نیاز دارد.
- `Secondary Index` — index اضافی غیر از primary key index؛ برای query بر اساس fieldهای دیگر و اجرای efficient join استفاده می‌شود.
- `Clustered Index` — indexای که data اصلی row را در خود index نگه می‌دارد؛ hop به heap file را حذف می‌کند، اما storage و write overhead بیشتری دارد.
- `Covering Index` — indexی که بخشی از columnهای لازم query را نیز در خود دارد؛ بعضی queryها را فقط با index پاسخ می‌دهد و نیاز به read از data اصلی را کم می‌کند.
- `Multi-Column Index` — indexی برای query هم‌زمان چند column یا field؛ برای شرط‌های ترکیبی و queryهای چندبعدی استفاده می‌شود.
- `Full-Text Index` — index تخصصی برای جست‌وجوی termها در documentهای متنی؛ search بر اساس word، synonym، variation و edit distance را پشتیبانی می‌کند.
- `In-Memory Index` — indexی که به‌طور کامل در memory نگهداری می‌شود؛ lookup را سریع می‌کند، اما اندازهٔ dataset به memory موجود محدود می‌شود.
- `Write Amplification` — تبدیل شدن یک logical write به چند physical write روی disk؛ مصرف bandwidth و فرسودگی SSD را افزایش می‌دهد و برای انتخاب storage engine مهم است.
- `Heap File` — محل نگهداری rowها در order غیرمشخص، جدا از index؛ از duplicate شدن data میان secondary indexها جلوگیری می‌کند، اما ممکن است hop اضافی ایجاد کند.
- `Multi-Dimensional Index` — index برای query هم‌زمان چند dimension؛ برای geospatial data و queryهایی مانند range روی latitude و longitude مناسب است.
- `Fuzzy Search` — جست‌وجوی valueهای مشابه، نه فقط valueهای دقیق؛ typo، synonym، variation زبانی و edit distance را پوشش می‌دهد.
- `In-Memory Database` — databaseای که read و processing آن عمدتاً از memory انجام می‌شود؛ latency پایین‌تری دارد و durability آن با log، snapshot، replication یا hardware ویژه تأمین می‌شود.
- `Anti-Caching` — انتقال کم‌استفاده‌ترین data از memory به disk و بازگرداندن آن هنگام access؛ امکان مدیریت dataset بزرگ‌تر از memory را بدون معماری کاملاً disk-centric فراهم می‌کند.

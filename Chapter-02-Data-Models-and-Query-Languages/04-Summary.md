# Chapter 2 — Data Models and Query Languages

## Summary

Data modelها موضوع بسیار گسترده‌ای هستند و در این chapter نگاهی سریع به طیف متنوعی از modelهای مختلف انداختیم. فرصت بررسی تمام جزئیات هر model را نداشتیم، اما امیدواریم این overview به‌اندازه‌ای بوده باشد که علاقه‌مند شوید دربارهٔ modelی که بیشترین تناسب را با requirementهای application شما دارد، بیشتر بدانید.

از نظر تاریخی، داده ابتدا به‌شکل یک tree بزرگ نمایش داده می‌شد (`hierarchical model`)، اما این model برای نمایش many-to-many relationshipها مناسب نبود؛ بنابراین relational model برای حل این مسئله ابداع شد. در دوره‌های جدیدتر، developerها دریافتند که بعضی applicationها نیز به‌خوبی در relational model جا نمی‌گیرند. datastoreهای nonrelational جدید یا همان «NoSQL» در دو جهت اصلی تکامل پیدا کرده‌اند:

1. `Document database`ها use caseهایی را هدف می‌گیرند که داده در documentهای self-contained قرار دارد و relationship میان یک document و document دیگر نادر است.
2. `Graph database`ها در جهت مخالف حرکت می‌کنند و use caseهایی را هدف می‌گیرند که در آن‌ها هر چیزی ممکن است با هر چیز دیگری relationship داشته باشد.

هر سه model، یعنی document، relational و graph، امروزه به‌طور گسترده استفاده می‌شوند و هرکدام در domain مناسب خود عملکرد خوبی دارند. می‌توان یک model را با استفاده از modelی دیگر emulate کرد؛ برای مثال، graph data را می‌توان در یک relational database نمایش داد، اما result اغلب awkward و نامناسب می‌شود. به همین دلیل برای purposeهای مختلف systemهای متفاوتی داریم، نه یک راه‌حل واحد که برای همه‌چیز مناسب باشد.

یکی از ویژگی‌های مشترک document databaseها و graph databaseها این است که معمولاً schema را برای داده‌ای که ذخیره می‌کنند enforce نمی‌کنند؛ این موضوع می‌تواند سازگار کردن application با requirementهای در حال تغییر را آسان‌تر کند. بااین‌حال، application شما به احتمال زیاد همچنان فرض می‌کند که داده structure مشخصی دارد. تفاوت فقط در این است که schema صریح باشد و هنگام write enforce شود، یا implicit باشد و هنگام read مدیریت شود.

هر data model query language یا framework مخصوص خود را دارد و ما چند نمونه را بررسی کردیم: SQL، MapReduce، `aggregation pipeline` در MongoDB، Cypher، SPARQL و Datalog. همچنین به CSS و XSL/XPath نیز اشاره کردیم؛ این‌ها database query language نیستند، اما parallelهای جالبی با آن‌ها دارند.

با وجود اینکه مسیر زیادی را پوشش دادیم، data modelهای زیادی همچنان بررسی‌نشده باقی مانده‌اند. برای نمونه، چند مثال کوتاه:

- پژوهشگرانی که با genome data کار می‌کنند، اغلب باید `sequence-similarity search` انجام دهند. این کار یعنی یک string بسیار طولانی (که نمایندهٔ یک DNA molecule است) را با database بزرگی از stringهایی مقایسه کنند که مشابه آن هستند، اما کاملاً یکسان نیستند. هیچ‌کدام از databaseهایی که در این chapter توضیح داده شدند برای چنین use caseای مناسب نیستند؛ به همین دلیل پژوهشگران software تخصصی genome database مانند `GenBank` ساخته‌اند [48].
- particle physicistها دهه‌هاست که تحلیل large-scale به سبک `Big Data` انجام می‌دهند و projectهایی مانند `Large Hadron Collider (LHC)` اکنون با صدها petabyte داده کار می‌کنند. در چنین scaleای به solutionهای سفارشی نیاز است تا هزینهٔ hardware از کنترل خارج نشود [49].
- `Full-text search` را می‌توان نوعی data model دانست که اغلب در کنار databaseها استفاده می‌شود. `Information retrieval` موضوع تخصصی بزرگی است که در این book با جزئیات زیاد پوشش داده نمی‌شود؛ اما در فصل ۳ و Part III به search indexها خواهیم پرداخت.

فعلاً بحث را همین‌جا متوقف می‌کنیم. در chapter بعد trade-offهایی را بررسی خواهیم کرد که هنگام implementation data modelهای توضیح‌داده‌شده در این chapter مطرح می‌شوند.

## Key Terms

- `Data Model` — مدل سازمان‌دهی و نمایش داده؛ مشخص می‌کند داده چگونه structure پیدا کند و relationshipهای میان داده‌ها چگونه بیان و query شوند.
- `Relational Model` — مدل مبتنی بر table، row و column؛ برای داده‌های structureیافته و relationshipهایی که با join قابل بیان‌اند مناسب است.
- `Document Model` — مدل مبتنی بر documentهای self-contained؛ برای داده‌هایی با structure درختی و relationshipهای کم میان documentها مناسب است.
- `Graph Model` — مدل مبتنی بر vertex و edge؛ برای داده‌های interconnected که relationshipها بخش اصلی آن‌ها هستند مناسب است.
- `Query Language` — زبان بیان خواستهٔ application از datastore؛ روش دسترسی، filter، ترکیب و پردازش داده را مشخص می‌کند.
- `Declarative Query` — queryای که result موردنظر را توصیف می‌کند، نه مراحل اجرای آن را؛ به database اجازه می‌دهد execution plan و optimization را خودش انتخاب کند.
- `Data Processing` — اجرای operationها برای تبدیل، تحلیل یا استخراج داده؛ می‌تواند به‌صورت transactional، batch، online یا distributed انجام شود.
- `Schema` — structure و قواعد مورد انتظار برای داده؛ مشخص می‌کند چه fieldها، typeها و relationshipهایی معتبر هستند و چه زمانی enforce شوند.
- `MapReduce` — programming model پردازش داده در مراحل map و reduce؛ برای پردازش distributed دادهٔ بزرگ استفاده می‌شود و در برخی NoSQL systemها query ارائه می‌کند.
- `Graph Query` — query برای match کردن pattern یا traverse کردن graph؛ برای یافتن entityها و relationshipهای مستقیم یا چندمرحله‌ای به کار می‌رود.
- `Query Optimizer` — جزء database برای انتخاب strategy اجرای query؛ indexها، joinها و ترتیب operationها را برای performance بهتر انتخاب می‌کند.
- `Sequence-Similarity Search` — جست‌وجوی stringهایی که از نظر sequence شبیه یکدیگرند؛ در genome analysis برای مقایسهٔ DNA sequenceها با datasetهای بزرگ استفاده می‌شود.
- `Genome Database` — database تخصصی برای ذخیره و query کردن داده‌های genome؛ برای داده‌هایی طراحی شده که queryهای آن‌ها با relational، document یا graph database معمولی به‌خوبی پوشش داده نمی‌شود.
- `Full-Text Search` — جست‌وجو در متن بر اساس کلمه، عبارت یا الگوی زبانی؛ معمولاً با search index انجام می‌شود و در کنار database اصلی قرار می‌گیرد.
- `Information Retrieval` — حوزهٔ پیدا کردن اطلاعات مرتبط از میان مجموعه‌ای از documentها؛ مبنای فنی search engineها و systemهای جست‌وجوی متن است.

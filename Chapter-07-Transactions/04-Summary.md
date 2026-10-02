# Chapter 7 — Transactions

## Summary

Transactionها یک abstraction layer هستند که به application اجازه می‌دهند وانمود کند بعضی concurrency problemها و بعضی انواع hardware و software fault وجود ندارند. بخش بزرگی از errorها به یک transaction abort ساده تقلیل پیدا می‌کند و application فقط باید دوباره تلاش کند.

در این chapter exampleهای زیادی از problemهایی دیدیم که transactionها به جلوگیری از آن‌ها کمک می‌کنند. همهٔ applicationها در معرض همهٔ این problemها نیستند: applicationای با access pattern بسیار ساده، مانند applicationای که فقط یک record را read و write می‌کند، احتمالاً می‌تواند بدون transaction کار کند. اما برای access patternهای پیچیده‌تر، transactionها تعداد caseهای بالقوهٔ error را که باید دربارهٔ آن‌ها فکر کنید به‌شدت کاهش می‌دهند.

بدون transaction، scenarioهای مختلف error—از جمله crash کردن process، network interruption، power outage، پر شدن disk و concurrency غیرمنتظره—می‌توانند data را به روش‌های مختلف inconsistent کنند. برای مثال، data denormalized به‌سادگی ممکن است با source data خود out of sync شود. بدون transaction، reasoning دربارهٔ اثر accessهای پیچیده و تعاملی روی database بسیار دشوار می‌شود.

در این chapter به‌طور ویژه روی concurrency control عمیق شدیم. چند isolation level پرکاربرد را بررسی کردیم؛ به‌خصوص read committed، snapshot isolation—که گاهی repeatable read نامیده می‌شود—و serializable. این isolation levelها را با بررسی exampleهای مختلف race condition توصیف کردیم:

**Dirty reads**

یک client writeهای client دیگری را پیش از commit شدن آن‌ها read می‌کند. Read committed و levelهای قوی‌تر از dirty read جلوگیری می‌کنند.

**Dirty writes**

یک client dataای را overwrite می‌کند که client دیگری آن را write کرده، اما هنوز commit نکرده است. تقریباً تمام implementationهای transaction از dirty write جلوگیری می‌کنند.

**Read skew (nonrepeatable reads)**

یک client بخش‌های مختلف database را در pointهای زمانی متفاوت می‌بیند. این مسئله معمولاً با snapshot isolation جلوگیری می‌شود؛ snapshot isolation اجازه می‌دهد transaction از یک snapshot consistent در یک point in time read کند و معمولاً با multi-version concurrency control (MVCC) implement می‌شود.

**Lost updates**

دو client به‌صورت concurrent یک read-modify-write cycle انجام می‌دهند. یکی write دیگری را بدون وارد کردن changeهای آن overwrite می‌کند و در نتیجه data از دست می‌رود. بعضی implementationهای snapshot isolation این anomaly را به‌صورت automatic prevent می‌کنند، درحالی‌که بعضی دیگر به lock دستی مانند `SELECT FOR UPDATE` نیاز دارند.

**Write skew**

یک transaction چیزی را read می‌کند، بر اساس value مشاهده‌شده تصمیم می‌گیرد و تصمیم خود را در database write می‌کند. بااین‌حال، تا زمان انجام write ممکن است premise تصمیم دیگر true نباشد. فقط serializable isolation از این anomaly جلوگیری می‌کند.

**Phantom reads**

یک transaction objectهایی را read می‌کند که با search condition مشخصی match می‌شوند. Client دیگری writeای انجام می‌دهد که روی result آن search اثر می‌گذارد. Snapshot isolation از phantom readهای ساده جلوگیری می‌کند، اما phantomها در context مربوط به write skew به treatment ویژه‌ای مانند index-range lock نیاز دارند.

Weak isolation levelها در برابر بعضی از این anomalyها محافظت می‌کنند، اما handling بعضی دیگر را به application developer واگذار می‌کنند؛ برای مثال، با استفاده از explicit locking. فقط serializable isolation در برابر تمام این issueها محافظت می‌کند. سه approach مختلف برای implement کردن serializable transactionها را بررسی کردیم:

**اجرای literal transactionها به‌ترتیب serial**

اگر بتوانید هر transaction را بسیار سریع اجرا کنید و transaction throughput آن‌قدر پایین باشد که روی یک CPU core process شود، این approach گزینه‌ای ساده و effective است.

**Two-phase locking**

این approach برای چند دهه روش standard implement کردن serializability بود، اما بسیاری از applicationها به‌دلیل ویژگی‌های performance آن از استفاده‌اش اجتناب می‌کنند.

**Serializable snapshot isolation (SSI)**

Algorithmی نسبتاً جدید که بیشتر downsideهای approachهای قبلی را ندارد. SSI از approach optimistic استفاده می‌کند و به transactionها اجازه می‌دهد بدون block شدن ادامه دهند. وقتی transaction می‌خواهد commit شود، check می‌شود و اگر execution آن serializable نبوده باشد، abort می‌شود.

Exampleهای این chapter از relational data model استفاده می‌کردند. بااین‌حال، همان‌طور که در بخش «The Need for Multi-Object Transactions» در صفحهٔ ۲۳۱ گفتیم، transactionها بدون توجه به data model، feature ارزشمندی برای database هستند.

در این chapter، ideaها و algorithmها را عمدتاً در context مربوط به databaseای که روی یک machine اجرا می‌شود بررسی کردیم. Transactionها در distributed databaseها مجموعهٔ جدیدی از challengeهای دشوار ایجاد می‌کنند که در دو chapter بعدی دربارهٔ آن‌ها صحبت خواهیم کرد.

## Key Terms

- `Transaction` — abstractionای برای گروه‌بندی operationهای data و مدیریت آن‌ها به‌صورت یک واحد all-or-nothing.
- `ACID` — مجموعهٔ guaranteeهای Atomicity، Consistency، Isolation و Durability.
- `Isolation` — کنترل visibility و interference میان transactionهای concurrent.
- `Isolation Levels` — سطوح مختلف guarantee برای کنترل concurrency و anomalyهای transaction.
- `Read Committed` — isolation levelی که dirty read و dirty write را منع می‌کند.
- `Snapshot Isolation` — اجرای transaction روی snapshot consistent مربوط به شروع آن.
- `Serializability` — guarantee معادل بودن result transactionهای concurrent با اجرای serial آن‌ها.
- `Two-Phase Locking (2PL)` — روش locking برای دستیابی به serializability با shared و exclusive lock.
- `Serializable Snapshot Isolation (SSI)` — روش optimistic مبتنی بر snapshot isolation برای detect کردن serialization conflict.
- `Concurrency Control` — mechanismهای مدیریت اجرای هم‌زمان transactionها.
- `Lost Update` — از دست رفتن change یک transaction در اثر overwrite شدن توسط write concurrent.
- `Write Skew` — نقض invariant در اثر تصمیم‌گیری transaction بر اساس premiseای که دیگر معتبر نیست.
- `Phantom` — تغییر result یک search query در اثر write transaction دیگر.
- `Deadlock` — انتظار چرخه‌ای transactionها برای lockهای یکدیگر.
- `Atomicity` — guarantee اجرای all-or-nothing transaction.
- `Durability` — guarantee باقی ماندن data پس از commit و failure.

# Chapter 9 — Consistency and Consensus

## Summary

در این chapter، consistency و consensus را از چند زاویه بررسی کردیم. ابتدا به linearizability پرداختیم؛ consistency model پرکاربردی که می‌خواهد data تکثیرشده طوری به‌نظر برسد که انگار فقط یک copy از آن وجود دارد و همهٔ operationها روی همان copy به‌شکل اتمیک اجرا می‌شوند. Linearizability به‌سادگی قابل‌فهم است—database مانند variableای در یک program تک‌نخی رفتار می‌کند—اما به‌خصوص در محیط‌هایی با network delay زیاد، کند است.

همچنین causality را بررسی کردیم که بر اساس رابطهٔ علت و معلول، برای eventهای system ترتیب تعیین می‌کند. برخلاف linearizability که همهٔ operationها را در یک timeline با total order قرار می‌دهد، causality consistency model ضعیف‌تری می‌دهد: بعضی eventها می‌توانند concurrent باشند و تاریخچهٔ versionها شبیه timelineای شاخه‌دار و ادغام‌شونده می‌شود. Causal consistency سربار coordination مربوط به linearizability را ندارد و در برابر مشکل‌های network حساسیت کمتری نشان می‌دهد.

بااین‌حال، حتی ثبت causal order—مثلاً با Lamport timestamp—برای پیاده‌سازی همه‌چیز کافی نیست. تضمین یکتایی username و رد کردن ثبت‌نام‌های هم‌زمان با نام یکسان، نمونه‌ای از این محدودیت است. اگر nodeای قرار است ثبت‌نامی را بپذیرد، باید به‌نحوی مطمئن شود node دیگری هم‌زمان همان نام را ثبت نمی‌کند. این مسئله ما را به consensus رساند.

رسیدن به consensus یعنی تصمیمی گرفته شود که همهٔ nodeها بر سر آن توافق کنند و آن تصمیم برگشت‌ناپذیر باشد. مسئله‌های گوناگونی را می‌توان به consensus تقلیل داد و از نظر حل‌پذیری معادل آن دانست؛ یعنی راه‌حل یکی را می‌توان به راه‌حل دیگری تبدیل کرد. از جمله:

- **Linearizable compare-and-set register:** register باید با توجه به برابر بودن مقدار فعلی با پارامتر operation، به‌صورت اتمیک تصمیم بگیرد value را تغییر دهد یا نه.
- **Atomic transaction commit:** database باید تصمیم بگیرد distributed transaction commit شود یا abort.
- **Total order broadcast:** messaging system باید ترتیب تحویل messageها را تعیین کند.
- **Lock و lease:** وقتی چند client برای گرفتن lock یا lease رقابت می‌کنند، باید مشخص شود کدام‌یک موفق شده است.
- **Membership/coordination service:** با داشتن failure detectorای مانند timeout، باید دربارهٔ زنده‌بودن nodeها و مرده محسوب‌شدن nodeهایی که sessionشان timeout شده تصمیم گرفت.
- **Uniqueness constraint:** وقتی چند transaction هم‌زمان می‌خواهند recordهای متعارض با key یکسان بسازند، باید مشخص شود کدام‌یک مجاز است و کدام‌یک constraint violation می‌گیرد.

تمام این مسئله‌ها با یک node ساده‌اند، یا وقتی تصمیم‌گیری را به یک node واگذار کنیم. در single-leader database، اختیار تصمیم‌گیری به leader سپرده می‌شود؛ به همین دلیل چنین databaseای می‌تواند operationهای linearizable، uniqueness constraint و replication log با total order فراهم کند.

اما اگر leader از کار بیفتد یا network interruption دسترسی به آن را قطع کند، system دیگر نمی‌تواند پیشرفت کند. سه راه برای مواجهه با این وضعیت وجود دارد:

1. منتظر بمانیم leader recover شود و در این فاصله بپذیریم system block است. بسیاری از XA/JTA transaction coordinatorها این روش را انتخاب می‌کنند. این راه consensus را به‌طور کامل حل نمی‌کند، چون property مربوط به termination را ندارد: اگر leader برنگردد، system می‌تواند برای همیشه متوقف بماند.
2. انسان‌ها دستی failover کنند، leader node جدیدی برگزینند و system را برای استفاده از آن پیکربندی کنند. بسیاری از relational databaseها چنین می‌کنند. این نوعی تصمیم‌گیری با دخالت عامل بیرونی است: operator انسانی، خارج از computer system، تصمیم را می‌گیرد. سرعت failover به سرعت عمل انسان محدود است و معمولاً از computer کندتر است.
3. از algorithmی برای انتخاب خودکار leader جدید استفاده کنیم. این روش به consensus algorithm نیاز دارد و بهتر است از algorithm اثبات‌شده‌ای استفاده شود که شرایط نامساعد network را درست مدیریت می‌کند.

Single-leader database می‌تواند بدون اجرای consensus روی هر write، linearizability فراهم کند؛ اما برای حفظ رهبری و تغییر leader همچنان به consensus نیاز دارد. به‌بیان دیگر، داشتن leader فقط مسئله را عقب می‌اندازد: consensus همچنان لازم است، اما در جای دیگری و با دفعات کمتر. خوشبختانه algorithmها و systemهای fault-tolerant برای consensus وجود دارند و در این chapter با آن‌ها آشنا شدیم.

ابزارهایی مانند ZooKeeper امکان واگذاری بخشی از consensus، failure detection و membership service را به یک سرویس بیرونی فراهم می‌کنند. استفاده از آن ساده نیست، اما از پیاده‌سازی الگوریتم‌های خودمان برای تحمل تمام دشواری‌های فصل ۸ بهتر است. اگر مسئله‌ای دارید که به consensus تقلیل‌پذیر است و باید fault-tolerant باشد، استفاده از ابزاری مانند ZooKeeper انتخاب مناسب‌تری است.

بااین‌حال، همهٔ systemها الزاماً به consensus نیاز ندارند. برای نمونه، systemهای leaderless و multi-leader replication معمولاً consensus سراسری ندارند. conflictهای این systemها پیامد نبود consensus میان leaderهای مختلف‌اند؛ شاید این قابل‌قبول باشد و لازم باشد به‌جای linearizability، با dataای کار کنیم که تاریخچهٔ versionهایش شاخه می‌زند و دوباره merge می‌شود.

در این chapter به پژوهش‌های نظری گسترده‌ای دربارهٔ distributed systemها ارجاع دادیم. مقاله‌ها و proofهای نظری همیشه آسان‌فهم نیستند و گاهی فرض‌های غیرواقع‌بینانه دارند، اما برای کار عملی در این حوزه بسیار ارزشمندند: کمک می‌کنند دربارهٔ آنچه شدنی و ناشدنی است استدلال کنیم و راه‌های خلاف‌انتظار failure در distributed systemها را پیدا کنیم. اگر فرصت دارید، مرور منابع این chapter ارزشمند است.

با اینجا Part II کتاب—شامل replication، partitioning، transactionها، مدل‌های failure در distributed systemها و در نهایت consistency و consensus—به پایان می‌رسد. در Part III، پس از این بنیان نظری، دوباره به systemهای عملی بازمی‌گردیم و می‌بینیم چگونه می‌توان از building blockهای ناهمگون applicationهای قدرتمند ساخت.

## Key Terms

- `Consistency` — مجموعهٔ قواعدی که تعیین می‌کند clientها چه state و ترتیبی را از data مشاهده کنند.
- `Linearizability` — guaranteeای که operationها را در یک ترتیب واحد و سازگار با real time قرار می‌دهد؛ درک آن ساده است، اما می‌تواند در networkهای کند هزینهٔ performance داشته باشد.
- `Causality` — رابطهٔ علت‌ومعلولی میان eventها که ترتیب لازم را بدون مرتب‌کردن اجباری eventهای concurrent مشخص می‌کند.
- `Consensus` — توافق nodeها بر یک تصمیم برگشت‌ناپذیر؛ بسیاری از primitiveها و capabilityهای distributed system را می‌توان به آن تقلیل داد.
- `Consensus-Equivalent Problem` — مسئله‌ای مانند atomic commit، total order broadcast یا leader election که حل آن را می‌توان به حل consensus یا مسئلهٔ معادل دیگری تبدیل کرد.
- `Single-Leader Replication` — الگویی که در آن leader ترتیب writeها را تعیین می‌کند؛ برای تغییر یا بازیابی leader همچنان به consensus نیاز دارد.
- `Membership and Coordination Service` — سرویسی مانند ZooKeeper که هماهنگی، failure detection و تعیین اعضای فعال cluster را به applicationها واگذار می‌کند.

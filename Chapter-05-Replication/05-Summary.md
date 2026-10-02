# Chapter 5 — Replication

## Summary

در این chapter مسئلهٔ replication را بررسی کردیم. Replication می‌تواند چند هدف مهم داشته باشد:

- **High availability**: system حتی وقتی یک machine، چند machine یا حتی یک datacenter کامل از کار می‌افتد، به کار خود ادامه دهد.
- **Disconnected operation**: application بتواند هنگام قطع شدن network همچنان به کار ادامه دهد.
- **Latency**: data در موقعیت جغرافیایی نزدیک به userها قرار بگیرد تا interaction آن‌ها با system سریع‌تر باشد.
- **Scalability**: system بتواند حجم read بیشتری از ظرفیت یک machine را مدیریت کند؛ برای مثال با انجام readها روی replicaها.

با اینکه هدف replication ساده به نظر می‌رسد—نگه‌داشتن copy یکسانی از data روی چند machine—در عمل مسئله‌ای بسیار پیچیده است. برای طراحی درست باید با دقت به concurrency و تمام چیزهایی فکر کنیم که ممکن است اشتباه پیش بروند، و پیامدهای این failureها را مدیریت کنیم. دست‌کم باید با unavailable شدن nodeها و network interruptionها کنار بیاییم؛ آن هم بدون در نظر گرفتن failureهای پنهان‌تری مانند silent data corruption ناشی از software bugها.

سه approach اصلی برای replication را بررسی کردیم:

**Single-leader replication**

Clientها تمام writeها را به یک node واحد، یعنی leader، می‌فرستند. Leader جریان eventهای مربوط به data change را برای nodeهای دیگر، یعنی followerها، ارسال می‌کند. Readها می‌توانند از هر replica انجام شوند، اما read از followerها ممکن است stale باشد.

**Multi-leader replication**

Clientها هر write را به یکی از چند leader node می‌فرستند و هرکدام از این leaderها می‌توانند write را بپذیرند. Leaderها streamهای مربوط به data change را برای یکدیگر و برای follower nodeهای احتمالی ارسال می‌کنند.

**Leaderless replication**

Clientها هر write را برای چند node می‌فرستند و از چند node به‌صورت parallel read می‌کنند تا nodeهایی را که data stale دارند detect و correct کنند.

هر approach مزایا و معایب خود را دارد. Single-leader replication محبوب است، چون درک آن نسبتاً ساده است و معمولاً نگرانی‌ای بابت conflict resolution وجود ندارد. Multi-leader و leaderless replication می‌توانند در برابر faulty nodeها، network interruptionها و latency spikeها robustتر باشند؛ اما در مقابل، reasoning دربارهٔ رفتار آن‌ها دشوارتر است و فقط consistency guaranteeهای بسیار ضعیفی ارائه می‌کنند.

Replication می‌تواند synchronous یا asynchronous باشد و این انتخاب هنگام رخ دادن failure اثر عمیقی بر رفتار system دارد. Asynchronous replication در شرایط عادی ممکن است سریع باشد، اما باید دقیقاً مشخص کنیم وقتی replication lag افزایش پیدا می‌کند و serverها fail می‌شوند چه اتفاقی رخ می‌دهد. اگر leader fail شود و followerای که به‌صورت asynchronous update شده به leader جدید تبدیل شود، ممکن است dataهایی که اخیراً commit شده‌اند از دست بروند.

همچنین effectهای عجیبی را که replication lag می‌تواند ایجاد کند بررسی کردیم و چند consistency model را برای تعیین رفتار application هنگام وجود replication lag دیدیم:

**Read-after-write consistency**

Userها باید همیشه dataای را که خودشان submit کرده‌اند مشاهده کنند.

**Monotonic reads**

پس از آنکه userها data را در یک نقطهٔ زمانی مشاهده کردند، نباید بعداً همان data را از نقطه‌ای قدیمی‌تر ببینند.

**Consistent prefix reads**

Userها باید data را در stateای ببینند که از نظر causal معنا داشته باشد؛ برای مثال، یک question و reply آن باید به ترتیب درست نمایش داده شوند.

در پایان، مسئله‌های concurrency ذاتی در multi-leader و leaderless replication را بررسی کردیم: چون این approachها اجازه می‌دهند چند write به‌صورت concurrent رخ دهد، conflict ممکن است ایجاد شود. Algorithmی را بررسی کردیم که database می‌تواند با استفاده از آن تعیین کند آیا یک operation پیش از operation دیگر رخ داده است یا هر دو operation concurrent بوده‌اند. همچنین با روش‌هایی برای resolve کردن conflict از طریق merge کردن updateهای concurrent آشنا شدیم.

در chapter بعد، بررسی dataای را که میان چند machine توزیع شده است ادامه می‌دهیم؛ این بار با counterpart مربوط به replication، یعنی تقسیم کردن یک dataset بزرگ به partitionها.

## Key Terms

- `Replication` — نگه‌داری copyهای یکسان از data روی چند machine یا node برای بهبود availability، latency یا read scalability.
- `Single-Leader Replication` — مدلی که تمام writeها را به یک leader می‌فرستد و changeها را برای followerها replicate می‌کند.
- `Multi-Leader Replication` — مدلی که چند leader می‌توانند write بپذیرند و changeهای خود را میان یکدیگر replicate کنند.
- `Leaderless Replication` — مدلی که client write را برای چند node می‌فرستد و با read از چند node، stale data را detect و correct می‌کند.
- `Replication Lag` — فاصلهٔ میان state جدیدتر و state عقب‌ماندهٔ replicaها.
- `Eventual Consistency` — مدلی که replicaها در نهایت به state سازگار می‌رسند، بدون اینکه convergence فوری تضمین شود.
- `Read-After-Write Consistency` — guarantee مشاهدهٔ dataای که user خودش submit کرده است.
- `Monotonic Reads` — guarantee اینکه user پس از مشاهدهٔ state جدید، دوباره state قدیمی‌تر را نبیند.
- `Consistent Prefix Reads` — guarantee مشاهدهٔ writeهای مرتبط در ترتیبی که از نظر causal معنادار باشد.
- `Quorum` — حداقل تعداد responseهای replica که برای معتبر دانستن یک read یا write لازم است.
- `Conflict Resolution` — انتخاب یا merge کردن versionهای متناقض برای رسیدن به state مشترک.
- `Causality` — رابطهٔ علت و معلولی میان eventها که ترتیب منطقی مشاهده یا apply شدن آن‌ها را مشخص می‌کند.
- `Failover` — انتقال نقش leader به replicaی دیگر پس از failure برای حفظ availability.
- `Fault Tolerance` — توانایی system برای ادامهٔ کار با وجود برخی faultها و failureهای داخلی.

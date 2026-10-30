# Chapter 9 — Consistency and Consensus

## Consistency Guarantees

در بخش «Problems with Replication Lag» در صفحهٔ ۱۶۱، بعضی مسئله‌های زمانی را در replicated database بررسی کردیم. اگر در یک لحظهٔ واحد به دو node از database نگاه کنید، احتمالاً data متفاوتی می‌بینید؛ چون write requestها در زمان‌های متفاوت به nodeهای مختلف می‌رسند. این ناسازگاری‌ها مستقل از روش replication database رخ می‌دهند؛ چه single-leader باشد، چه multi-leader یا leaderless.

بیشتر replicated databaseها دست‌کم **eventual consistency** را فراهم می‌کنند. یعنی اگر نوشتن در database را متوقف کنید و مدتی نامشخص منتظر بمانید، سرانجام تمام read requestها یک value یکسان برمی‌گردانند [1]. به‌عبارت دیگر، ناسازگاری موقتی است و در نهایت برطرف می‌شود؛ البته به این فرض که faultهای network نیز سرانجام رفع شوند. شاید نام دقیق‌تری برای eventual consistency، **convergence** باشد؛ چون انتظار داریم تمام replicaها نهایتاً به یک value واحد برسند [2].

بااین‌حال، این guarantee بسیار ضعیف است و دربارهٔ زمان convergence چیزی نمی‌گوید. تا پیش از آن، readها ممکن است هر چیزی یا حتی هیچ چیزی برنگردانند [1]. برای مثال، اگر valueای بنویسید و بلافاصله آن را بخوانید، تضمینی نیست همان valueای را ببینید که نوشته‌اید؛ چون ممکن است read به replica دیگری route شود. به بخش «Reading Your Own Writes» در صفحهٔ ۱۶۲ مراجعه کنید.

Eventual consistency برای application developerها دشوار است، چون رفتار آن با variable در یک program معمولی single-threaded تفاوت زیادی دارد. اگر valueای را به variable نسبت دهید و کمی بعد آن را بخوانید، انتظار ندارید value قبلی را ببینید یا read با failure مواجه شود. Database در ظاهر شبیه variableای است که می‌توان از آن read کرد و در آن write کرد، اما semantics آن در واقع بسیار پیچیده‌تر است [3].

هنگام کار با databaseای که فقط guaranteeهای ضعیف ارائه می‌دهد، باید همیشه محدودیت‌های آن را در نظر داشته باشید و ناخواسته فرض بیش از حد نکنید. Bugها اغلب subtle هستند و با test به‌سختی پیدا می‌شوند، چون application ممکن است بیشتر اوقات درست کار کند. حالت‌های مرزی eventual consistency فقط هنگام رخ دادن fault در system—مثلاً قطع network—یا در شرایط concurrency بالا آشکار می‌شوند.

در این chapter، consistency modelهای قوی‌تری را بررسی می‌کنیم که data systemها می‌توانند ارائه دهند. این guaranteeها رایگان نیستند: systemهایی با guarantee قوی‌تر ممکن است performance پایین‌تری داشته باشند یا نسبت به systemهایی با guarantee ضعیف‌تر، fault tolerance کمتری ارائه کنند. بااین‌حال، guaranteeهای قوی‌تر جذاب‌اند، چون استفادهٔ درست از آن‌ها آسان‌تر است. پس از دیدن چند consistency model متفاوت، بهتر می‌توانید تشخیص دهید کدام‌یک با نیازهایتان سازگارتر است.

میان consistency modelهای distributed و سلسله‌مراتب transaction isolation levelهایی که پیش‌تر بررسی کردیم شباهت‌هایی وجود دارد [4, 5]؛ به بخش «Weak Isolation Levels» در صفحهٔ ۲۳۳ مراجعه کنید. بااین‌حال، با وجود مقداری هم‌پوشانی، این‌ها عمدتاً دو دغدغهٔ مستقل‌اند: transaction isolation بیشتر به جلوگیری از race conditionهای ناشی از اجرای هم‌زمان transactionها می‌پردازد؛ درحالی‌که distributed consistency بیشتر دربارهٔ هماهنگ کردن state replicaها در برابر delay و fault است.

این chapter موضوع‌های گسترده‌ای را پوشش می‌دهد، اما همان‌طور که خواهیم دید، این حوزه‌ها پیوند عمیقی با هم دارند:

- ابتدا یکی از قوی‌ترین consistency modelهای رایج، یعنی linearizability، و مزایا و معایب آن را بررسی می‌کنیم.
- سپس به مسئلهٔ ترتیب eventها در distributed system می‌پردازیم؛ به‌ویژه causality و total ordering. این موضوع در بخش «Ordering Guarantees» در صفحهٔ ۳۳۹ آمده است.
- در بخش سوم، «Distributed Transactions and Consensus» در صفحهٔ ۳۵۲، روش commit اتمیک distributed transaction را بررسی می‌کنیم که در نهایت ما را به راه‌حل‌های مسئلهٔ consensus می‌رساند.

## Key Terms

- `Consistency Guarantee` — قاعده‌ای که مشخص می‌کند clientها پس از read و write چه state یا ترتیبی را می‌توانند مشاهده کنند.
- `Consistency Model` — مجموعه‌ای از semantics و guaranteeها که رفتار read و write را در system توصیف می‌کند.
- `Eventual Consistency` — guarantee ضعیفی که می‌گوید پس از توقف writeها و رفع faultها، replicaها نهایتاً به value یکسان می‌رسند، اما زمان convergence را مشخص نمی‌کند.
- `Convergence` — رسیدن replicaها به state یا value یکسان پس از آنکه تغییرات متوقف و faultها رفع شوند.
- `Stale Read` — خواندن value قدیمی از replicaای که هنوز write جدید را دریافت نکرده است.
- `Strong Consistency` — guarantee قوی‌تری که رفتار سیستم را برای clientها قابل‌پیش‌بینی‌تر می‌کند، اما ممکن است هزینهٔ performance یا fault tolerance داشته باشد.
- `Transaction Isolation` — guaranteeهایی که کنترل می‌کنند transactionهای هم‌زمان چگونه بر read و write یکدیگر اثر بگذارند؛ دغدغه‌ای جدا از هماهنگی replicaها در distributed consistency.

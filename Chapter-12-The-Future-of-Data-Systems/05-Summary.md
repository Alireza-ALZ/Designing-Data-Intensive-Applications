# Chapter 12 — The Future of Data Systems

## Summary

در این chapter رویکردهای تازه‌ای برای طراحی data systemها را بررسی کردیم و نظرها و گمانه‌زنی‌های شخصی‌ام دربارهٔ آینده را هم مطرح کردم. بحث را با این مشاهده آغاز کردیم که هیچ ابزار واحدی نمی‌تواند همهٔ use caseهای ممکن را به‌طور کارآمد پشتیبانی کند؛ بنابراین applicationها برای رسیدن به هدف‌هایشان ناچارند چند نرم‌افزار متفاوت را با هم ترکیب کنند. دیدیم می‌توان مسئلهٔ data integration را با batch processing و event streamها حل کرد تا data changeها میان systemهای مختلف جریان پیدا کنند.

در این رویکرد، بعضی systemها به‌عنوان system of record تعیین می‌شوند و dataهای دیگر با transformation از آن‌ها مشتق می‌شوند. به این ترتیب می‌توان indexها، materialized viewها، machine learning modelها، خلاصه‌های آماری و موارد دیگر را نگهداری کرد. اگر این derivation و transformationها را asynchronous و با coupling سست انجام دهیم، مشکل در یک بخش به قسمت‌های نامرتبط system سرایت نمی‌کند و در نتیجه کل system robustتر و fault-tolerantتر می‌شود.

نمایش dataflow به‌صورت transformation از یک dataset به dataset دیگر، تکامل application را هم آسان می‌کند: اگر بخواهید یکی از مرحله‌های processing را تغییر دهید ــ مثلاً ساختار index یا cache را عوض کنید ــ کافی است code transformation تازه را روی کل input dataset دوباره اجرا کنید تا output از نو مشتق شود. به همین شکل، اگر مشکلی پیش آمد، می‌توانید code را اصلاح و data را دوباره پردازش کنید تا بازیابی شود. این فرایندها به کارهایی شباهت دارند که databaseها از قبل درون خود انجام می‌دهند؛ بنابراین ایدهٔ applicationهای dataflow را به‌صورت بازکردن componentهای database و ساخت application از راه ترکیب componentهای loosely coupled صورت‌بندی کردیم.

می‌توان derived state را با مشاهدهٔ changeهای data زیربنایی به‌روز کرد؛ خود derived state نیز می‌تواند در دسترس consumerهای پایین‌دستی قرار گیرد تا changeهایش را مشاهده کنند. حتی می‌توان این dataflow را تا device کاربر نهایی که data را نمایش می‌دهد ادامه داد و UIهایی ساخت که با تغییر data به‌طور پویا به‌روز شوند و در حالت offline هم به کار ادامه دهند.

سپس بررسی کردیم چگونه می‌توان در حضور fault مطمئن شد این پردازش‌ها درست باقی می‌مانند. دیدیم با event processing asynchronous نیز می‌توان integrity guaranteeهای قوی را در scale حفظ کرد: operation ID سرتاسری، operationها را idempotent می‌کند و constraintها را می‌توان asynchronous بررسی کرد. clientها می‌توانند تا موفق شدن بررسی منتظر بمانند، یا بدون انتظار ادامه دهند و احتمال عذرخواهی به‌خاطر نقض constraint را بپذیرند. این رویکرد از روش سنتی distributed transactionها مقیاس‌پذیرتر و robustتر است و با شیوهٔ کار بسیاری از فرایندهای کسب‌وکار نیز سازگاری دارد.

با ساخت applicationها حول dataflow و بررسی asynchronous constraintها می‌توان از بیشتر coordinationها دوری کرد و systemهایی ساخت که integrity را حفظ می‌کنند و با وجود faultها، حتی در محیط‌های geographically distributed، performance خوبی دارند. همچنین دربارهٔ استفاده از audit برای بررسی integrity data و تشخیص corruption صحبت کردیم.

در پایان، کمی عقب‌تر ایستادیم و جنبه‌های اخلاقی ساخت applicationهای data-intensive را بررسی کردیم. دیدیم data می‌تواند برای کارهای خوب به کار رود، اما آسیب‌های جدی هم به بار آورد: تصمیم‌هایی را توجیه کند که عمیقاً بر زندگی مردم اثر می‌گذارند و اعتراض به آن‌ها دشوار است؛ به تبعیض و بهره‌کشی بینجامد؛ surveillance را عادی کند؛ و اطلاعات خصوصی را آشکار سازد. خطر data breach نیز وجود دارد و ممکن است استفاده‌ای که با نیت خیر از data می‌شود پیامدهای ناخواسته داشته باشد.

از آنجا که نرم‌افزار و data اثر بزرگی بر جهان دارند، ما مهندسان باید به یاد داشته باشیم مسئولیم برای ساخت جهانی تلاش کنیم که خودمان می‌خواهیم در آن زندگی کنیم؛ جهانی که با انسان‌ها انسانی و محترمانه رفتار می‌کند. امیدوارم بتوانیم برای رسیدن به این هدف با هم همکاری کنیم.

## Key Terms

- `System of Record` — منبع معتبر اصلی data که derived data و representationهای دیگر از آن ساخته می‌شوند.
- `Dataflow` — جریان تغییرهای data میان systemها و componentها، از مبدأ تا مصرف‌کننده.
- `Derived Data` — dataای مانند index، materialized view یا model که با transformation از منبع اصلی تولید می‌شود.
- `Integrity` — حفظ کامل‌بودن و درستی data، حتی هنگام fault و پردازش asynchronous.
- `Idempotence` — خاصیتی که باعث می‌شود retry یک operation، effect آن را دوباره اعمال نکند.
- `Auditing` — بررسی data برای کشف corruption یا نقض integrity.
- `Data Integration` — ترکیب systemهای متفاوت و هماهنگ‌کردن dataflow میان آن‌ها.


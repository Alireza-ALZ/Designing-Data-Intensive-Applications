# Chapter 6 — Partitioning

## Summary

در این chapter روش‌های مختلف partition کردن یک dataset بزرگ به subsetهای کوچک‌تر را بررسی کردیم. وقتی data آن‌قدر زیاد است که ذخیره و پردازش آن روی یک machine دیگر عملی نیست، partitioning ضروری می‌شود.

هدف partitioning این است که data و query load را به‌صورت یکنواخت میان چند machine توزیع کنیم و از ایجاد hot spot—یعنی nodeهایی با load نامتناسباً زیاد—جلوگیری کنیم. برای رسیدن به این هدف باید partitioning schemeای متناسب با data خود انتخاب کنید و هر زمان nodeهایی به cluster اضافه یا از آن حذف می‌شوند، partitionها را rebalancing کنید.

دو approach اصلی برای partitioning را بررسی کردیم:

- **Key-range partitioning**: keyها مرتب می‌شوند و هر partition مالک تمام keyهایی است که از یک minimum مشخص تا یک maximum مشخص قرار دارند. مرتب بودن keyها این مزیت را دارد که range queryهای efficient امکان‌پذیر می‌شوند، اما اگر application اغلب به keyهایی دسترسی داشته باشد که در sort order به یکدیگر نزدیک‌اند، خطر ایجاد hot spot وجود دارد. در این approach معمولاً وقتی یک partition بیش از حد بزرگ می‌شود، range آن به دو subrange split می‌شود و partitionها به‌صورت dynamic rebalanced می‌شوند.
- **Hash partitioning**: یک hash function روی هر key اعمال می‌شود و هر partition مالک rangeای از hashهاست. این روش order مربوط به keyها را از بین می‌برد و range queryها را inefficient می‌کند، اما ممکن است load را یکنواخت‌تر توزیع کند. هنگام استفاده از hash partitioning معمول است که تعداد ثابتی partition از ابتدا ایجاد شود، چند partition به هر node assign شود و هنگام اضافه یا حذف شدن nodeها، partitionهای کامل از یک node به node دیگر منتقل شوند. Dynamic partitioning نیز در این approach ممکن است.

Hybrid approachها نیز ممکن‌اند؛ برای مثال، می‌توان از compound key استفاده کرد و یک بخش key را برای تعیین partition و بخش دیگر را برای sort order به‌کار برد.

همچنین interaction میان partitioning و secondary indexها را بررسی کردیم. یک secondary index نیز باید partition شود و برای این کار دو روش وجود دارد:

- **Document-partitioned indexها (local indexها)**: secondary indexها در همان partitionای ذخیره می‌شوند که primary key و value در آن قرار دارند. در نتیجه هنگام write فقط یک partition باید update شود، اما read کردن از secondary index به scatter/gather میان تمام partitionها نیاز دارد.
- **Term-partitioned indexها (global indexها)**: secondary indexها به‌صورت جداگانه و بر اساس valueهای indexشده partition می‌شوند. یک entry در secondary index ممکن است recordهایی از تمام partitionهای primary key را شامل شود. وقتی documentای write می‌شود، چند partition از secondary index باید update شوند؛ بااین‌حال، read می‌تواند از یک partition واحد پاسخ داده شود.

در پایان، تکنیک‌های route کردن queryها به partition مناسب را بررسی کردیم؛ این تکنیک‌ها از partition-aware load balancing ساده تا sophisticatedترین parallel query execution engineها را شامل می‌شوند.

از نظر design، هر partition تا حد زیادی مستقل از partitionهای دیگر عمل می‌کند و همین ویژگی است که اجازه می‌دهد یک partitioned database روی چند machine scale شود. بااین‌حال، reasoning دربارهٔ operationهایی که باید روی چند partition write انجام دهند دشوار است. برای مثال، اگر write روی یک partition موفق شود اما روی partition دیگری fail شود، چه اتفاقی باید رخ دهد؟ در chapterهای بعدی به این سؤال خواهیم پرداخت.

## Key Terms

- `Partitioning` — تقسیم dataset بزرگ به بخش‌های کوچک‌تر برای توزیع data و query load میان چند machine.
- `Key-Range Partitioning` — partition کردن بر اساس range مرتب‌شدهٔ keyها، با امکان اجرای efficient range query.
- `Hash Partitioning` — partition کردن بر اساس range مقدار hash key برای توزیع یکنواخت‌تر load.
- `Secondary Index` — index اضافی برای search بر اساس fieldها یا valueهایی غیر از primary key.
- `Rebalancing` — جابه‌جا کردن partitionها و load میان nodeها پس از تغییر cluster.
- `Request Routing` — route کردن request یا query به partition و node مناسب.
- `Query Routing` — تعیین محل اجرای query بر اساس partition assignment.
- `Scalability` — توانایی partitioned database برای افزایش ظرفیت با توزیع data و processing روی machineهای بیشتر.
- `Load Distribution` — توزیع متوازن data، read و write requestها میان nodeها.
- `Hot Spot` — node یا partitionی که load نامتناسباً زیادی دریافت می‌کند.
- `Fault Tolerance` — توانایی ادامهٔ کار با وجود failure برخی nodeها یا machineها.
- `Hybrid Partitioning` — ترکیب چند روش partitioning در یک key، مانند استفاده از بخشی از compound key برای partition و بخش دیگر برای sort order.

# Chapter 3 — Storage and Retrieval

## Summary

در این chapter تلاش کردیم بفهمیم databaseها چگونه storage و retrieval را مدیریت می‌کنند. وقتی data را در یک database ذخیره می‌کنید چه اتفاقی می‌افتد و database بعداً، زمانی که دوباره برای آن data query می‌زنید، چه کاری انجام می‌دهد؟

در سطحی کلی دیدیم که storage engineها به دو دستهٔ اصلی تقسیم می‌شوند: engineهایی که برای transaction processing (OLTP) و engineهایی که برای analytics (OLAP) optimize شده‌اند. access patternها در این دو use case تفاوت‌های بزرگی دارند:

- سیستم‌های OLTP معمولاً user-facing هستند؛ یعنی ممکن است حجم بسیار زیادی request دریافت کنند. برای مدیریت این load، applicationها معمولاً در هر query فقط تعداد کمی record را لمس می‌کنند. Application recordها را با استفاده از نوعی key درخواست می‌کند و storage engine برای پیدا کردن data مربوط به آن key از index استفاده می‌کند. در اینجا disk seek time اغلب bottleneck است.
- Data warehouseها و systemهای تحلیلی مشابه کمتر شناخته‌شده‌اند، چون عمدتاً business analystها، نه end userها، از آن‌ها استفاده می‌کنند. این systemها در مقایسه با OLTP queryهای بسیار کمتری را پردازش می‌کنند، اما هر query معمولاً demanding است و لازم دارد در زمان کوتاهی millions record scan شود. در اینجا disk bandwidth، نه seek time، اغلب bottleneck است و column-oriented storage به راه‌حلی increasingly popular برای این نوع workload تبدیل شده است.

در سمت OLTP، storage engineهایی از دو school of thought اصلی دیدیم:

- school مبتنی بر log که فقط append کردن به fileها و delete کردن fileهای obsolete را مجاز می‌داند و هرگز fileای را که write شده است update نمی‌کند. `Bitcask`، `SSTable`ها، `LSM-tree`ها، `LevelDB`، `Cassandra`، `HBase`، `Lucene` و موارد دیگر در این گروه قرار می‌گیرند.
- school مبتنی بر update-in-place که disk را مجموعه‌ای از pageهای fixed-size در نظر می‌گیرد که می‌توان آن‌ها را overwrite کرد. `B-tree`ها بزرگ‌ترین نمونهٔ این فلسفه هستند و در تمام relational databaseهای اصلی و همچنین بسیاری از databaseهای غیررابطه‌ای استفاده می‌شوند.

Log-structured storage engineها توسعه‌ای نسبتاً جدید هستند. ایدهٔ اصلی آن‌ها این است که به‌صورت systematic، writeهای random-access را روی disk به writeهای sequential تبدیل کنند. این کار با توجه به ویژگی‌های performance هارددیسک‌ها و SSDها، throughput بالاتری برای write فراهم می‌کند.

در ادامهٔ بررسی بخش OLTP، مرور کوتاهی بر indexing structureهای پیچیده‌تر و databaseهایی داشتیم که برای نگهداری تمام data در memory optimize شده‌اند. سپس از internals مربوط به storage engineها فاصله گرفتیم تا architecture سطح بالای یک data warehouse معمولی را بررسی کنیم. این زمینه نشان داد چرا workloadهای تحلیلی با OLTP تفاوت زیادی دارند: وقتی queryهای شما نیاز دارند تعداد زیادی row را به‌صورت sequential scan کنند، indexها اهمیت بسیار کمتری پیدا می‌کنند. در عوض، مهم می‌شود که data را به‌شکل بسیار compact encode کنیم تا مقدار dataای که query باید از disk بخواند به حداقل برسد. بررسی کردیم که column-oriented storage چگونه به رسیدن به این هدف کمک می‌کند.

اگر به‌عنوان application developer این دانش را دربارهٔ internals storage engineها داشته باشید، موقعیت بسیار بهتری برای تشخیص مناسب‌ترین tool برای application خود خواهید داشت. اگر لازم باشد tuning parameterهای یک database را تنظیم کنید، این درک به شما اجازه می‌دهد تصور کنید افزایش یا کاهش هر parameter چه اثری ممکن است داشته باشد.

این chapter نمی‌توانست شما را در tuning یک storage engine خاص expert کند، اما امیدواریم vocabulary و ایده‌های کافی در اختیارتان گذاشته باشد تا بتوانید documentation مربوط به database مورد انتخاب خود را درک کنید.

## Key Terms

- `Storage Engine` — بخش داخلی database که مسئول سازمان‌دهی storage و اجرای read و write است.
- `Log-Structured Storage` — رویکردی که writeها را به append کردن به log و ساخت fileهای جدید متکی می‌کند و fileهای قبلی را update نمی‌کند.
- `Update-in-Place Storage` — رویکردی که data را در pageهای fixed-size نگه می‌دارد و همان pageها را هنگام تغییر overwrite می‌کند.
- `Query Performance` — سرعت و هزینهٔ اجرای query، از جمله زمان پاسخ و resource مصرف‌شده.
- `Query Optimization` — انتخاب plan و روش اجرای مناسب برای کاهش هزینه و افزایش performance query.
- `Disk Seek Time` — زمان لازم برای رسیدن head دیسک به location موردنظر؛ در workloadهای random-access می‌تواند bottleneck باشد.
- `Disk Bandwidth` — نرخ انتقال data بین disk و memory؛ در queryهای تحلیلی و scanهای بزرگ اهمیت زیادی دارد.
- `OLTP` — workload تعاملی با requestهای زیاد و دسترسی به تعداد کمی record در هر query.
- `OLAP` — workload تحلیلی با queryهای سنگین که معمولاً تعداد زیادی record را scan و aggregate می‌کند.
- `LSM-Tree` — storage structure مبتنی بر writeهای sequential، فایل‌های sorted و compaction.
- `B-Tree` — index متوازن مبتنی بر page که از update-in-place برای lookup و range query استفاده می‌کند.
- `Column-Oriented Storage` — layoutای که data را بر اساس column ذخیره می‌کند تا فقط columnهای لازم query خوانده شوند.

# Chapter 11 — Stream Processing

## Summary

در این chapter دربارهٔ event streamها، کاربردهایشان و روش پردازش آن‌ها صحبت کردیم. stream processing از بعضی جهت‌ها بسیار شبیه batch processing در Chapter 10 است، با این تفاوت که به‌طور پیوسته روی streamهای نامحدود و پایان‌ناپذیر انجام می‌شود، نه روی inputای با اندازهٔ ثابت. از این دیدگاه، message brokerها و event logها معادل streamیِ filesystem هستند.

دو نوع message broker را با هم مقایسه کردیم:

**Message broker به سبک AMQP/JMS**

broker هر پیام را به یک consumer اختصاص می‌دهد و consumer پس از پردازش موفق، همان پیام را جداگانه acknowledgment می‌کند. broker پس از acknowledgment پیام را حذف می‌کند. این رویکرد برای RPCای asynchronous، مثلاً task queue، مناسب است؛ به‌ویژه وقتی ترتیب دقیق پردازش پیام‌ها اهمیت ندارد و نیازی نیست پس از پردازش دوباره پیام‌های قدیمی را بخوانیم.

**Message broker مبتنی بر log**

broker همهٔ پیام‌های هر partition را به یک consumer node اختصاص می‌دهد و آن‌ها را همیشه به همان ترتیب تحویل می‌دهد. partitioning امکان parallelism را فراهم می‌کند و consumer با checkpoint کردن offset آخرین پیام پردازش‌شده، میزان پیشرفتش را دنبال می‌کند. broker پیام‌ها را روی disk نگه می‌دارد؛ بنابراین در صورت نیاز می‌توان به عقب برگشت و پیام‌های قدیمی را دوباره خواند.

رویکرد مبتنی بر log به replication logهای databaseها و log-structured storage engineها شباهت دارد. این رویکرد به‌ویژه برای stream processing systemهایی مناسب است که input stream مصرف می‌کنند و state یا output stream مشتق‌شده می‌سازند.

streamها می‌توانند از جاهای مختلف بیایند: eventهای فعالیت user، sensorهایی که به‌طور دوره‌ای اندازه‌گیری می‌کنند و data feedهایی مانند data بازارهای مالی، همگی به‌طور طبیعی stream هستند. همچنین دیدیم writeهای database را هم می‌توان stream در نظر گرفت: changelog ــ یعنی تاریخچهٔ همهٔ تغییرهای database ــ را می‌توان به‌طور ضمنی با Change Data Capture یا به‌طور صریح با Event Sourcing ثبت کرد. Log compaction این امکان را می‌دهد که stream یک copy کامل از محتوای database را در خود نگه دارد.

نمایش database به‌شکل stream فرصت‌های قدرتمندی برای یکپارچه‌سازی systemها ایجاد می‌کند. می‌توان با مصرف log تغییرها و apply کردنشان در systemهای derived data، search index، cache و analytics systemها را پیوسته به‌روز نگه داشت. حتی می‌توان viewهای تازه‌ای از data موجود ساخت: از ابتدا شروع کرد و log تغییرها را تا زمان حال مصرف کرد.

قابلیت نگهداری state به‌شکل stream و replay کردن پیام‌ها، پایهٔ تکنیک‌هایی است که stream join و fault tolerance را در frameworkهای مختلف ممکن می‌کنند. از کاربردهای stream processing نیز گفتیم: جست‌وجوی patternهای event با Complex Event Processing، محاسبهٔ aggregationهای windowed با stream analytics و به‌روز نگه‌داشتن systemهای derived data با materialized viewها.

سپس دشواری‌های استدلال دربارهٔ زمان در stream processor را بررسی کردیم؛ از جمله تفاوت processing time با timestamp eventها و مسئلهٔ مدیریت eventهای عقب‌مانده‌ای که پس از تصور پایان window می‌رسند.

سه نوع join در stream processها را از هم متمایز کردیم:

- **Stream-stream join:** هر دو input، streamای از activity eventها هستند و join operator به‌دنبال eventهای مرتبطی می‌گردد که در یک window زمانی رخ داده‌اند؛ مثلاً دو action یک user که حداکثر ۳۰ دقیقه با هم فاصله دارند. اگر هدف پیدا کردن eventهای مرتبط در همان stream باشد، دو ورودی join می‌توانند در واقع یک stream واحد باشند؛ یعنی self-join.
- **Stream-table join:** یک input، streamای از activity eventها و input دیگر changelog یک database است. changelog یک copy محلی از database را به‌روز نگه می‌دارد. join operator برای هر activity event، database را query می‌کند و activity event غنی‌شده‌ای در output می‌گذارد.
- **Table-table join:** هر دو input streamهایی از changelogهای database هستند. هر تغییر در یک طرف با تازه‌ترین state طرف دیگر join می‌شود و نتیجه، stream تغییرهای materialized view حاصل از join دو table است.

در پایان، روش‌های fault tolerance و exactly-once semantics در stream processor را بررسی کردیم. مانند batch processing باید output ناقص taskهای شکست‌خورده را کنار بگذاریم؛ اما چون stream process درازمدت اجرا می‌شود و پیوسته output تولید می‌کند، نمی‌توانیم همهٔ outputها را صرفاً دور بریزیم. در عوض، می‌توان از سازوکارهای recovery دقیق‌تر، مبتنی بر microbatching، checkpointing، transaction یا writeهای idempotent استفاده کرد.

## Key Terms

- `AMQP/JMS-Style Message Broker` — brokerای که پیام‌ها را جداگانه به consumerها می‌دهد و پس از acknowledgment حذف می‌کند.
- `Log-Based Message Broker` — brokerای که پیام‌ها را در partitionهای پایدار نگه می‌دارد تا consumer ترتیبی بخواند و در صورت نیاز replay کند.
- `Stream-Stream Join` — تطبیق eventهای مرتبط از دو stream، معمولاً در یک window زمانی.
- `Stream-Table Join` — پیوستن eventهای stream به state جاری یک table که می‌تواند با changelog به‌روز شود.
- `Table-Table Join` — نگهداری پیوستهٔ view حاصل از join دو table با اعمال changeهای هر دو طرف.
- `Exactly-Once Semantics` — تضمین effect قابل‌مشاهدهٔ یک‌باره برای هر input، حتی اگر task در عمل retry شود.
- `Microbatching` — تقسیم stream به batchهای کوچک برای اجرای دوره‌ای و recovery دقیق‌تر.
- `Idempotent Write` — writeای که اجرای دوباره‌اش همان state را باقی می‌گذارد و effect تکراری ایجاد نمی‌کند.

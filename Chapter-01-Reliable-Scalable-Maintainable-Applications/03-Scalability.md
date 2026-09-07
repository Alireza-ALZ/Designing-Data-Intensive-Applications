# Chapter 1 — Reliable, Scalable, and Maintainable Applications

## Scalability

حتی اگر سیستمی امروز reliable کار کند، این موضوع لزوماً به این معنا نیست که در آینده هم همین‌طور کار خواهد کرد. یکی از علت‌های رایج degradation، افزایش `load` است: شاید تعداد کاربران هم‌زمان سیستم از ۱۰٬۰۰۰ به ۱۰۰٬۰۰۰ رسیده باشد، یا از ۱ میلیون به ۱۰ میلیون افزایش یافته باشد. شاید سیستم اکنون حجم دادهٔ بسیار بزرگ‌تری نسبت به گذشته پردازش کند.

از واژهٔ `scalability` برای توصیف توانایی سیستم در کنار آمدن با افزایش load استفاده می‌کنیم. بااین‌حال، scalability برچسبی تک‌بعدی نیست که بتوانیم به یک سیستم بزنیم؛ گفتن اینکه «X scalable است» یا «Y scale نمی‌شود» معنای دقیقی ندارد. در عوض، بحث دربارهٔ scalability یعنی بررسی پرسش‌هایی مانند این‌ها: «اگر سیستم به شکل مشخصی رشد کند، چه گزینه‌هایی برای مدیریت این رشد داریم؟» و «چگونه می‌توانیم resourceهای محاسباتی اضافه کنیم تا load بیشتر را مدیریت کنیم؟»

### Describing Load

ابتدا باید load فعلی سیستم را به‌صورت مختصر توصیف کنیم؛ تنها پس از آن می‌توانیم دربارهٔ پرسش‌های مربوط به رشد صحبت کنیم، مثلاً اینکه اگر load ما دو برابر شود چه اتفاقی خواهد افتاد. می‌توان load را با چند عدد توصیف کرد که آن‌ها را `load parameter` می‌نامیم. بهترین انتخاب parameterها به architecture سیستم شما بستگی دارد: ممکن است تعداد requestها در ثانیه برای یک web server، نسبت readها به writeها در یک database، تعداد userهای فعال هم‌زمان در یک chat room، `hit rate` یک cache یا چیز دیگری باشد. شاید برای شما حالت average اهمیت داشته باشد، یا شاید bottleneck عمدتاً تحت تأثیر تعداد کمی از حالت‌های extreme باشد.

برای ملموس‌تر شدن این ایده، Twitter را به‌عنوان مثال در نظر بگیرید و از داده‌هایی استفاده کنیم که در نوامبر ۲۰۱۲ منتشر شده‌اند [16]. دو operation اصلی Twitter عبارت‌اند از:

**Post tweet**

کاربر می‌تواند message جدیدی برای followerهای خود منتشر کند؛ این operation به‌طور میانگین ۴٫۶k request در ثانیه و در peak بیش از ۱۲k request در ثانیه دارد.

**Home timeline**

کاربر می‌تواند tweetهایی را ببیند که افرادی که آن‌ها را follow می‌کند منتشر کرده‌اند؛ این operation ۳۰۰k request در ثانیه دارد.

پردازش سادهٔ ۱۲٬۰۰۰ write در ثانیه - نرخ peak برای post کردن tweet - نسبتاً آسان است. بااین‌حال، چالش اصلی Twitter در زمینهٔ scaling، حجم tweetها نیست؛ بلکه `fan-out` است [یادداشت ۱]. هر user افراد زیادی را follow می‌کند و افراد زیادی نیز هر user را follow می‌کنند. به‌طور کلی، دو روش برای پیاده‌سازی این دو operation وجود دارد:

1. هنگام post کردن tweet، tweet جدید را صرفاً در یک collection سراسری از tweetها insert کنید. وقتی user، home timeline خود را request می‌کند، همهٔ افرادی را که follow می‌کند پیدا کنید، tweetهای هرکدام را به دست آورید و آن‌ها را بر اساس زمان merge کنید. در یک relational database، مانند آنچه در شکل ۱-۲ می‌بینید، می‌توانید queryای شبیه این بنویسید:

```sql
SELECT tweets.*, users.* FROM tweets
  JOIN users ON tweets.sender_id      = users.id
  JOIN follows ON follows.followee_id = users.id
  WHERE follows.follower_id = current_user
```

2. برای home timeline هر user یک cache نگه دارید؛ چیزی شبیه mailboxای از tweetها برای هر user گیرنده، همان‌طور که در شکل ۱-۳ می‌بینید. وقتی user یک tweet post می‌کند، همهٔ افرادی را که آن user را follow می‌کنند پیدا کنید و tweet جدید را در cache مربوط به home timeline تک‌تک آن‌ها insert کنید. در این حالت، request برای خواندن home timeline ارزان است، چون نتیجه از قبل محاسبه شده است.

*شکل ۱-۲. یک relational schema ساده برای پیاده‌سازی home timeline در Twitter.*

*شکل ۱-۳. data pipeline مربوط به Twitter برای رساندن tweetها به followerها، همراه با load parameterهای نوامبر ۲۰۱۲ [16].*

نسخهٔ اول Twitter از روش ۱ استفاده می‌کرد، اما سیستم‌ها برای همگام ماندن با load مربوط به queryهای home timeline با مشکل روبه‌رو شدند؛ بنابراین شرکت به روش ۲ تغییر مسیر داد. این روش بهتر کار می‌کند، چون نرخ average انتشار tweet تقریباً دو order of magnitude کمتر از نرخ read کردن home timeline است. بنابراین در این مورد بهتر است در زمان write کار بیشتری انجام دهیم و در زمان read کار کمتری.

بااین‌حال، نقطه‌ضعف روش ۲ این است که post کردن یک tweet اکنون به کار اضافی زیادی نیاز دارد. هر tweet به‌طور میانگین برای حدود ۷۵ follower ارسال می‌شود؛ بنابراین ۴٫۶k tweet در ثانیه به ۳۴۵k write در ثانیه روی cacheهای home timeline تبدیل می‌شود. اما این average پنهان می‌کند که تعداد followerهای هر user می‌تواند بسیار متفاوت باشد و بعضی userها بیش از ۳۰ میلیون follower دارند. این یعنی یک tweet منفرد ممکن است به بیش از ۳۰ میلیون write در home timelineها منجر شود! انجام این کار در زمان مناسب - Twitter تلاش می‌کند tweetها را ظرف پنج ثانیه به followerها برساند - چالش قابل‌توجهی است.

در مثال Twitter، توزیع followerها برای هر user - که شاید بر اساس تعداد دفعات tweet کردن userها وزن‌دهی شده باشد - یک load parameter کلیدی برای بحث دربارهٔ scalability است، چون load مربوط به fan-out را تعیین می‌کند. application شما ممکن است ویژگی‌های کاملاً متفاوتی داشته باشد، اما می‌توانید از اصول مشابهی برای reasoning دربارهٔ load آن استفاده کنید.

پیچش نهایی داستان Twitter این است که اکنون، پس از آنکه روش ۲ به‌شکل robust پیاده‌سازی شده، Twitter به سمت ترکیبی از هر دو روش حرکت می‌کند. tweetهای بیشتر userها همچنان در لحظهٔ post شدن به home timelineها fan-out می‌شوند، اما تعداد کمی از userها که followerهای بسیار زیادی دارند - یعنی celebrityها - از این fan-out مستثنا می‌شوند. tweetهای celebrityهایی که ممکن است یک user آن‌ها را follow کند، جداگانه fetch می‌شوند و هنگام read شدن با home timeline آن user merge می‌شوند؛ مشابه روش ۱. این رویکرد hybrid می‌تواند performance را به‌طور پیوسته در سطح خوبی نگه دارد. در فصل ۱۲، پس از آنکه مباحث فنی بیشتری را پوشش دادیم، دوباره به این مثال برمی‌گردیم.

**یادداشت ۱ - fan-out:** این اصطلاح از electronic engineering گرفته شده است؛ جایی که تعداد inputهای logic gate متصل به output یک gate دیگر را توصیف می‌کند. output باید جریان کافی برای راه‌اندازی همهٔ inputهای متصل فراهم کند. در transaction processing systemها، از این اصطلاح برای توصیف تعداد requestهایی استفاده می‌کنیم که برای پاسخ دادن به یک request ورودی باید به serviceهای دیگر ارسال کنیم.

### Describing Performance

پس از آنکه load سیستم را توصیف کردید، می‌توانید بررسی کنید که با افزایش load چه اتفاقی می‌افتد. این موضوع را می‌توان از دو زاویه بررسی کرد:

- وقتی یک load parameter را افزایش می‌دهید و resourceهای سیستم - مانند CPU، memory و network bandwidth - را ثابت نگه می‌دارید، performance سیستم چه تغییری می‌کند؟
- وقتی یک load parameter را افزایش می‌دهید، اگر بخواهید performance ثابت بماند، resourceها را به چه میزان باید افزایش دهید؟

هر دو پرسش به عددهایی دربارهٔ performance نیاز دارند؛ بنابراین بیایید کوتاه بررسی کنیم که performance یک سیستم را چگونه توصیف می‌کنیم.

در یک `batch processing system` مانند `Hadoop`، معمولاً `throughput` برای ما مهم است؛ یعنی تعداد recordهایی که می‌توانیم در هر ثانیه پردازش کنیم، یا کل زمانی که اجرای یک job روی datasetی با اندازهٔ مشخص طول می‌کشد [یادداشت ۲]. در سیستم‌های `online processing`، آنچه معمولاً اهمیت بیشتری دارد `response time` سرویس است؛ یعنی زمانی که از ارسال یک request توسط client تا دریافت response طول می‌کشد.

**یادداشت ۲:** در یک دنیای ایده‌آل، زمان اجرای یک batch job برابر است با `dataset size / throughput`. در عمل، زمان اجرا اغلب طولانی‌تر است؛ به‌دلیل `skew` - یعنی داده به‌طور یکنواخت میان worker processها توزیع نشده است - و همچنین به‌دلیل نیاز به منتظر ماندن برای تکمیل کندترین task.

#### Latency and Response Time

واژه‌های `latency` و `response time` اغلب به‌صورت مترادف استفاده می‌شوند، اما یکسان نیستند. response time همان چیزی است که client می‌بیند: علاوه بر زمان واقعی پردازش request، یعنی `service time`، شامل delayهای network و `queueing delay` نیز می‌شود. latency مدت‌زمانی است که request در انتظار رسیدگی شدن است؛ یعنی زمانی که request در حالت انتظار برای دریافت service قرار دارد [17].

حتی اگر بارها و بارها دقیقاً یک request یکسان را ارسال کنید، response time در هر تلاش کمی متفاوت خواهد بود. در عمل، در سیستمی که انواع مختلف request را پردازش می‌کند، response time می‌تواند بسیار متغیر باشد. بنابراین نباید response time را یک عدد منفرد بدانیم؛ بلکه باید آن را توزیعی از مقدارهای قابل‌اندازه‌گیری در نظر بگیریم.

در شکل ۱-۴، هر نوار خاکستری نشان‌دهندهٔ یک request به service است و ارتفاع آن مدت‌زمان اجرای request را نشان می‌دهد. بیشتر requestها نسبتاً سریع‌اند، اما گاهی `outlier`هایی وجود دارند که بسیار بیشتر طول می‌کشند. شاید requestهای کند ذاتاً پرهزینه‌تر باشند؛ مثلاً چون دادهٔ بیشتری پردازش می‌کنند. اما حتی در شرایطی که تصور می‌کنید همهٔ requestها باید زمان یکسانی ببرند، باز هم variation خواهید داشت: `latency` اضافی و تصادفی ممکن است بر اثر context switch به یک background process، از دست رفتن یک network packet و `TCP retransmission`، توقف `garbage collection`، `page fault`ای که باعث خواندن از disk می‌شود، لرزش‌های مکانیکی در server rack [18] یا علت‌های متعدد دیگر ایجاد شود.

*شکل ۱-۴. نمایش mean و percentileها: response time مربوط به نمونه‌ای شامل ۱۰۰ request به یک service.*

رایج است که average response time یک service گزارش شود. از نظر دقیق، واژهٔ average به فرمول مشخصی اشاره نمی‌کند، اما در عمل معمولاً منظور `arithmetic mean` است: اگر n مقدار داشته باشیم، همهٔ مقدارها را با هم جمع می‌کنیم و حاصل را بر n تقسیم می‌کنیم. بااین‌حال، mean برای فهمیدن response time «معمول» معیار چندان خوبی نیست، چون نشان نمی‌دهد چند user واقعاً آن delay را تجربه کرده‌اند.

معمولاً استفاده از `percentile`ها بهتر است. اگر فهرست response timeها را از سریع‌ترین تا کندترین مرتب کنید، `median` نقطهٔ میانی است. برای مثال، اگر median response time برابر ۲۰۰ میلی‌ثانیه باشد، یعنی نیمی از requestها در کمتر از ۲۰۰ میلی‌ثانیه برمی‌گردند و نیم دیگر بیشتر از آن زمان طول می‌کشند.

این ویژگی median را به معیار خوبی برای فهمیدن زمان انتظار معمول userها تبدیل می‌کند: نیمی از requestهای user در کمتر از median response time پاسخ داده می‌شوند و نیم دیگر بیشتر طول می‌کشند. median را `50th percentile` نیز می‌نامند و گاهی به‌صورت `p50` کوتاه می‌کنند. توجه کنید که median به یک request منفرد اشاره دارد؛ اگر user چند request ارسال کند - در طول یک session یا چون چند resource در یک page وجود دارد - احتمال اینکه دست‌کم یکی از آن‌ها کندتر از median باشد، بسیار بیشتر از ۵۰ درصد است.

برای فهمیدن شدت outlierها، می‌توانید percentileهای بالاتر را بررسی کنید: percentileهای ۹۵، ۹۹ و ۹۹٫۹ رایج‌اند و به‌ترتیب به‌صورت `p95`، `p99` و `p999` کوتاه می‌شوند. این‌ها thresholdهای response time هستند که در آن‌ها ۹۵، ۹۹ یا ۹۹٫۹ درصد requestها سریع‌تر از threshold مربوطه‌اند. برای مثال، اگر 95th percentile response time برابر ۱٫۵ ثانیه باشد، یعنی ۹۵ request از هر ۱۰۰ request کمتر از ۱٫۵ ثانیه طول می‌کشند و ۵ request از هر ۱۰۰ request، ۱٫۵ ثانیه یا بیشتر زمان می‌برند. این موضوع در شکل ۱-۴ نشان داده شده است.

percentileهای بالای response time که `tail latency` نیز نامیده می‌شوند مهم‌اند، چون مستقیماً بر تجربهٔ user از service اثر می‌گذارند. برای مثال، Amazon نیازمندی‌های response time مربوط به serviceهای داخلی را با `99.9th percentile` توصیف می‌کند، حتی اگر این معیار فقط روی یک request از هر ۱٬۰۰۰ request اثر بگذارد. دلیلش این است که userهایی با کندترین requestها اغلب بیشترین داده را در account خود دارند، چون خریدهای زیادی انجام داده‌اند؛ یعنی ارزشمندترین customerها هستند [19]. حفظ رضایت این customerها با سریع نگه داشتن website برای آن‌ها اهمیت دارد: Amazon همچنین مشاهده کرده است که افزایش ۱۰۰ میلی‌ثانیه‌ای در response time، فروش را ۱ درصد کاهش می‌دهد [20] و گزارش‌های دیگر نشان می‌دهند که کند شدن یک‌ثانیه‌ای، یکی از metricهای رضایت customer را ۱۶ درصد کاهش می‌دهد [21, 22].

از سوی دیگر، بهینه‌سازی `99.99th percentile` - یعنی کندترین یک request از هر ۱۰٬۰۰۰ request - برای هدف‌های Amazon بیش از حد پرهزینه و کم‌فایده تشخیص داده شد. کاهش response time در percentileهای بسیار بالا دشوار است، چون این مقدارها به‌سادگی تحت تأثیر رویدادهای تصادفی خارج از کنترل شما قرار می‌گیرند و مزیت حاصل نیز به‌تدریج کاهش می‌یابد.

برای مثال، percentileها اغلب در `service level objective (SLO)` و `service level agreement (SLA)` استفاده می‌شوند؛ قراردادهایی که performance و availability مورد انتظار یک service را تعریف می‌کنند. ممکن است یک SLA اعلام کند service زمانی up محسوب می‌شود که median response time آن کمتر از ۲۰۰ میلی‌ثانیه و 99th percentile آن کمتر از ۱ ثانیه باشد؛ اگر response time طولانی‌تر شود، می‌توان service را عملاً down در نظر گرفت. همچنین ممکن است service ملزم باشد دست‌کم ۹۹٫۹ درصد مواقع up باشد. این metricها انتظارات clientهای service را مشخص می‌کنند و به customerها اجازه می‌دهند اگر SLA رعایت نشد، درخواست refund کنند.

در percentileهای بالا، `queueing delay` اغلب بخش بزرگی از response time را تشکیل می‌دهد. یک server فقط می‌تواند تعداد کمی کار را به‌صورت parallel پردازش کند؛ برای مثال، این تعداد ممکن است با تعداد coreهای CPU محدود شود. بنابراین فقط چند request کند کافی است تا پردازش requestهای بعدی متوقف یا معطل شود؛ اثری که گاهی `head-of-line blocking` نامیده می‌شود. حتی اگر پردازش requestهای بعدی روی server سریع باشد، client به‌دلیل زمانی که منتظر تکمیل request قبلی می‌ماند، response time کلی کندی مشاهده خواهد کرد. به‌دلیل این اثر، اندازه‌گیری response time در سمت client اهمیت دارد.

وقتی برای آزمودن scalability سیستم به‌صورت مصنوعی load تولید می‌کنید، client تولیدکنندهٔ load باید مستقل از response time به ارسال request ادامه دهد. اگر client پیش از ارسال request بعدی منتظر تکمیل request قبلی بماند، صف‌ها در test به‌طور مصنوعی کوتاه‌تر از حالت واقعی باقی می‌مانند و این موضوع measurementها را منحرف می‌کند [23].

#### Percentiles in Practice

percentileهای بالا به‌خصوص در backend serviceهایی اهمیت دارند که برای پاسخ دادن به یک request واحد از end user، چند بار فراخوانی می‌شوند. حتی اگر این callها را parallel انجام دهید، request end user همچنان باید منتظر کندترین call parallel بماند. فقط یک call کند کافی است تا کل request end user کند شود؛ همان‌طور که در شکل ۱-۵ نشان داده شده است. حتی اگر فقط درصد کوچکی از backend callها کند باشند، وقتی یک request end user به چند backend call نیاز دارد، احتمال دریافت یک call کند افزایش می‌یابد و در نتیجه سهم بیشتری از requestهای end user کند می‌شوند. این اثر `tail latency amplification` نام دارد [24].

اگر بخواهید percentileهای response time را به dashboardهای monitoring serviceهای خود اضافه کنید، باید آن‌ها را به‌صورت مداوم و مؤثر محاسبه کنید. برای مثال، ممکن است بخواهید یک `rolling window` از response timeهای requestهای ۱۰ دقیقهٔ اخیر نگه دارید. هر دقیقه، median و percentileهای مختلف را روی مقدارهای آن window محاسبه و این metricها را روی یک graph رسم می‌کنید.

پیاده‌سازی naïve این است که فهرستی از response time همهٔ requestها را در window زمانی نگه دارید و هر دقیقه آن فهرست را sort کنید. اگر این کار بیش از حد inefficient باشد، algorithmهایی وجود دارند که با حداقل هزینهٔ CPU و memory، تقریب خوبی از percentileها محاسبه می‌کنند؛ مانند `forward decay` [25]، `t-digest` [26] یا `HdrHistogram` [27]. توجه کنید که average گرفتن از percentileها - مثلاً برای کاهش resolution زمانی یا ترکیب داده‌های چند machine - از نظر ریاضی بی‌معناست. روش درست برای aggregate کردن داده‌های response time، جمع کردن histogramهاست [28].

*شکل ۱-۵. وقتی برای پاسخ دادن به یک request به چند backend call نیاز است، فقط یک backend request کند می‌تواند کل request end user را کند کند.*

### Approaches for Coping with Load

اکنون که parameterهای توصیف load و metricهای اندازه‌گیری performance را بررسی کردیم، می‌توانیم جدی‌تر دربارهٔ scalability صحبت کنیم: چگونه حتی وقتی load parameterهای ما به میزان مشخصی افزایش می‌یابند، performance را در سطح خوبی نگه داریم؟

معماری‌ای که برای یک سطح از load مناسب است، احتمالاً نمی‌تواند با ۱۰ برابر آن load کنار بیاید. بنابراین اگر روی serviceای با رشد سریع کار می‌کنید، احتمالاً باید با هر افزایش یک order of magnitude در load - یا حتی بیشتر از آن - معماری خود را از نو بررسی کنید.

افراد اغلب از دوگانه‌ای میان `scaling up` یا `vertical scaling` - انتقال به machine قدرتمندتر - و `scaling out` یا `horizontal scaling` - توزیع load میان چند machine کوچک‌تر - صحبت می‌کنند. توزیع load میان چند machine را `shared-nothing architecture` نیز می‌نامند. سیستمی که می‌تواند روی یک machine اجرا شود معمولاً ساده‌تر است، اما machineهای high-end می‌توانند بسیار گران شوند؛ بنابراین workloadهای بسیار سنگین اغلب نمی‌توانند از scaling out اجتناب کنند. در عمل، معماری‌های خوب معمولاً ترکیبی pragmatic از این رویکردها هستند. برای مثال، استفاده از چند machine نسبتاً قدرتمند همچنان می‌تواند ساده‌تر و ارزان‌تر از استفاده از تعداد زیادی virtual machine کوچک باشد.

بعضی سیستم‌ها `elastic` هستند؛ یعنی وقتی افزایش load را تشخیص می‌دهند، می‌توانند به‌طور خودکار resourceهای محاسباتی اضافه کنند. در مقابل، سیستم‌های دیگر به‌صورت manual scale می‌شوند؛ یعنی یک انسان capacity را تحلیل می‌کند و تصمیم می‌گیرد machineهای بیشتری به سیستم اضافه کند. یک elastic system زمانی مفید است که load بسیار unpredictable باشد، اما سیستم‌های manual ساده‌ترند و ممکن است operational surpriseهای کمتری داشته باشند.

در حالی که توزیع stateless serviceها میان چند machine نسبتاً ساده است، انتقال یک stateful data system از یک node منفرد به یک setup توزیع‌شده می‌تواند پیچیدگی زیادی ایجاد کند. به همین دلیل، تا همین اواخر wisdom رایج این بود که database را روی یک node منفرد نگه داریم و آن را scale up کنیم؛ مگر اینکه هزینهٔ scaling یا نیازهای high availability ما را مجبور کند آن را distributed کنیم.

با بهتر شدن ابزارها و abstractionهای مربوط به distributed systemها، این wisdom رایج ممکن است - دست‌کم برای بعضی applicationها - تغییر کند. قابل تصور است که در آینده distributed data systemها به انتخاب پیش‌فرض تبدیل شوند، حتی برای use caseهایی که حجم زیادی از data یا traffic را مدیریت نمی‌کنند. در ادامهٔ این کتاب، انواع مختلف distributed data systemها را بررسی می‌کنیم و می‌بینیم که آن‌ها نه‌فقط از نظر scalability، بلکه از نظر ease of use و maintainability چگونه عمل می‌کنند.

معماری سیستم‌هایی که در scale بزرگ کار می‌کنند معمولاً کاملاً به application خاص وابسته است؛ چیزی به نام معماری scalable عمومی و مناسب همه وجود ندارد؛ اصطلاح غیررسمی آن `magic scaling sauce` است. مشکل ممکن است حجم readها، حجم writeها، حجم دادهٔ قابل ذخیره، پیچیدگی داده، نیازمندی‌های response time، access patternها یا - معمولاً - ترکیبی از همهٔ این موارد و مسائل متعدد دیگر باشد.

برای مثال، سیستمی که برای پردازش ۱۰۰٬۰۰۰ request در ثانیه طراحی شده و اندازهٔ هر request یک kB است، با سیستمی که برای ۳ request در دقیقه طراحی شده و اندازهٔ هر request دو GB است، تفاوت زیادی دارد؛ حتی اگر throughput دادهٔ هر دو سیستم یکسان باشد.

معماری‌ای که برای یک application مشخص به‌خوبی scale می‌شود، بر اساس assumptionهایی دربارهٔ operationهای رایج و operationهای نادر ساخته می‌شود؛ این‌ها همان load parameterها هستند. اگر این assumptionها اشتباه از آب دربیایند، تلاش engineering برای scaling در بهترین حالت هدر رفته و در بدترین حالت counterproductive است. در یک startup نوپا یا محصولی که هنوز اعتبارسنجی نشده است، معمولاً توانایی iterate کردن سریع روی featureهای محصول مهم‌تر از scale کردن برای load فرضی آینده است.

هرچند معماری‌های scalable به application مشخصی وابسته‌اند، معمولاً از building blockهای general-purpose تشکیل می‌شوند که در patternهای آشنا کنار هم قرار گرفته‌اند. در این کتاب building blockها و patternهای مذکور را بررسی می‌کنیم.

## Key Terms

- `Scalability` — توانایی کنار آمدن سیستم با افزایش load و حفظ performance قابل‌قبول با اضافه کردن resource.
- `Load` — حجم request، داده، user یا operationهایی که سیستم باید در یک بازهٔ زمانی پردازش کند.
- `Load Parameter` — عدد یا شاخص توصیف‌کنندهٔ load، مانند request در ثانیه، نسبت read به write یا cache hit rate.
- `Performance` — کیفیت و سرعت انجام کار توسط سیستم که با metricهایی مانند throughput، response time و latency سنجیده می‌شود.
- `Batch Processing` — پردازش مجموعه‌ای از داده در قالب job که معمولاً throughput یا زمان اجرای کامل job معیار اصلی آن است.
- `Online Processing` — پردازش requestهای تعاملی و جاری که در آن response time و latency اهمیت بیشتری دارند.
- `Throughput` — تعداد record، request یا job پردازش‌شده در واحد زمان.
- `Response Time` — فاصلهٔ ارسال request تا دریافت response از دید client، شامل service time و delayهای network و queue.
- `Service Time` — زمان واقعی پردازش request در service که بخشی از response time است.
- `Latency` — مدت انتظار request برای رسیدگی؛ مفهومی متفاوت از response time.
- `Queueing Delay` — زمان انتظار request در queue پیش از شروع پردازش، به‌دلیل اشغال بودن ظرفیت.
- `Percentile` — آستانه‌ای برای توصیف توزیع response time؛ مثلاً p95 زمانی است که ۹۵ درصد requestها سریع‌تر از آن پاسخ می‌گیرند.
- `Median` — مقدار میانی یک توزیع مرتب‌شده؛ همان p50 که نیمی از مقدارها کمتر و نیمی بیشتر از آن هستند.
- `Tail Latency` — latency در percentileهای بالا که کندی بخش کوچکی از requestها را نشان می‌دهد.
- `Tail Latency Amplification` — بزرگ‌تر شدن اثر tail latency در requestهایی که چند backend call دارند.
- `Head-of-Line Blocking` — معطل شدن requestهای بعدی پشت یک request کند به‌دلیل محدودیت parallelism.
- `Fan-out` — تعداد مقصدها یا callهای ایجادشده از یک operation؛ مانند تبدیل یک tweet به writeهای متعدد برای followerها.
- `Horizontal Scaling` — افزایش ظرفیت با اضافه کردن machine یا node و توزیع load میان آن‌ها؛ همان scaling out.
- `Vertical Scaling` — افزایش ظرفیت با استفاده از machine قدرتمندتر؛ همان scaling up.
- `Shared-Nothing Architecture` — معماری توزیع‌شده‌ای که در آن هر node resourceهای خود را دارد و load میان nodeها توزیع می‌شود.
- `Elasticity` — توانایی افزودن یا حذف خودکار resource بر اساس load؛ برای loadهای unpredictable مفید است.
- `Resource Utilization` — میزان استفاده از resourceهای محاسباتی برای تحلیل capacity و تصمیم‌گیری دربارهٔ افزایش منابع.
- `Service Level Objective (SLO)` — هدف قابل‌اندازه‌گیری برای سطح service، مانند p99 response time یا درصد availability.
- `Service Level Agreement (SLA)` — توافق قراردادی دربارهٔ performance و availability مورد انتظار یک service.

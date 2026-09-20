# DDIA Terminology

## Chapter 1

| English Term | Persian Explanation | Engineering Meaning |
|---|---|---|
| Reliability | توانایی ادامهٔ عملکرد صحیح سیستم در شرایط نامساعد | سیستم باید function مورد انتظار را حتی در برابر hardware fault، software fault و خطای انسانی درست اجرا کند. |
| Scalability | توانایی مدیریت رشد سیستم | با افزایش حجم داده، ترافیک یا پیچیدگی، راهی معقول برای حفظ عملکرد و توسعهٔ سیستم وجود داشته باشد. |
| Maintainability | قابلیت نگهداری و تغییرپذیری سیستم | افراد مختلف در تیم‌های engineering و operations بتوانند سیستم را به‌صورت productive نگهداری و توسعه دهند. |
| Data System | سیستمی برای ذخیره‌سازی و پردازش داده | ممکن است از database، cache، search index و message queue تشکیل شود و از طریق یک API واحد ارائه شود. |
| Database | سیستم اصلی برای ذخیره و بازیابی داده | داده را با الگوهای دسترسی و ضمانت‌های مشخص ذخیره می‌کند و معمولاً منبع اصلی دادهٔ application است. |
| Datastore | ابزار عمومی نگهداری داده | هر سامانهٔ ذخیره‌سازی داده که الزاماً در دسته‌بندی سنتی database قرار نمی‌گیرد. |
| Storage Engine | موتور داخلی ذخیره و بازیابی داده | ساختارها و algorithmهایی که database برای نوشتن، خواندن و سازمان‌دهی داده روی storage استفاده می‌کند. |
| Data Model | مدل سازمان‌دهی و نمایش داده | مشخص می‌کند داده چگونه ساختاربندی شود و رابطهٔ میان داده‌ها چگونه بیان و استفاده شود. |
| Query | درخواست خواندن یا پردازش داده | عملیاتی declarative یا procedural برای انتخاب، فیلتر، ترکیب یا تغییر داده. |
| Transaction | واحد منطقی اجرای عملیات داده | مجموعه‌ای از عملیات که باید با semantics مشخص و معمولاً به‌صورت اتمیک اجرا شود. |
| Consistency | سازگاری مشاهده‌شده در داده | تضمینی دربارهٔ اینکه clientها پس از عملیات، وضعیت داده را مطابق قواعد سیستم و ترتیب معتبر ببینند. |
| Fault Tolerance | توانایی ادامهٔ کار با وجود fault | طراحی سیستم برای شناسایی یا تحمل برخی hardware faultها، software faultها و failureهای داخلی بدون از دست دادن عملکرد موردنیاز. |
| Latency | مدت انتظار request برای رسیدگی | با response time یکی نیست و فقط بخش انتظار برای دریافت service را توصیف می‌کند. |
| Throughput | نرخ پردازش کار در واحد زمان | تعداد requestها، پیام‌ها یا عملیات پردازش‌شده در واحد زمان. |
| Message Queue | صف ذخیره و انتقال پیام میان componentها | producer پیام را در صف قرار می‌دهد و consumer می‌تواند آن را مستقل یا به‌صورت asynchronous پردازش کند. |
| Durability Guarantee | تضمین باقی‌ماندن داده پس از write | قراردادی دربارهٔ اینکه دادهٔ پذیرفته‌شده پس از restart، crash یا failure تا چه حد حفظ می‌شود. |
| Access Pattern | الگوی خواندن و نوشتن داده | شکل و توزیع عملیاتی که application روی داده انجام می‌دهد و مبنای انتخاب storage و index است. |
| Cache | محل نگهداری موقت داده یا نتیجه | با نگهداری نتیجهٔ عملیات پرهزینه، latency خواندن را کاهش می‌دهد؛ اما باید با منبع اصلی داده همگام بماند. |
| Search Index | ساختار بهینه برای جست‌وجوی داده | امکان جست‌وجوی سریع بر اساس keyword یا معیارهای دیگر را فراهم می‌کند و معمولاً باید با database هماهنگ بماند. |
| API | رابط قابل استفاده برای clientها | قرارداد دسترسی به یک service که جزئیات implementation داخلی را از client پنهان می‌کند. |
| Load | میزان کاری که به سیستم وارد می‌شود | معمولاً با حجم request، ترافیک، داده یا عملیات در یک بازهٔ زمانی توصیف می‌شود و مبنای تحلیل Scalability است. |
| Fault | انحراف یک component از specification خود | وضعیتی در یک جزء سیستم که ممکن است، اما لزوماً نباید، به failure کل سیستم منجر شود. |
| Failure | از کار افتادن service در سطح کل سیستم | زمانی که سیستم دیگر service موردنیاز کاربر را ارائه نمی‌کند. |
| Fault-Tolerant | مقاوم در برابر برخی faultها | صفت سیستمی که می‌تواند faultهای مشخصی را تحمل کند و service را ادامه دهد. |
| Resilient | توانمند در بازگشت یا ادامهٔ کار پس از اختلال | سیستمی که در برابر اختلال‌ها رفتار پایدار دارد و می‌تواند از failureهای موقت عبور کند. |
| Hardware Fault | خرابی در componentهای سخت‌افزاری | خرابی disk، RAM، منبع تغذیه، network یا machine که می‌تواند به failure منجر شود. |
| Software Error | خطای داخلی در software | bug یا رفتار نادرستی که ممکن است در چند node تکرار شود و failureهای متعدد ایجاد کند. |
| Systematic Error | خطایی وابسته به یک علت مشترک یا الگوی تکرارشونده | faultی که برخلاف خرابی تصادفی hardware، می‌تواند هم‌زمان چند node یا component را تحت تأثیر قرار دهد. |
| Cascading Failure | زنجیره‌ای از failureهای وابسته | fault یک component باعث fault در component دیگر می‌شود و این روند در سیستم گسترش می‌یابد. |
| Mean Time to Failure (MTTF) | میانگین زمان مورد انتظار تا خرابی | معیاری برای برآورد عمر متوسط یک component پیش از failure. |
| Redundancy | داشتن component یا مسیر جایگزین | چند component می‌توانند در صورت خرابی یکی از آن‌ها، service را ادامه دهند. |
| High Availability | در دسترس بودن مداوم service | هدفی برای کاهش downtime و حفظ دسترسی service، حتی هنگام maintenance یا failure برخی nodeها. |
| Rolling Upgrade | ارتقای تدریجی nodeها بدون توقف کل سیستم | تغییر version یا patch هر بار روی بخشی از nodeها اعمال می‌شود تا service در دسترس بماند. |
| Process Isolation | جدا کردن فرایندها از یکدیگر | خطا یا مصرف resource یک process نباید مستقیماً processهای دیگر را از کار بیندازد. |
| Configuration Error | خطای ناشی از تنظیمات نادرست | یکی از علت‌های مهم outage که باید با validation، محیط آزمایشی و rollback کنترل شود. |
| Sandbox | محیط جدا و امن برای آزمایش | محیطی غیر-production که امکان کار با دادهٔ واقعی را بدون اثرگذاری بر کاربران فراهم می‌کند. |
| Automated Testing | اجرای خودکار testها | روشی برای کشف bugها و پوشش corner caseها در سطح unit، integration و کل سیستم. |
| Rollback | بازگرداندن تغییر به version یا configuration قبلی | راهی برای recovery سریع پس از deployment یا configuration change ناموفق. |
| Monitoring | مشاهده و اندازه‌گیری وضعیت سیستم | جمع‌آوری metricها و signalها برای تشخیص زودهنگام مشکل و تحلیل failure. |
| Telemetry | دادهٔ عملیاتی برای مشاهدهٔ رفتار سیستم | metric، log و signalهایی که برای پیگیری وضعیت و فهمیدن failureها جمع‌آوری می‌شوند. |
| Error Rate | نرخ رخداد خطا در یک بازهٔ زمانی | تعداد errorها نسبت به کل عملیات یا requestها؛ شاخصی برای سلامت service. |
| Backup | نسخهٔ ذخیره‌شده برای بازگردانی داده | کپی قابل استفاده برای restore داده پس از corruption یا failure. |
| Recovery | بازگرداندن سیستم یا داده به وضعیت قابل استفاده | اقدام‌ها و ابزارهایی برای ادامهٔ service یا restore داده پس از failure. |
| Failure Detection | شناسایی خرابی یا از دسترس خارج شدن component | پایش health workerها و serviceها برای تشخیص سریع failure و آغاز recovery. |
| Performance | کیفیت و سرعت انجام کار توسط سیستم | با metricهایی مانند throughput، response time و latency سنجیده می‌شود. |
| Load Parameter | عدد یا شاخص توصیف‌کنندهٔ load | مانند request در ثانیه، نسبت read به write، تعداد user فعال یا cache hit rate. |
| Fan-out | تعداد مقصدها یا callهای ایجادشده از یک operation | یک operation مانند post کردن tweet می‌تواند به تعداد زیادی write یا call تبدیل شود. |
| Batch Processing | پردازش مجموعه‌ای از داده در قالب job | معمولاً throughput یا زمان کامل اجرای job معیار اصلی آن است. |
| Online Processing | پردازش requestهای تعاملی و جاری | معمولاً response time و latency برای تجربهٔ client اهمیت بیشتری دارند. |
| Response Time | زمان مشاهده‌شده از دید client | فاصلهٔ ارسال request تا دریافت response، شامل service time و delayهای network و queue. |
| Service Time | زمان واقعی پردازش request | بخشی از response time که صرف پردازش request در service می‌شود. |
| Queueing Delay | زمان انتظار request در queue | تأخیری که پیش از شروع پردازش request و به‌دلیل اشغال بودن ظرفیت ایجاد می‌شود. |
| Percentile | آستانه‌ای برای توصیف توزیع response time | مثلاً p95 زمانی است که ۹۵ درصد requestها سریع‌تر از آن پاسخ می‌گیرند. |
| Median | مقدار میانی یک توزیع مرتب‌شده | همان p50؛ نیمی از مقدارها کمتر و نیمی بیشتر از آن هستند. |
| Tail Latency | latency در percentileهای بالا | کندی بخش کوچکی از requestها که می‌تواند مستقیماً تجربهٔ user را خراب کند. |
| Tail Latency Amplification | بزرگ‌تر شدن اثر tail latency در callهای زنجیره‌ای | در requestهایی با چند backend call، فقط یک call کند می‌تواند کل request را کند کند. |
| Head-of-Line Blocking | معطل شدن requestهای بعدی پشت یک request کند | محدودیت parallelism باعث می‌شود چند request کند، response time requestهای سریع بعدی را هم افزایش دهند. |
| Horizontal Scaling | افزایش ظرفیت با اضافه کردن machine یا node | load میان چند machine توزیع می‌شود؛ همان scaling out. |
| Vertical Scaling | افزایش ظرفیت با استفاده از machine قدرتمندتر | منابع یک machine افزایش می‌یابد؛ همان scaling up. |
| Shared-Nothing Architecture | معماری توزیع‌شده بدون resource مشترک مرکزی | هر node منابع خود را دارد و load میان nodeها توزیع می‌شود. |
| Elasticity | توانایی افزودن یا حذف خودکار resource بر اساس load | برای loadهای unpredictable مفید است، اما می‌تواند پیچیدگی عملیاتی ایجاد کند. |
| Resource Utilization | میزان استفاده از resourceهای محاسباتی | برای تحلیل capacity و تصمیم‌گیری دربارهٔ افزایش CPU، memory، network یا nodeها استفاده می‌شود. |
| Service Level Objective (SLO) | هدف قابل‌اندازه‌گیری برای سطح service | معیار داخلی مانند p99 response time یا درصد availability که service باید به آن برسد. |
| Service Level Agreement (SLA) | توافق قراردادی دربارهٔ سطح service | performance و availability مورد انتظار را مشخص می‌کند و ممکن است در صورت نقض، جبران تعیین کند. |
| Operability | آسان بودن اجرای پایدار و روزمرهٔ سیستم | سیستم باید visibility، automation و control کافی برای تیم operations فراهم کند. |
| Simplicity | کاهش complexity غیرضروری بدون حذف functionality لازم | سیستم ساده‌تر راحت‌تر فهمیده، نگهداری و تغییر داده می‌شود. |
| Complexity | دشواری فهم، تغییر یا پیش‌بینی رفتار سیستم | complexity هزینهٔ maintenance و احتمال bugهای ناشی از change را افزایش می‌دهد. |
| Accidental Complexity | complexity ناشی از implementation، نه خود مسئله | بخشی از دشواری که با abstraction و طراحی بهتر می‌توان حذف کرد. |
| Essential Complexity | complexity ذاتی خود مسئله | دشواری‌ای که از requirement و domain مسئله می‌آید و با حذف implementation از بین نمی‌رود. |
| Abstraction | پنهان کردن جزئیات پشت یک interface ساده | جزئیات implementation را مخفی و component را برای applicationهای مختلف reusable می‌کند. |
| Coupling | وابستگی میان moduleها یا componentها | coupling شدید باعث می‌شود change در یک بخش، بخش‌های دیگر را نیز تحت تأثیر قرار دهد. |
| Dependency | رابطهٔ نیازمندی یک component به component دیگر | dependencyهای درهم‌تنیده فهم، تست و تغییر سیستم را دشوار می‌کنند. |
| Automation | انجام خودکار taskها و processهای عملیاتی | خطای انسانی را کاهش می‌دهد، اما نیازمند setup، monitoring و نگهداری صحیح است. |
| Operational Model | مدل قابل‌فهم برای رفتار عملیاتی سیستم | توضیح می‌دهد عملیات‌هایی مانند تغییر configuration یا restart چه نتیجه‌ای دارند. |
| Self-Healing | بازگردانی خودکار سیستم از برخی وضعیت‌های خراب | سیستم می‌تواند برخی failureها را بدون intervention دستی شناسایی و اصلاح کند. |
| Evolvability | توانایی سازگار شدن سیستم با requirementهای جدید | agility در سطح data system و امکان تغییر architecture یا behavior در طول زمان. |
| Legacy System | سیستم قدیمی و دشوار برای نگهداری یا تغییر | معمولاً نتیجهٔ تصمیم‌ها و محدودیت‌های گذشته است و maintenance آن هزینهٔ زیادی دارد. |
| Technical Debt | هزینهٔ آیندهٔ تصمیم‌های فنی کوتاه‌مدت یا ناقص | changeهای بعدی را دشوارتر می‌کند و باید آگاهانه مدیریت و بازپرداخت شود. |
| Agile | رویکرد کاری برای سازگاری سریع با change | processها و practiceهایی برای iteration سریع و پاسخ‌گویی به requirementهای جدید. |
| Test-Driven Development (TDD) | توسعهٔ code با شروع از test | با تعریف behavior مورد انتظار پیش از implementation، feedback سریع و تغییرپذیری را بهبود می‌دهد. |
| Refactoring | تغییر ساختار داخلی بدون تغییر behavior قابل‌مشاهده | complexity را کاهش می‌دهد و code را برای changeهای بعدی آماده‌تر می‌کند. |
| Change Management | مدیریت کنترل‌شدهٔ تغییرات سیستم | changeها را با توجه به اثر، ریسک، deployment و recovery برنامه‌ریزی و اجرا می‌کند. |
| Configuration Management | مدیریت version و تغییرات configuration | از تغییرات ناسازگار جلوگیری و امکان audit و بازگردانی configuration را فراهم می‌کند. |
| Capacity Planning | پیش‌بینی resource موردنیاز در آینده | با بررسی رشد load، ظرفیت لازم برای جلوگیری از degradation را برآورد می‌کند. |
| Big Ball of Mud | سیستم یا codebase‌ای گرفتار complexity و dependencyهای درهم‌تنیده | maintenance و تغییر چنین سیستمی دشوار و پرهزینه است. |
| Modularity | تقسیم سیستم به moduleهای مستقل و قابل‌مدیریت | مرزهای روشن میان بخش‌ها coupling را کاهش و تغییرپذیری را افزایش می‌دهد. |
| Data-Intensive Application | applicationای که چالش اصلی آن مقدار، complexity یا سرعت تغییر داده است | طراحی آن بیشتر تحت تأثیر data، storage و processing قرار دارد تا محدودیت CPU. |
| Functional Requirement | requirement مربوط به کاری که application باید انجام دهد | مانند ذخیره، بازیابی، جست‌وجو یا پردازش داده. |
| Nonfunctional Requirement | ویژگی کلی و کیفی سیستم | مانند security، reliability، scalability، compatibility یا maintainability. |
| Compliance | رعایت الزام‌های قانونی، regulatory یا سیاست‌های سازمانی | رفتار و دادهٔ سیستم باید با قوانین و policyهای لازم سازگار باشد. |
| Compatibility | توانایی کار کردن با platform، component یا interface دیگر | تغییر یک جزء نباید بدون دلیل، integration با اجزای سازگار را مختل کند. |
| Processing Capacity | مقدار ظرفیت محاسباتی موجود برای پردازش load | با افزایش ظرفیت می‌توان در load بالا performance و reliability را حفظ کرد. |

## Chapter 2

| English Term | Persian Explanation | Engineering Meaning |
|---|---|---|
| Relational Model | مدل سازمان‌دهی داده در relationها یا tableها | داده را به row و column تقسیم می‌کند و access path را از application پنهان می‌سازد. |
| Document Model | مدل سازمان‌دهی داده در documentهای معمولاً nested | برای داده‌های tree-like و one-to-many می‌تواند locality و سادگی بیشتری فراهم کند. |
| NoSQL | عنوانی کلی برای databaseهای nonrelational | معمولاً برای scalability، queryهای تخصصی یا schemaهای dynamic استفاده می‌شود. |
| Relational Database Management System (RDBMS) | نرم‌افزار مدیریت database بر پایهٔ relational model | ذخیره، query و مدیریت دادهٔ ساخت‌یافته را با table، row و column انجام می‌دهد. |
| Schema | ساختار و قواعد شکل دادهٔ ذخیره‌شده | مشخص می‌کند داده چه fieldها، typeها و رابطه‌هایی داشته باشد. |
| Polyglot Persistence | استفاده از چند نوع datastore برای نیازهای مختلف | فناوری ذخیره‌سازی بر اساس use case انتخاب می‌شود، نه با یک راه‌حل واحد برای همه. |
| Object-Relational Mapping (ORM) | نگاشت objectهای application به ساختار relational | translation layer میان objectهای code و tableهای database را ساده‌تر می‌کند. |
| Impedance Mismatch | ناهماهنگی میان مدل object-oriented و relational | تفاوت دو مدل باعث translation layer و boilerplate code می‌شود. |
| Nested Data | دادهٔ تو‌در‌تو درون یک record یا document | داده‌های مرتبط را در یک ساختار واحد نگه می‌دارد و می‌تواند queryهای چندگانه را کاهش دهد. |
| Relation | مجموعه‌ای از tupleها در relational model | در SQL معمولاً با table نمایش داده می‌شود. |
| Tuple | یک عضو از relation در relational model | در SQL معمولاً با row نمایش داده می‌شود. |
| One-to-Many Relationship | رابطهٔ یک entity با چند entity مرتبط | مانند یک user با چند position یا یک document والد با چند record فرزند. |
| Many-to-One Relationship | رابطهٔ چند entity با یک entity مشترک | مانند چند user که به یک region یا industry اشاره می‌کنند. |
| Many-to-Many Relationship | رابطهٔ چند entity از هر دو طرف | هر entity می‌تواند با چند entity از طرف مقابل مرتبط باشد و معمولاً به reference و join نیاز دارد. |
| Foreign Key | identifierای برای reference به row یک table دیگر | رابطهٔ میان tableها را برقرار می‌کند و ممکن است با constraint محدود شود. |
| Join | ترکیب دادهٔ مرتبط از چند table | هنگام query، rowهای مرتبط را بر اساس key یا شرط مشترک به هم متصل می‌کند. |
| Normalization | حذف duplication با ذخیرهٔ هر value در یک محل | write overhead و خطر inconsistency ناشی از copyهای متعدد را کاهش می‌دهد. |
| Denormalization | duplicate کردن کنترل‌شدهٔ داده | با کاهش join یا بهبود locality می‌تواند read را سریع‌تر کند، اما update و consistency را دشوارتر می‌کند. |
| Hierarchical Model | مدل درختیِ recordهای nested | هر record معمولاً یک parent دارد و برای one-to-many مناسب است. |
| Network Model | مدل graph-like با امکان چند parent برای هر record | many-to-one و many-to-many را با link و access path مدل می‌کند. |
| Access Path | مسیر مشخص برای رسیدن به record | در مدل‌های قدیمی با دنبال کردن linkها از root به داده می‌رسیدند. |
| Query Optimizer | جزء database برای انتخاب روش اجرای query | ترتیب اجرای operationها و indexهای مناسب را خودکار تعیین می‌کند. |
| Document Reference | identifierای برای اشاره به document مرتبط | در document model نقش مشابه foreign key را دارد و هنگام read resolve می‌شود. |
| Data Locality | نزدیک بودن داده‌های مرتبط در یک محل ذخیره‌سازی | می‌تواند تعداد queryها و joinهای لازم برای خواندن یک entity را کاهش دهد. |
| Schema Flexibility | آزادی در تغییر structure documentها بدون migration هم‌زمان در کل داده‌ها | برای داده‌های متغیر مفید است، اما بخشی از اعتبارسنجی schema را به application منتقل می‌کند. |
| Schema-on-Read | تفسیر structure داده هنگام read، بدون enforce شدن کامل توسط database | تغییر format داده را ساده‌تر می‌کند، اما client باید با versionهای مختلف داده سازگار باشد. |
| Schema-on-Write | enforce کردن schema هنگام write و الزام سازگاری دادهٔ ذخیره‌شده با آن | ساختار داده را صریح و قابل‌کنترل می‌کند، اما تغییر schema ممکن است به migration نیاز داشته باشد. |
| Schema Evolution | تغییر کنترل‌شدهٔ schema در طول عمر application | code و داده باید بتوانند با schemaهای جدید و قدیمی به‌صورت سازگار کار کنند. |
| Migration | تغییر structure یا انتقال داده برای هماهنگی با schema جدید | معمولاً برای تغییر table، اضافه کردن field یا بازنویسی recordهای موجود استفاده می‌شود. |
| Dynamic Type Checking | بررسی type داده در زمان اجرای program | انعطاف‌پذیری بیشتری می‌دهد، اما برخی خطاها دیرتر و هنگام runtime آشکار می‌شوند. |
| Static Type Checking | بررسی type داده پیش از اجرای program، معمولاً در compile time | بسیاری از ناسازگاری‌های type را زودتر آشکار می‌کند، اما تغییر structure ممکن است نیازمند تغییرات صریح باشد. |
| Heterogeneous Data | داده‌ای که itemهای آن structure یا type یکسانی ندارند | در collectionهای متنوع یا دادهٔ ورودی از systemهای خارجی رایج است و ممکن است schema ثابت را نامناسب کند. |
| Client-Side Join | اجرای join در application با دریافت داده از چند request | complexity و round-tripهای network را افزایش می‌دهد و معمولاً از join داخل database کندتر است. |
| Hybrid Data Model | ترکیب قابلیت‌های relational و document | امکان استفادهٔ هم‌زمان از nested data و queryهای relational را فراهم می‌کند. |
| Query Language | زبان بیان query برای خواندن یا پردازش داده | interfaceی برای انتقال خواستهٔ application به database با syntax مشخص است. |
| Declarative Query | queryای که result موردنظر را مشخص می‌کند، نه مراحل رسیدن به آن | database می‌تواند execution plan، index و ترتیب operationها را خودش انتخاب و optimize کند. |
| Imperative Query | query یا APIای که مراحل و ترتیب اجرای operationها را مشخص می‌کند | کنترل اجرایی بیشتری می‌دهد، اما coupling با implementation و access path را افزایش می‌دهد. |
| Declarative Programming | برنامه‌نویسی بر اساس توصیف result یا rule، بدون تعیین algorithm دقیق | abstraction و امکان optimization خودکار را افزایش می‌دهد. |
| Imperative Programming | برنامه‌نویسی با تعیین گام‌ها و ترتیب اجرای آن‌ها | کنترل رفتار اجرایی را بیشتر می‌کند، اما parallelization و تغییر implementation دشوارتر می‌شود. |
| Relational Algebra | مجموعه‌ای formal از operationها برای کار با relationها | مبنای نظری بسیاری از queryهای relational و SQL است. |
| Query Execution | اجرای عملی query روی داده | شامل انتخاب plan، خواندن index، join، filter، grouping و تولید result است. |
| Parallel Processing | اجرای هم‌زمان بخش‌های یک کار روی چند core یا machine | با تقسیم کار می‌تواند throughput را افزایش دهد، اگر taskها قابلیت parallel شدن داشته باشند. |
| MapReduce | programming model برای پردازش داده در دو مرحلهٔ map و reduce | اجرای distributed روی دادهٔ بزرگ را ممکن می‌کند و در برخی NoSQL datastoreها برای query استفاده می‌شود. |
| Map Function | functionی که ورودی‌ها را می‌خواند و key-value تولید می‌کند | داده را به خروجی‌های قابل group شدن برای مرحلهٔ reduce تبدیل می‌کند. |
| Reduce Function | functionی که valueهای مربوط به یک key را ترکیب می‌کند | aggregation یا محاسبهٔ نهایی هر گروه را انجام می‌دهد. |
| Pure Function | functionی بدون وابستگی بیرونی و بدون side effect | اجرای مجدد، جابه‌جایی و parallel کردن آن در distributed system امن‌تر است. |
| Side Effect | تغییری خارج از result function، مانند write یا تغییر state مشترک | اجرای مجدد یا موازی function را دشوار و احتمال inconsistency را بیشتر می‌کند. |
| Distributed Query Execution | اجرای query روی چند machine یا node | برای پردازش datasetهای بزرگ استفاده می‌شود و به تقسیم کار و coordination نیاز دارد. |
| Aggregation Pipeline | query language مرحله‌ای MongoDB برای filter و group کردن داده | جایگزینی declarativeتر برای بسیاری از queryهای MapReduce است. |
| Composability | قابلیت ترکیب componentها یا operationها برای ساخت behavior پیچیده‌تر | abstractionها را reusable می‌کند و طراحی pipelineهای قابل‌گسترش را آسان‌تر می‌سازد. |
| Graph Database | databaseای که داده را به‌صورت vertex و edge ذخیره و query می‌کند | برای داده‌های به‌شدت interconnected و traversalهای چندمرحله‌ای مناسب است. |
| Graph Model | مدل نمایش داده به‌صورت node و relationship | ارتباط میان entityها را به‌عنوان بخش اصلی data model در نظر می‌گیرد. |
| Property Graph | graphی که vertex و edge در آن identifier، label و property دارند | انعطاف‌پذیری زیادی برای مدل کردن entityها و relationshipهای متنوع فراهم می‌کند. |
| Vertex | node یا entity در graph | نقطه‌ای که object، person، location یا هر entity دیگر را نمایش می‌دهد. |
| Edge | connection جهت‌دار یا رابطه میان دو vertex | نوع و جهت relationship میان entityها را نشان می‌دهد و می‌تواند property داشته باشد. |
| Graph Query | query برای پیدا کردن vertex، edge یا pattern در graph | معمولاً شامل pattern matching یا traversal در مسیرهای چندمرحله‌ای است. |
| Graph Traversal | پیمایش graph با دنبال کردن edgeها از یک vertex به vertexهای دیگر | برای پیدا کردن connectionهای مستقیم یا زنجیره‌ای و حل queryهای relationshipمحور استفاده می‌شود. |
| Cypher | declarative query language مربوط به Neo4j و property graphها | patternهای graph را با syntax خوانا برای match، create و return بیان می‌کند. |
| Triple-Store | datastoreای که اطلاعات را به‌صورت subject، predicate و object ذخیره می‌کند | data model ساده‌ای برای graph data و RDF فراهم می‌کند. |
| Subject | بخش اول یک RDF triple | vertex یا entityای را مشخص می‌کند که statement دربارهٔ آن است. |
| Predicate | بخش دوم یک RDF triple | نوع property یا relationship میان subject و object را مشخص می‌کند. |
| Object | بخش سوم یک RDF triple | value یک property یا vertex مقصد یک relationship است. |
| SPARQL | declarative query language برای triple-storeهای مبتنی بر RDF | patternهای RDF را برای جست‌وجو و ترکیب graph data بیان می‌کند. |
| RDF | data model استاندارد برای بیان resourceها و relationshipهای آن‌ها | امکان تبادل machine-readable data میان systemها و namespaceهای مستقل را فراهم می‌کند. |
| Turtle | syntax خوانا برای نوشتن RDF tripleها | نمایش compactتری از RDF است و برای خواندن و نوشتن انسانی مناسب‌تر از RDF/XML است. |
| Datalog | query language rule-based و declarative، مبتنی بر predicate و rule | queryهای recursive و قابل‌ترکیب را با derive کردن factهای جدید از داده و ruleها بیان می‌کند. |
| Pattern Matching | پیدا کردن بخش‌هایی از graph یا data که با یک pattern مشخص سازگارند | پایهٔ queryهایی مانند MATCH در Cypher و patternهای SPARQL است. |
| Recursive Query | queryای که برای رسیدن به result به خودش یا نتیجهٔ مرحلهٔ قبل reference می‌دهد | برای traversal با عمق نامشخص و common table expressionهای recursive استفاده می‌شود. |
| Graph Processing | پردازش الگوریتمی روی مجموعه‌ای از vertexها و edgeها | برای کارهایی مانند shortest path، ranking و تحلیل connectionها به کار می‌رود. |
| Sequence-Similarity Search | جست‌وجوی stringهایی که از نظر sequence شبیه یکدیگرند | در genome analysis برای مقایسهٔ DNA sequenceها با datasetهای بزرگ استفاده می‌شود. |
| Genome Database | database تخصصی برای ذخیره و query کردن داده‌های genome | برای داده‌هایی طراحی شده که queryهای آن‌ها با databaseهای عمومی به‌خوبی پوشش داده نمی‌شود. |
| Full-Text Search | جست‌وجو در متن بر اساس کلمه، عبارت یا الگوی زبانی | معمولاً با search index انجام می‌شود و در کنار database اصلی قرار می‌گیرد. |
| Information Retrieval | حوزهٔ پیدا کردن اطلاعات مرتبط از میان مجموعه‌ای از documentها | مبنای فنی search engineها و systemهای جست‌وجوی متن است. |
| Data Processing | اجرای operationها برای تبدیل، تحلیل یا استخراج داده | می‌تواند به‌صورت transactional، batch، online یا distributed انجام شود. |

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

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
| Latency | مدت‌زمان پاسخ‌گویی به یک درخواست | زمان سپری‌شده از ارسال request تا دریافت response؛ معمولاً برای سنجش responsiveness استفاده می‌شود. |
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

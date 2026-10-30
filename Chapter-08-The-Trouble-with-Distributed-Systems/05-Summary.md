# Chapter 8 — The Trouble with Distributed Systems

## Summary

در این chapter طیف گسترده‌ای از مشکل‌هایی را بررسی کردیم که ممکن است در distributed systemها رخ دهند؛ از جمله:

- هر بار که packetی از طریق network می‌فرستید، ممکن است گم شود یا به‌شکلی نامحدود به تأخیر بیفتد. reply هم ممکن است گم یا delayed شود؛ بنابراین اگر پاسخی نگیرید، نمی‌دانید message به مقصد رسیده است یا نه.
- ممکن است clock یک node، با وجود تلاش‌های شما برای راه‌اندازی NTP، اختلاف زیادی با clock nodeهای دیگر داشته باشد یا ناگهان به جلو و عقب بپرد. اتکا به clock خطرناک است، چون معمولاً اندازه‌گیری دقیقی از بازهٔ خطای آن ندارید.
- process ممکن است در هر نقطه‌ای از اجرا برای مدت زیادی pause شود؛ مثلاً به‌دلیل stop-the-world garbage collection. nodeهای دیگر ممکن است آن را dead اعلام کنند و process بعداً دوباره فعال شود، بی‌آنکه بداند pause شده بوده است.

امکان رخ دادن چنین **partial failure**هایی ویژگی تعریف‌کنندهٔ distributed systemهاست. هر زمان software کاری را با nodeهای دیگر انجام می‌دهد، ممکن است گاهی fail کند، به‌شکلی تصادفی کند شود یا اصلاً پاسخ ندهد و در نهایت timeout شود. در distributed system می‌کوشیم تحمل partial failure را درون software بسازیم تا کل system، حتی وقتی بعضی اجزای آن خراب‌اند، بتواند به کار ادامه دهد.

برای تحمل faultها، نخست باید آن‌ها را detect کرد؛ اما همین کار هم دشوار است. بیشتر systemها mechanism دقیقی برای فهمیدن اینکه node واقعاً fail شده یا نه ندارند. به همین دلیل، بیشتر distributed algorithmها با timeout تصمیم می‌گیرند که remote node هنوز available است یا خیر. بااین‌حال، timeout نمی‌تواند failure شبکه را از failure خود node تشخیص دهد و variable network delay گاهی باعث می‌شود nodeی را که crash نکرده، به‌اشتباه مشکوک به crash بدانیم. گاهی نیز node در state تنزل‌یافته‌ای قرار دارد: مثلاً ممکن است به‌دلیل bug در driver، throughput یک network interface با ظرفیت یک گیگابیت در ثانیه ناگهان به ۱ کیلوبیت در ثانیه کاهش یابد [94]. برخورد با چنین nodeای که «به‌سختی کار می‌کند» اما کاملاً از کار نیفتاده، حتی از برخورد با nodeای که به‌طور کامل fail شده دشوارتر است.

پس از detect کردن fault هم tolerate کردن آن آسان نیست: میان machineها نه global variable وجود دارد، نه shared memory، نه common knowledge و نه هیچ state مشترک دیگری. nodeها حتی نمی‌توانند روی این توافق کنند که ساعت چند است، چه رسد به موضوع‌های پیچیده‌تر. تنها راه انتقال اطلاعات از یک node به node دیگر، فرستادن آن از طریق network غیرقابل‌اعتماد است. یک node به‌تنهایی نمی‌تواند تصمیم‌های مهم را با اطمینان بگیرد؛ به همین دلیل به protocolهایی نیاز داریم که nodeهای دیگر را نیز درگیر کنند و برای رسیدن به توافق، quorum تشکیل دهند.

اگر به نوشتن software در محیط ریاضیاتی و ایده‌آل یک کامپیوتر عادت داشته باشید—جایی که یک operation یکسان همیشه همان نتیجهٔ قطعی را برمی‌گرداند—ورود به واقعیت فیزیکی و آشفتهٔ distributed systemها می‌تواند شوکه‌کننده باشد. از سوی دیگر، مهندسان distributed system اغلب مسئله‌ای را که روی یک کامپیوتر حل شود ساده می‌دانند [5]؛ به‌خصوص که یک کامپیوتر امروزی کارهای زیادی می‌تواند انجام دهد [95]. اگر می‌توانید از باز کردن این جعبهٔ پاندورا اجتناب کنید و system را روی یک machine نگه دارید، معمولاً این انتخاب ارزشمند است.

بااین‌حال، همان‌طور که در مقدمهٔ بخش دوم گفتیم، scalability تنها دلیل استفاده از distributed system نیست. Fault tolerance و latency پایین—با نزدیک قرار دادن data به کاربران از نظر جغرافیایی—نیز هدف‌های مهمی هستند که با یک node به دست نمی‌آیند.

در این chapter همچنین از بحث اصلی فاصله گرفتیم تا بپرسیم آیا غیرقابل‌اعتماد بودن network، clock و process قانون اجتناب‌ناپذیر طبیعت است یا نه. دیدیم که چنین نیست: می‌توان hard real-time response guarantee و bounded delay در network فراهم کرد، اما این کار بسیار پرهزینه است و utilization منابع hardware را کاهش می‌دهد. بیشتر systemهایی که safety-critical نیستند، گزینهٔ ارزان و غیرقابل‌اعتماد را به گزینهٔ گران و reliable ترجیح می‌دهند.

همچنین به supercomputerها پرداختیم که روی reliable بودن componentها حساب می‌کنند؛ بنابراین اگر componentی fail شود، کل workload باید متوقف و از ابتدا restart شود. در مقابل، distributed systemها—دست‌کم در تئوری—می‌توانند بدون وقفه در سطح service برای همیشه کار کنند، چون تمام faultها و maintenanceها را می‌توان در سطح node مدیریت کرد. در عمل، اگر یک تغییر configuration نامناسب روی همهٔ nodeها deploy شود، distributed system نیز از کار می‌افتد.

این chapter دربارهٔ مشکل‌ها بود و تصویری تیره‌وتار ترسیم کرد. در chapter بعد به سراغ راه‌حل‌ها می‌رویم و algorithmهایی را بررسی می‌کنیم که برای کنار آمدن با مشکل‌های distributed system طراحی شده‌اند.

## Key Terms

- `Partial Failure` — وضعیتی که بخشی از distributed system fail شده، درحالی‌که بخش‌های دیگر همچنان کار می‌کنند.
- `Fault Tolerance` — توانایی system برای ادامهٔ ارائهٔ عملکرد موردنیاز، با وجود fault در بعضی componentها.
- `Unreliable Network` — networkی که ممکن است packetها را گم کند یا با delay نامحدود تحویل دهد؛ نبود response وضعیت node مقابل را روشن نمی‌کند.
- `Network Delay` — مدت زمانی که packet یا response در مسیر network در انتظار انتقال یا تحویل می‌ماند.
- `Timeout` — مهلتی که پس از آن system، در نبود response، node را unavailable یا faulty فرض می‌کند.
- `Unbounded Delay` — نبود upper bound تضمین‌شده برای زمان رسیدن packet یا پاسخ‌گویی node.
- `Clock Skew` — اختلاف time گزارش‌شده میان clockهای nodeهای مختلف.
- `Clock Synchronization` — تنظیم clockها با time source مشترک برای کاهش اختلاف؛ تضمینی برای دقت کامل نیست.
- `Process Pause` — توقف موقت اجرای process یا thread، درحالی‌که بقیهٔ system به کار خود ادامه می‌دهد.
- `Distributed Knowledge` — دانشی که هر node از state محلی و messageهای دریافتی دارد؛ هیچ nodeای دید قطعی از کل system ندارد.
- `Majority Quorum` — توافق بیش از نیمی از nodeها برای گرفتن تصمیمی که یک node به‌تنهایی نباید بگیرد.
- `Byzantine Fault` — رفتار nodeای که ممکن است messageهای دلخواه یا فریب‌کارانه بفرستد و از protocol پیروی نکند.
- `System Model` — abstractionای که timing و faultهای ممکن را مشخص می‌کند و فرض‌های algorithm را محدود می‌سازد.
- `Timing Assumptions` — فرض‌ها دربارهٔ bound مربوط به network delay، process pause و clock error.
- `Safety Property` — شرطی که باید همیشه برقرار بماند و نقض آن در نقطه‌ای مشخص، قابل‌برگشت نیست.
- `Liveness Property` — شرطی دربارهٔ پیشرفت system که انتظار می‌رود با برقرار بودن فرض‌های لازم، سرانجام محقق شود.
- `Degraded Node` — nodeای که کاملاً از کار نیفتاده، اما performance آن به‌شدت افت کرده و ممکن است رسیدگی به آن از failure کامل دشوارتر باشد.

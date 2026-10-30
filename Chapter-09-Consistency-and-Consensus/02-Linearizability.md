# Chapter 9 — Consistency and Consensus

## Linearizability

در databaseای با eventual consistency، اگر هم‌زمان از دو replica متفاوت یک سؤال یکسان بپرسید، ممکن است دو پاسخ متفاوت بگیرید. این گیج‌کننده است. آیا ساده‌تر نبود اگر database این تصور را ایجاد می‌کرد که فقط یک replica—یعنی یک copy از data—وجود دارد؟ در آن صورت همهٔ clientها view یکسانی از data داشتند و لازم نبود نگران replication lag باشید.

این ایده پشت linearizability است؛ مفهومی که با نام‌های atomic consistency، strong consistency، immediate consistency و external consistency نیز شناخته می‌شود. تعریف دقیق linearizability ظریف است و در ادامهٔ این بخش آن را بررسی می‌کنیم؛ اما ایدهٔ پایه این است که system طوری به نظر برسد که انگار فقط یک copy از data وجود دارد و همهٔ operationها روی آن atomic هستند. با وجود چند replica در واقعیت، application با این guarantee لازم نیست نگران آن‌ها باشد.

برای روشن شدن موضوع، وب‌سایتی ورزشی را در نظر بگیرید که linearizable نیست. Alice و Bob در یک اتاق نشسته‌اند و هر دو نتیجهٔ فینال جام جهانی ۲۰۱۴ را در گوشی‌هایشان دنبال می‌کنند. درست پس از اعلام نتیجهٔ نهایی، Alice صفحه را refresh می‌کند، برنده را می‌بیند و با هیجان نتیجه را به Bob می‌گوید. Bob با ناباوری صفحه را دوباره بارگذاری می‌کند، اما request او به database replicaای می‌رسد که عقب‌تر است؛ بنابراین هنوز می‌بیند مسابقه ادامه دارد.

اگر Alice و Bob هم‌زمان صفحه را refresh می‌کردند، دریافت دو نتیجهٔ متفاوت کمتر عجیب بود؛ چون نمی‌دانستند requestهایشان دقیقاً چه زمانی روی server پردازش شده است. اما Bob می‌داند که پس از شنیدن نتیجه از Alice دکمهٔ refresh را زده است، پس انتظار دارد نتیجهٔ query او دست‌کم به‌اندازهٔ نتیجهٔ Alice به‌روز باشد. برگرداندن نتیجه‌ای قدیمی، نقض linearizability است.

### What Makes a System Linearizable?

ایدهٔ اصلی linearizability ساده است: system باید طوری رفتار کند که انگار فقط یک copy از data وجود دارد. اما برای تعریف دقیق این معنا باید کمی دقت کرد.

در یک database خطی‌پذیر، سه client به‌طور هم‌زمان یک key به‌نام `x` را می‌خوانند و می‌نویسند. در ادبیات distributed systemها، `x` را **register** می‌نامند؛ در عمل می‌تواند یک key در key-value store، یک row در relational database یا یک document در document database باشد.

هر bar در نمودار، request یک client را نشان می‌دهد: ابتدای آن زمان ارسال request و انتهای آن زمان دریافت response است. به‌دلیل تغییرپذیری network delay، client نمی‌داند database دقیقاً چه زمانی request را پردازش کرده است؛ فقط می‌داند این اتفاق باید بین ارسال request و دریافت response رخ داده باشد.

نمودار زمانی، محور افقی را مانند یک global clock در نظر می‌گیرد. در systemهای واقعی معمولاً ساعت دقیق و مشترکی نداریم، اما برای تحلیل یک distributed algorithm می‌توان وجود چنین ساعتی را فرض کرد، به‌شرط آنکه خود algorithm به آن دسترسی نداشته باشد. algorithm فقط تقریب نادقیقی از زمان واقعی را می‌بیند که از oscillator کوارتز و NTP به‌دست می‌آید.

در این مثال، register دو نوع operation دارد:

- `read(x) ⇒ v` یعنی client مقدار register با نام `x` را خوانده و database مقدار `v` را برگردانده است.
- `write(x, v) ⇒ r` یعنی client خواسته مقدار `x` را به `v` تغییر دهد و database پاسخ `r` را برگردانده است؛ این پاسخ می‌تواند `ok` یا `error` باشد.

در ابتدا مقدار `x` برابر `0` است و client C آن را به `1` تغییر می‌دهد. هم‌زمان، clientهای A و B مرتب database را poll می‌کنند تا جدیدترین مقدار را بخوانند. پاسخ‌های ممکن چنین‌اند:

- نخستین read از سوی A پیش از شروع write کامل شده است؛ پس حتماً باید مقدار قدیمی `0` را برگرداند.
- آخرین read از سوی A پس از پایان write شروع می‌شود؛ اگر database خطی‌پذیر باشد، حتماً باید مقدار جدید `1` را برگرداند. پردازش write جایی بین شروع و پایان operation آن رخ داده و پردازش read نیز جایی بین شروع و پایان read؛ بنابراین اگر read پس از پایان write آغاز شده باشد، باید بعد از write پردازش شود و مقدار جدید را ببیند.
- readهایی که از نظر زمانی با write هم‌پوشانی دارند ممکن است `0` یا `1` برگردانند، چون نمی‌دانیم هنگام پردازش read، write اعمال شده بوده یا نه. این operationها با write concurrent هستند.

این شرط به‌تنهایی برای توصیف کامل linearizability کافی نیست. اگر readهای هم‌زمان با write بتوانند هرکدام مقدار قدیمی یا جدید را برگردانند، ممکن است خواننده‌ها هنگام انجام write چند بار شاهد رفت‌وبرگشت مقدار میان نسخهٔ قدیمی و جدید باشند. چنین رفتاری با انتظار ما از systemی که «یک copy از data» را شبیه‌سازی می‌کند سازگار نیست.

registerای که readهای هم‌زمان با write می‌توانند از آن مقدار قدیمی یا جدید را برگردانند، **regular register** نامیده می‌شود؛ این تضمین به‌تنهایی linearizability نیست.

برای خطی‌پذیر بودن، قید دیگری لازم است: پس از آنکه هر readای مقدار جدید را برگرداند، تمام readهای بعدی—چه از همان client و چه از clientهای دیگر—نیز باید مقدار جدید را برگردانند.

در یک system خطی‌پذیر فرض می‌کنیم نقطه‌ای در بازهٔ شروع تا پایان write وجود دارد که مقدار `x` به‌صورت اتمیک از `0` به `1` تغییر می‌کند. بنابراین اگر read یک client مقدار جدید `1` را برگرداند، همهٔ readهای بعدی نیز باید `1` را ببینند؛ حتی اگر write هنوز تمام نشده باشد. مثلاً اگر A نخستین clientی باشد که `1` را می‌خواند و بلافاصله پس از پایان read آن، B یک read جدید شروع کند، read مربوط به B هم باید `1` برگرداند. این وضعیت همان انتظار Bob پس از شنیدن نتیجه از Alice است.

می‌توان نمودار زمانی را طوری دقیق‌تر کرد که هر operation در یک نقطهٔ زمانی اتمیک‌شدنش را نشان دهد. علاوه بر read و write، نوع operation دیگری هم داریم:

- `cas(x, vold, vnew) ⇒ r` یک operation اتمیک **compare-and-set** است. اگر مقدار فعلی `x` برابر `vold` باشد، آن را اتمیک به `vnew` تغییر می‌دهد. اگر `x ≠ vold` باشد، register بدون تغییر می‌ماند و operation خطا برمی‌گرداند. `r` پاسخ database، یعنی `ok` یا `error`، است.

در نمودار، هر operation با یک خط عمودی درون bar آن مشخص می‌شود که لحظهٔ اجرای فرضی آن را نشان می‌دهد. این نقاط به‌ترتیب به هم وصل می‌شوند و باید یک دنبالهٔ معتبر از readها و writeهای register بسازند؛ هر read باید مقداری را برگرداند که آخرین write تنظیم کرده است. شرط linearizability این است که خط اتصال این نقاط همیشه در زمان رو به جلو حرکت کند و هرگز به عقب برنگردد. به این ترتیب، recency guarantee برقرار می‌ماند: پس از آنکه valueای نوشته یا خوانده شد، تمام readهای بعدی همان value را می‌بینند تا وقتی که دوباره overwrite شود.

چند نکته دربارهٔ این مدل مهم است:

- ممکن است B ابتدا درخواست read مربوط به `x` را بفرستد، سپس D درخواست کند `x` را روی `0` بگذارد و بعد A درخواست کند آن را روی `1` بگذارد؛ بااین‌حال read مربوط به B می‌تواند `1` برگرداند. این یعنی database ابتدا write از D، سپس write از A و در پایان read از B را پردازش کرده است. با اینکه ترتیب پردازش با ترتیب ارسال فرق دارد، این ترتیب مجاز است، چون سه request با هم concurrent هستند؛ شاید read از B در network کمی تأخیر خورده و پس از دو write به database رسیده باشد.
- ممکن است read مربوط به B مقدار `1` را پیش از آن برگرداند که A response موفقیت write را از database دریافت کند. این به معنای خوانده‌شدن مقدار پیش از نوشتن آن نیست؛ ممکن است فقط response مربوط به A در network تأخیر کرده باشد.
- این مدل transaction isolation را فرض نمی‌کند؛ client دیگری می‌تواند هر لحظه value را تغییر دهد. مثلاً C ابتدا `1` و سپس `2` را می‌خواند، چون B بین دو read مقدار را تغییر داده است. operation اتمیک `cas` برای بررسی این به‌کار می‌رود که client دیگری هم‌زمان مقدار را تغییر نداده باشد. درخواست‌های `cas` از B و C موفق می‌شوند، اما درخواست D شکست می‌خورد، چون زمانی که database آن را پردازش می‌کند، `x` دیگر `0` نیست.
- آخرین read از سوی B خطی‌پذیر نیست. این operation با write از نوع `cas` متعلق به C هم‌زمان است که `x` را از `2` به `4` تغییر می‌دهد؛ در نبود requestهای دیگر، برگرداندن `2` برای B قابل‌قبول بود. اما A پیش از شروع read مربوط به B، مقدار جدید `4` را خوانده است. بنابراین B مجاز نیست مقداری قدیمی‌تر از مقداری را ببیند که A دیده است؛ باز هم همان وضعیت Alice و Bob رخ می‌دهد.

این‌ها شهود اصلی linearizability هستند؛ تعریف رسمی آن دقیق‌تر است. هرچند از نظر محاسباتی پرهزینه است، می‌توان با ثبت زمان requestها و responseها و بررسی امکان مرتب‌کردن آن‌ها در یک ترتیب ترتیبی معتبر، آزمود که آیا رفتار system خطی‌پذیر است یا نه.

#### Linearizability در برابر Serializability

Linearizability را به‌آسانی با serializability اشتباه می‌گیرند؛ چون هر دو ظاهراً به معنای قابل‌مرتب‌شدن در یک ترتیب ترتیبی‌اند. اما این دو guarantee کاملاً متفاوت‌اند:

- **Serializability** یک isolation property برای transactionهاست. هر transaction می‌تواند چند object (مانند row، document یا record) را بخواند و بنویسد. تضمین می‌کند رفتار transactionها معادل اجرای آن‌ها در یک ترتیب serial باشد؛ یعنی هر transaction پیش از آغاز transaction بعدی کامل شود. این ترتیب لازم نیست با ترتیب واقعی اجرای transactionها یکسان باشد.
- **Linearizability** یک recency guarantee برای read و writeهای یک register، یعنی یک object منفرد، است. operationها را در قالب transaction گروه‌بندی نمی‌کند؛ بنابراین به‌تنهایی جلوی مسائلی مثل write skew را نمی‌گیرد، مگر اینکه اقدام دیگری مانند materializing conflicts انجام شود.

یک database می‌تواند هم serializability و هم linearizability را فراهم کند. این ترکیب **strict serializability** یا **strong one-copy serializability (strong-1SR)** نام دارد. پیاده‌سازی‌های serializability مبتنی بر Two-Phase Locking یا اجرای serial واقعی معمولاً linearizable هم هستند.

اما **Serializable Snapshot Isolation (SSI)** خطی‌پذیر نیست: طراحی آن بر این اساس است که readها از یک consistent snapshot انجام شوند تا contention میان readerها و writerها به‌دلیل lock کاهش یابد. هدف snapshot سازگار این است که writeهای جدیدتر از زمان snapshot را در بر نگیرد؛ بنابراین read از آن snapshot خطی‌پذیر نیست.

### Relying on Linearizability

Linearizability چه وقت مفید است؟ دیدن نتیجهٔ مسابقه مثالی کم‌اهمیت است؛ نتیجه‌ای که چند ثانیه قدیمی باشد احتمالاً زیان واقعی ایجاد نمی‌کند. اما در برخی موارد، linearizability برای درست کار کردن system ضروری است.

#### Locking و leader election

systemی که از single-leader replication استفاده می‌کند باید مطمئن شود واقعاً فقط یک leader دارد، نه چند leader هم‌زمان (split brain). یک راه انتخاب leader استفاده از lock است: هر node که راه‌اندازی می‌شود برای گرفتن lock تلاش می‌کند و node موفق، leader می‌شود. مستقل از نحوهٔ پیاده‌سازی lock، این operation باید linearizable باشد؛ تمام nodeها باید بر سر مالک lock توافق داشته باشند، وگرنه lock فایده‌ای ندارد.

سرویس‌های coordination مانند Apache ZooKeeper و etcd اغلب برای پیاده‌سازی distributed lock و leader election استفاده می‌شوند. آن‌ها با الگوریتم‌های consensus، operationهای خطی‌پذیر و fault-tolerant فراهم می‌کنند. بااین‌حال، پیاده‌سازی درست lock و leader election جزئیات ظریفی دارد؛ برای نمونه، مسئلهٔ fencing که در بخش «The leader and the lock» بررسی شد. کتابخانه‌هایی مانند Apache Curator با ارائهٔ recipeهای سطح‌بالاتر روی ZooKeeper کمک می‌کنند. با وجود این، یک linearizable storage service زیرساخت پایهٔ این کارهای coordination است.

نکتهٔ اجرایی: ZooKeeper و etcd به‌طور دقیق‌تر writeهای linearizable فراهم می‌کنند، اما readها ممکن است stale باشند؛ چون به‌صورت پیش‌فرض هر replica می‌تواند آن‌ها را پاسخ دهد. برای linearizable read باید صریحاً آن را درخواست کرد: etcd این قابلیت را quorum read می‌نامد و در ZooKeeper باید پیش از read، `sync()` را فراخوانی کرد.

در بعضی distributed databaseها، مانند Oracle Real Application Clusters (RAC)، distributed locking در سطح بسیار ریزتری هم استفاده می‌شود. RAC برای هر disk page یک lock دارد و چند node به یک disk storage system مشترک دسترسی دارند. چون این lockهای خطی‌پذیر در مسیر حیاتی اجرای transaction قرار دارند، RAC معمولاً برای ارتباط میان database nodeها از cluster interconnect اختصاصی استفاده می‌کند.

#### Constraintها و uniqueness guaranteeها

uniqueness constraint در databaseها رایج است: مثلاً username یا email باید یک user را به‌طور یکتا مشخص کند؛ یا در file storage service نباید دو file با path و filename یکسان وجود داشته باشد. اگر بخواهید هنگام نوشتن data این constraint را اعمال کنید—طوری که اگر دو نفر هم‌زمان بخواهند user یا fileای با نام یکسان بسازند، یکی از آن‌ها error بگیرد—به linearizability نیاز دارید.

این وضعیت شبیه lock است: هنگام ثبت‌نام user در service، می‌توان تصور کرد که او روی username انتخابی خود lock می‌گیرد. این operation همچنین شبیه atomic compare-and-set است که username را به ID کاربری اختصاص می‌دهد که آن را claim کرده، مشروط بر اینکه username از قبل گرفته نشده باشد.

مسائل مشابهی هنگام تضمین این شرط‌ها پیش می‌آیند: موجودی حساب بانکی هیچ‌وقت منفی نشود؛ بیش از موجودی انبار کالا فروخته نشود؛ یا دو نفر هم‌زمان یک صندلی در هواپیما یا سالن تئاتر رزرو نکنند. همهٔ این constraintها نیازمند یک value به‌روز و یگانه‌اند—مانند موجودی حساب، تعداد کالا یا وضعیت اشغال صندلی—که تمام nodeها بر سر آن توافق داشته باشند.

در applicationهای واقعی گاهی می‌توان این constraintها را سهل‌گیرانه‌تر تفسیر کرد؛ برای مثال، اگر پرواز بیش از ظرفیت فروخته شد، می‌توان مسافران را به پرواز دیگری منتقل کرد و بابت زحمتشان غرامت داد. در چنین مواردی شاید linearizability لازم نباشد. اما hard uniqueness constraint، مانند constraint رایج در relational databaseها، به linearizability نیاز دارد. برخی constraintهای دیگر، مانند foreign key یا constraintهای attribute، بدون نیاز به linearizability قابل‌پیاده‌سازی‌اند.

#### وابستگی زمانی میان channelها

به نکته‌ای در مثال Alice و Bob توجه کنید: اگر Alice نتیجه را فریاد نزده بود، Bob نمی‌فهمید نتیجهٔ query او قدیمی است. او چند ثانیه بعد دوباره صفحه را refresh می‌کرد و سرانجام نتیجهٔ نهایی را می‌دید. نقض linearizability فقط به‌خاطر وجود یک communication channel اضافی در system آشکار شد: صدای Alice که به گوش Bob رسید.

وضعیت مشابهی در systemهای کامپیوتری هم پیش می‌آید. فرض کنید وب‌سایتی دارید که userها می‌توانند در آن عکس upload کنند و یک background process عکس‌ها را برای download سریع‌تر به resolution پایین‌تری تبدیل می‌کند و thumbnail می‌سازد.

image resizer باید صریحاً برای انجام کار resize مطلع شود و این دستور از web server از طریق message queue به آن می‌رسد. web server کل عکس را در queue قرار نمی‌دهد، چون بیشتر message brokerها برای messageهای کوچک طراحی شده‌اند و حجم عکس ممکن است چندین megabyte باشد. در عوض، ابتدا عکس در file storage service نوشته می‌شود و پس از تکمیل write، دستور اجرای resize در queue قرار می‌گیرد.

اگر file storage service خطی‌پذیر باشد، system باید درست کار کند. در غیر این صورت احتمال race condition وجود دارد: ممکن است message queue سریع‌تر از replication داخلی storage service عمل کند. در آن صورت، وقتی resizer عکس را دریافت می‌کند، ممکن است نسخه‌ای قدیمی از آن یا اصلاً هیچ فایلی را نبیند. اگر نسخهٔ قدیمی پردازش شود، نسخهٔ کامل و نسخهٔ resizeشدهٔ عکس در file storage برای همیشه ناسازگار می‌مانند.

این مشکل از وجود دو communication channel متفاوت میان web server و resizer ناشی می‌شود: file storage و message queue. بدون recency guarantee مربوط به linearizability، میان این دو channel امکان race condition وجود دارد. این وضعیت مانند مثال Alice و Bob است که در آن میان دو channel—replication در database و ارتباط صوتی واقعی میان Alice و Bob—race condition رخ داد.

linearizability تنها راه جلوگیری از این race condition نیست، اما ساده‌ترین راه برای فهم آن است. اگر communication channel اضافی را کنترل می‌کنید—مانند message queue، برخلاف ارتباط Alice و Bob—می‌توانید از روش‌های جایگزینی شبیه read-your-writes استفاده کنید؛ البته به بهای پیچیدگی بیشتر.

### Implementing Linearizable Systems

با توجه به کاربردهای linearizability، ببینیم چطور می‌توان systemی با semantics خطی‌پذیر ساخت. از آنجا که linearizability اساساً یعنی «طوری رفتار کن که انگار فقط یک copy از data وجود دارد و تمام operationهای آن atomic هستند»، ساده‌ترین پاسخ استفادهٔ واقعی از فقط یک copy از data است. اما این روش faultها را تحمل نمی‌کند: اگر node نگهدارندهٔ آن copy از کار بیفتد، data از بین می‌رود یا دست‌کم تا بازگشت node در دسترس نخواهد بود.

رایج‌ترین روش برای fault-tolerant کردن system استفاده از replication است. روش‌های replication فصل ۵ را از نظر امکان خطی‌پذیر بودن مقایسه کنیم:

- **Single-leader replication (بالقوه خطی‌پذیر):** در این system، leader نسخهٔ اصلی data را برای writeها نگه می‌دارد و followerها نسخه‌های پشتیبان را روی nodeهای دیگر دارند. اگر readها از leader یا followerهایی انجام شوند که به‌صورت synchronous به‌روز می‌شوند، امکان linearizability وجود دارد. اما همهٔ single-leader databaseها واقعاً خطی‌پذیر نیستند؛ گاهی طراحی آن‌ها (مثلاً استفاده از snapshot isolation) مانع می‌شود و گاهی concurrency bugها.

  read از leader متکی به این فرض است که دقیقاً می‌دانیم leader کدام node است. ممکن است nodeای تصور کند leader است، درحالی‌که واقعاً چنین نیست؛ اگر leader خیالی همچنان requestها را پاسخ دهد، احتمال نقض linearizability زیاد است. در asynchronous replication، failover حتی ممکن است writeهای commitشده را از دست بدهد که هم durability و هم linearizability را نقض می‌کند.

  اگر database تک‌رهبر را partition یا shard کنید، وجود یک leader جداگانه برای هر partition، linearizability را تغییر نمی‌دهد؛ چون این guarantee دربارهٔ یک object منفرد است. اما transactionهای میان چند partition مسئلهٔ دیگری هستند.

- **Consensus algorithmها (خطی‌پذیر):** برخی consensus algorithmها شباهت‌هایی به single-leader replication دارند، اما protocolهای consensus سازوکارهایی برای جلوگیری از split brain و stale replica دارند. همین جزئیات باعث می‌شود بتوانند storage خطی‌پذیر را به‌شکل ایمن پیاده‌سازی کنند؛ ZooKeeper و etcd از همین روش استفاده می‌کنند.

- **Multi-leader replication (غیرخطی‌پذیر):** systemهای multi-leader عموماً linearizable نیستند، چون writeها را هم‌زمان روی چند node پردازش و سپس به‌صورت asynchronous به nodeهای دیگر replicate می‌کنند. در نتیجه ممکن است writeهای متعارض تولید شوند که نیاز به conflict resolution دارند. این conflictها حاصل نبود یک copy واحد از data هستند.

- **Leaderless replication (احتمالاً غیرخطی‌پذیر):** گاهی ادعا می‌شود در systemهای سبک Dynamo می‌توان با الزام به quorum read و write، یعنی `w + r > n`، به «strong consistency» رسید. بسته به پیکربندی دقیق quorumها و تعریف strong consistency، این ادعا کاملاً درست نیست.

  روش‌های conflict resolution از نوع **last write wins** که بر ساعت‌های time-of-day تکیه می‌کنند—مثلاً در Cassandra—تقریباً قطعاً nonlinearizable هستند؛ چون به‌دلیل clock skew نمی‌توان تضمین کرد timestampهای ساعت با ترتیب واقعی eventها سازگار باشند. sloppy quorumها هم امکان linearizability را از بین می‌برند. حتی با strict quorum نیز رفتار nonlinearizable ممکن است رخ دهد.

#### Linearizability و quorumها

در نگاه اول به نظر می‌رسد strict quorum برای read و write باید در مدل Dynamo خطی‌پذیر باشد. اما با network delayهای متغیر امکان race condition وجود دارد.

فرض کنید مقدار اولیهٔ `x` برابر `0` باشد و writer آن را با ارسال write به هر سه replica تغییر دهد (`n = 3`, `w = 3`). هم‌زمان، client A از quorum دو node می‌خواند (`r = 2`) و در یکی از آن‌ها مقدار جدید `1` را می‌بیند. هم‌زمان با write، client B از quorum متفاوتی شامل دو node می‌خواند و از هر دو مقدار قدیمی `0` را می‌گیرد.

شرط quorum برقرار است (`w + r > n`)، اما این اجرا خطی‌پذیر نیست: request مربوط به B پس از پایان request مربوط به A آغاز می‌شود، ولی B مقدار قدیمی را می‌گیرد، درحالی‌که A مقدار جدید را دیده است. این دوباره همان وضعیت Alice و Bob است.

با کاهش performance می‌توان quorumهای سبک Dynamo را خطی‌پذیر کرد: reader باید پیش از برگرداندن نتیجه به application، read repair را به‌صورت synchronous انجام دهد؛ writer نیز پیش از ارسال writeهای خود باید جدیدترین state را از quorumای از nodeها بخواند. Riak به‌دلیل جریمهٔ performance، read repair را به‌صورت synchronous انجام نمی‌دهد. Cassandra در quorum readها تا پایان read repair صبر می‌کند، اما اگر چند write هم‌زمان روی یک key انجام شوند، به‌دلیل last-write-wins conflict resolution، linearizability را از دست می‌دهد.

افزون بر این، با این روش فقط operationهای خطی‌پذیر read و write را می‌توان پیاده‌سازی کرد؛ operation خطی‌پذیر compare-and-set قابل‌پیاده‌سازی نیست، چون به consensus algorithm نیاز دارد. در نتیجه، امن‌ترین فرض این است که system بدون leader و با replication سبک Dynamo، linearizability فراهم نمی‌کند.

### The Cost of Linearizability

چون بعضی روش‌های replication می‌توانند linearizability را فراهم کنند و بعضی نمی‌توانند، بررسی دقیق‌تر مزایا و هزینه‌های آن مفید است.

در فصل ۵ دیدیم که روش‌های مختلف replication کاربردهای متفاوتی دارند؛ برای مثال، multi-leader replication اغلب برای replication میان چند datacenter انتخاب مناسبی است. حالا قطع‌شدن ارتباط network میان دو datacenter را در نظر بگیرید. فرض می‌کنیم network داخل هر datacenter فعال است و clientها می‌توانند به datacenterها دسترسی داشته باشند، اما datacenterها قادر به ارتباط با یکدیگر نیستند.

با multi-leader database، هر datacenter می‌تواند به کار عادی خود ادامه دهد: writeهای یک datacenter به‌صورت asynchronous به دیگری replicate می‌شوند، پس فقط در صف می‌مانند تا پس از برقراری دوبارهٔ اتصال، مبادله شوند.

اما اگر single-leader replication داشته باشیم، leader باید در یکی از datacenterها باشد. تمام writeها و readهای خطی‌پذیر باید به leader فرستاده شوند؛ بنابراین requestهای clientهای متصل به datacenterهای follower باید به‌صورت synchronous از network عبور کنند تا به datacenter leader برسند.

اگر network میان datacenterها در این setup قطع شود، clientهای متصل به datacenterهای follower نمی‌توانند به leader برسند؛ در نتیجه نه می‌توانند write انجام دهند و نه read خطی‌پذیر. آن‌ها همچنان می‌توانند از follower بخوانند، اما ممکن است نتیجه قدیمی و nonlinearizable باشد. اگر application به read و write خطی‌پذیر نیاز داشته باشد، قطع network باعث unavailable شدن application در datacenterهایی می‌شود که به leader دسترسی ندارند. clientهایی که مستقیماً به datacenter leader وصل‌اند مشکلی نخواهند داشت؛ اما برای clientهایی که فقط به datacenter follower دسترسی دارند، outage تا زمان ترمیم اتصال ادامه پیدا می‌کند.

#### قضیهٔ CAP

این مسئله فقط پیامد single-leader و multi-leader replication نیست: هر database خطی‌پذیری، صرف‌نظر از نحوهٔ پیاده‌سازی، با آن روبه‌روست. موضوع هم منحصر به deployment چند datacenter نیست و در هر network غیرقابل‌اعتماد، حتی داخل یک datacenter، رخ می‌دهد. trade-off چنین است:

- اگر application به linearizability نیاز داشته باشد و بعضی replicaها به‌دلیل مشکل network از بقیه جدا شوند، replicaهای جداشده نمی‌توانند requestها را پردازش کنند. باید تا رفع مشکل network صبر کنند یا error برگردانند؛ در هر دو حالت unavailable می‌شوند.
- اگر application به linearizability نیاز نداشته باشد، می‌توان آن را طوری طراحی کرد که هر replica مستقلاً requestها را پردازش کند، حتی اگر از replicaهای دیگر جدا شده باشد؛ مثلاً با multi-leader. در این حالت application هنگام مشکل network می‌تواند available بماند، اما رفتارش linearizable نیست.

بنابراین applicationهایی که به linearizability نیاز ندارند، در برابر مشکل‌های network تحمل بیشتری دارند. این بینش به‌طور گسترده با نام **قضیهٔ CAP** شناخته می‌شود؛ نامی که از Eric Brewer در سال ۲۰۰۰ گرفته شده، هرچند طراحان distributed databaseها از دههٔ ۱۹۷۰ این trade-off را می‌شناختند.

CAP در ابتدا به‌عنوان یک قاعدهٔ سرانگشتی و بدون تعریف‌های دقیق مطرح شد تا بحث دربارهٔ trade-offهای databaseها را آغاز کند. در آن زمان، بسیاری از distributed databaseها روی ارائهٔ semantics خطی‌پذیر در clusterهایی از machineها با storage مشترک متمرکز بودند. CAP مهندسان database را به بررسی فضای طراحی گسترده‌تری از systemهای distributed و shared-nothing تشویق کرد؛ systemهایی که برای پیاده‌سازی web serviceهای بزرگ‌مقیاس مناسب‌تر بودند. CAP در این تغییر فرهنگ و رشد فناوری‌های جدید database از میانهٔ دههٔ ۲۰۰۰—که NoSQL نامیده شدند—نقش تاریخی مهمی داشت.

#### چرا CAP چندان راهگشا نیست

گاهی CAP را این‌طور بیان می‌کنند: Consistency، Availability و Partition tolerance؛ از میان این سه، دو مورد را انتخاب کنید. این بیان گمراه‌کننده است، چون network partition نوعی fault است و چیزی نیست که بتوان انتخاب کرد؛ چه بخواهیم چه نخواهیم رخ می‌دهد.

وقتی network درست کار می‌کند، system می‌تواند هم consistency (یعنی linearizability) و هم availability کامل داشته باشد. هنگام وقوع network fault باید میان linearizability و availability کامل انتخاب کرد. بنابراین بیان بهتر CAP این است: هنگام partition، یا consistent باش یا available. network قابل‌اعتمادتر باعث می‌شود کمتر با این انتخاب روبه‌رو شویم، اما در نهایت گریزناپذیر است.

در بحث‌های CAP تعریف‌های متناقضی برای availability وجود دارد و صورت‌بندی theorem با معنای رایج این واژه هم‌خوان نیست. بسیاری از systemهایی که «highly available» یا fault-tolerant نامیده می‌شوند، تعریف خاص CAP از availability را برآورده نمی‌کنند. در مجموع، CAP محل سوءبرداشت و سردرگمی زیادی است و به فهم بهتر systemها کمک چندانی نمی‌کند؛ بنابراین بهتر است از آن به‌عنوان ابزار طراحی استفاده نشود.

CAP آن‌گونه که به‌صورت رسمی تعریف شده، دامنهٔ بسیار محدودی دارد: فقط یک consistency model، یعنی linearizability، و یک نوع fault، یعنی network partition—nodeهایی که زنده‌اند اما از یکدیگر جدا شده‌اند—را بررسی می‌کند. دربارهٔ network delay، nodeهای ازکارافتاده یا trade-offهای دیگر چیزی نمی‌گوید. پس CAP با وجود اثر تاریخی‌اش، برای طراحی systemها ارزش عملی کمی دارد. نتیجه‌های ناممکن‌بودن جالب‌تری در distributed systemها وجود دارند و CAP اکنون با نتیجه‌های دقیق‌تری جایگزین شده است؛ امروزه بیشتر اهمیت تاریخی دارد.

#### Linearizability و network delay

با اینکه linearizability guarantee مفیدی است، در عمل systemهای نسبتاً کمی واقعاً خطی‌پذیرند. برای نمونه، حتی RAM در CPUهای چند‌هسته‌ای مدرن هم linearizable نیست: اگر threadای روی یک CPU core در آدرسی از memory بنویسد و threadای روی core دیگر کمی بعد همان آدرس را بخواند، تضمینی نیست که مقدار نوشته‌شده را ببیند؛ مگر آنکه از memory barrier یا fence استفاده شود.

دلیل این رفتار آن است که هر CPU core، memory cache و store buffer خودش را دارد. دسترسی به memory به‌طور پیش‌فرض ابتدا به cache می‌رود و تغییرها به‌صورت asynchronous در main memory نوشته می‌شوند. دسترسی به cache بسیار سریع‌تر از دسترسی به main memory است و این ویژگی برای performance مناسب CPUهای مدرن ضروری است. اما اکنون چند copy از data وجود دارد—یکی در main memory و احتمالاً چند copy دیگر در cacheهای مختلف—و این copyها به‌صورت asynchronous به‌روز می‌شوند؛ در نتیجه linearizability از بین می‌رود.

چرا چنین trade-offی را می‌پذیریم؟ قضیهٔ CAP توضیحی برای مدل consistency حافظه در سیستم چند‌هسته‌ای نیست: درون یک computer معمولاً ارتباط را قابل‌اعتماد فرض می‌کنیم و انتظار نداریم یک CPU core پس از قطع ارتباط با بقیهٔ computer به کار عادی ادامه دهد. دلیل کنارگذاشتن linearizability در اینجا performance است، نه fault tolerance.

همین موضوع دربارهٔ بسیاری از distributed databaseهایی صدق می‌کند که linearizability ارائه نمی‌دهند: انگیزهٔ اصلی‌شان افزایش performance است، نه چندان افزایش fault tolerance. Linearizability کند است و این کندی همیشه وجود دارد، نه فقط هنگام network fault.

آیا می‌توان storage خطی‌پذیر را کارآمدتر پیاده‌سازی کرد؟ به‌نظر می‌رسد پاسخ منفی است: Attiya و Welch ثابت کرده‌اند که اگر linearizability بخواهید، response time مربوط به read و write دست‌کم متناسب با میزان عدم‌قطعیت network delay است. در networkهایی با delay بسیار متغیر—مانند بیشتر computer networkها—response time مربوط به read و write خطی‌پذیر ناگزیر زیاد خواهد بود. الگوریتم سریع‌تری برای linearizability وجود ندارد؛ اما consistency modelهای ضعیف‌تر می‌توانند بسیار سریع‌تر باشند. این trade-off برای systemهای حساس به latency اهمیت زیادی دارد. در فصل ۱۲ روش‌هایی را بررسی می‌کنیم که بدون قربانی‌کردن correctness، از linearizability اجتناب می‌کنند.

## Key Terms

- `Linearizability` — guaranteeای که هر operation را چنان جلوه می‌دهد که گویی در لحظه‌ای اتمیک میان آغاز و پایانش، روی یک copy واحد از data اجرا شده است؛ ترتیب real-time operationهای غیرهم‌پوشان نیز حفظ می‌شود.
- `Linearizable System` — systemی که read و writeهایش با semantics خطی‌پذیر سازگارند و پس از مشاهدهٔ یک value جدید، readهای بعدی نمی‌توانند value قدیمی‌تری ببینند.
- `Real-Time Ordering` — قید ترتیب که می‌گوید اگر یک operation پیش از شروع operation دیگری تمام شود، اثر اولی باید در ترتیب مشاهده‌شده پیش از دومی باشد.
- `Atomic Compare-and-Set (CAS)` — operationای که مقدار را فقط در صورت برابر بودن با مقدار مورد انتظار، به‌صورت اتمیک تغییر می‌دهد؛ برای جلوگیری از update بر مبنای state قدیمی کاربرد دارد.
- `Strict Serializability` — ترکیب serializability و linearizability؛ transactionها هم معادل یک ترتیب serial اجرا می‌شوند و هم ترتیب real-time عملیات رعایت می‌شود.
- `Quorum` — زیرمجموعه‌ای از replicaها که برای read یا write باید پاسخ دهند؛ برقرار بودن شرط quorum به‌تنهایی linearizability را تضمین نمی‌کند.
- `Consensus` — سازوکار توافق nodeها بر سر تصمیمی مشترک، مانند انتخاب leader؛ برای پیاده‌سازی operationهای خطی‌پذیر و جلوگیری از split brain به‌کار می‌رود.
- `Network Partition` — وضعیتی که nodeها همچنان فعال‌اند اما به‌دلیل اختلال network نمی‌توانند با بخشی از replicaها ارتباط بگیرند؛ می‌تواند انتخاب میان linearizability و availability را تحمیل کند.
- `Cross-Channel Race Condition` — raceای که وقتی دو مسیر مستقل ارتباطی، مانند storage و message queue، با delayهای متفاوت پیش می‌روند ایجاد می‌شود و ممکن است consumer پیش از قابل‌مشاهده‌شدن write، notification را دریافت کند.
- `CAP Theorem` — نتیجه‌ای دربارهٔ trade-off هنگام network partition: system نمی‌تواند در آن وضعیت هم linearizability و هم availability کامل را تضمین کند؛ این theorem به‌تنهایی ابزار جامعی برای طراحی system نیست.
- `Memory Barrier` — دستور یا سازوکار هماهنگ‌سازی دسترسی‌های حافظه که ترتیب مشاهدهٔ read و writeها را میان CPU coreها کنترل می‌کند.
- `Performance Overhead` — هزینهٔ latency و هماهنگی لازم برای حفظ linearizability، به‌ویژه وقتی operationها باید میان nodeها توافق یا منتظر پاسخ بمانند.

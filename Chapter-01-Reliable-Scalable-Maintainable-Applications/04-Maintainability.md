# Chapter 1 — Reliable, Scalable, and Maintainable Applications

## Maintainability

همه می‌دانیم که بخش عمدهٔ هزینهٔ software صرف development اولیه نمی‌شود، بلکه مربوط به maintenance مداوم آن است: رفع bugها، operational نگه داشتن سیستم‌ها، بررسی failureها، سازگار کردن سیستم با platformهای جدید، تغییر آن برای `use case`های تازه، بازپرداخت `technical debt` و اضافه کردن featureهای جدید.

بااین‌حال، متأسفانه بسیاری از افرادی که روی software systemها کار می‌کنند از maintenanceِ به‌اصطلاح `legacy system`ها خوششان نمی‌آید؛ شاید چون باید اشتباه‌های دیگران را اصلاح کنند، با platformهایی کار کنند که اکنون قدیمی شده‌اند، یا با سیستم‌هایی کار کنند که مجبور شده‌اند کارهایی را انجام دهند که از ابتدا برای آن‌ها طراحی نشده بودند. هر legacy system به شیوهٔ خاص خودش ناخوشایند است؛ بنابراین ارائهٔ توصیه‌های عمومی برای کار با همهٔ آن‌ها دشوار است.

با وجود این، می‌توانیم و باید software را طوری طراحی کنیم که امیدوار باشیم دردسرهای دورهٔ maintenance را به حداقل برساند و در نتیجه خودمان software legacy جدیدی تولید نکنیم. برای این منظور، توجه ویژه‌ای به سه اصل طراحی برای software systemها خواهیم داشت:

**Operability**

کار را برای تیم‌های operations آسان کنیم تا بتوانند سیستم را روان و پایدار اجرا کنند.

**Simplicity**

با حذف هرچه بیشتر complexity از سیستم، درک آن را برای engineerهای جدید آسان کنیم. توجه کنید که منظور، سادگی user interface نیست.

**Evolvability**

تغییر دادن سیستم در آینده را برای engineerها آسان کنیم تا بتوانند با تغییر requirementها، سیستم را برای `use case`های پیش‌بینی‌نشده سازگار کنند. از این مفهوم با نام‌های `extensibility`، `modifiability` یا `plasticity` نیز یاد می‌شود.

همانند reliability و scalability، برای دستیابی به این هدف‌ها راه‌حل ساده و یک‌مرحله‌ای وجود ندارد. در عوض، تلاش می‌کنیم هنگام فکر کردن دربارهٔ سیستم‌ها، operability، simplicity و evolvability را در نظر داشته باشیم.

### Operability: Making Life Easy for Operations

گفته شده است که «operations خوب اغلب می‌تواند محدودیت‌های software بد یا ناقص را دور بزند، اما software خوب نمی‌تواند با operations بد به‌طور reliable اجرا شود» [12]. هرچند بعضی جنبه‌های operations را می‌توان و باید automated کرد، در نهایت این انسان‌ها هستند که باید آن automation را در ابتدا راه‌اندازی کنند و مطمئن شوند که درست کار می‌کند.

تیم‌های operations برای روان اجرا شدن software system حیاتی‌اند. یک تیم operations خوب معمولاً مسئولیت‌های زیر و موارد دیگری را بر عهده دارد [29]:

- health سیستم را monitoring کند و اگر سیستم وارد وضعیت نامناسبی شد، service را به‌سرعت restore کند.
- علت مشکل‌هایی مانند system failure یا degraded performance را پیدا کند.
- software و platformها، از جمله security patchها، را به‌روز نگه دارد.
- اثر متقابل systemهای مختلف بر یکدیگر را زیر نظر داشته باشد تا پیش از آنکه یک change مشکل‌ساز آسیب ایجاد کند، بتوان از آن جلوگیری کرد.
- مشکل‌های آینده را پیش‌بینی و پیش از وقوع حل کند؛ برای مثال، با `capacity planning`.
- practiceها و toolهای مناسبی برای deployment، `configuration management` و موارد دیگر ایجاد کند.
- taskهای پیچیدهٔ maintenance را انجام دهد؛ مانند انتقال یک application از یک platform به platform دیگر.
- هنگام اعمال configuration changeها، security سیستم را حفظ کند.
- processهایی تعریف کند که operations را predictable کنند و به پایدار ماندن `production environment` کمک کنند.
- دانش سازمان دربارهٔ سیستم را حفظ کند، حتی وقتی افراد مختلف سازمان را ترک می‌کنند یا افراد جدید جای آن‌ها می‌آیند.

Operability خوب یعنی taskهای routine را آسان کنیم تا تیم operations بتواند تلاش خود را روی فعالیت‌های high-value متمرکز کند. Data systemها می‌توانند کارهای مختلفی برای آسان کردن taskهای routine انجام دهند، از جمله:

- با monitoring مناسب، visibility خوبی نسبت به رفتار runtime و internals سیستم فراهم کنند.
- از automation و integration با toolهای استاندارد پشتیبانی خوبی داشته باشند.
- از وابستگی به machineهای منفرد دوری کنند؛ به‌گونه‌ای که بتوان machineها را برای maintenance از مدار خارج کرد، در حالی که کل سیستم بدون interruption به کار خود ادامه می‌دهد.
- documentation مناسب و یک `operational model` قابل‌فهم ارائه کنند؛ مدلی از جنس «اگر X را انجام دهم، Y رخ می‌دهد».
- behavior پیش‌فرض مناسبی داشته باشند، اما در صورت نیاز آزادی override کردن defaultها را نیز به administratorها بدهند.
- در جاهایی که مناسب است `self-healing` داشته باشند، اما در صورت نیاز، کنترل دستی state سیستم را نیز در اختیار administratorها بگذارند.
- behavior قابل‌پیش‌بینی داشته باشند و surpriseها را به حداقل برسانند.

### Simplicity: Managing Complexity

پروژه‌های کوچک software ممکن است codeهایی لذت‌بخش، ساده و expressive داشته باشند؛ اما با بزرگ‌تر شدن پروژه، code اغلب بسیار complex و درک آن دشوار می‌شود. این complexity سرعت همهٔ افرادی را که باید روی سیستم کار کنند کاهش می‌دهد و در نتیجه هزینهٔ maintenance را بیشتر می‌کند. پروژهٔ softwareای که در complexity گرفتار شده باشد، گاهی `big ball of mud` نامیده می‌شود [30].

complexity می‌تواند نشانه‌های مختلفی داشته باشد: انفجار `state space`، `tight coupling` میان moduleها، dependencyهای درهم‌تنیده، نام‌گذاری و terminology ناسازگار، hackهایی با هدف حل مشکل‌های performance، special-caseهایی برای دور زدن مشکل‌های بخش‌های دیگر و موارد متعدد دیگر. دربارهٔ این موضوع پیش‌تر مطالب زیادی نوشته شده است [31, 32, 33].

وقتی complexity، maintenance را دشوار می‌کند، budgetها و scheduleها اغلب از حد تعیین‌شده فراتر می‌روند. در software complex، هنگام ایجاد change خطر وارد کردن bug نیز بیشتر است: وقتی درک و reasoning دربارهٔ سیستم برای developerها دشوارتر باشد، assumptionهای پنهان، پیامدهای ناخواسته و interactionهای پیش‌بینی‌نشده راحت‌تر نادیده گرفته می‌شوند. برعکس، کاهش complexity، maintainability software را به‌طور چشمگیری بهتر می‌کند؛ بنابراین simplicity باید یکی از هدف‌های اصلی سیستم‌هایی باشد که می‌سازیم.

ساده‌تر کردن سیستم لزوماً به معنای کاهش functionality آن نیست؛ این کار می‌تواند به معنای حذف `accidental complexity` نیز باشد. Moseley و Marks [32] complexity را accidental می‌دانند اگر در مسئله‌ای که software حل می‌کند - از دید userها - ذاتی نباشد و فقط از implementation ناشی شود.

یکی از بهترین toolهایی که برای حذف accidental complexity در اختیار داریم `abstraction` است. یک abstraction خوب می‌تواند جزئیات implementation زیادی را پشت یک `façade` تمیز و ساده برای فهمیدن پنهان کند. یک abstraction خوب همچنین می‌تواند برای طیف گسترده‌ای از applicationهای مختلف استفاده شود. این reuse نه‌تنها از پیاده‌سازی دوبارهٔ یک چیز مشابه در چند محل کارآمدتر است، بلکه به software باکیفیت‌تری نیز منجر می‌شود؛ چون بهبودهای quality در component abstractشده، به نفع همهٔ applicationهایی است که از آن استفاده می‌کنند.

برای مثال، high-level programming languageها abstractionهایی هستند که machine code، registerهای CPU و syscallها را پنهان می‌کنند. `SQL` abstractionی است که data structureهای پیچیدهٔ روی disk و در memory، requestهای concurrent از clientهای دیگر و inconsistencyهای پس از crash را پنهان می‌کند. البته وقتی با یک high-level language برنامه‌نویسی می‌کنیم، همچنان از machine code استفاده می‌کنیم؛ فقط مستقیماً با آن کار نمی‌کنیم، چون abstraction زبان برنامه‌نویسی ما را از فکر کردن دربارهٔ آن بی‌نیاز می‌کند.

بااین‌حال، پیدا کردن abstractionهای خوب بسیار دشوار است. در حوزهٔ distributed systemها، با اینکه algorithmهای خوب زیادی وجود دارد، هنوز به‌روشنی نمی‌دانیم چگونه آن‌ها را در قالب abstractionهایی package کنیم که به ما کمک کنند complexity سیستم را در سطح قابل‌مدیریتی نگه داریم.

در سراسر این کتاب، به دنبال abstractionهای خوبی خواهیم بود که اجازه دهند بخش‌هایی از یک سیستم بزرگ را به componentهایی well-defined و reusable تبدیل کنیم.

### Evolvability: Making Change Easy

بسیار بعید است که requirementهای سیستم شما برای همیشه بدون تغییر باقی بمانند. احتمال بسیار بیشتری دارد که requirementها دائماً در حال تغییر باشند: واقعیت‌های جدیدی یاد می‌گیرید، use caseهایی که قبلاً پیش‌بینی نشده بودند پدیدار می‌شوند، اولویت‌های business تغییر می‌کنند، userها featureهای جدیدی درخواست می‌کنند، platformهای جدید جای platformهای قدیمی را می‌گیرند، requirementهای قانونی یا regulatory تغییر می‌کنند، رشد سیستم باعث changeهای معماری می‌شود و موارد دیگر.

از نظر processهای سازمانی، الگوهای کاری `Agile` چارچوبی برای سازگار شدن با change فراهم می‌کنند. جامعهٔ Agile همچنین toolها و patternهای فنی‌ای توسعه داده است که هنگام توسعهٔ software در محیطی با changeهای مکرر مفیدند؛ مانند `test-driven development (TDD)` و `refactoring`.

بیشتر بحث‌ها دربارهٔ این تکنیک‌های Agile روی scaleای نسبتاً کوچک و local تمرکز دارند؛ مثلاً چند source code file درون یک application. در این کتاب، به دنبال راه‌هایی برای افزایش agility در سطح یک data system بزرگ‌تر هستیم؛ سیستمی که شاید از چند application یا service مختلف با ویژگی‌های متفاوت تشکیل شده باشد. برای مثال، چگونه می‌توان architecture مربوط به assemble کردن home timeline در Twitter را - که در بخش «Describing Load» توضیح داده شد - از روش ۱ به روش ۲ `refactor` کرد؟

سهولت modify کردن یک data system و سازگار کردن آن با requirementهای در حال تغییر، ارتباط نزدیکی با simplicity و abstractionهای آن دارد: سیستم‌های ساده و قابل‌فهم معمولاً راحت‌تر از سیستم‌های complex modify می‌شوند. اما چون این ایده اهمیت زیادی دارد، برای اشاره به agility در سطح data system از واژهٔ متفاوتی استفاده می‌کنیم: `evolvability` [34].

## Key Terms

- `Maintainability` — قابلیت نگهداری، فهم، تغییر و توسعهٔ سیستم با هدف کاهش هزینه و دردسر maintenance.
- `Operability` — آسان بودن اجرای پایدار و روزمرهٔ سیستم با visibility، automation و control کافی برای operations.
- `Simplicity` — کاهش complexity غیرضروری بدون حذف functionality لازم، برای آسان‌تر شدن فهم، نگهداری و تغییر سیستم.
- `Complexity` — دشواری فهم، تغییر یا پیش‌بینی رفتار سیستم که هزینهٔ maintenance و احتمال bug را افزایش می‌دهد.
- `Accidental Complexity` — complexity ناشی از implementation، نه خود مسئله، که با abstraction و طراحی بهتر قابل حذف است.
- `Essential Complexity` — complexity ذاتی مسئله که از requirement و domain می‌آید و با حذف implementation از بین نمی‌رود.
- `Abstraction` — پنهان کردن جزئیات implementation پشت یک interface ساده و reusable.
- `Coupling` — وابستگی میان moduleها یا componentها که در صورت شدید بودن، اثر change را میان بخش‌ها گسترش می‌دهد.
- `Dependency` — رابطهٔ نیازمندی یک component به component دیگر که در صورت درهم‌تنیدگی، فهم و تغییر سیستم را دشوار می‌کند.
- `Automation` — انجام خودکار taskها و processهای عملیاتی برای کاهش خطای انسانی، همراه با نیاز به setup و monitoring صحیح.
- `Operational Model` — مدل قابل‌فهمی از رفتار عملیاتی سیستم؛ مانند نتیجهٔ تغییر configuration یا restart.
- `Self-Healing` — بازگردانی خودکار سیستم از برخی وضعیت‌های خراب بدون intervention دستی.
- `Evolvability` — توانایی سازگار شدن data system با requirementهای جدید و تغییر architecture یا behavior در طول زمان.
- `Legacy System` — سیستم قدیمی و دشوار برای نگهداری یا تغییر که معمولاً هزینهٔ maintenance بالایی دارد.
- `Technical Debt` — هزینهٔ آیندهٔ تصمیم‌های فنی کوتاه‌مدت یا ناقص که changeهای بعدی را دشوارتر می‌کند.
- `Agile` — رویکرد کاری برای سازگاری سریع با change و iteration مداوم روی requirementها و featureها.
- `Test-Driven Development (TDD)` — توسعهٔ code با تعریف test و behavior مورد انتظار پیش از implementation.
- `Refactoring` — تغییر ساختار داخلی code بدون تغییر behavior قابل‌مشاهده، برای کاهش complexity و آماده‌سازی changeهای بعدی.
- `Change Management` — مدیریت کنترل‌شدهٔ تغییرات با توجه به اثر، ریسک، deployment و recovery.
- `Configuration Management` — مدیریت version و تغییرات configuration برای جلوگیری از ناسازگاری و امکان audit و rollback.
- `Capacity Planning` — پیش‌بینی resource موردنیاز آینده بر اساس رشد load برای جلوگیری از degradation.

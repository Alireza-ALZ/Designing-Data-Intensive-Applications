# Chapter 2 — Data Models and Query Languages

## Relational Model Versus Document Model

احتمالاً شناخته‌شده‌ترین `data model` امروزی، مدل `SQL` است که بر پایهٔ `relational model` پیشنهادی Edgar Codd در سال ۱۹۷۰ بنا شده است [1]. در این مدل، داده‌ها در `relation`ها سازمان‌دهی می‌شوند که در SQL آن‌ها را `table` می‌نامیم؛ هر relation نیز مجموعه‌ای نامرتب از `tuple`هاست که در SQL به آن‌ها `row` می‌گوییم.

relational model در ابتدا یک پیشنهاد نظری بود و بسیاری از افراد در آن زمان تردید داشتند که بتوان آن را به‌طور کارآمد پیاده‌سازی کرد. بااین‌حال، تا اواسط دههٔ ۱۹۸۰، `relational database management system (RDBMS)`ها و SQL به ابزار انتخابی بیشتر افرادی تبدیل شده بودند که به ذخیره و query کردن داده‌های دارای ساختار منظم نیاز داشتند. سلطهٔ relational databaseها حدود ۲۵ تا ۳۰ سال ادامه پیدا کرد؛ در تاریخ محاسبات، این مدت تقریباً یک ابدیت است.

ریشهٔ relational databaseها در `business data processing` است؛ کاری که در دهه‌های ۱۹۶۰ و ۱۹۷۰ روی `mainframe` computerها انجام می‌شد. use caseهای آن زمان از دید امروز معمولی به نظر می‌رسند: معمولاً `transaction processing`، مانند ثبت فروش یا تراکنش‌های بانکی، رزرو بلیت هواپیما و کنترل موجودی انبار؛ و `batch processing`، مانند صدور صورت‌حساب مشتری، محاسبهٔ حقوق و گزارش‌گیری.

Databaseهای دیگرِ آن دوره، application developerها را مجبور می‌کردند دربارهٔ نمایش داخلی داده در database زیاد فکر کنند. هدف relational model این بود که این جزئیات implementation را پشت یک interface تمیزتر پنهان کند.

در طول سال‌ها، رویکردهای رقیب زیادی برای ذخیره و query کردن داده وجود داشته‌اند. در دهه‌های ۱۹۷۰ و اوایل ۱۹۸۰، `network model` و `hierarchical model` جایگزین‌های اصلی بودند، اما relational model بر آن‌ها غلبه کرد. `object database`ها در اواخر دههٔ ۱۹۸۰ و اوایل دههٔ ۱۹۹۰ ظهور کردند و دوباره کنار رفتند. `XML database`ها در اوایل دههٔ ۲۰۰۰ پدیدار شدند، اما فقط در حوزه‌های محدودی پذیرفته شدند. هر رقیب relational model در زمان خود hype زیادی ایجاد کرد، اما هیچ‌کدام دوام نیاورد [2].

با بسیار قدرتمندتر و networked شدن computerها، از آن‌ها برای هدف‌های بسیار متنوع‌تری استفاده شد. شگفت‌آور اینکه relational databaseها فراتر از دامنهٔ اولیهٔ خود، یعنی business data processing، به‌خوبی generalize شدند و برای use caseهای گسترده‌ای مناسب بودند. بخش بزرگی از چیزهایی که امروز در web می‌بینید همچنان با relational databaseها کار می‌کنند: از online publishing، forum و social networking گرفته تا ecommerce، gameها، applicationهای productivity به‌صورت software-as-a-service و موارد بسیار دیگر.

### The Birth of NoSQL

اکنون در دههٔ ۲۰۱۰، `NoSQL` تازه‌ترین تلاش برای کنار زدن سلطهٔ relational model است. نام NoSQL چندان خوش‌انتخاب نیست، چون به فناوری مشخصی اشاره نمی‌کند؛ این نام در ابتدا فقط یک Twitter hashtag جذاب برای یک meetup دربارهٔ databaseهای open source، distributed و nonrelational در سال ۲۰۰۹ بود [3]. بااین‌حال، این اصطلاح توجه‌ها را جلب کرد و به‌سرعت در جامعهٔ web startupها و فراتر از آن گسترش یافت. اکنون چندین database system جالب با hashtagِ `#NoSQL` شناخته می‌شوند و این نام بعدها به‌صورت `Not Only SQL` بازتفسیر شد [4].

چند نیروی اصلی پشت پذیرش NoSQL databaseها وجود دارد:

- نیاز به scalabilityای بیشتر از آنچه relational databaseها به‌سادگی فراهم می‌کنند، از جمله برای datasetهای بسیار بزرگ یا `write throughput` بسیار بالا.
- ترجیح گستردهٔ free و open source software نسبت به database productهای تجاری.
- query operationهای تخصصی‌ای که relational model پشتیبانی خوبی از آن‌ها ندارد.
- نارضایتی از محدودیت relational schemaها و تمایل به data modelای پویاتر و expressiveتر [5].

applicationهای مختلف requirementهای متفاوتی دارند و بهترین فناوری برای یک use case ممکن است با بهترین فناوری برای use case دیگری کاملاً فرق داشته باشد. بنابراین به نظر می‌رسد در آیندهٔ قابل پیش‌بینی، relational databaseها در کنار مجموعهٔ متنوعی از nonrelational datastoreها به استفاده ادامه دهند؛ ایده‌ای که گاهی `polyglot persistence` نامیده می‌شود [3].

### The Object-Relational Mismatch

امروزه بخش زیادی از application development با `object-oriented programming language`ها انجام می‌شود و همین موضوع به یک انتقاد رایج از SQL data model منجر شده است: اگر داده در relational tableها ذخیره شود، میان objectهای `application code` و مدل database شامل table، row و column به یک translation layer ناخوشایند نیاز داریم. این گسست میان دو model گاهی `impedance mismatch` نامیده می‌شود [یادداشت ۱].

frameworkهای `object-relational mapping (ORM)` مانند `ActiveRecord` و `Hibernate` مقدار boilerplate code موردنیاز برای این translation layer را کاهش می‌دهند، اما نمی‌توانند تفاوت‌های میان دو model را به‌طور کامل پنهان کنند.

برای مثال، شکل ۲-۱ نشان می‌دهد که یک résumé، یعنی یک LinkedIn profile، چگونه می‌تواند در یک relational schema بیان شود. کل profile را می‌توان با یک identifier یکتا به نام `user_id` شناسایی کرد. fieldهایی مانند `first_name` و `last_name` برای هر user دقیقاً یک‌بار وجود دارند؛ بنابراین می‌توان آن‌ها را به‌صورت columnهایی در tableِ users مدل کرد. بااین‌حال، بیشتر افراد در طول career خود بیش از یک شغل داشته‌اند (`positions`) و ممکن است تعداد متفاوتی دورهٔ تحصیل و هر تعداد اطلاعات تماس داشته باشند.

میان user و این itemها یک `one-to-many relationship` وجود دارد که می‌توان آن را به روش‌های مختلف نمایش داد:

- در مدل سنتی SQL، یعنی پیش از SQL:1999، رایج‌ترین نمایش `normalized` این است که positions، education و contact information را در tableهای جداگانه قرار دهیم و با یک `foreign key` به tableِ users ارجاع دهیم؛ همان‌طور که در شکل ۲-۱ دیده می‌شود.
- نسخه‌های بعدی SQL standard پشتیبانی از `structured datatype`ها و XML data را اضافه کردند. این قابلیت اجازه می‌داد داده‌های چندمقداری در یک row واحد ذخیره شوند و امکان query و index کردن درون آن documentها وجود داشته باشد. Oracle، IBM DB2، MS SQL Server و PostgreSQL از این قابلیت‌ها با درجات متفاوتی پشتیبانی می‌کنند [6, 7]. چندین database از جمله IBM DB2، MySQL و PostgreSQL از `JSON datatype` نیز پشتیبانی می‌کنند [8].
- گزینهٔ سوم این است که jobها، education و contact info را به‌صورت یک JSON یا XML document encode کنیم، آن را در یک text column از database ذخیره کنیم و تفسیر structure و content آن را به application بسپاریم. در این setup معمولاً نمی‌توانید از database برای query کردن valueهای داخل آن column encodeشده استفاده کنید.

*شکل ۲-۱. نمایش یک LinkedIn profile با استفاده از relational schema.*

برای data structureای مانند résumé که عمدتاً یک document مستقل و خودکفاست، نمایش JSON می‌تواند کاملاً مناسب باشد؛ به مثال ۲-۱ توجه کنید. JSON این مزیت را دارد که بسیار ساده‌تر از XML است. Document-oriented databaseهایی مانند `MongoDB` [9]، `RethinkDB` [10]، `CouchDB` [11] و `Espresso` [12] از این data model پشتیبانی می‌کنند.

**مثال ۲-۱. نمایش یک LinkedIn profile به‌صورت JSON document**

```json
{
"user_id":     251,
"first_name": "Bill",
"last_name":   "Gates",
"summary":     "Co-chair of the Bill & Melinda Gates... Active blogger.",
"region_id":   "us:91",
"industry_id": 131,
"photo_url":   "/p/7/000/253/05b/308dd6e.jpg",
"positions": [
    {"job_title": "Co-chair", "organization": "Bill & Melinda Gates Foundation"},
    {"job_title": "Co-founder, Chairman", "organization": "Microsoft"}
],
"education": [
    {"school_name": "Harvard University",       "start": 1973, "end": 1975},
    {"school_name": "Lakeside School, Seattle", "start": null, "end": null}
],
"contact_info": {
    "blog":    "http://thegatesnotes.com",
    "twitter": "http://twitter.com/BillGates"
    }
}
```

بعضی developerها احساس می‌کنند JSON `impedance mismatch` میان application code و storage layer را کاهش می‌دهد. بااین‌حال، همان‌طور که در فصل ۴ خواهیم دید، JSON به‌عنوان data encoding format مشکلاتی هم دارد. نبود schema اغلب به‌عنوان یک مزیت مطرح می‌شود؛ در بخش «Schema flexibility in the document model» در صفحهٔ ۳۹ دربارهٔ آن صحبت خواهیم کرد.

نمایش JSON نسبت به schema چندجدولیِ شکل ۲-۱ `locality` بهتری دارد. اگر بخواهید در مثال relational یک profile را fetch کنید، باید یا چند query اجرا کنید، یعنی هر table را با user_id query کنید، یا یک `multi-way join` نامرتب میان tableِ users و tableهای فرعی آن انجام دهید. در نمایش JSON، تمام اطلاعات مرتبط در یک جا قرار دارند و یک query کافی است.

رابطه‌های one-to-many میان user profile و positions، سوابق تحصیلی و contact information کاربر، در داده یک structure درختی ایجاد می‌کنند و نمایش JSON این tree structure را به‌صراحت نشان می‌دهد؛ به شکل ۲-۲ توجه کنید.

*شکل ۲-۲. رابطه‌های one-to-many که یک tree structure تشکیل می‌دهند.*

### Many-to-One and Many-to-Many Relationships

در مثال ۲-۱ بخش قبل، `region_id` و `industry_id` به‌صورت ID داده شده‌اند، نه به‌صورت stringهای plain-text مانند `"Greater Seattle Area"` و `"Philanthropy"`. چرا؟

اگر user interface برای وارد کردن region و industry fieldهای free-text داشته باشد، ذخیره کردن آن‌ها به‌صورت plain-text منطقی است. اما استفاده از فهرست‌های استاندارد regionهای جغرافیایی و industryها و اجازه دادن به userها برای انتخاب از یک drop-down list یا autocompleter مزیت‌هایی دارد:

- style و spelling یکسان در profileها.
- جلوگیری از ambiguity؛ مثلاً وقتی چند city نام یکسانی دارند.
- به‌روزرسانی آسان: نام فقط در یک محل ذخیره می‌شود، بنابراین اگر روزی لازم باشد تغییر کند، update کردن آن در همه‌جا آسان است؛ مثلاً در صورت تغییر نام یک شهر به‌دلیل رویدادهای سیاسی.
- پشتیبانی از localization: وقتی site به زبان‌های دیگر ترجمه می‌شود، فهرست‌های استاندارد را می‌توان localize کرد تا region و industry به زبان viewer نمایش داده شوند.
- search بهتر: مثلاً search برای philanthropistها در ایالت Washington می‌تواند این profile را پیدا کند، چون فهرست regionها می‌تواند این واقعیت را encode کند که Seattle در Washington قرار دارد؛ چیزی که از stringِ `"Greater Seattle Area"` مشخص نیست.

اینکه ID ذخیره کنید یا text string، مسئله‌ای دربارهٔ `duplication` است. وقتی از ID استفاده می‌کنید، اطلاعاتی که برای انسان معنا دارد، مانند کلمهٔ Philanthropy، فقط در یک محل ذخیره می‌شود و هر چیزی که به آن اشاره می‌کند از یک ID استفاده می‌کند؛ IDای که فقط درون database معنا دارد. وقتی text را مستقیماً ذخیره می‌کنید، اطلاعات معنادار برای انسان را در هر recordای که از آن استفاده می‌کند duplicate می‌کنید.

مزیت استفاده از ID این است که چون برای انسان معنایی ندارد، هیچ‌وقت لازم نیست تغییر کند. ID می‌تواند ثابت بماند، حتی اگر اطلاعاتی که شناسایی می‌کند تغییر کند. هر چیزی که برای انسان معنا دارد ممکن است زمانی در آینده تغییر کند؛ و اگر چنین اطلاعاتی duplicate شده باشد، باید همهٔ copyهای redundant آن update شوند. این کار `write overhead` ایجاد می‌کند و خطر inconsistency را به همراه دارد؛ یعنی ممکن است بعضی copyهای اطلاعات update شوند و بعضی دیگر نشوند. حذف این duplication ایدهٔ اصلی پشت `normalization` در databaseهاست [یادداشت ۲].

*یادداشت ۲: ادبیات مربوط به relational model، چند `normal form` متفاوت را از هم متمایز می‌کند، اما این تفاوت‌ها از نظر عملی اهمیت چندانی ندارند. به‌عنوان یک قاعدهٔ سرانگشتی، اگر valueهایی را duplicate می‌کنید که می‌توانستند فقط در یک محل ذخیره شوند، schema شما normalized نیست.*

Database administratorها و developerها معمولاً دربارهٔ normalization و `denormalization` بحث زیادی می‌کنند، اما فعلاً دربارهٔ آن قضاوتی نمی‌کنیم. در بخش سوم این کتاب به این موضوع برمی‌گردیم و روش‌های systematic برای مدیریت caching، denormalization و `derived data` را بررسی می‌کنیم.

متأسفانه normalized کردن این داده به `many-to-one relationship`ها نیاز دارد؛ مثلاً افراد زیادی در یک region مشخص زندگی می‌کنند و افراد زیادی در یک industry مشخص کار می‌کنند. این رابطه‌ها به‌خوبی با document model جور درنمی‌آیند. در relational databaseها، ارجاع دادن به rowهای tableهای دیگر با ID امری عادی است، چون `join`ها آسان‌اند. در document databaseها برای tree structureهای one-to-many به join نیاز نیست و پشتیبانی از join نیز اغلب ضعیف است [یادداشت ۳].

اگر خود database از join پشتیبانی نکند، باید join را در application code با اجرای چند query به database شبیه‌سازی کنید. در این مورد، فهرست regionها و industryها احتمالاً آن‌قدر کوچک و کم‌تغییر هستند که application می‌تواند آن‌ها را به‌سادگی در memory نگه دارد. اما بااین‌حال، کار ساختن join از database به application code منتقل شده است.

علاوه بر این، حتی اگر نسخهٔ اولیهٔ یک application به‌خوبی در document model بدون join جا بگیرد، داده با اضافه شدن featureهای بیشتر به applicationها معمولاً interconnectedتر می‌شود. برای مثال، چند تغییر احتمالی در مثال résumé را در نظر بگیرید:

**Organizations and schools as entities**

در توضیح قبلی، organization، یعنی شرکتی که user در آن کار کرده است، و `school_name`، یعنی محل تحصیل user، فقط string هستند. شاید بهتر باشد آن‌ها به‌جای string، referenceهایی به entityها باشند. در این صورت هر organization، school یا university می‌تواند web page خودش را داشته باشد؛ با logo، news feed و غیره. هر résumé نیز می‌تواند به organizationها و schoolهایی که در آن نام برده شده‌اند link شود و logo و اطلاعات دیگر آن‌ها را شامل شود؛ به شکل ۲-۳ برای نمونه‌ای از LinkedIn توجه کنید.

**Recommendations**

فرض کنید بخواهید feature جدیدی اضافه کنید: یک user بتواند برای user دیگری recommendation بنویسد. این recommendation در résumé کاربری که recommendation دریافت کرده است، همراه با name و photo کاربر پیشنهاددهنده نمایش داده می‌شود. اگر recommender photo خود را update کند، هر recommendationای که نوشته است باید photo جدید را نشان دهد. بنابراین recommendation باید referenceای به profile نویسنده داشته باشد.

*شکل ۲-۳. نام شرکت فقط یک string نیست، بلکه linkای به یک company entity است. تصویر از linkedin.com.*

شکل ۲-۴ نشان می‌دهد که این featureهای جدید به `many-to-many relationship` نیاز دارند. داده‌های داخل هر مستطیل نقطه‌چین را می‌توان در یک document گروه‌بندی کرد، اما referenceها به organizationها، schoolها و userهای دیگر باید به‌صورت reference نمایش داده شوند و هنگام query کردن به join نیاز دارند.

*شکل ۲-۴. گسترش résuméها با many-to-many relationship.*

### Are Document Databases Repeating History?

در حالی که many-to-many relationshipها و joinها به‌طور معمول در relational databaseها استفاده می‌شوند، document databaseها و NoSQL دوباره بحث دربارهٔ بهترین روش نمایش چنین رابطه‌هایی در database را زنده کرده‌اند. این بحث بسیار قدیمی‌تر از NoSQL است؛ در واقع، به نخستین database systemهای computerای برمی‌گردد.

محبوب‌ترین database برای business data processing در دههٔ ۱۹۷۰، `IBM Information Management System (IMS)` بود که در ابتدا برای stock-keeping در برنامهٔ فضایی Apollo توسعه داده شد و نخستین بار در سال ۱۹۶۸ به‌صورت تجاری عرضه شد [13]. این سیستم هنوز هم استفاده و نگهداری می‌شود و روی OS/390 در IBM mainframeها اجرا می‌شود [14].

طراحی IMS از data model نسبتاً ساده‌ای به نام `hierarchical model` استفاده می‌کرد که شباهت‌های قابل‌توجهی با JSON model مورد استفاده در document databaseها دارد [2]. IMS همهٔ داده‌ها را به‌صورت درختی از recordهایی نمایش می‌داد که درون recordهای دیگر nested شده بودند؛ تقریباً مشابه structure JSON در شکل ۲-۲.

IMS نیز مانند document databaseها برای one-to-many relationshipها خوب کار می‌کرد، اما many-to-many relationshipها را دشوار می‌کرد و از join پشتیبانی نمی‌کرد. developerها باید تصمیم می‌گرفتند که داده را duplicate یا denormalize کنند، یا referenceها را از یک record به record دیگر به‌صورت دستی resolve کنند. این مشکل‌های دهه‌های ۱۹۶۰ و ۱۹۷۰ بسیار شبیه مشکل‌هایی بودند که developerهای امروز هنگام کار با document databaseها با آن‌ها روبه‌رو هستند [15].

برای حل محدودیت‌های hierarchical model، راه‌حل‌های مختلفی پیشنهاد شد. دو راه‌حل برجسته‌تر relational model بود که به SQL تبدیل شد و جهان را درنوردید، و `network model` که ابتدا طرفداران زیادی داشت اما سرانجام به فراموشی سپرده شد. «بحث بزرگ» میان این دو جبهه بخش زیادی از دههٔ ۱۹۷۰ ادامه داشت [2].

چون مسئله‌ای که این دو model حل می‌کردند هنوز هم بسیار مرتبط است، ارزش دارد این بحث را با نگاه امروزی به‌اختصار مرور کنیم.

#### The Network Model

network model توسط کمیته‌ای به نام `Conference on Data Systems Languages (CODASYL)` استاندارد شد و چند database vendor مختلف آن را پیاده‌سازی کردند؛ این مدل با نام CODASYL model نیز شناخته می‌شود [16].

CODASYL model تعمیمی از hierarchical model بود. در tree structure مربوط به hierarchical model، هر record دقیقاً یک parent دارد؛ اما در network model، یک record می‌تواند چند parent داشته باشد. برای مثال، می‌توان یک record برای regionِ `"Greater Seattle Area"` داشت و هر userای را که در آن region زندگی می‌کند به آن link کرد. این ساختار اجازه می‌داد many-to-one و many-to-many relationshipها مدل شوند.

linkهای میان recordها در network model `foreign key` نبودند، بلکه بیشتر به pointer در یک programming language شباهت داشتند؛ هرچند همچنان روی disk ذخیره می‌شدند. تنها راه دسترسی به یک record، دنبال کردن مسیری از یک root record در امتداد این زنجیره‌های link بود. این مسیر را `access path` می‌نامیدند.

در ساده‌ترین حالت، access path می‌توانست شبیه پیمایش یک linked list باشد: از head فهرست شروع کنید و هر بار یک record را بررسی کنید تا record موردنظر را پیدا کنید. اما در دنیایی با many-to-many relationshipها، چند path متفاوت ممکن است به یک record واحد برسند و programmerای که با network model کار می‌کرد باید این access pathهای مختلف را در ذهن خود نگه می‌داشت.

یک query در CODASYL با حرکت دادن cursor در database انجام می‌شد؛ یعنی با پیمایش فهرست recordها و دنبال کردن access pathها. اگر record چند parent داشت، یعنی چند pointer ورودی از recordهای دیگر، application code باید تمام رابطه‌های مختلف را track می‌کرد. حتی اعضای کمیتهٔ CODASYL نیز پذیرفته بودند که این کار شبیه navigation در یک `n-dimensional data space` است [17].

انتخاب دستی access path می‌توانست در دههٔ ۱۹۷۰ از قابلیت‌های بسیار محدود hardware، مانند tape driveهایی که seek آن‌ها بسیار کند بود، به کارآمدترین شکل استفاده کند. اما مشکل این بود که این روش code مربوط به query و update کردن database را پیچیده و انعطاف‌ناپذیر می‌کرد. در hierarchical model و network model، اگر pathای به دادهٔ موردنظر نداشتید، در موقعیت دشواری قرار می‌گرفتید. می‌توانستید access pathها را تغییر دهید، اما در این صورت باید حجم زیادی از database query code دست‌نویس را بررسی و برای پشتیبانی از access pathهای جدید بازنویسی می‌کردید. تغییر data model یک application دشوار بود.

#### The Relational Model

در مقابل، relational model تمام داده‌ها را آشکار و بدون ساختارهای تو‌در‌تو در اختیار می‌گذاشت: یک relation یا table صرفاً مجموعه‌ای از tupleها یا rowهاست، همین. اگر بخواهید داده‌ها را ببینید، نه با structureهای nested تو‌در‌تو و هزارتووار روبه‌رو هستید و نه لازم است access pathهای پیچیده‌ای را دنبال کنید. می‌توانید هر تعداد یا همهٔ rowهای یک table را بخوانید و rowهایی را انتخاب کنید که با یک condition دلخواه match می‌شوند. می‌توانید با مشخص کردن بعضی columnها به‌عنوان key و match کردن بر اساس آن‌ها، یک row خاص را بخوانید. همچنین می‌توانید یک row جدید در هر table insert کنید، بدون اینکه نگران relationshipهای foreign key از آن table به tableهای دیگر یا از tableهای دیگر به آن باشید [یادداشت ۴].

در relational database، `query optimizer` به‌طور خودکار تصمیم می‌گیرد کدام بخش‌های query و با چه ترتیبی اجرا شوند و از کدام indexها استفاده شود. این انتخاب‌ها عملاً همان access path هستند، اما تفاوت مهم این است که آن‌ها را query optimizer به‌صورت خودکار انجام می‌دهد، نه application developer؛ بنابراین ما به‌ندرت لازم است دربارهٔ آن‌ها فکر کنیم.

اگر بخواهید دادهٔ خود را به روش‌های جدیدی query کنید، کافی است یک index جدید declare کنید و queryها به‌طور خودکار از مناسب‌ترین indexها استفاده خواهند کرد. لازم نیست queryهای خود را برای استفاده از index جدید تغییر دهید. به این ترتیب relational model اضافه کردن featureهای جدید به applicationها را بسیار آسان‌تر کرد.

query optimizerهای relational databaseها موجودات پیچیده‌ای هستند و سال‌ها تلاش پژوهشی و development صرف آن‌ها شده است [18]. اما یکی از insightهای کلیدی relational model این بود: کافی است query optimizer را یک‌بار بسازید تا همهٔ applicationهایی که از database استفاده می‌کنند از آن بهره‌مند شوند. اگر query optimizer نداشته باشید، handcode کردن access path برای یک query مشخص، از نوشتن یک optimizer general-purpose آسان‌تر است؛ اما راه‌حل general-purpose در بلندمدت برنده می‌شود.

*یادداشت ۴: foreign key constraintها اجازه می‌دهند modificationها را محدود کنید، اما چنین constraintهایی برای relational model الزامی نیستند. حتی با وجود constraintها، join روی foreign key در زمان query انجام می‌شود، در حالی که در CODASYL، join عملاً در زمان insert انجام می‌شد.*

#### Comparison to Document Databases

document databaseها از یک جنبه به hierarchical model برگشته‌اند: recordهای nested را درون record والد خود ذخیره می‌کنند، نه در یک table جداگانه. این همان one-to-many relationshipهایی است که positions، education و contact_info در شکل ۲-۱ نمونه‌ای از آن هستند.

بااین‌حال، در نمایش many-to-one و many-to-many relationshipها، relational databaseها و document databaseها از اساس تفاوتی ندارند: در هر دو، item مرتبط با یک identifier یکتا reference می‌شود. این identifier در relational model `foreign key` و در document model `document reference` نام دارد [9]. این identifier هنگام read شدن، با استفاده از یک join یا follow-up query resolve می‌شود. تا امروز، document databaseها مسیر CODASYL را دنبال نکرده‌اند.

*یادداشت ۳: در زمان نگارش کتاب، join در RethinkDB پشتیبانی می‌شد، در MongoDB پشتیبانی نمی‌شد و در CouchDB فقط در viewهای از پیش تعریف‌شده پشتیبانی می‌شد.*

### Relational Versus Document Databases Today

برای مقایسهٔ relational databaseها با document databaseها، تفاوت‌های زیادی وجود دارد؛ از جمله ویژگی‌های `Fault Tolerance` آن‌ها (به فصل ۵ مراجعه کنید) و نحوهٔ مدیریت `concurrency` (به فصل ۷ مراجعه کنید). در این فصل فقط روی تفاوت‌های `data model` تمرکز می‌کنیم.

مهم‌ترین استدلال‌ها به نفع document data model عبارت‌اند از: انعطاف‌پذیری schema، performance بهتر به‌دلیل locality، و نزدیک‌تر بودن آن به data structureهایی که application استفاده می‌کند. relational model در مقابل، از joinها و رابطه‌های many-to-one و many-to-many پشتیبانی بهتری ارائه می‌دهد.

#### Which data model leads to simpler application code?

اگر دادهٔ application شما ساختاری شبیه document دارد ــ یعنی treeای از رابطه‌های one-to-many که معمولاً کل آن یک‌جا load می‌شود ــ استفاده از document model احتمالاً انتخاب خوبی است. روش relational برای خرد کردن این ساختار شبیه document، یعنی تقسیم آن به چند table (مانند positions، education و contact_info در شکل ۲-۱)، می‌تواند به schemaهای دست‌وپاگیر و application code غیرضروری پیچیده منجر شود.

Document model محدودیت‌هایی هم دارد. برای مثال، نمی‌توانید مستقیماً به یک item تو‌در‌تو درون document reference بدهید؛ در عوض باید چیزی شبیه «دومین item در فهرست positions مربوط به user 251» را مشخص کنید؛ روشی شبیه access path در hierarchical model. بااین‌حال، تا وقتی documentها بیش از حد عمیق nested نشده باشند، این معمولاً مشکل بزرگی نیست.

پشتیبانی ضعیف document databaseها از join ممکن است بسته به application مشکل‌ساز باشد یا نباشد. برای مثال، در یک analytics application که با document database ثبت می‌کند چه eventهایی در چه زمان‌هایی رخ داده‌اند، شاید هرگز به many-to-many relationship نیاز نباشد [19].

اما اگر application شما از many-to-many relationship استفاده کند، document model جذابیت کمتری خواهد داشت. می‌توان با denormalization نیاز به join را کاهش داد، اما در این صورت application code باید برای سازگار نگه داشتن دادهٔ denormalized کار بیشتری انجام دهد. همچنین می‌توان joinها را با ارسال چند request به database در application code شبیه‌سازی کرد؛ اما این کار complexity را به application منتقل می‌کند و معمولاً از joinای که code تخصصی داخل database انجام می‌دهد کندتر است.

در چنین شرایطی، استفاده از document model می‌تواند به application code بسیار پیچیده‌تر و performance ضعیف‌تر منجر شود [15]. بنابراین نمی‌توان به‌طور کلی گفت کدام data model به application code ساده‌تری منجر می‌شود؛ پاسخ به نوع رابطه‌هایی بستگی دارد که میان data itemها وجود دارد. برای داده‌هایی که ارتباط‌های زیادی با هم دارند، document model نامناسب و دست‌وپاگیر است، relational model قابل‌قبول است و graph modelها طبیعی‌ترین انتخاب هستند (به بخش «Graph-Like Data Models» در صفحهٔ ۴۹ مراجعه کنید).

#### Schema flexibility in the document model

بیشتر document databaseها و قابلیت JSON در relational databaseها، هیچ schemaای را برای دادهٔ درون documentها enforce نمی‌کنند. پشتیبانی XML در relational databaseها معمولاً همراه با `schema validation` اختیاری ارائه می‌شود. نبود schema یعنی می‌توان keyها و valueهای دلخواهی به یک document اضافه کرد و client هنگام read کردن، هیچ تضمینی ندارد که documentها چه fieldهایی داشته باشند.

گاهی document databaseها را `schemaless` می‌نامند، اما این تعبیر گمراه‌کننده است؛ چون codeای که داده را read می‌کند معمولاً نوعی structure را فرض می‌گیرد. یعنی schemaای implicit وجود دارد، اما database آن را enforce نمی‌کند [20]. اصطلاح دقیق‌تر `schema-on-read` است: structure داده implicit است و فقط هنگام read تفسیر می‌شود. این رویکرد در مقابل `schema-on-write` قرار دارد؛ رویکرد سنتی relational databaseها که در آن schema explicit است و database تضمین می‌کند تمام داده‌های نوشته‌شده با آن سازگار باشند [21].

schema-on-read شبیه `dynamic type checking` در programming languageهاست، در حالی که schema-on-write به `static type checking` شباهت دارد. همان‌طور که طرفداران static و dynamic type checking دربارهٔ مزیت‌های نسبی آن‌ها بحث‌های زیادی دارند [22]، enforce کردن schema در database نیز موضوعی بحث‌برانگیز است و در حالت کلی پاسخ درست یا غلط مطلقی ندارد.

تفاوت این دو رویکرد به‌خصوص زمانی آشکار می‌شود که application بخواهد format دادهٔ خود را تغییر دهد. فرض کنید در حال حاضر نام کامل هر user را در یک field ذخیره می‌کنید، اما تصمیم گرفته‌اید first name و last name را جداگانه ذخیره کنید [23]. در document database کافی است نوشتن documentهای جدید با fieldهای جدید را شروع کنید و در application code حالتی را مدیریت کنید که documentهای قدیمی read می‌شوند. برای مثال:

```javascript
if (user && user.name && !user.first_name) {
    // Documents written before Dec 8, 2013 don't have first_name
    user.first_name = user.name.split(" ")[0];
}
```

در مقابل، در یک database با schema «statically typed» معمولاً باید `migration`ای شبیه این اجرا کنید:

```sql
ALTER TABLE users ADD COLUMN first_name text;
UPDATE users SET first_name = split_part(name, ' ', 1);      -- PostgreSQL
UPDATE users SET first_name = substring_index(name, ' ', 1);      -- MySQL
```

تغییر schema reputation بدی دارد، چون تصور می‌شود کند است و به downtime نیاز دارد. این reputation کاملاً منصفانه نیست: بیشتر relational database systemها دستور `ALTER TABLE` را در چند millisecond اجرا می‌کنند. `MySQL` یک استثنای مهم است؛ این سیستم هنگام `ALTER TABLE` کل table را copy می‌کند و در زمان تغییر یک table بزرگ ممکن است چند دقیقه یا حتی چند ساعت downtime ایجاد شود. البته ابزارهای مختلفی برای دور زدن این محدودیت وجود دارد [24, 25, 26].

اجرای دستور `UPDATE` روی یک table بزرگ احتمالاً در هر databaseای کند است، چون باید هر row بازنویسی شود. اگر این کار قابل‌قبول نباشد، application می‌تواند `first_name` را با مقدار پیش‌فرض `NULL` نگه دارد و آن را هنگام read پر کند؛ درست مانند کاری که در document database انجام می‌دهد.

رویکرد schema-on-read زمانی مزیت دارد که itemهای یک collection به دلیلی structure یکسانی نداشته باشند؛ یعنی داده `heterogeneous` باشد. برای مثال:

- objectها typeهای بسیار متفاوتی دارند و عملی نیست که برای هر type یک table جداگانه بسازیم.
- structure داده را systemهای خارجی تعیین می‌کنند؛ systemهایی که کنترلی بر آن‌ها نداریم و ممکن است هر زمان تغییر کنند.

در چنین شرایطی، schema ممکن است بیشتر از آنکه کمک کند مانع ایجاد کند و schemaless documentها می‌توانند data model طبیعی‌تری باشند. اما وقتی انتظار می‌رود همهٔ recordها structure یکسانی داشته باشند، schema mechanism مفیدی برای مستندسازی و enforce کردن آن structure است. در فصل ۴، schema و `schema evolution` را با جزئیات بیشتری بررسی می‌کنیم.

#### Data locality for queries

یک document معمولاً به‌صورت یک رشتهٔ پیوسته ذخیره می‌شود که با `JSON`، `XML` یا یک variant باینری آن‌ها (مانند `BSON` در MongoDB) encode شده است. اگر application شما اغلب لازم دارد کل document را access کند ــ مثلاً برای render کردن آن در یک web page ــ این locality در storage یک مزیت performance ایجاد می‌کند. اگر داده مانند شکل ۲-۱ میان چند table تقسیم شده باشد، برای بازیابی همهٔ آن به چند index lookup نیاز است؛ این کار ممکن است به disk seekهای بیشتر و زمان طولانی‌تر منجر شود.

مزیت locality فقط زمانی وجود دارد که لازم باشد بخش بزرگی از document را هم‌زمان بخوانید. database معمولاً باید کل document را load کند، حتی اگر فقط به بخش کوچکی از آن access داشته باشید؛ بنابراین برای documentهای بزرگ ممکن است این کار wasteful باشد. هنگام update کردن document نیز معمولاً باید کل document دوباره write شود؛ فقط modificationهایی که اندازهٔ encodeشدهٔ document را تغییر نمی‌دهند به‌سادگی می‌توانند in-place انجام شوند [19]. به همین دلیل معمولاً توصیه می‌شود documentها را نسبتاً کوچک نگه دارید و از writeهایی که اندازهٔ document را افزایش می‌دهند پرهیز کنید [9].

این محدودیت‌های performance، مجموعهٔ شرایطی را که document databaseها در آن مفید هستند به‌طور چشمگیری کوچک می‌کنند.

باید توجه کرد که ایدهٔ گروه‌بندی داده‌های مرتبط برای بهبود locality به document model محدود نیست. برای مثال، databaseِ `Google Spanner` همین ویژگی locality را در یک relational data model فراهم می‌کند؛ به این صورت که schema اجازه می‌دهد rowهای یک table به‌صورت interleaved و nested درون یک parent table قرار بگیرند [27]. `Oracle` نیز با قابلیتی به نام `multi-table index cluster tables` امکان مشابهی ارائه می‌دهد [28]. مفهوم `column-family` در Bigtable data model، که در Cassandra و HBase استفاده می‌شود، هدف مشابهی برای مدیریت locality دارد [29]. در فصل ۳ بیشتر دربارهٔ locality صحبت خواهیم کرد.

#### Convergence of document and relational databases

بیشتر relational database systemها (به‌جز MySQL) از اواسط دههٔ ۲۰۰۰ از XML پشتیبانی کرده‌اند. این پشتیبانی شامل functionهایی برای modificationهای local در XML documentها و قابلیت index و query کردن داخل XML documentهاست؛ بنابراین applicationها می‌توانند data modelهایی بسیار شبیه document databaseها داشته باشند.

`PostgreSQL` از نسخهٔ 9.3، `MySQL` از نسخهٔ 5.7 و `IBM DB2` از نسخهٔ 10.5 سطح مشابهی از پشتیبانی برای JSON documentها دارند [30]. با توجه به محبوبیت JSON در web APIها، احتمالاً relational databaseهای دیگری نیز همین مسیر را دنبال می‌کنند و پشتیبانی از JSON را اضافه خواهند کرد.

در سمت document databaseها، `RethinkDB` از joinهای شبیه relational در query language خود پشتیبانی می‌کند و بعضی driverهای MongoDB به‌طور خودکار database referenceها را resolve می‌کنند؛ این کار عملاً یک client-side join است، هرچند احتمالاً از joinای که داخل database انجام می‌شود کندتر است، چون به round-tripهای network اضافی نیاز دارد و optimization کمتری روی آن انجام می‌شود.

به نظر می‌رسد relational و document databaseها به‌مرور شبیه‌تر می‌شوند و این اتفاق خوبی است؛ data modelهای آن‌ها مکمل یکدیگرند. اگر database بتواند هم دادهٔ شبیه document را handle کند و هم queryهای relational را روی آن اجرا کند، application می‌تواند ترکیبی از قابلیت‌ها را به کار بگیرد که بیشترین تناسب را با نیازش دارد.

ترکیبی از relational model و document model مسیر مناسبی برای databaseهای آینده است.

*یادداشت ۵: توصیف اولیهٔ Codd از relational model [1] در واقع چیزی بسیار شبیه JSON document را درون یک relational schema مجاز می‌دانست. او این مفهوم را `nonsimple domains` نامید. ایده این بود که value یک row مجبور نیست فقط یک primitive datatype مانند number یا string باشد؛ این value می‌تواند یک relation تو‌در‌تو (table) نیز باشد. بنابراین می‌توان structure درختی‌ای با عمق دلخواه را به‌عنوان value داشت؛ چیزی شبیه پشتیبانی JSON یا XML که بیش از ۳۰ سال بعد به SQL اضافه شد.*

## Key Terms

- `Relational Model` — مدلی که داده را در relationها یا tableها به‌صورت row و column سازمان‌دهی می‌کند و access path را از application developer پنهان می‌سازد.
- `Document Model` — مدلی که داده را در documentهای معمولاً nested و شبیه JSON سازمان‌دهی می‌کند.
- `NoSQL` — عنوانی کلی برای مجموعه‌ای متنوع از nonrelational databaseها؛ این نام بعدها به `Not Only SQL` تعبیر شد.
- `Relational Database Management System (RDBMS)` — نرم‌افزاری برای ذخیره و query کردن داده بر پایهٔ relational model.
- `Schema` — ساختار و قواعدی که شکل دادهٔ ذخیره‌شده را مشخص می‌کند.
- `Polyglot Persistence` — استفادهٔ هم‌زمان از relational databaseها و datastoreهای متنوع، متناسب با requirement هر use case.
- `Object-Relational Mapping (ORM)` — framework یا روشی برای نگاشت objectهای application به tableها، rowها و columnهای relational database.
- `Impedance Mismatch` — ناهماهنگی میان مدل object-oriented application و مدل relational database.
- `Nested Data` — داده‌ای که به‌صورت ساختار درونی و تو‌در‌تو داخل یک record یا document قرار گرفته است.
- `One-to-Many Relationship` — رابطه‌ای که در آن یک entity با چند entity مرتبط است؛ مانند یک user و چند position.
- `Many-to-One Relationship` — رابطه‌ای که در آن چند entity به یک entity مشترک اشاره می‌کنند؛ مانند چند user در یک region.
- `Many-to-Many Relationship` — رابطه‌ای که در آن هر طرف می‌تواند با چند entity از طرف دیگر مرتبط باشد.
- `Foreign Key` — identifierای در یک table که به recordای در table دیگر reference می‌دهد.
- `Join` — عملیاتی برای ترکیب recordهای مرتبط از چند table هنگام query.
- `Normalization` — حذف duplication داده با ذخیرهٔ هر value در یک محل و reference دادن به آن.
- `Denormalization` — duplicate کردن کنترل‌شدهٔ داده برای کاهش join یا بهبود الگوی دسترسی.
- `Hierarchical Model` — مدلی درختی که recordها را درون recordهای دیگر nested می‌کند.
- `Network Model` — مدلی که اجازه می‌دهد یک record چند parent داشته باشد و رابطه‌های many-to-many را با link مدل کند.
- `Access Path` — مسیر مشخصی برای رسیدن از یک record ریشه به record موردنظر.
- `Query Optimizer` — جزء database که ترتیب اجرای query و indexهای مناسب را به‌طور خودکار انتخاب می‌کند.
- `Document Reference` — identifierای که در document model به یک document مرتبط اشاره می‌کند.
- `Data Locality` — نزدیک بودن داده‌های مرتبط به یکدیگر در یک document یا محل ذخیره‌سازی، برای کاهش query و join.
- `Schema Flexibility` — آزادی در تغییر structure documentها بدون نیاز به migration هم‌زمان در کل داده‌ها.
- `Schema-on-Read` — رویکردی که در آن structure داده هنگام read تفسیر می‌شود و database آن را از پیش enforce نمی‌کند.
- `Schema-on-Write` — رویکردی که در آن schema هنگام write enforce می‌شود و دادهٔ ذخیره‌شده باید با آن سازگار باشد.
- `Schema Evolution` — تغییر کنترل‌شدهٔ schema در طول عمر application و سازگار نگه داشتن code و داده با versionهای مختلف.
- `Migration` — فرایند تغییر structure یا انتقال داده برای هماهنگ شدن با schema یا version جدید.
- `Dynamic Type Checking` — بررسی type داده هنگام اجرا، نه در زمان compile.
- `Static Type Checking` — بررسی type داده پیش از اجرا، معمولاً در زمان compile.
- `Heterogeneous Data` — مجموعه‌ای از داده که itemهای آن structure یا type یکسانی ندارند.
- `Client-Side Join` — اجرای join در application یا client با دریافت داده از چند request، به‌جای اجرای آن داخل database.
- `Hybrid Data Model` — ترکیب قابلیت‌های relational و document برای پشتیبانی هم‌زمان از دادهٔ nested و queryهای relational.

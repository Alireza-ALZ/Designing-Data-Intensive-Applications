# Chapter 4 — Encoding and Evolution

## Formats for Encoding Data

Programها معمولاً data را دست‌کم در دو representation متفاوت نگهداری می‌کنند:

1. در memory، data در objectها، structها، listها، arrayها، hash tableها، treeها و موارد مشابه نگهداری می‌شود. این data structureها برای access و manipulation efficient توسط CPU (معمولاً با استفاده از pointerها) بهینه شده‌اند.
2. وقتی می‌خواهید data را در یک file بنویسید یا آن را از طریق network بفرستید، باید آن را به نوعی sequence مستقل از byteها تبدیل کنید (برای مثال، یک JSON document). چون pointer برای process دیگری معنایی ندارد، این representation مبتنی بر sequence byte با data structureهایی که معمولاً در memory استفاده می‌شوند کاملاً متفاوت است.*

بنابراین به نوعی translation میان این دو representation نیاز داریم. تبدیل representation درون memory به یک sequence از byteها `encoding` نام دارد (که با نام‌های `serialization` یا `marshalling` نیز شناخته می‌شود) و عمل معکوس `decoding` نام دارد (که به آن `parsing`، `deserialization` یا `unmarshalling` نیز گفته می‌شود).†

#### تداخل اصطلاحات

واژهٔ `serialization` متأسفانه در context مربوط به transactionها نیز استفاده می‌شود (به Chapter 7 مراجعه کنید)، اما در آنجا معنای کاملاً متفاوتی دارد. برای جلوگیری از overloading شدن این واژه، در این کتاب از `encoding` استفاده می‌کنیم، هرچند `serialization` شاید اصطلاح رایج‌تری باشد.

از آنجا که این مسئله بسیار رایج است، libraryها و encoding formatهای بسیار متنوعی برای انتخاب وجود دارد. ابتدا overview کوتاهی از آن‌ها داشته باشیم.

*پاورقی: استثناهایی مانند برخی memory-mapped fileها یا کار کردن مستقیم روی data فشرده وجود دارد؛ همان‌طور که در بخش «Column Compression» در صفحهٔ ۹۷ توضیح داده شد.*

*پاورقی: encoding هیچ ارتباطی با encryption ندارد. در این کتاب دربارهٔ encryption بحث نمی‌کنیم.*

### Language-Specific Formats

بسیاری از programming languageها پشتیبانی built-in برای encode کردن objectهای درون memory به sequenceهای byte دارند. برای مثال، Java دارای `java.io.Serializable` [1]، Ruby دارای `Marshal` [2] و Python دارای `pickle` [3] است. libraryهای third-party زیادی نیز وجود دارند؛ مانند `Kryo` برای Java [4].

این encoding libraryها بسیار convenient هستند، چون اجازه می‌دهند objectهای درون memory با حداقل code اضافی save و restore شوند. بااین‌حال، چند مشکل عمیق نیز دارند:

- encoding اغلب به یک programming language مشخص وابسته است و خواندن data در language دیگر بسیار دشوار می‌شود. اگر data را با چنین encodingای ذخیره یا منتقل کنید، ممکن است برای مدت بسیار طولانی به programming language فعلی خود متعهد شوید و امکان integration سیستم‌های خود با سیستم‌های سازمان‌های دیگر را از بین ببرید؛ چون آن سازمان‌ها ممکن است از languageهای متفاوتی استفاده کنند.
- برای restore کردن data در همان object typeها، فرآیند decoding باید بتواند classهای arbitrary را instantiate کند. این موضوع اغلب source مشکل‌های security است [5]: اگر attacker بتواند application شما را وادار کند یک byte sequence arbitrary را decode کند، می‌تواند classهای arbitrary را instantiate کند و این کار در بسیاری از موارد به او اجازه می‌دهد کارهای بسیار خطرناکی مانند remote execution کردن code arbitrary انجام دهد [6, 7].
- versioning data در این libraryها اغلب یک موضوع فرعی است. از آنجا که این libraryها برای encoding سریع و ساده طراحی شده‌اند، معمولاً مشکل‌های ناخوشایند مربوط به forward و backward compatibility را نادیده می‌گیرند.
- efficiency (زمان CPU لازم برای encode یا decode کردن و اندازهٔ structure encoded) نیز اغلب در اولویت بعدی قرار دارد. برای مثال، serialization built-in در Java به performance ضعیف و encoding حجیم خود مشهور است [8].

به این دلایل، معمولاً استفاده از encoding built-in language برای چیزی بیش از موارد بسیار transient ایدهٔ خوبی نیست.

### JSON, XML, and Binary Variants

اگر به‌سراغ encodingهای standardized برویم که توسط programming languageهای متعدد قابل write و read باشند، JSON و XML گزینه‌های obvious هستند. این formatها widely known و widely supported هستند و تقریباً به همان اندازه نیز مورد انتقاد قرار گرفته‌اند. XML اغلب به بیش‌ازحد verbose بودن و پیچیدگی غیرضروری نقد می‌شود [9]. محبوبیت JSON عمدتاً به پشتیبانی built-in آن در web browserها (به‌دلیل subset بودن از JavaScript) و سادگی آن در مقایسه با XML برمی‌گردد. CSV نیز format محبوب دیگری است که language-independent است، هرچند قدرت کمتری دارد.

JSON، XML و CSV formatهای متنی هستند و بنابراین تا حدی human-readable محسوب می‌شوند (هرچند syntax آن‌ها موضوع بحث‌های فراوان است). جدا از مسئله‌های ظاهری syntax، این formatها مشکل‌های ظریف‌تری نیز دارند:

- دربارهٔ encoding عددها ابهام زیادی وجود دارد. در XML و CSV نمی‌توانید میان یک number و stringای که از digitها تشکیل شده است تفاوت بگذارید، مگر اینکه به schemaای external مراجعه کنید. JSON میان string و number تفاوت می‌گذارد، اما integer و floating-point number را از هم متمایز نمی‌کند و precision را نیز مشخص نمی‌کند.

  این موضوع هنگام کار با numberهای بزرگ مشکل‌ساز می‌شود. برای مثال، integerهای بزرگ‌تر از `2^53` را نمی‌توان به‌صورت دقیق در یک floating-point number از نوع IEEE 754 double-precision نمایش داد؛ بنابراین وقتی چنین numberهایی در languageای که از floating-point number استفاده می‌کند (مانند JavaScript) parse شوند، نادقیق خواهند شد. نمونه‌ای از numberهای بزرگ‌تر از `2^53` در Twitter دیده می‌شود که برای شناسایی هر tweet از یک number ۶۴بیتی استفاده می‌کند. JSON برگردانده‌شده توسط API توییتر، ID مربوط به tweet را دو بار شامل می‌شود: یک بار به‌صورت JSON number و یک بار به‌صورت decimal string، تا با این واقعیت کنار بیاید که JavaScript applicationها این numberها را به‌درستی parse نمی‌کنند [10].

- JSON و XML از Unicode character stringها (یعنی textهای human-readable) پشتیبانی خوبی دارند، اما binary stringها را پشتیبانی نمی‌کنند؛ binary string sequenceای از byteهاست که character encoding ندارد. Binary stringها feature مفیدی هستند، بنابراین معمولاً برای دور زدن این محدودیت، binary data را با استفاده از `Base64` به text encode می‌کنند. سپس schema مشخص می‌کند که value باید به‌عنوان Base64-encoded تفسیر شود. این روش کار می‌کند، اما تا حدی hacky است و اندازهٔ data را ۳۳٪ افزایش می‌دهد.
- برای XML [11] و JSON [12] پشتیبانی optional از schema وجود دارد. این schema languageها بسیار قدرتمند و در نتیجه یادگیری و implementation آن‌ها نسبتاً پیچیده است. استفاده از XML schemaها نسبتاً رایج است، اما بسیاری از toolهای مبتنی بر JSON زحمت استفاده از schema را به خود نمی‌دهند. از آنجا که تفسیر صحیح data (مانند numberها و binary stringها) به اطلاعات موجود در schema وابسته است، applicationهایی که از XML/JSON schema استفاده نمی‌کنند ممکن است مجبور شوند logic مناسب برای encoding و decoding را به‌صورت hardcode در code خود قرار دهند.
- CSV هیچ schemaای ندارد، بنابراین application باید معنای هر row و column را تعریف کند. اگر تغییر application یک row یا column جدید اضافه کند، باید این تغییر را به‌صورت دستی handle کنید. CSV همچنین format نسبتاً مبهمی است (اگر value شامل comma یا newline باشد چه اتفاقی می‌افتد؟). هرچند ruleهای escaping آن به‌صورت رسمی مشخص شده‌اند [13]، همهٔ parserها آن‌ها را به‌درستی implement نمی‌کنند.

با وجود این ضعف‌ها، JSON، XML و CSV برای بسیاری از هدف‌ها به‌اندازهٔ کافی خوب هستند. احتمالاً این formatها، به‌خصوص به‌عنوان data interchange formatها، همچنان محبوب باقی می‌مانند؛ یعنی برای ارسال data از یک سازمان به سازمان دیگر. در چنین موقعیت‌هایی، تا وقتی افراد بر سر format توافق داشته باشند، اغلب مهم نیست format چقدر زیبا یا efficient باشد. دشواری توافق دادن سازمان‌های مختلف بر سر هر چیزی، از بیشتر نگرانی‌های دیگر مهم‌تر است.

#### Binary encoding

برای dataای که فقط در داخل سازمان شما استفاده می‌شود، فشار کمتری برای استفاده از یک encoding format با کمترین مخرج مشترک وجود دارد. برای مثال، می‌توانید formatی را انتخاب کنید که compactتر باشد یا سریع‌تر parse شود. برای dataset کوچک، این دستاوردها ناچیزند؛ اما وقتی حجم data به terabyte برسد، انتخاب data format می‌تواند اثر بزرگی داشته باشد.

JSON از XML کم‌حجم‌تر است، اما هر دوی آن‌ها در مقایسه با binary formatها همچنان فضای زیادی مصرف می‌کنند. این مشاهده به توسعهٔ binary encodingهای متعددی برای JSON (مانند `MessagePack`، `BSON`، `BJSON`، `UBJSON`، `BISON` و `Smile`) و برای XML (برای مثال `WBXML` و `Fast Infoset`) منجر شد. این formatها در nicheهای مختلف پذیرفته شده‌اند، اما هیچ‌کدام به اندازهٔ نسخه‌های متنی JSON و XML به‌طور گسترده adopted نشده‌اند.

برخی از این formatها مجموعهٔ datatypeها را گسترش می‌دهند (برای مثال با متمایز کردن integer و floating-point number یا اضافه کردن پشتیبانی از binary string)، اما در بقیهٔ موارد data model مربوط به JSON/XML را بدون تغییر حفظ می‌کنند. به‌طور خاص، چون این formatها schemaای را اجباری نمی‌کنند، باید تمام نام‌های field مربوط به object را درون data encodedشده قرار دهند. یعنی در یک binary encoding از JSON document مثال ۴-۱، باید stringهای `userName`، `favoriteNumber` و `interests` در جایی از data قرار داشته باشند.

**مثال ۴-۱.** رکورد نمونه‌ای که در چند binary format مختلف در این chapter encode خواهد شد.

```json
{
      "userName": "Martin",
      "favoriteNumber": 1337,
      "interests": ["daydreaming", "hacking"]
}
```

بیایید نمونهٔ `MessagePack` را بررسی کنیم؛ binary encodingای برای JSON. شکل ۴-۱ byte sequenceای را نشان می‌دهد که با encode کردن JSON document مثال ۴-۱ با `MessagePack` به دست می‌آید [14]. چند byte اول آن چنین‌اند:

1. byte اول، `0x83`، مشخص می‌کند چیزی که در ادامه می‌آید یک object است (چهار bit بالایی برابر `0x80`) که سه field دارد (چهار bit پایینی برابر `0x03`). اگر می‌پرسید وقتی object بیش از ۱۵ field داشته باشد چه اتفاقی می‌افتد و تعداد fieldها در چهار bit جا نشود، در آن صورت از type indicator دیگری استفاده می‌شود و تعداد fieldها در دو یا چهار byte encode می‌شود.
2. byte دوم، `0xa8`، مشخص می‌کند چیزی که در ادامه می‌آید یک string است (چهار bit بالایی برابر `0xa0`) که هشت byte طول دارد (چهار bit پایینی برابر `0x08`).
3. هشت byte بعدی نام field یعنی `userName` را در ASCII نگهداری می‌کنند. چون length از قبل مشخص شده است، نیازی به marker برای تعیین پایان string (یا escaping) وجود ندارد.
4. هفت byte بعدی string شش‌حرفی `Martin` را با prefix برابر `0xa6` encode می‌کنند و به همین ترتیب ادامه می‌یابد.

Binary encoding، ۶۶ byte طول دارد که فقط اندکی کمتر از ۸۱ byte موردنیاز برای encoding متنی JSON است (با حذف whitespace). تمام binary encodingهای JSON از این نظر مشابه‌اند. مشخص نیست چنین کاهش اندکی در فضا (و شاید افزایش سرعت parsing) ارزش از دست دادن human-readability را داشته باشد.

در بخش‌های بعدی خواهیم دید که چگونه می‌توان این کار را بسیار بهتر انجام داد و همین record را فقط در ۳۲ byte encode کرد.

**شکل ۴-۱.** رکورد نمونه (مثال ۴-۱) که با `MessagePack` encode شده است.

### Thrift and Protocol Buffers

`Apache Thrift` [15] و `Protocol Buffers` (`protobuf`) [16]، binary encoding libraryهایی هستند که بر اساس یک اصل یکسان ساخته شده‌اند. Protocol Buffers ابتدا در Google و Thrift ابتدا در Facebook توسعه داده شد و هر دو در سال‌های ۲۰۰۷ و ۲۰۰۸ open source شدند [17].

Thrift و Protocol Buffers هر دو برای هر dataای که encode می‌شود به schema نیاز دارند. برای encode کردن data مثال ۴-۱ در Thrift، schema را در `Thrift interface definition language (IDL)` به‌شکل زیر توصیف می‌کنید:

```thrift
struct Person {
  1: required string       userName,
  2: optional i64          favoriteNumber,
  3: optional list<string> interests
}
```

تعریف schema معادل برای Protocol Buffers بسیار شبیه است:

```protobuf
message Person {
    required string user_name      = 1;
    optional int64 favorite_number = 2;
    repeated string interests      = 3;
}
```

Thrift و Protocol Buffers هرکدام یک code generation tool دارند که تعریف schema مانند نمونه‌های بالا را دریافت می‌کند و classهایی را تولید می‌کند که schema را در programming languageهای مختلف implement می‌کنند [18]. application code شما می‌تواند برای encode یا decode کردن recordهای این schema، code تولیدشده را call کند.

Data encodedشده با این schema چه شکلی دارد؟ به‌شکل گیج‌کننده‌ای، Thrift دو binary encoding format متفاوت دارد که به‌ترتیب `BinaryProtocol` و `CompactProtocol` نامیده می‌شوند. ابتدا `BinaryProtocol` را بررسی کنیم. encoding کردن مثال ۴-۱ با این format به ۵۹ byte نیاز دارد؛ همان‌طور که در شکل ۴-۲ نشان داده شده است [19].

*پاورقی: در واقع Thrift سه format دارد: `BinaryProtocol`، `CompactProtocol` و `DenseProtocol`. بااین‌حال، `DenseProtocol` فقط در implementation مربوط به C++ پشتیبانی می‌شود و بنابراین cross-language محسوب نمی‌شود [18]. علاوه بر این‌ها، دو encoding format مبتنی بر JSON نیز دارد.*

**شکل ۴-۲.** رکورد نمونه که با `BinaryProtocol` مربوط به Thrift encode شده است.

مانند شکل ۴-۱، هر field دارای یک type annotation است (برای مشخص کردن اینکه field از نوع string، integer، list یا چیز دیگری است) و در صورت نیاز length آن نیز مشخص می‌شود (length یک string یا تعداد itemهای یک list). stringهایی که در data ظاهر می‌شوند (`Martin`، `daydreaming` و `hacking`) نیز مانند قبل به‌صورت ASCII (یا دقیق‌تر، UTF-8) encode می‌شوند.

تفاوت بزرگ در مقایسه با شکل ۴-۱ این است که نام fieldها (`userName`، `favoriteNumber` و `interests`) در data وجود ندارند. در عوض، data encodedشده شامل field tagهایی است که number هستند (۱، ۲ و ۳). این همان numberهایی است که در تعریف schema ظاهر می‌شوند. Field tagها مانند alias برای fieldها هستند؛ یعنی روشی compact برای اشاره به field موردنظر، بدون spell کردن نام آن.

encoding مربوط به `CompactProtocol` از نظر semantics معادل `BinaryProtocol` است، اما همان‌طور که در شکل ۴-۳ می‌بینید، همان اطلاعات را فقط در ۳۴ byte جا می‌دهد. این کار با pack کردن type مربوط به field و tag number در یک byte و استفاده از variable-length integer انجام می‌شود. به‌جای استفاده از هشت byte کامل برای number `1337`، این number در دو byte encode می‌شود و bit بالایی هر byte مشخص می‌کند آیا byteهای بیشتری در ادامه وجود دارند یا نه. بنابراین numberهای بین `-64` و `63` در یک byte، numberهای بین `-8192` و `8191` در دو byte و به همین ترتیب encode می‌شوند. Numberهای بزرگ‌تر byteهای بیشتری مصرف می‌کنند.

**شکل ۴-۳.** رکورد نمونه که با `CompactProtocol` مربوط به Thrift encode شده است.

در نهایت، Protocol Buffers (که فقط یک binary encoding format دارد) همان data را مانند شکل ۴-۴ encode می‌کند. نحوهٔ bit packing در آن اندکی با Thrift متفاوت است، اما در بقیهٔ موارد به `CompactProtocol` شباهت زیادی دارد. Protocol Buffers همین record را در ۳۳ byte جا می‌دهد.

**شکل ۴-۴.** رکورد نمونه که با Protocol Buffers encode شده است.

یک نکتهٔ مهم این است که در schemaهای بالا هر field با `required` یا `optional` علامت‌گذاری شده است، اما این موضوع هیچ تفاوتی در نحوهٔ encoding field ایجاد نمی‌کند (در binary data چیزی مشخص نمی‌کند که field required بوده یا نه). تفاوت صرفاً این است که `required` یک runtime check را فعال می‌کند که اگر field set نشده باشد fail می‌شود؛ این قابلیت می‌تواند برای پیدا کردن bugها مفید باشد.

#### Field tags و schema evolution

پیش‌تر گفتیم که schemaها ناگزیر در طول زمان تغییر می‌کنند. به این فرایند `schema evolution` می‌گوییم. Thrift و Protocol Buffers چگونه تغییر schema را مدیریت می‌کنند و در عین حال backward و forward compatibility را حفظ می‌کنند؟

همان‌طور که از مثال‌ها مشخص است، یک record encodedشده فقط concatenation مربوط به fieldهای encodedشده است. هر field با tag number خود (numberهای ۱، ۲ و ۳ در schemaهای نمونه) شناسایی می‌شود و یک datatype نیز دارد (برای مثال string یا integer). اگر value یک field set نشده باشد، آن field به‌سادگی از record encodedشده حذف می‌شود. بنابراین field tagها برای معنای data encodedشده حیاتی هستند. می‌توانید نام field را در schema تغییر دهید، چون data encodedشده هیچ‌گاه به نام fieldها reference نمی‌دهد؛ اما نمی‌توانید tag مربوط به field را تغییر دهید، چون با این کار تمام data encodedشدهٔ قبلی invalid می‌شود.

می‌توانید fieldهای جدیدی به schema اضافه کنید، به‌شرطی که به هر field یک tag number جدید بدهید. اگر code قدیمی (که tag numberهای جدید شما را نمی‌شناسد) data نوشته‌شده توسط code جدید را بخواند و با field جدیدی روبه‌رو شود که tag number آن را نمی‌شناسد، می‌تواند آن field را به‌سادگی ignore کند. type annotation به parser اجازه می‌دهد مشخص کند چند byte را باید skip کند. این کار forward compatibility را حفظ می‌کند: code قدیمی می‌تواند recordهایی را بخواند که code جدید آن‌ها را نوشته است.

backward compatibility چطور حفظ می‌شود؟ تا زمانی که هر field یک tag number یکتا داشته باشد، code جدید همیشه می‌تواند data قدیمی را بخواند، چون tag numberها همچنان همان معنا را دارند. تنها نکته این است که اگر field جدیدی اضافه می‌کنید، نمی‌توانید آن را required کنید. اگر field را required کنید، زمانی که code جدید data نوشته‌شده توسط code قدیمی را بخواند، check شکست می‌خورد؛ چون code قدیمی field جدیدی را که اضافه کرده‌اید write نکرده است. بنابراین برای حفظ backward compatibility، هر fieldی که پس از deployment اولیهٔ schema اضافه می‌کنید باید optional باشد یا default value داشته باشد.

حذف کردن field نگرانی‌های backward و forward compatibility را برعکسِ اضافه کردن field ایجاد می‌کند. یعنی فقط می‌توانید fieldی را حذف کنید که optional باشد (field required را هرگز نمی‌توان حذف کرد) و هرگز نمی‌توانید از همان tag number دوباره استفاده کنید؛ چون ممکن است هنوز dataای در جایی وجود داشته باشد که tag number قدیمی را شامل شود و code جدید باید آن field را ignore کند.

#### Datatypes و schema evolution

تغییر datatype یک field چطور؟ این کار ممکن است امکان‌پذیر باشد؛ برای جزئیات باید documentation را بررسی کنید، اما خطر از دست رفتن precision یا truncate شدن valueها وجود دارد. برای مثال فرض کنید یک integer ۳۲بیتی را به integer ۶۴بیتی تغییر دهید. code جدید به‌سادگی می‌تواند data نوشته‌شده توسط code قدیمی را بخواند، چون parser می‌تواند bitهای missing را با صفر پر کند. اما اگر code قدیمی data نوشته‌شده توسط code جدید را بخواند، همچنان از variable ۳۲بیتی برای نگهداری value استفاده می‌کند. اگر value decodeشدهٔ ۶۴بیتی در ۳۲ bit جا نشود، truncate خواهد شد.

نکتهٔ جالب دربارهٔ Protocol Buffers این است که datatypeای به نام list یا array ندارد، بلکه برای fieldها markerای به نام `repeated` دارد (که گزینهٔ سومی در کنار `required` و `optional` است). همان‌طور که در شکل ۴-۴ می‌بینید، encoding یک field `repeated` دقیقاً مطابق نامش است: همان field tag چند بار در record ظاهر می‌شود. نتیجهٔ خوب این است که تبدیل یک field optional (تک‌مقداری) به field repeated (چندمقداری) اشکالی ندارد. code جدید هنگام خواندن data قدیمی، listای با صفر یا یک element می‌بیند (بسته به اینکه field وجود داشته یا نه)؛ code قدیمی هنگام خواندن data جدید فقط آخرین element list را می‌بیند.

Thrift datatype اختصاصی `list` دارد که با datatype مربوط به elementهای list parameterize می‌شود. این ویژگی اجازهٔ همان evolution از single-valued به multi-valued را که Protocol Buffers ارائه می‌دهد نمی‌دهد، اما مزیت آن پشتیبانی از nested listهاست.

### Avro

`Apache Avro` [20] binary encoding format دیگری است که تفاوت جالبی با Protocol Buffers و Thrift دارد. این project در سال ۲۰۰۹ به‌عنوان subprojectای از Hadoop آغاز شد؛ چون Thrift برای use caseهای Hadoop fit مناسبی نداشت [21].

Avro نیز برای مشخص کردن structure مربوط به data encodedشده از schema استفاده می‌کند. Avro دو schema language دارد: یکی (`Avro IDL`) که برای ویرایش توسط انسان طراحی شده و دیگری (مبتنی بر JSON) که machine-readableتر است.

Schema نمونهٔ ما در Avro IDL می‌تواند چنین باشد:

```avro
record Person {
    string               userName;
    union { null, long } favoriteNumber = null;
    array<string>        interests;
}
```

نمایش JSON معادل این schema به‌صورت زیر است:

```json
{
    "type": "record",
    "name": "Person",
    "fields": [
        {"name": "userName",       "type": "string"},
        {"name": "favoriteNumber", "type": ["null", "long"], "default": null},
        {"name": "interests",      "type": {"type": "array", "items": "string"}}
    ]
}
```

اول از همه توجه کنید که در schema هیچ tag numberای وجود ندارد. اگر record نمونهٔ خود (مثال ۴-۱) را با این schema encode کنیم، Avro binary encoding فقط ۳۲ byte طول خواهد داشت؛ compactترین encoding در میان تمام encodingهایی که تاکنون دیده‌ایم. breakdown مربوط به byte sequence encodedشده در شکل ۴-۵ نشان داده شده است.

اگر byte sequence را بررسی کنید، می‌بینید چیزی وجود ندارد که fieldها یا datatype آن‌ها را identify کند. encoding فقط از valueهایی تشکیل شده است که پشت سر هم concatenate شده‌اند. یک string صرفاً length prefixای است که بعد از آن byteهای UTF-8 می‌آیند، اما در data encodedشده چیزی وجود ندارد که به شما بگوید این value یک string است. این value می‌توانست به همان اندازه integer یا هر چیز دیگری باشد. یک integer با variable-length encoding encode می‌شود (همان encodingای که `CompactProtocol` مربوط به Thrift استفاده می‌کند).

**شکل ۴-۵.** رکورد نمونه که با Avro encode شده است.

برای parse کردن binary data، fieldها را به همان orderی که در schema آمده‌اند طی می‌کنید و از schema برای تعیین datatype هر field استفاده می‌کنید. این یعنی binary data فقط زمانی به‌درستی decode می‌شود که code خواننده دقیقاً از همان schemaای استفاده کند که code نویسنده استفاده کرده است. هر mismatch میان schema خواننده و نویسنده باعث decode نادرست data خواهد شد.

پس Avro چگونه از schema evolution پشتیبانی می‌کند؟

#### Writer’s schema و reader’s schema

در Avro، وقتی application می‌خواهد dataای را encode کند (برای write کردن در file یا database یا ارسال از طریق network و غیره)، data را با هر versionای از schema که می‌شناسد encode می‌کند؛ برای مثال، این schema ممکن است در application compile شده باشد. به این schema، `writer’s schema` گفته می‌شود.

وقتی application می‌خواهد dataای را decode کند (آن را از file یا database بخواند یا از network دریافت کند و غیره)، انتظار دارد data بر اساس schema مشخصی باشد که `reader’s schema` نام دارد. این همان schemaای است که application code به آن متکی است؛ ممکن است code در جریان build application از روی همین schema generate شده باشد.

ایدهٔ اصلی Avro این است که writer’s schema و reader’s schema لازم نیست یکسان باشند؛ فقط باید با هم compatible باشند. هنگام decode (read) کردن data، Avro library تفاوت‌ها را با قرار دادن writer’s schema و reader’s schema در کنار یکدیگر resolve می‌کند و data را از schema نویسنده به schema خواننده تبدیل می‌کند. specification مربوط به Avro [20] دقیقاً مشخص می‌کند این resolution چگونه انجام می‌شود و شکل ۴-۶ آن را نشان می‌دهد.

برای مثال، اگر fieldهای writer’s schema و reader’s schema در order متفاوتی باشند مشکلی وجود ندارد، چون schema resolution fieldها را بر اساس نام field با یکدیگر match می‌کند. اگر code خواننده با fieldی روبه‌رو شود که در writer’s schema وجود دارد اما در reader’s schema وجود ندارد، آن را ignore می‌کند. اگر code خواننده انتظار fieldی را داشته باشد اما writer’s schema fieldی با آن نام نداشته باشد، آن field با default valueای که در reader’s schema تعریف شده است پر می‌شود.

**شکل ۴-۶.** یک Avro reader تفاوت‌های writer’s schema و reader’s schema را resolve می‌کند.

#### Ruleهای schema evolution

در Avro، forward compatibility یعنی می‌توانید version جدید schema را به‌عنوان writer و version قدیمی schema را به‌عنوان reader داشته باشید. برعکس، backward compatibility یعنی version جدید schema reader و version قدیمی schema writer باشد.

برای حفظ compatibility، فقط می‌توانید fieldی را اضافه یا حذف کنید که default value داشته باشد. (field مربوط به `favoriteNumber` در schema Avro ما default value برابر `null` دارد.) برای مثال فرض کنید fieldی با default value اضافه کنید؛ در این صورت این field در schema جدید وجود دارد اما در schema قدیمی وجود ندارد. وقتی readerای که از schema جدید استفاده می‌کند recordی را بخواند که با schema قدیمی نوشته شده است، default value برای field missing قرار داده می‌شود.

اگر fieldی را اضافه کنید که default value نداشته باشد، readerهای جدید نمی‌توانند data نوشته‌شده توسط writerهای قدیمی را بخوانند و backward compatibility را می‌شکنید. اگر fieldی را حذف کنید که default value نداشته باشد، readerهای قدیمی نمی‌توانند data نوشته‌شده توسط writerهای جدید را بخوانند و forward compatibility را می‌شکنید.

در برخی programming languageها، `null` برای هر variableای default قابل‌قبولی است، اما در Avro چنین نیست. اگر می‌خواهید field بتواند `null` باشد، باید از union type استفاده کنید. برای مثال، `union { null, long, string } field;` مشخص می‌کند `field` می‌تواند number، string یا `null` باشد. فقط زمانی می‌توانید `null` را به‌عنوان default value استفاده کنید که یکی از branchهای union باشد.‡ این روش کمی verboseتر از nullable بودن همه‌چیز به‌صورت پیش‌فرض است، اما با صریح کردن اینکه چه چیزی می‌تواند `null` باشد و چه چیزی نمی‌تواند، به جلوگیری از bugها کمک می‌کند [22].

در نتیجه، Avro markerهای optional و required را به همان شکلی که Protocol Buffers و Thrift دارند ندارد؛ در عوض از union typeها و default valueها استفاده می‌کند.

تغییر datatype یک field ممکن است، به‌شرطی که Avro بتواند type را convert کند. تغییر نام یک field نیز امکان‌پذیر است، اما کمی پیچیده‌تر است: reader’s schema می‌تواند aliasهایی برای نام fieldها داشته باشد و بنابراین نام fieldهای schema مربوط به writer قدیمی را با aliasها match کند. این یعنی تغییر نام field backward compatible است، اما forward compatible نیست. به‌طور مشابه، اضافه کردن یک branch به union type backward compatible است، اما forward compatible نیست.

*پاورقی: دقیق‌تر بگوییم، default value باید از type مربوط به اولین branch union باشد؛ اما این محدودیت خاص Avro است و feature عمومی union typeها محسوب نمی‌شود.*

#### اما writer’s schema چیست؟

یک سؤال مهم که تاکنون از آن عبور کرده‌ایم این است: reader چگونه writer’s schemaای را می‌داند که یک قطعهٔ مشخص از data با آن encode شده است؟ نمی‌توانیم کل schema را با هر record همراه کنیم، چون احتمالاً schema از data encodedشده بزرگ‌تر خواهد بود و تمام صرفه‌جویی حاصل از binary encoding را بی‌اثر می‌کند.

پاسخ به contextی بستگی دارد که Avro در آن استفاده می‌شود. چند مثال را بررسی کنیم:

##### Large file with lots of records

یک use case رایج برای Avro، به‌خصوص در context Hadoop، ذخیره کردن file بزرگی است که millions record دارد و همهٔ recordها با یک schema یکسان encode شده‌اند. (در Chapter 10 دربارهٔ چنین موقعیتی بیشتر صحبت می‌کنیم.) در این حالت، writer آن file می‌تواند writer’s schema را فقط یک بار در ابتدای file قرار دهد. Avro برای این کار file formatای به نام `object container files` مشخص می‌کند.

##### Database with individually written records

در database، recordهای مختلف ممکن است در زمان‌های متفاوت و با writer’s schemaهای متفاوت write شوند؛ بنابراین نمی‌توان فرض کرد تمام recordها schema یکسانی دارند. ساده‌ترین راه‌حل این است که در ابتدای هر record encodedشده یک version number قرار دهیم و فهرستی از schema versionها را در database نگه داریم. Reader می‌تواند record را fetch کند، version number را استخراج کند و سپس writer’s schema مربوط به آن version number را از database fetch کند. با استفاده از آن writer’s schema، می‌تواند بقیهٔ record را decode کند. برای مثال، `Espresso` [23] به این روش کار می‌کند.

##### Sending records over a network connection

وقتی دو process از طریق یک network connection دوطرفه با یکدیگر ارتباط برقرار می‌کنند، می‌توانند هنگام setup connection دربارهٔ schema version مذاکره کنند و سپس در طول عمر connection از همان schema استفاده کنند. `Avro RPC protocol` (به بخش «Dataflow Through Services: REST and RPC» در صفحهٔ ۱۳۱ مراجعه کنید) به همین روش کار می‌کند.

در هر صورت داشتن databaseای از schema versionها مفید است، چون هم به‌عنوان documentation عمل می‌کند و هم فرصتی برای بررسی schema compatibility فراهم می‌کند [24]. به‌عنوان version number می‌توانید از یک integer ساده و افزایشی یا hash مربوط به schema استفاده کنید.

#### Dynamically generated schemas

یکی از مزیت‌های رویکرد Avro در مقایسه با Protocol Buffers و Thrift این است که schema شامل tag number نیست. اما چرا این موضوع مهم است؟ مشکل نگه داشتن چند number در schema چیست؟

تفاوت این است که Avro با dynamically generated schemaها سازگارتر است. برای مثال، فرض کنید یک relational database دارید و می‌خواهید محتوای آن را در file dump کنید و برای دوری از مشکلاتی که دربارهٔ textual formatها (JSON، CSV و SQL) گفتیم، از binary format استفاده کنید. اگر از Avro استفاده کنید، می‌توانید نسبتاً به‌سادگی یک Avro schema (در همان JSON representation که پیش‌تر دیدیم) از relational schema تولید کنید و محتوای database را با آن schema encode کرده و همهٔ data را در یک Avro object container file dump کنید [25]. برای هر database table یک record schema تولید می‌کنید و هر column به یک field در آن record تبدیل می‌شود. نام column در database به نام field در Avro map می‌شود.

حالا اگر database schema تغییر کند (برای مثال، یک column به table اضافه و یک column از آن حذف شود)، می‌توانید به‌سادگی Avro schema جدیدی از database schema به‌روزشده تولید کنید و data را در schema جدید Avro export کنید. فرآیند data export لازم نیست به schema change توجه خاصی داشته باشد؛ هر بار که اجرا می‌شود فقط schema conversion را انجام می‌دهد. هر کسی که data fileهای جدید را بخواند، تغییر fieldهای record را می‌بیند، اما چون fieldها با نام شناسایی می‌شوند، writer’s schema به‌روزشده همچنان می‌تواند با reader’s schema قدیمی match شود.

در مقابل، اگر برای این کار از Thrift یا Protocol Buffers استفاده می‌کردید، احتمالاً باید field tagها را به‌صورت دستی تعیین می‌کردید. هر بار که database schema تغییر می‌کرد، administrator مجبور بود mapping میان نام columnهای database و field tagها را به‌صورت دستی update کند. (ممکن است بتوان این کار را automate کرد، اما schema generator باید بسیار مراقب باشد که tagهایی را که قبلاً استفاده شده‌اند دوباره assign نکند.) چنین dynamically generated schemaای هرگز هدف طراحی Thrift یا Protocol Buffers نبوده است، درحالی‌که برای Avro هدفی مهم محسوب می‌شده است.

#### Code generation و dynamically typed languageها

Thrift و Protocol Buffers به code generation متکی هستند: پس از تعریف schema، می‌توانید codeای تولید کنید که این schema را در programming language انتخابی شما implement کند. این قابلیت در statically typed languageهایی مانند Java، C++ یا C# مفید است، چون اجازه می‌دهد برای data decodeشده از in-memory structureهای efficient استفاده کنید و هنگام نوشتن programهایی که به data structureها دسترسی دارند، type checking و autocompletion در IDE داشته باشید.

در dynamically typed programming languageهایی مانند JavaScript، Ruby یا Python، code generation چندان فایده‌ای ندارد، چون compile-time type checkerای وجود ندارد که بخواهد آن را satisfy کند. در این languageها معمولاً به code generation با دید خوبی نگاه نمی‌شود، چون این languageها اساساً از یک compilation step صریح اجتناب می‌کنند. علاوه بر این، در مورد dynamically generated schema (مانند Avro schemaای که از database table تولید شده است)، code generation مانعی غیرضروری برای دسترسی به data است.

Avro برای statically typed programming languageها code generation optional ارائه می‌دهد، اما بدون code generation نیز به همان خوبی قابل استفاده است. اگر object container fileای داشته باشید که writer’s schema را در خود embed کرده باشد، می‌توانید آن را با Avro library باز کنید و data را تقریباً به همان شکلی ببینید که یک JSON file را مشاهده می‌کنید. این file self-describing است، چون تمام metadata لازم را در خود دارد.

این property در کنار dynamically typed data processing languageهایی مانند `Apache Pig` [26] بسیار مفید است. در Pig می‌توانید چند Avro file را باز کنید، تحلیل آن‌ها را شروع کنید و datasetهای مشتق‌شده را بدون اینکه حتی لازم باشد دربارهٔ schema فکر کنید، در output fileهایی با Avro format بنویسید.

### The Merits of Schemas

همان‌طور که دیدیم، Protocol Buffers، Thrift و Avro همگی از schema برای توصیف یک binary encoding format استفاده می‌کنند. schema languageهای آن‌ها بسیار ساده‌تر از `XML Schema` یا `JSON Schema` هستند؛ آن دو از ruleهای validation بسیار جزئی‌تری پشتیبانی می‌کنند (برای مثال: «value رشته‌ای این field باید با این regular expression match شود» یا «value integer این field باید بین ۰ و ۱۰۰ باشد»). چون Protocol Buffers، Thrift و Avro ساده‌تر implement و ساده‌تر استفاده می‌شوند، توانسته‌اند از programming languageهای نسبتاً متنوعی پشتیبانی کنند.

ایده‌هایی که این encodingها بر آن‌ها بنا شده‌اند به‌هیچ‌وجه جدید نیستند. برای مثال، این encodingها شباهت‌های زیادی با `ASN.1` دارند؛ یک schema definition language که نخستین بار در سال ۱۹۸۴ استاندارد شد [27]. ASN.1 برای تعریف network protocolهای مختلف استفاده شده است و binary encoding آن (`DER`) هنوز هم برای encode کردن SSL certificateها (برای مثال `X.509`) استفاده می‌شود [28]. ASN.1 با استفاده از tag numberها، مشابه Protocol Buffers و Thrift، از schema evolution پشتیبانی می‌کند [29]. بااین‌حال، بسیار پیچیده و بد documentation شده است؛ بنابراین ASN.1 احتمالاً انتخاب خوبی برای applicationهای جدید نیست.

بسیاری از data systemها نیز نوعی proprietary binary encoding برای data خود implement می‌کنند. برای مثال، بیشتر relational databaseها network protocolای دارند که از طریق آن می‌توانید queryها را به database بفرستید و response دریافت کنید. این protocolها معمولاً مخصوص یک database مشخص هستند و vendor database driverای (برای مثال با استفاده از APIهای `ODBC` یا `JDBC`) ارائه می‌دهد که responseها را از network protocol database به in-memory data structure decode می‌کند.

بنابراین می‌بینیم که اگرچه textual data formatهایی مانند JSON، XML و CSV widely used هستند، binary encodingهای مبتنی بر schema نیز گزینه‌ای viable محسوب می‌شوند. این encodingها چند property خوب دارند:

- می‌توانند بسیار compactتر از variantهای مختلف «binary JSON» باشند، چون می‌توانند نام fieldها را از data encodedشده حذف کنند.
- schema نوع ارزشمندی از documentation است و چون برای decoding به schema نیاز داریم، می‌توان مطمئن بود schema به‌روز است (درحالی‌که documentationای که به‌صورت دستی نگهداری می‌شود به‌راحتی ممکن است از واقعیت فاصله بگیرد).
- نگهداری databaseای از schemaها اجازه می‌دهد پیش از deploy شدن هر چیزی، forward و backward compatibility مربوط به schema changeها را بررسی کنید.
- برای کاربرانی که از statically typed programming languageها استفاده می‌کنند، امکان generate کردن code از روی schema مفید است، چون type checking را در compile time ممکن می‌کند.

خلاصه اینکه schema evolution همان نوع flexibilityای را فراهم می‌کند که schemaless یا schema-on-read JSON databaseها ارائه می‌دهند (به بخش «Schema flexibility in the document model» در صفحهٔ ۳۹ مراجعه کنید)، درحالی‌که guaranteeهای بهتری دربارهٔ data و tooling بهتری در اختیار شما می‌گذارد.

## Key Terms

- `Encoding` — تبدیل data از representation درون memory به sequence مستقل از byteها برای storage یا انتقال.
- `Decoding` — بازسازی representation قابل استفاده از byte sequence encodedشده.
- `Serialization` — نام رایج دیگری برای تبدیل object به byte sequence؛ در این کتاب برای جلوگیری از ابهام معمولاً از encoding استفاده می‌شود.
- `Deserialization` — بازگرداندن data از representation serialized یا encoded به object یا data structure قابل استفاده.
- `Data Encoding Format` — قرارداد و ساختار byte sequenceای که برای نمایش، storage یا انتقال data استفاده می‌شود.
- `Language-Specific Format` — encodingای که به object model یا runtime یک programming language خاص وابسته است.
- `Binary Encoding` — نمایش data به‌صورت byteهای فشرده و machine-oriented، نه متن human-readable.
- `Schema` — تعریف structure، fieldها، datatypeها و ruleهای data encodedشده.
- `Schema Evolution` — تغییر کنترل‌شدهٔ schema در طول زمان، همراه با حفظ compatibility میان code و data versionهای مختلف.
- `Schema Compatibility` — توانایی schemaهای جدید و قدیمی برای read و write کردن data یکدیگر.
- `Backward Compatibility` — توانایی code یا schema جدید برای خواندن data نوشته‌شده توسط version قدیمی.
- `Forward Compatibility` — توانایی code یا schema قدیمی برای خواندن data نوشته‌شده توسط version جدید.
- `Field Identifier` — tag number یا شناسه‌ای که field را در binary encoding مشخص می‌کند.
- `Field Tag` — number کوتاهی که در Thrift و Protocol Buffers به‌عنوان شناسهٔ پایدار field استفاده می‌شود.
- `Optional Field` — fieldای که نبودن آن در record مجاز است و معمولاً default value یا رفتار مشخصی دارد.
- `Required Field` — fieldای که باید وجود داشته باشد؛ اضافه کردن آن پس از deployment اولیه می‌تواند backward compatibility را بشکند.
- `Default Value` — valueای که هنگام نبودن یک field در data قدیمی برای آن استفاده می‌شود.
- `Type System` — مجموعهٔ datatypeها و ruleهایی که مشخص می‌کند valueها چگونه تفسیر و validate شوند.
- `Data Representation` — شکل داخلی یا external data، مانند object در memory یا byte sequence در file و network.
- `Thrift` — binary encoding و interface definition system مبتنی بر schema که از code generation و field tag استفاده می‌کند.
- `Protocol Buffers` — binary encoding system مبتنی بر schema که برای fieldها tag number و برای evolution markerهایی مانند optional و repeated دارد.
- `Avro` — binary encoding format مبتنی بر schema که resolution را میان writer’s schema و reader’s schema انجام می‌دهد.
- `Writer’s Schema` — schemaای که producer هنگام encode کردن data از آن استفاده کرده است.
- `Reader’s Schema` — schemaای که consumer هنگام decode کردن data انتظار دارد.
- `Union Type` — typeای که اجازه می‌دهد یک field یکی از چند type مشخص، مانند `null`، `long` یا `string` باشد.
- `Variable-Length Encoding` — encoding عددها با تعداد byte متناسب با اندازهٔ value، برای کاهش حجم numberهای کوچک.
- `Code Generation` — تولید خودکار class یا code از روی schema برای encode، decode، type checking و دسترسی به data.
- `Self-Describing File` — fileای که metadata یا schema لازم برای تفسیر محتوای خود را همراه data نگه می‌دارد.

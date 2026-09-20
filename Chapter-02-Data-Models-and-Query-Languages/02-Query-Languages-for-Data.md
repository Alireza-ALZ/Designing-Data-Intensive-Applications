# Chapter 2 — Data Models and Query Languages

## Query Languages for Data

وقتی relational model معرفی شد، روش جدیدی برای query کردن داده نیز همراه آن آمد: `SQL` یک `declarative query language` است، در حالی که `IMS` و `CODASYL` با استفاده از code امری (`imperative`) database را query می‌کردند. این تفاوت یعنی چه؟

بسیاری از programming languageهای رایج imperative هستند. برای مثال، اگر فهرستی از گونه‌های جانوری داشته باشید و بخواهید فقط کوسه‌های آن فهرست را برگردانید، ممکن است چیزی شبیه این بنویسید:

```javascript
function getSharks() {
    var sharks = [];
    for (var i = 0; i < animals.length; i++) {
        if (animals[i].family === "Sharks") {
            sharks.push(animals[i]);
        }
    }
    return sharks;
}
```

در `relational algebra`، به‌جای آن می‌نویسید:

```text
sharks = σfamily = “Sharks” (animals)
```

در این عبارت، σ (حرف یونانی sigma) عملگر `selection` است و فقط animalهایی را برمی‌گرداند که شرط `family = “Sharks”` را برآورده می‌کنند.

هنگام تعریف SQL، ساختار آن تا حد زیادی از relational algebra پیروی می‌کرد:

```sql
SELECT * FROM animals WHERE family = 'Sharks';
```

یک زبان imperative به computer می‌گوید عملیات مشخصی را با ترتیب مشخصی انجام دهد. می‌توانید تصور کنید که code را خط‌به‌خط اجرا می‌کنید، conditionها را ارزیابی می‌کنید، variableها را تغییر می‌دهید و تصمیم می‌گیرید loop یک بار دیگر اجرا شود یا نه.

در یک `declarative query language` مانند SQL یا relational algebra، فقط pattern داده‌ای را که می‌خواهید مشخص می‌کنید: اینکه resultها باید چه conditionهایی را داشته باشند و داده چگونه transform شود (برای مثال sort، group یا aggregate شود). اما مشخص نمی‌کنید این هدف چگونه به دست بیاید. تصمیم‌گیری دربارهٔ indexهای مورد استفاده، روش‌های join و ترتیب اجرای بخش‌های مختلف query بر عهدهٔ `query optimizer` در database system است.

یک declarative query language جذاب است، چون معمولاً conciseتر است و کار کردن با آن از یک imperative API ساده‌تر است. اما مهم‌تر اینکه جزئیات implementation database engine را پنهان می‌کند و به database system اجازه می‌دهد performance را بهبود دهد، بدون اینکه لازم باشد queryها تغییر کنند.

برای مثال، در imperative code ابتدای این بخش، فهرست animalها ترتیب مشخصی دارد. اگر database بخواهد در پس‌زمینه فضای بلااستفادهٔ disk را آزاد کند، ممکن است لازم باشد recordها را جابه‌جا کند و در نتیجه ترتیب نمایش animalها تغییر کند. آیا database می‌تواند این کار را بدون شکستن queryها با اطمینان انجام دهد؟

مثال SQL هیچ ترتیب خاصی را guarantee نمی‌کند، بنابراین اگر ترتیب تغییر کند مشکلی ندارد. اما اگر query به‌صورت imperative code نوشته شده باشد، database هرگز نمی‌تواند مطمئن باشد که code به آن ترتیب وابسته است یا نه. محدودتر بودن functionality در SQL، فضای بیشتری برای automatic optimization در اختیار database می‌گذارد.

در نهایت، declarative languageها معمولاً برای اجرای parallel مناسب‌ترند. امروزه CPUها بیشتر با اضافه کردن coreهای جدید سریع‌تر می‌شوند، نه با افزایش چشمگیر clock speed نسبت به گذشته [31]. parallel کردن imperative code روی چند core یا چند machine بسیار دشوار است، چون این code دستورهایی را مشخص می‌کند که باید به ترتیب خاصی اجرا شوند. declarative languageها شانس بیشتری برای سریع‌تر شدن از طریق parallel execution دارند، چون فقط pattern resultها را مشخص می‌کنند، نه algorithmی را که باید برای به‌دست آوردن result استفاده شود. اگر مناسب باشد، database آزاد است query language را با یک implementation موازی اجرا کند [32].

### Declarative Queries on the Web

مزیت‌های declarative query languageها فقط به databaseها محدود نمی‌شود. برای روشن شدن موضوع، رویکرد declarative و imperative را در محیطی کاملاً متفاوت مقایسه کنیم: یک web browser.

فرض کنید وب‌سایتی دربارهٔ animalهای اقیانوس دارید. کاربر اکنون صفحهٔ مربوط به کوسه‌ها را می‌بیند، بنابراین item مربوط به این صفحه را در navigation به‌صورت selected علامت می‌زنید:

```html
<ul>
    <li class="selected">
         <p>Sharks</p>
         <ul>
              <li>Great White Shark</li>
              <li>Tiger Shark</li>
              <li>Hammerhead Shark</li>
         </ul>
    </li>
    <li>
         <p>Whales</p>
         <ul>
              <li>Blue Whale</li>
              <li>Humpback Whale</li>
             <li>Fin Whale</li>
         </ul>
    </li>
</ul>
```

item انتخاب‌شده با CSS classِ `"selected"` علامت‌گذاری شده است. عبارت `<p>Sharks</p>` عنوان صفحهٔ انتخاب‌شدهٔ فعلی است.

اکنون فرض کنید می‌خواهید title صفحهٔ انتخاب‌شده background آبی داشته باشد تا به‌صورت visual highlight شود. این کار با CSS ساده است:

```css
li.selected > p {
    background-color: blue;
}
```

در اینجا CSS selectorِ `li.selected > p` pattern عناصری را اعلام می‌کند که می‌خواهیم style آبی روی آن‌ها اعمال شود: همهٔ elementهای `<p>` که parent مستقیم آن‌ها یک element از نوع `<li>` با CSS classِ `selected` است. در مثال، elementِ `<p>Sharks</p>` با این pattern match می‌شود، اما `<p>Whales</p>` match نمی‌شود، چون parentِ `<li>` آن classِ `class="selected"` ندارد.

اگر به‌جای CSS از XSL استفاده می‌کردید، می‌توانستید کار مشابهی انجام دهید:

```xml
<xsl:template match="li[@class='selected']/p">
    <fo:block background-color="blue">
        <xsl:apply-templates/>
    </fo:block>
</xsl:template>
```

در اینجا XPath expressionِ `li[@class='selected']/p` معادل CSS selectorِ `li.selected > p` در مثال قبل است. CSS و XSL هر دو declarative languageهایی هستند که برای مشخص کردن style یک document به کار می‌روند.

حالا تصور کنید اگر مجبور بودید از رویکرد imperative استفاده کنید، زندگی چگونه می‌شد. در JavaScript و با استفاده از core `Document Object Model (DOM) API`، result ممکن بود چنین شکلی داشته باشد:

```javascript
var liElements = document.getElementsByTagName("li");
for (var i = 0; i < liElements.length; i++) {
    if (liElements[i].className === "selected") {
        var children = liElements[i].childNodes;
        for (var j = 0; j < children.length; j++) {
            var child = children[j];
            if (child.nodeType === Node.ELEMENT_NODE && child.tagName === "P") {
                child.setAttribute("style", "background-color: blue");
            }
        }
    }
}
```

این JavaScript به‌صورت imperative، elementِ `<p>Sharks</p>` را دارای background آبی می‌کند، اما code بسیار نامناسب است. این code نه‌تنها بسیار طولانی‌تر و فهم آن دشوارتر از معادل‌های CSS و XSL است، بلکه چند مشکل جدی نیز دارد:

- اگر classِ `selected` حذف شود (مثلاً چون کاربر روی صفحهٔ دیگری کلیک کرده است)، رنگ آبی حذف نمی‌شود؛ حتی اگر code دوباره اجرا شود. بنابراین item تا زمان reload کامل page همچنان highlight باقی می‌ماند. در CSS، browser به‌طور خودکار تشخیص می‌دهد که ruleِ `li.selected > p` دیگر برقرار نیست و به‌محض حذف classِ selected، background آبی را حذف می‌کند.
- اگر بخواهید از API جدیدی مانند `document.getElementsByClassName("selected")` یا حتی `document.evaluate()` استفاده کنید که ممکن است performance را بهتر کند، باید code را بازنویسی کنید. در مقابل، browser vendorها می‌توانند performance CSS و XPath را بدون شکستن compatibility بهبود دهند.

در web browser، استفاده از declarative CSS styling بسیار بهتر از دستکاری imperative styleها در JavaScript است. به همین شکل، در databaseها نیز declarative query languageهایی مانند SQL در عمل بسیار بهتر از imperative query APIها هستند.

### MapReduce Querying

`MapReduce` یک programming model برای پردازش bulk حجم زیادی از داده روی machineهای متعدد است که Google آن را popular کرد [33]. برخی NoSQL datastoreها، از جمله MongoDB و CouchDB، شکل محدودی از MapReduce را به‌عنوان mechanism اجرای read-only query روی documentهای متعدد پشتیبانی می‌کنند.

MapReduce به‌طور کلی در فصل ۱۰ با جزئیات بیشتری توضیح داده می‌شود. فعلاً فقط استفادهٔ MongoDB از این model را به‌اختصار بررسی می‌کنیم.

MapReduce نه یک declarative query language است و نه یک imperative query API کامل؛ بلکه جایی بین این دو قرار می‌گیرد. logic مربوط به query با snippetهایی از code بیان می‌شود و processing framework آن snippetها را بارها فراخوانی می‌کند. این model بر پایهٔ functionهای `map` (که `collect` نیز نامیده می‌شود) و `reduce` (که `fold` یا `inject` نیز نامیده می‌شود) بنا شده است؛ functionهایی که در بسیاری از functional programming languageها وجود دارند.

برای مثال، فرض کنید marine biologist هستید و هر بار که animalهایی را در اقیانوس می‌بینید، یک observation record در database اضافه می‌کنید. اکنون می‌خواهید گزارشی بسازید که نشان دهد در هر ماه چند shark مشاهده کرده‌اید.

در PostgreSQL ممکن است این query را به شکل زیر بنویسید:

```sql
SELECT date_trunc('month', observation_timestamp) AS observation_month,
       sum(num_animals) AS total_animals
FROM observations
WHERE family = 'Sharks'
GROUP BY observation_month;
```

functionِ `date_trunc('month', timestamp)` ماه تقویمی‌ای را که timestamp در آن قرار دارد تعیین می‌کند و timestamp دیگری را برمی‌گرداند که آغاز آن ماه را نشان می‌دهد. به عبارت دیگر، timestamp را به نزدیک‌ترین ماه، رو به پایین، round می‌کند.

این query ابتدا observationها را filter می‌کند تا فقط گونه‌های خانوادهٔ Sharks باقی بمانند، سپس observationها را بر اساس ماه تقویمی وقوع آن‌ها group می‌کند و در نهایت تعداد animalهای مشاهده‌شده در تمام observationهای آن ماه را با هم جمع می‌کند.

همین کار را می‌توان با قابلیت MapReduce در MongoDB به شکل زیر بیان کرد:

```javascript
db.observations.mapReduce(
    function map() {
        var year = this.observationTimestamp.getFullYear();
        var month = this.observationTimestamp.getMonth() + 1;
        emit(year + "-" + month, this.numAnimals);
    },
    function reduce(key, values) {
        return Array.sum(values);
    },
    {
        query: { family: "Sharks" },
        out: "monthlySharkReport"
    }
);
```

filter مربوط به در نظر گرفتن فقط گونه‌های shark را می‌توان به‌صورت declarative مشخص کرد؛ این قابلیت، extension اختصاصی MongoDB برای MapReduce است.

functionِ JavaScript به نام `map` برای هر documentای که با query match شود یک بار فراخوانی می‌شود و `this` به object مربوط به همان document اشاره می‌کند.

functionِ map یک key (رشته‌ای شامل year و month، مانند `"2013-12"` یا `"2014-1"`) و یک value (تعداد animalهای آن observation) emit می‌کند.

جفت‌های key-value تولیدشده توسط map بر اساس key group می‌شوند. برای همهٔ جفت‌های key-valueای که key یکسان دارند (یعنی همان ماه و year)، functionِ reduce یک بار فراخوانی می‌شود.

functionِ reduce تعداد animalها را در تمام observationهای یک ماه مشخص با هم جمع می‌کند.

خروجی نهایی در collectionای به نام `monthlySharkReport` نوشته می‌شود.

برای مثال، فرض کنید collectionِ observations شامل این دو document باشد:

```javascript
{
    observationTimestamp: Date.parse("Mon, 25 Dec 1995 12:34:56 GMT"),
    family:     "Sharks",
    species:    "Carcharodon carcharias",
    numAnimals: 3
}
{
    observationTimestamp: Date.parse("Tue, 12 Dec 1995 16:17:18 GMT"),
    family:     "Sharks",
    species:    "Carcharias taurus",
    numAnimals: 4
}
```

functionِ map برای هر document یک بار فراخوانی می‌شود و در نتیجه این دو خروجی ایجاد می‌شوند:

```javascript
emit("1995-12", 3)
emit("1995-12", 4)
```

سپس functionِ reduce با این ورودی فراخوانی می‌شود:

```javascript
reduce("1995-12", [3, 4])
```

و مقدار `7` را برمی‌گرداند.

functionهای map و reduce در کارهایی که اجازه دارند انجام دهند تا حدی محدود هستند. آن‌ها باید `pure function` باشند؛ یعنی فقط از داده‌ای استفاده کنند که به‌عنوان input به آن‌ها داده شده است، نتوانند queryهای اضافی به database ارسال کنند و هیچ `side effect`ای نداشته باشند. این محدودیت‌ها به database اجازه می‌دهند functionها را در هر محل و با هر ترتیبی اجرا کنند و در صورت failure آن‌ها را دوباره اجرا کنند. بااین‌حال، این functionها همچنان قدرتمندند: می‌توانند stringها را parse کنند، functionهای library را صدا بزنند، محاسبه انجام دهند و کارهای دیگری انجام دهند.

MapReduce یک programming model نسبتاً low-level برای distributed execution روی clusterای از machineهاست. query languageهای سطح بالاتر مانند SQL می‌توانند به‌صورت pipelineای از operationهای MapReduce پیاده‌سازی شوند (به فصل ۱۰ مراجعه کنید)، اما distributed implementationهای زیادی از SQL نیز وجود دارند که از MapReduce استفاده نمی‌کنند. توجه کنید که هیچ چیزی در SQL آن را مجبور نمی‌کند فقط روی یک machine اجرا شود و MapReduce نیز انحصاری بر distributed query execution ندارد.

امکان استفاده از JavaScript code در میان query، قابلیت خوبی برای queryهای advanced است؛ اما این قابلیت به MapReduce محدود نمی‌شود. بعضی SQL databaseها نیز می‌توانند با JavaScript functionها extend شوند [34].

یکی از مشکلات usability در MapReduce این است که باید دو JavaScript function را با دقت هماهنگ و پیاده‌سازی کنید؛ کاری که اغلب از نوشتن یک query واحد دشوارتر است. علاوه بر این، یک declarative query language فرصت‌های بیشتری برای query optimizer فراهم می‌کند تا performance query را بهبود دهد. به همین دلیل MongoDB 2.2 پشتیبانی از declarative query languageای به نام `aggregation pipeline` را اضافه کرد [9]. در این language، query شمارش sharkها به شکل زیر نوشته می‌شود:

```javascript
db.observations.aggregate([
    { $match: { family: "Sharks" } },
    { $group: {
        _id: {
            year: { $year: "$observationTimestamp" },
            month: { $month: "$observationTimestamp" }
        },
        totalAnimals: { $sum: "$numAnimals" }
    } }
]);
```

زبان aggregation pipeline از نظر expressiveness شبیه subsetای از SQL است، اما به‌جای syntax جمله‌مانند انگلیسی SQL از syntax مبتنی بر JSON استفاده می‌کند. این تفاوت شاید بیشتر مسئله‌ای سلیقه‌ای باشد. نتیجهٔ کلی این است که یک NoSQL system ممکن است در عمل مشغول reinvent کردن SQL شود، هرچند آن را در ظاهری متفاوت ارائه کند.

*پاورقی: IMS و CODASYL هر دو imperative query API داشتند. Applicationها معمولاً از COBOL code برای پیمایش recordهای database، یک record در هر لحظه، استفاده می‌کردند [2, 16].*

## Key Terms

| English Term | Persian Explanation | Engineering Meaning |
|---|---|---|
| Query Language | زبان بیان query برای خواندن یا پردازش داده | interfaceی است که به application اجازه می‌دهد خواستهٔ خود را با syntax مشخص به database منتقل کند. |
| Declarative Query | queryای که نتیجهٔ موردنظر را مشخص می‌کند، نه مراحل رسیدن به آن | database می‌تواند execution plan، index و ترتیب operationها را خودش انتخاب و optimize کند. |
| Imperative Query | query یا APIای که مراحل و ترتیب اجرای operationها را مشخص می‌کند | کنترل بیشتری به application می‌دهد، اما coupling آن با implementation و access path بیشتر است. |
| Declarative Programming | برنامه‌نویسی بر اساس توصیف result یا rule، بدون تعیین algorithm دقیق | abstraction و امکان optimization خودکار را افزایش می‌دهد. |
| Imperative Programming | برنامه‌نویسی با تعیین گام‌ها و ترتیب اجرای آن‌ها | رفتار اجرایی را صریح کنترل می‌کند، اما parallelization و تغییر implementation دشوارتر می‌شود. |
| Relational Algebra | مجموعه‌ای از operationهای formal برای کار با relationها | مبنای نظری بسیاری از queryهای relational و SQL است. |
| Query Execution | اجرای عملی query روی داده | شامل انتخاب plan، خواندن index، join، filter، grouping و تولید result است. |
| Parallel Processing | اجرای هم‌زمان بخش‌های یک کار روی چند core یا machine | با تقسیم کار می‌تواند throughput را افزایش دهد، به شرطی که taskها قابلیت parallel شدن داشته باشند. |
| MapReduce | programming model برای پردازش داده در دو مرحلهٔ map و reduce | اجرای distributed روی دادهٔ بزرگ را ممکن می‌کند، اما نسبت به query languageهای declarative سطح پایین‌تری دارد. |
| Map Function | functionی که ورودی‌ها را می‌خواند و key-value تولید می‌کند | داده را به خروجی‌های قابل group شدن برای مرحلهٔ reduce تبدیل می‌کند. |
| Reduce Function | functionی که valueهای مربوط به یک key را ترکیب می‌کند | aggregation یا محاسبهٔ نهایی هر گروه را انجام می‌دهد. |
| Pure Function | functionی بدون وابستگی بیرونی و بدون side effect | اجرای مجدد، جابه‌جایی و parallel کردن آن در distributed system امن‌تر است. |
| Side Effect | تغییری خارج از result function، مانند write یا تغییر state مشترک | اجرای مجدد یا موازی function را دشوار و احتمال inconsistency را بیشتر می‌کند. |
| Distributed Query Execution | اجرای query روی چند machine یا node | برای پردازش datasetهای بزرگ استفاده می‌شود و به coordination و تقسیم کار نیاز دارد. |
| Aggregation Pipeline | query language مرحله‌ای MongoDB برای filter و group کردن داده | جایگزینی declarativeتر برای بسیاری از queryهای MapReduce است. |
| Composability | قابلیت ترکیب componentها یا operationها برای ساختن behavior پیچیده‌تر | abstractionها را reusable می‌کند و طراحی pipelineهای قابل‌گسترش را آسان‌تر می‌سازد. |

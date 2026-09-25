# Chapter 2 — Data Models and Query Languages

## Graph-Like Data Models

پیش‌تر دیدیم که `many-to-many relationship`ها یکی از ویژگی‌های مهم برای تمایز میان data modelهای مختلف هستند. اگر application شما عمدتاً relationshipهای one-to-many (دادهٔ tree-structured) داشته باشد یا میان recordها relationshipی وجود نداشته باشد، document model انتخاب مناسبی است.

اما اگر many-to-many relationshipها در دادهٔ شما بسیار رایج باشند چه؟ relational model می‌تواند حالت‌های سادهٔ many-to-many relationship را مدیریت کند، اما هرچه connectionهای درون داده پیچیده‌تر شوند، طبیعی‌تر است که داده را به‌صورت graph model کنیم.

یک graph از دو نوع object تشکیل می‌شود: `vertex`ها (که node یا entity نیز نامیده می‌شوند) و `edge`ها (که relationship یا arc نیز نامیده می‌شوند). انواع زیادی از داده را می‌توان به‌صورت graph مدل کرد. چند نمونهٔ رایج:

**Social graph**

Vertexها انسان‌ها هستند و edgeها نشان می‌دهند کدام افراد یکدیگر را می‌شناسند.

**Web graph**

Vertexها web pageها هستند و edgeها linkهای HTML به pageهای دیگر را نشان می‌دهند.

**Road or rail network**

Vertexها junctionها هستند و edgeها جاده‌ها یا خط‌های راه‌آهن میان آن‌ها را نشان می‌دهند.

الگوریتم‌های شناخته‌شده‌ای می‌توانند روی این graphها کار کنند. برای مثال، سیستم‌های navigation خودرو کوتاه‌ترین path میان دو نقطه را در یک road network پیدا می‌کنند و می‌توان از `PageRank` روی web graph برای تعیین محبوبیت یک web page و در نتیجه ranking آن در search resultها استفاده کرد.

در مثال‌های بالا، همهٔ vertexهای graph نمایندهٔ یک نوع object هستند؛ به‌ترتیب انسان، web page یا junction جاده. اما graphها به چنین دادهٔ homogeneousای محدود نیستند. یکی از کاربردهای قدرتمند graph این است که راهی یکپارچه برای ذخیرهٔ typeهای کاملاً متفاوت object در یک datastore فراهم کند. برای مثال، Facebook یک graph واحد با typeهای متنوعی از vertex و edge نگهداری می‌کند: vertexها نمایندهٔ انسان‌ها، locationها، eventها، check-inها و commentهای userها هستند؛ edgeها نشان می‌دهند کدام افراد دوست یکدیگرند، هر check-in در کدام location رخ داده، چه کسی روی کدام post comment گذاشته، چه کسانی در کدام event شرکت کرده‌اند و موارد دیگر [35].

در این بخش از مثالی استفاده می‌کنیم که در شکل ۲-۵ نشان داده شده است. این مثال می‌تواند از یک social network یا genealogical database گرفته شده باشد: دو نفر را نشان می‌دهد، Lucy از Idaho و Alain از Beaune در فرانسه. آن‌ها ازدواج کرده‌اند و در London زندگی می‌کنند.

*شکل ۲-۵. نمونه‌ای از دادهٔ graph-structured؛ boxها vertex و arrowها edge را نشان می‌دهند.*

راه‌های متفاوت اما مرتبطی برای structure دادن و query کردن داده در graphها وجود دارد. در این بخش `property graph model` (پیاده‌سازی‌شده در Neo4j، Titan و InfiniteGraph) و `triple-store model` (پیاده‌سازی‌شده در Datomic، AllegroGraph و موارد دیگر) را بررسی می‌کنیم. همچنین سه declarative query language برای graphها را می‌بینیم: Cypher، SPARQL و Datalog. علاوه بر این‌ها، imperative graph query languageهایی مانند Gremlin [36] و graph processing frameworkهایی مانند Pregel نیز وجود دارند (به فصل ۱۰ مراجعه کنید).

### Property Graphs

در property graph model، هر vertex شامل این موارد است:

- یک identifier یکتا
- مجموعه‌ای از outgoing edgeها
- مجموعه‌ای از incoming edgeها
- مجموعه‌ای از propertyها به‌صورت key-value pair

هر edge نیز شامل این موارد است:

- یک identifier یکتا
- vertexای که edge از آن شروع می‌شود (`tail vertex`)
- vertexای که edge به آن ختم می‌شود (`head vertex`)
- labelای برای توصیف نوع relationship میان دو vertex
- مجموعه‌ای از propertyها به‌صورت key-value pair

می‌توانید یک graph store را متشکل از دو relational table در نظر بگیرید: یکی برای vertexها و دیگری برای edgeها؛ همان‌طور که در مثال ۲-۲ نشان داده شده است. این schema از `json datatype` در PostgreSQL برای ذخیرهٔ propertyهای هر vertex یا edge استفاده می‌کند. head و tail هر edge ذخیره می‌شوند؛ اگر بخواهید مجموعهٔ incoming یا outgoing edgeهای یک vertex را پیدا کنید، می‌توانید table مربوط به edgeها را بر اساس `head_vertex` یا `tail_vertex` query کنید.

**مثال ۲-۲. نمایش یک property graph با استفاده از relational schema**

```sql
CREATE TABLE vertices (
    vertex_id integer PRIMARY KEY,
    properties json
);

CREATE TABLE edges (
    edge_id     integer PRIMARY KEY,
    tail_vertex integer REFERENCES vertices (vertex_id),
    head_vertex integer REFERENCES vertices (vertex_id),
    label       text,
    properties json
);

CREATE INDEX edges_tails ON edges (tail_vertex);
CREATE INDEX edges_heads ON edges (head_vertex);
```

چند ویژگی مهم این model عبارت‌اند از:

1. هر vertex می‌تواند edgeای به هر vertex دیگری داشته باشد. schemaای وجود ندارد که مشخص کند چه typeهایی از object می‌توانند یا نمی‌توانند به یکدیگر مرتبط شوند.
2. برای هر vertex می‌توانید incoming و outgoing edgeهای آن را به‌صورت کارآمد پیدا کنید و graph را traverse کنید؛ یعنی در امتداد زنجیره‌ای از vertexها، هم به سمت جلو و هم به سمت عقب حرکت کنید. به همین دلیل در مثال ۲-۲ روی هر دو columnِ `tail_vertex` و `head_vertex` index ایجاد شده است.
3. با استفاده از labelهای متفاوت برای typeهای مختلف relationship، می‌توانید چند نوع اطلاعات را در یک graph واحد ذخیره کنید و در عین حال data model تمیزی داشته باشید.

این ویژگی‌ها انعطاف‌پذیری زیادی برای data modeling در اختیار graphها می‌گذارند؛ همان‌طور که در شکل ۲-۵ دیده می‌شود. شکل، مواردی را نشان می‌دهد که بیان آن‌ها در یک relational schema سنتی دشوار است: typeهای متفاوت ساختارهای منطقه‌ای در کشورهای مختلف (فرانسه département و région دارد، در حالی که ایالات متحده county و state دارد)، پیچیدگی‌های تاریخی مانند وجود یک country درون country دیگر (فعلاً از جزئیات مربوط به sovereign stateها و nationها صرف‌نظر می‌کنیم) و granularity متفاوت داده. برای مثال، محل اقامت فعلی Lucy در سطح city مشخص شده، اما محل تولد او فقط در سطح state ثبت شده است.

می‌توانید graph را گسترش دهید تا factهای بیشتری دربارهٔ Lucy و Alain یا افراد دیگر در آن قرار دهید. برای نمونه، می‌توانید allergyهای غذایی آن‌ها را ثبت کنید: برای هر allergen یک vertex بسازید و با یک edge میان person و allergen نشان دهید که آن فرد به آن allergen حساسیت دارد. سپس می‌توانید allergenها را به مجموعه‌ای از vertexها متصل کنید که نشان می‌دهند کدام غذاها شامل کدام ماده هستند. در این صورت می‌توانید queryای بنویسید که برای هر فرد مشخص کند خوردن چه غذاهایی safe است.

Graphها برای `evolvability` مناسب‌اند: با اضافه شدن featureهای جدید به application، graph می‌تواند به‌سادگی گسترش یابد تا تغییرات data structureهای application را پوشش دهد.

### The Cypher Query Language

`Cypher` یک declarative query language برای property graphهاست که برای graph databaseِ Neo4j ساخته شده است [37]. نام آن از شخصیتی در فیلم The Matrix گرفته شده و ارتباطی با cipherهای رمزنگاری ندارد [38].

مثال ۲-۳ query مربوط به Cypher را برای وارد کردن بخش سمت چپ شکل ۲-۵ در یک graph database نشان می‌دهد. بقیهٔ graph را می‌توان به همین شکل اضافه کرد، اما برای خوانایی حذف شده است. به هر vertex یک نام نمادین مانند USA یا Idaho داده می‌شود و بخش‌های دیگر query می‌توانند با استفاده از این نام‌ها edge میان vertexها را بسازند. برای مثال، `(Idaho) -[:WITHIN]-> (USA)` edgeای با labelِ `WITHIN` ایجاد می‌کند که Idaho در آن tail node و USA head node است.

**مثال ۲-۳. بخشی از دادهٔ شکل ۲-۵، به‌صورت Cypher query**

```cypher
CREATE
  (NAmerica:Location {name:'North America', type:'continent'}),
  (USA:Location      {name:'United States', type:'country' }),
  (Idaho:Location    {name:'Idaho',         type:'state'    }),
  (Lucy:Person       {name:'Lucy' }),
  (Idaho) -[:WITHIN]-> (USA) -[:WITHIN]-> (NAmerica),
  (Lucy) -[:BORN_IN]-> (Idaho)
```

وقتی همهٔ vertexها و edgeهای شکل ۲-۵ به database اضافه شدند، می‌توانیم سؤال‌های جالبی بپرسیم. برای مثال، نام تمام افرادی را پیدا کنیم که از ایالات متحده به اروپا مهاجرت کرده‌اند. دقیق‌تر بگوییم، می‌خواهیم همهٔ vertexهایی را پیدا کنیم که هم edgeای از نوع `BORN_IN` به یک location درون ایالات متحده دارند و هم edgeای از نوع `LIVING_IN` به یک location درون اروپا؛ سپس propertyِ `name` مربوط به هر یک از آن vertexها را برگردانیم.

مثال ۲-۴ نشان می‌دهد این query در Cypher چگونه بیان می‌شود. در clauseِ `MATCH` نیز از همان arrow notation برای پیدا کردن patternها در graph استفاده می‌شود: `(person) -[:BORN_IN]-> ()` هر دو vertexای را match می‌کند که با edgeای با labelِ `BORN_IN` به هم مرتبط‌اند. tail vertex آن edge به variableای به نام `person` bind می‌شود و head vertex بدون نام باقی می‌ماند.

**مثال ۲-۴. Cypher query برای پیدا کردن افرادی که از ایالات متحده به اروپا مهاجرت کرده‌اند**

```cypher
MATCH
  (person) -[:BORN_IN]-> () -[:WITHIN*0..]-> (us:Location {name:'United States'}),
  (person) -[:LIVES_IN]-> () -[:WITHIN*0..]-> (eu:Location {name:'Europe'})
RETURN person.name
```

می‌توان query را این‌گونه خواند:

هر vertexای را (با نام `person`) پیدا کن که هر دو شرط زیر را داشته باشد:

1. `person` یک outgoing edge از نوع `BORN_IN` به vertexای دارد. از آن vertex می‌توان زنجیره‌ای از outgoing edgeهای `WITHIN` را دنبال کرد تا در نهایت به vertexای از typeِ `Location` برسیم که propertyِ `name` آن برابر با `"United States"` است.
2. همان vertex مربوط به person یک outgoing edge از نوع `LIVES_IN` نیز دارد. با دنبال کردن آن edge و سپس زنجیره‌ای از outgoing edgeهای `WITHIN` در نهایت به vertexای از typeِ `Location` می‌رسیم که propertyِ `name` آن برابر با `"Europe"` است.

برای هر vertex مربوط به person که این شرایط را دارد، propertyِ `name` را برگردان.

راه‌های مختلفی برای اجرای این query وجود دارد. توضیح بالا پیشنهاد می‌کند کار را با scan کردن تمام افراد database شروع کنید، محل تولد و محل اقامت هر فرد را بررسی کنید و فقط افرادی را برگردانید که شرایط را دارند. اما به‌طور معادل می‌توانید از دو vertex مربوط به Location شروع کنید و به سمت عقب حرکت کنید.

اگر روی propertyِ `name` index وجود داشته باشد، احتمالاً می‌توانید دو vertex نمایندهٔ ایالات متحده و اروپا را به‌صورت کارآمد پیدا کنید. سپس با دنبال کردن تمام incoming edgeهای `WITHIN`، همهٔ locationهای مربوط به ایالات متحده و اروپا را پیدا می‌کنید؛ locationهایی مانند state، region و city. در نهایت می‌توانید افرادی را پیدا کنید که با incoming edgeای از نوع `BORN_IN` یا `LIVES_IN` به یکی از این vertexهای location متصل هستند.

مانند هر declarative query language معمول، هنگام نوشتن query لازم نیست این جزئیات execution را مشخص کنید. query optimizer به‌طور خودکار strategyای را انتخاب می‌کند که پیش‌بینی می‌شود کارآمدترین باشد و شما می‌توانید روی نوشتن بقیهٔ application تمرکز کنید.

### Graph Queries in SQL

مثال ۲-۲ نشان داد که graph data را می‌توان در یک relational database نمایش داد. اما اگر graph data را در یک ساختار relational قرار دهیم، آیا می‌توانیم آن را با SQL نیز query کنیم؟

پاسخ مثبت است، اما با دشواری‌هایی همراه است. در یک relational database معمولاً از قبل می‌دانید در query خود به کدام joinها نیاز دارید. در یک graph query ممکن است پیش از پیدا کردن vertex موردنظر مجبور باشید تعداد متغیری edge را traverse کنید؛ یعنی تعداد joinها از قبل ثابت نیست.

در مثال ما، این موضوع در ruleِ `() -[:WITHIN*0..]-> ()` در Cypher query دیده می‌شود. edgeِ `LIVES_IN` یک person ممکن است به هر نوع locationای اشاره کند: street، city، district، region، state و غیره. یک city ممکن است درون یک region باشد، region درون یک state و state درون یک country. بنابراین edgeِ `LIVES_IN` ممکن است مستقیماً به location vertex موردنظر اشاره کند یا آن vertex چند level دورتر در hierarchy قرار گرفته باشد.

در Cypher، عبارت `:WITHIN*0..` این موضوع را بسیار concise بیان می‌کند: یعنی «یک edge از نوع WITHIN را صفر بار یا بیشتر دنبال کن». این عبارت شبیه operatorِ `*` در regular expression است.

از SQL:1999 به بعد، چنین pathهای traversal با طول متغیر را می‌توان با چیزی به نام `recursive common table expression` و syntaxِ `WITH RECURSIVE` در query بیان کرد. مثال ۲-۵ همان query قبلی، یعنی پیدا کردن نام افرادی که از ایالات متحده به اروپا مهاجرت کرده‌اند، را با این تکنیک در SQL نشان می‌دهد. این syntax در PostgreSQL، IBM DB2، Oracle و SQL Server پشتیبانی می‌شود، اما در مقایسه با Cypher بسیار دست‌وپاگیر است.

**مثال ۲-۵. همان query مثال ۲-۴، این بار در SQL و با استفاده از recursive common table expression**

```sql
WITH RECURSIVE

     -- in_usa is the set of vertex IDs of all locations within the United States
     in_usa(vertex_id) AS (
          SELECT vertex_id FROM vertices WHERE properties->>'name' = 'United States'
        UNION
          SELECT edges.tail_vertex FROM edges
            JOIN in_usa ON edges.head_vertex = in_usa.vertex_id
            WHERE edges.label = 'within'
     ),

     -- in_europe is the set of vertex IDs of all locations within Europe
     in_europe(vertex_id) AS (
          SELECT vertex_id FROM vertices WHERE properties->>'name' = 'Europe'
        UNION
          SELECT edges.tail_vertex FROM edges
            JOIN in_europe ON edges.head_vertex = in_europe.vertex_id
            WHERE edges.label = 'within'
     ),

     -- born_in_usa is the set of vertex IDs of all people born in the US
     born_in_usa(vertex_id) AS (
        SELECT edges.tail_vertex FROM edges
          JOIN in_usa ON edges.head_vertex = in_usa.vertex_id
          WHERE edges.label = 'born_in'
     ),

  -- lives_in_europe is the set of vertex IDs of all people living in Europe
  lives_in_europe(vertex_id) AS (
    SELECT edges.tail_vertex FROM edges
      JOIN in_europe ON edges.head_vertex = in_europe.vertex_id
      WHERE edges.label = 'lives_in'
  )

SELECT vertices.properties->>'name'
FROM vertices
-- join to find those people who were both born in the US *and* live in Europe
JOIN born_in_usa     ON vertices.vertex_id = born_in_usa.vertex_id
JOIN lives_in_europe ON vertices.vertex_id = lives_in_europe.vertex_id;
```

مراحل اجرای query به این شکل است:

1. ابتدا vertexای را پیدا کن که propertyِ `name` آن مقدار `"United States"` دارد و آن را اولین عضو مجموعهٔ vertexهای `in_usa` قرار بده.
2. تمام incoming edgeهای `within` را از vertexهای موجود در مجموعهٔ `in_usa` دنبال کن و آن‌ها را به همان مجموعه اضافه کن؛ این کار را تا زمانی ادامه بده که تمام incoming edgeهای `within` بررسی شده باشند.
3. همین کار را از vertexای شروع کن که propertyِ `name` آن مقدار `"Europe"` دارد و مجموعهٔ `in_europe` را بساز.
4. برای هر vertex موجود در مجموعهٔ `in_usa`، incoming edgeهای `born_in` را دنبال کن تا افرادی را پیدا کنی که در مکانی داخل ایالات متحده متولد شده‌اند.
5. به‌طور مشابه، برای هر vertex موجود در مجموعهٔ `in_europe`، incoming edgeهای `lives_in` را دنبال کن تا افرادی را پیدا کنی که در اروپا زندگی می‌کنند.
6. در نهایت مجموعهٔ افرادی را که در ایالات متحده متولد شده‌اند با مجموعهٔ افرادی که در اروپا زندگی می‌کنند intersect کن؛ این کار با join کردن آن‌ها انجام می‌شود.

اگر همان query در یک query language به ۴ خط و در query language دیگری به ۲۹ خط نیاز داشته باشد، این فقط نشان می‌دهد که data modelهای مختلف برای use caseهای متفاوت طراحی شده‌اند. انتخاب data model مناسب برای application اهمیت زیادی دارد.

### Triple-Stores and SPARQL

`triple-store model` تقریباً معادل property graph model است و فقط برای بیان ایده‌های یکسان از واژه‌های متفاوتی استفاده می‌کند. بااین‌حال، بررسی آن ارزشمند است، چون ابزارها و languageهای متنوعی برای triple-storeها وجود دارد که می‌توانند به toolbox شما برای ساخت application اضافه شوند.

در یک triple-store، تمام اطلاعات به‌شکل statementهای بسیار سادهٔ سه‌بخشی ذخیره می‌شوند: `(subject, predicate, object)`. برای مثال، در tripleِ `(Jim, likes, bananas)`، Jim subject، likes predicate (فعل) و bananas object است.

subject یک triple معادل vertex در graph است. object یکی از دو حالت زیر را دارد:

1. یک value از primitive datatype، مانند string یا number. در این حالت، predicate و object triple به‌ترتیب معادل key و value یک property روی subject vertex هستند. برای مثال، `(lucy, age, 33)` مانند vertexای به نام lucy با propertyهای `{"age":33}` است.
2. یک vertex دیگر در graph. در این حالت، predicate یک edge در graph است، subject tail vertex و object head vertex محسوب می‌شود. برای مثال، در `(lucy, marriedTo, alain)`، subject و object یعنی lucy و alain هر دو vertex هستند و predicateِ marriedTo، labelِ edgeای است که آن‌ها را به هم متصل می‌کند.

مثال ۲-۶ همان دادهٔ مثال ۲-۳ را به‌صورت triple و در formتی به نام `Turtle` نشان می‌دهد. Turtle زیرمجموعه‌ای از Notation3 یا N3 است [39].

**مثال ۲-۶. بخشی از دادهٔ شکل ۲-۵، به‌صورت Turtle triple**

```turtle
@prefix : <urn:example:>.
_:lucy     a       :Person.
_:lucy     :name   "Lucy".
_:lucy     :bornIn _:idaho.
_:idaho    a       :Location.
_:idaho    :name   "Idaho".
_:idaho    :type   "state".
_:idaho    :within _:usa.
_:usa      a       :Location.
_:usa      :name   "United States".
_:usa      :type   "country".
_:usa      :within _:namerica.
_:namerica a       :Location.
_:namerica :name   "North America".
_:namerica :type   "continent".
```

در این مثال، vertexهای graph با فرم `_:someName` نوشته شده‌اند. این name خارج از همین file معنایی ندارد و فقط به این دلیل وجود دارد که در غیر این صورت نمی‌دانستیم کدام tripleها به یک vertex واحد اشاره می‌کنند. وقتی predicate نمایندهٔ یک edge باشد، object یک vertex است؛ مانند `_:idaho :within _:usa`. وقتی predicate یک property باشد، object یک string literal است؛ مانند `_:usa :name "United States"`.

تکرار کردن یک subject در چند خط کمی تکراری است، اما خوشبختانه می‌توانید با استفاده از semicolon چند statement دربارهٔ یک subject بنویسید. این کار Turtle را خوانا و مناسب می‌کند؛ مثال ۲-۷ را ببینید.

**مثال ۲-۷. شکل کوتاه‌تر نوشتن دادهٔ مثال ۲-۶**

```turtle
@prefix : <urn:example:>.
_:lucy     a :Person;   :name "Lucy";          :bornIn _:idaho.
_:idaho    a :Location; :name "Idaho";         :type "state";   :within _:usa.
_:usa      a :Location; :name "United States"; :type "country"; :within _:namerica.
_:namerica a :Location; :name "North America"; :type "continent".
```

#### The semantic web

اگر دربارهٔ triple-storeها بیشتر بخوانید، ممکن است در گردابی از مقاله‌هایی دربارهٔ `semantic web` گرفتار شوید. triple-store data model کاملاً مستقل از semantic web است؛ برای مثال، Datomic [40] یک triple-store است که ادعا نمی‌کند ارتباطی با semantic web دارد. اما چون این دو در ذهن بسیاری از افراد ارتباط نزدیکی دارند، باید به‌اختصار دربارهٔ آن‌ها صحبت کنیم.

ایدهٔ اصلی semantic web ساده و منطقی است: وب‌سایت‌ها همین حالا اطلاعات را به‌صورت text و picture برای خواندن انسان‌ها منتشر می‌کنند؛ پس چرا همین اطلاعات را به‌صورت machine-readable data برای خواندن computerها منتشر نکنند؟ `Resource Description Framework (RDF)` [41] قرار بود mechanismی باشد که web siteهای مختلف با استفاده از آن data را در formتی یکسان منتشر کنند تا داده‌های سایت‌های مختلف به‌صورت خودکار با هم ترکیب شوند و یک web of data، یعنی نوعی «database سراسری از همه‌چیز»، بسازند.

متأسفانه semantic web در اوایل دههٔ ۲۰۰۰ بیش از حد hype شد، اما تاکنون نشانه‌ای از تحقق عملی آن دیده نشده است و همین موضوع بسیاری از افراد را نسبت به آن بدبین کرده است. این پروژه همچنین از انبوه گیج‌کننده‌ای از acronymها، پیشنهادهای بیش از حد پیچیده برای standardها و نوعی hubris رنج برده است.

بااین‌حال، اگر از این ضعف‌ها عبور کنیم، کارهای ارزشمند زیادی از پروژهٔ semantic web به‌وجود آمده است. حتی اگر علاقه‌ای به انتشار RDF data روی semantic web نداشته باشید، tripleها می‌توانند data model داخلی مناسبی برای applicationها باشند.

#### The RDF data model

زبان Turtle که در مثال ۲-۷ استفاده کردیم، formتی human-readable برای RDF data است. گاهی RDF در formت XML نیز نوشته می‌شود که همان کار را به‌شکل بسیار verboseتری انجام می‌دهد؛ مثال ۲-۸ را ببینید. Turtle/N3 ترجیح داده می‌شود، چون خواندن آن برای انسان بسیار آسان‌تر است. ابزارهایی مانند Apache Jena [42] نیز در صورت نیاز می‌توانند formت‌های مختلف RDF را به‌صورت خودکار به یکدیگر تبدیل کنند.

**مثال ۲-۸. دادهٔ مثال ۲-۷، با استفاده از RDF/XML syntax**

```xml
<rdf:RDF xmlns="urn:example:"
    xmlns:rdf="http://www.w3.org/1999/02/22-rdf-syntax-ns#">

     <Location rdf:nodeID="idaho">
       <name>Idaho</name>
       <type>state</type>
       <within>
         <Location rdf:nodeID="usa">
           <name>United States</name>
           <type>country</type>
           <within>
             <Location rdf:nodeID="namerica">
                <name>North America</name>
                <type>continent</type>
             </Location>
           </within>
         </Location>
       </within>
     </Location>

  <Person rdf:nodeID="lucy">
    <name>Lucy</name>
    <bornIn rdf:nodeID="idaho"/>
  </Person>
</rdf:RDF>
```

RDF به‌دلیل اینکه برای data exchange در مقیاس کل internet طراحی شده، چند ویژگی خاص دارد. subject، predicate و object یک triple اغلب `URI` هستند. برای مثال، predicate ممکن است URIای مانند `<http://my-company.com/namespace#within>` یا `<http://my-company.com/namespace#lives_in>` باشد، نه فقط WITHIN یا LIVES_IN.

منطق این طراحی آن است که بتوانید دادهٔ خود را با دادهٔ شخص دیگری ترکیب کنید. اگر آن شخص معنای متفاوتی برای واژهٔ within یا lives_in در نظر گرفته باشد، conflictی رخ نمی‌دهد، چون predicateهای او در واقع `<http://other.org/foo#within>` و `<http://other.org/foo#lives_in>` هستند.

لازم نیست URLِ `<http://my-company.com/namespace>` حتماً به چیزی resolve شود؛ از دید RDF، این URL فقط یک namespace است. برای جلوگیری از ابهام با URLهای `http://`، مثال‌های این بخش از URIهای non-resolvable مانند `urn:example:within` استفاده می‌کنند. خوشبختانه کافی است این prefix را یک بار در ابتدای file مشخص کنید و بعد آن را فراموش کنید.

#### The SPARQL query language

`SPARQL` یک query language برای triple-storeهایی است که از RDF data model استفاده می‌کنند [43]. این نام acronym عبارت SPARQL Protocol and RDF Query Language است و «sparkle» تلفظ می‌شود. SPARQL پیش از Cypher به‌وجود آمده است و چون pattern matching در Cypher از SPARQL گرفته شده، این دو بسیار شبیه به نظر می‌رسند [37].

همان query قبلی، یعنی پیدا کردن افرادی که از ایالات متحده به اروپا مهاجرت کرده‌اند، در SPARQL حتی conciseتر از Cypher است؛ مثال ۲-۹ را ببینید.

**مثال ۲-۹. همان query مثال ۲-۴، این بار در SPARQL**

```sparql
PREFIX : <urn:example:>

SELECT ?personName WHERE {
  ?person :name ?personName.
  ?person :bornIn / :within* / :name "United States".
  ?person :livesIn / :within* / :name "Europe".
}
```

ساختار بسیار مشابه است. دو expression زیر معادل یکدیگرند؛ variableها در SPARQL با علامت سؤال شروع می‌شوند:

```text
(person) -[:BORN_IN]-> () -[:WITHIN*0..]-> (location)   # Cypher

?person :bornIn / :within* ?location.                   # SPARQL
```

چون RDF میان property و edge تفاوتی قائل نمی‌شود و برای هر دو از predicate استفاده می‌کند، می‌توانید برای match کردن propertyها نیز از همین syntax استفاده کنید. در expressionهای زیر، variableِ `usa` به هر vertexای bind می‌شود که propertyای به نام name دارد و value آن stringِ `"United States"` است:

```text
(usa {name:'United States'})   # Cypher

?usa :name "United States".    # SPARQL
```

SPARQL query language خوبی است. حتی اگر semantic web هرگز تحقق پیدا نکند، SPARQL می‌تواند ابزار قدرتمندی برای استفادهٔ داخلی applicationها باشد.

#### Graph Databases Compared to the Network Model

در بخش «Are Document Databases Repeating History?» در صفحهٔ ۳۶، دربارهٔ رقابت CODASYL و relational model برای حل مسئلهٔ many-to-many relationship در IMS صحبت کردیم. در نگاه اول، network model مربوط به CODASYL شبیه graph model به نظر می‌رسد. آیا graph databaseها همان CODASYL هستند که در ظاهری جدید بازگشته‌اند؟

خیر. این دو در چند جنبهٔ مهم با هم تفاوت دارند:

- در CODASYL، database schemaای داشت که مشخص می‌کرد کدام نوع record می‌تواند درون کدام نوع record دیگر nested شود. در graph database چنین محدودیتی وجود ندارد و هر vertex می‌تواند به هر vertex دیگری edge داشته باشد. این ویژگی انعطاف‌پذیری بسیار بیشتری برای سازگار شدن application با requirementهای در حال تغییر فراهم می‌کند.
- در CODASYL، تنها راه رسیدن به یک record مشخص، traverse کردن یکی از access pathهای آن بود. در graph database می‌توانید مستقیماً با unique ID به هر vertex reference بدهید یا با استفاده از index، vertexهایی را پیدا کنید که value مشخصی دارند.
- در CODASYL، childهای یک record مجموعه‌ای ordered بودند. بنابراین database باید این order را حفظ می‌کرد و layout مربوط به storage تحت تأثیر قرار می‌گرفت. همچنین applicationهایی که record جدیدی در database insert می‌کردند باید نگران position آن record در این مجموعه‌ها می‌بودند. در graph database، vertexها و edgeها ordered نیستند و فقط هنگام query می‌توانید resultها را sort کنید.
- در CODASYL، همهٔ queryها imperative بودند، نوشتن آن‌ها دشوار بود و تغییر schema به‌سادگی آن‌ها را می‌شکست. در graph database، اگر بخواهید می‌توانید traversal را با imperative code بنویسید، اما بیشتر graph databaseها از high-level declarative query languageهایی مانند Cypher یا SPARQL نیز پشتیبانی می‌کنند.

### The Foundation: Datalog

`Datalog` language بسیار قدیمی‌تری از SPARQL یا Cypher است و دانشگاهیان در دههٔ ۱۹۸۰ آن را به‌طور گسترده مطالعه کرده‌اند [44, 45, 46]. Datalog در میان software engineerها کمتر شناخته شده است، اما مهم است، چون foundationای را فراهم می‌کند که query languageهای بعدی بر آن بنا شده‌اند.

در عمل، Datalog در چند data system استفاده می‌شود. برای مثال، query languageِ Datomic [40] است و `Cascalog` [47] یک implementation از Datalog برای query کردن datasetهای بزرگ در Hadoop است.

data model در Datalog شبیه triple-store model است، با این تفاوت که کمی generalize شده است. به‌جای نوشتن triple به‌صورت `(subject, predicate, object)`، آن را به شکل `predicate(subject, object)` می‌نویسیم. مثال ۲-۱۰ نشان می‌دهد دادهٔ مثال خود را در Datalog چگونه بنویسیم.

**مثال ۲-۱۰. بخشی از دادهٔ شکل ۲-۵، به‌صورت Datalog fact**

```prolog
name(namerica, 'North America').
type(namerica, continent).

name(usa, 'United States').
type(usa, country).
within(usa, namerica).

name(idaho, 'Idaho').
type(idaho, state).
within(idaho, usa).

name(lucy, 'Lucy').
born_in(lucy, idaho).
```

حالا که data را تعریف کرده‌ایم، می‌توانیم همان query قبلی را بنویسیم؛ همان‌طور که در مثال ۲-۱۱ نشان داده شده است. این query با معادل خود در Cypher یا SPARQL کمی متفاوت به نظر می‌رسد، اما این تفاوت نباید مانع شود. Datalog زیرمجموعه‌ای از Prolog است که اگر computer science خوانده باشید ممکن است قبلاً آن را دیده باشید.

**مثال ۲-۱۱. همان query مثال ۲-۴، این بار در Datalog**

```prolog
within_recursive(Location, Name) :- name(Location, Name).          /* Rule 1 */

within_recursive(Location, Name) :- within(Location, Via),    /* Rule 2 */
                                    within_recursive(Via, Name).

migrated(Name, BornIn, LivingIn) :- name(Person, Name),       /* Rule 3 */
                                    born_in(Person, BornLoc),
                                    within_recursive(BornLoc, BornIn),
                                    lives_in(Person, LivingLoc),
                                    within_recursive(LivingLoc, LivingIn).

?- migrated(Who, 'United States', 'Europe').
/* Who = 'Lucy'. */
```

Cypher و SPARQL بلافاصله با `SELECT` شروع می‌کنند، اما Datalog قدم‌به‌قدم پیش می‌رود. در Datalog ruleهایی تعریف می‌کنیم که database را دربارهٔ predicateهای جدید آگاه می‌کنند. در اینجا دو predicate جدید به نام‌های `within_recursive` و `migrated` تعریف کرده‌ایم. این predicateها tripleهایی نیستند که در database ذخیره شده باشند، بلکه از داده یا ruleهای دیگر derive می‌شوند.

Ruleها می‌توانند به ruleهای دیگر reference دهند؛ درست مانند functionهایی که functionهای دیگر را call می‌کنند یا به‌صورت recursive خودشان را call می‌کنند. به این شکل می‌توان queryهای پیچیده را قدم‌به‌قدم و با ترکیب بخش‌های کوچک ساخت.

در ruleها، کلماتی که با حرف بزرگ شروع می‌شوند variable هستند و predicateها مانند Cypher و SPARQL match می‌شوند. برای مثال، `name(Location, Name)` با tripleِ `name(namerica, 'North America')` match می‌شود و bindingهای `Location = namerica` و `Name = 'North America'` را برای variableها ایجاد می‌کند.

یک rule زمانی apply می‌شود که system بتواند برای تمام predicateهای سمت راست operatorِ `:-` یک match پیدا کند. در این صورت گویی سمت چپ `:-` با جایگزین شدن variableها با valueهای match‌شده به database اضافه شده است.

یکی از روش‌های ممکن برای apply کردن ruleها چنین است:

1. `name(namerica, 'North America')` در database وجود دارد، پس rule 1 apply می‌شود و `within_recursive(namerica, 'North America')` را تولید می‌کند.
2. `within(usa, namerica)` در database وجود دارد و مرحلهٔ قبل `within_recursive(namerica, 'North America')` را تولید کرده است، پس rule 2 apply می‌شود و `within_recursive(usa, 'North America')` را تولید می‌کند.
3. `within(idaho, usa)` در database وجود دارد و مرحلهٔ قبل `within_recursive(usa, 'North America')` را تولید کرده است، پس rule 2 apply می‌شود و `within_recursive(idaho, 'North America')` را تولید می‌کند.

با apply کردن مکرر ruleهای 1 و 2، predicateِ `within_recursive` می‌تواند تمام locationهای موجود در North America (یا هر location name دیگری) را در database به ما بگوید. این فرایند در شکل ۲-۶ نشان داده شده است.

*شکل ۲-۶. تعیین اینکه Idaho در North America قرار دارد، با استفاده از Datalog ruleهای مثال ۲-۱۱.*

حالا rule 3 می‌تواند افرادی را پیدا کند که در locationای به نام `BornIn` متولد شده‌اند و در locationای به نام `LivingIn` زندگی می‌کنند. با query کردن و قرار دادن `BornIn = 'United States'` و `LivingIn = 'Europe'` و رها کردن person به‌صورت variableای به نام `Who`، از Datalog system می‌خواهیم پیدا کند چه valueهایی می‌توانند برای variableِ `Who` ظاهر شوند. در نهایت همان پاسخی را می‌گیریم که در queryهای قبلی Cypher و SPARQL گرفتیم.

رویکرد Datalog به نوع متفاوتی از تفکر نسبت به query languageهای دیگر این chapter نیاز دارد، اما بسیار قدرتمند است؛ چون ruleها را می‌توان در queryهای مختلف با هم ترکیب و دوباره استفاده کرد. Datalog برای queryهای ساده و یک‌باره راحت‌ترین گزینه نیست، اما وقتی دادهٔ شما پیچیده باشد، بهتر می‌تواند از عهدهٔ آن برآید.

*پاورقی: Datomic و Cascalog برای Datalog از syntax مربوط به Clojure S-expression استفاده می‌کنند. در مثال‌های این بخش از syntax مربوط به Prolog استفاده کرده‌ایم که خواندن آن کمی آسان‌تر است، اما از نظر functionality تفاوتی ایجاد نمی‌کند.*

## Key Terms

- `Graph Database` — databaseای که داده را به‌صورت vertex و edge ذخیره و query می‌کند؛ برای داده‌های به‌شدت interconnected و traversalهای چندمرحله‌ای مناسب است.
- `Graph Model` — مدل نمایش داده به‌صورت node و relationship؛ ارتباط میان entityها را به‌عنوان بخش اصلی data model در نظر می‌گیرد.
- `Property Graph` — graphی که vertex و edge در آن identifier، label و property دارند؛ انعطاف‌پذیری زیادی برای مدل کردن entityها و relationshipهای متنوع فراهم می‌کند.
- `Vertex` — node یا entity در graph؛ نقطه‌ای که object، person، location یا هر entity دیگر را نمایش می‌دهد.
- `Edge` — connection جهت‌دار یا رابطه میان دو vertex؛ نوع و جهت relationship میان entityها را نشان می‌دهد و می‌تواند property داشته باشد.
- `Graph Query` — query برای پیدا کردن vertex، edge یا pattern در graph؛ معمولاً شامل pattern matching یا traversal در مسیرهای چندمرحله‌ای است.
- `Graph Traversal` — پیمایش graph با دنبال کردن edgeها از یک vertex به vertexهای دیگر؛ برای پیدا کردن connectionهای مستقیم یا زنجیره‌ای و حل queryهای relationshipمحور استفاده می‌شود.
- `Cypher` — declarative query language مربوط به Neo4j و property graphها؛ patternهای graph را با syntax خوانا برای match، create و return بیان می‌کند.
- `Triple-Store` — datastoreای که اطلاعات را به‌صورت subject، predicate و object ذخیره می‌کند؛ data model ساده‌ای برای graph data و RDF فراهم می‌کند.
- `Subject` — بخش اول یک RDF triple؛ vertex یا entityای را مشخص می‌کند که statement دربارهٔ آن است.
- `Predicate` — بخش دوم یک RDF triple؛ نوع property یا relationship میان subject و object را مشخص می‌کند.
- `Object` — بخش سوم یک RDF triple؛ value یک property یا vertex مقصد یک relationship است.
- `SPARQL` — declarative query language برای triple-storeهای مبتنی بر RDF؛ patternهای RDF را برای جست‌وجو و ترکیب graph data بیان می‌کند.
- `RDF` — data model استاندارد برای بیان resourceها و relationshipهای آن‌ها؛ امکان تبادل machine-readable data میان systemها و namespaceهای مستقل را فراهم می‌کند.
- `Turtle` — syntax خوانا برای نوشتن RDF tripleها؛ نمایش compactتری از RDF است و برای خواندن و نوشتن انسانی مناسب‌تر از RDF/XML است.
- `Datalog` — query language rule-based و declarative، مبتنی بر predicate و rule؛ queryهای recursive و قابل‌ترکیب را با derive کردن factهای جدید از داده و ruleها بیان می‌کند.
- `Pattern Matching` — پیدا کردن بخش‌هایی از graph یا data که با یک pattern مشخص سازگارند؛ پایهٔ queryهایی مانند MATCH در Cypher و patternهای SPARQL است.
- `Recursive Query` — queryای که برای رسیدن به result به خودش یا نتیجهٔ مرحلهٔ قبل reference می‌دهد؛ برای traversal با عمق نامشخص و common table expressionهای recursive استفاده می‌شود.
- `Graph Processing` — پردازش الگوریتمی روی مجموعه‌ای از vertexها و edgeها؛ برای کارهایی مانند shortest path، ranking و تحلیل connectionها به کار می‌رود.
- `Query Language` — زبان بیان query برای خواندن یا پردازش داده؛ interfaceی است که خواستهٔ application را با syntax مشخص به datastore منتقل می‌کند.

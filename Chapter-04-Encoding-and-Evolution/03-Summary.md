# Chapter 4 — Encoding and Evolution

## Summary

در این chapter چند روش برای تبدیل data structureها به byteهایی روی network یا byteهایی روی disk بررسی کردیم. دیدیم جزئیات این encodingها نه‌فقط بر efficiency آن‌ها، بلکه مهم‌تر از آن، بر architecture applicationها و گزینه‌های شما برای deploy کردن آن‌ها اثر می‌گذارد.

به‌طور خاص، بسیاری از serviceها باید از rolling upgrade پشتیبانی کنند؛ یعنی version جدید service به‌تدریج روی چند node در هر مرحله deploy شود، نه اینکه هم‌زمان روی تمام nodeها deploy گردد. Rolling upgrade اجازه می‌دهد versionهای جدید service بدون downtime release شوند (و در نتیجه releaseهای کوچک و مکرر به releaseهای بزرگ و نادر ترجیح داده شوند) و deploymentها را کم‌ریسک‌تر می‌کند؛ چون releaseهای faulty را می‌توان پیش از اثر گذاشتن بر تعداد زیادی user شناسایی و rollback کرد. این ویژگی‌ها برای evolvability، یعنی آسان بودن ایجاد change در application، بسیار مفید هستند.

در طول rolling upgrade، یا به دلایل مختلف دیگر، باید فرض کنیم nodeهای مختلف versionهای متفاوتی از code application ما را اجرا می‌کنند. بنابراین مهم است که تمام dataای که در system flow می‌کند به روشی encode شود که backward compatibility (code جدید بتواند data قدیمی را بخواند) و forward compatibility (code قدیمی بتواند data جدید را بخواند) فراهم کند.

چند data encoding format و propertyهای compatibility آن‌ها را بررسی کردیم:

- Encodingهای مخصوص یک programming language به یک language واحد محدود هستند و اغلب نمی‌توانند forward و backward compatibility را فراهم کنند.
- Formatهای متنی مانند JSON، XML و CSV گسترده استفاده می‌شوند و compatibility آن‌ها به نحوهٔ استفاده از آن‌ها بستگی دارد. این formatها schema languageهای optional دارند که گاهی مفید و گاهی مانع هستند. این formatها دربارهٔ datatypeها تا حدی مبهم‌اند؛ بنابراین باید هنگام کار با چیزهایی مانند numberها و binary stringها دقت کنید.
- Formatهای binary مبتنی بر schema مانند Thrift، Protocol Buffers و Avro encodingای compact و efficient با semantics کاملاً تعریف‌شده برای forward و backward compatibility فراهم می‌کنند. Schemaها می‌توانند برای documentation و code generation در statically typed languageها مفید باشند. بااین‌حال، نقطه‌ضعف آن‌ها این است که data پیش از human-readable شدن باید decode شود.

همچنین چند mode از dataflow را بررسی کردیم که scenarioهای متفاوتی را نشان می‌دهند؛ scenarioهایی که در آن‌ها data encoding اهمیت دارد:

- Databaseها؛ processی که در database write می‌کند data را encode می‌کند و processی که آن را read می‌کند data را decode می‌کند.
- RPC و REST APIها؛ client یک request را encode می‌کند، server request را decode و response را encode می‌کند و در نهایت client response را decode می‌کند.
- Asynchronous message passing (با استفاده از message brokerها یا actorها)؛ nodeها با ارسال message به یکدیگر ارتباط برقرار می‌کنند و sender message را encode و recipient آن را decode می‌کند.

می‌توان نتیجه گرفت که با کمی دقت، backward و forward compatibility و rolling upgrade کاملاً دست‌یافتنی هستند. امیدواریم evolution application شما سریع و deploymentهای شما مکرر باشند.

## Key Terms

- `Data Encoding` — تبدیل data به byte sequence برای انتقال یا storage میان componentهای مستقل.
- `Schema Evolution` — تغییر کنترل‌شدهٔ schema در طول زمان، همراه با حفظ compatibility میان code و data versionهای مختلف.
- `Backward Compatibility` — توانایی code یا schema جدید برای خواندن data قدیمی.
- `Forward Compatibility` — توانایی code یا schema قدیمی برای خواندن data جدید.
- `Dataflow` — مسیر انتقال data میان processها، serviceها، databaseها یا nodeها.
- `REST` — design philosophy مبتنی بر اصول HTTP برای ساخت APIهای resource-oriented.
- `RPC (Remote Procedure Call)` — abstractionای برای فراخوانی یک service remote شبیه function یا method محلی.
- `Message-Passing` — ارتباط میان componentها با ارسال و دریافت message، معمولاً بدون memory مشترک.
- `Message Broker` — واسطه‌ای برای ذخیرهٔ موقت، routing و delivery کردن messageها.
- `Asynchronous Communication` — ارتباطی که sender بدون انتظار برای response یا delivery کامل ادامه می‌دهد.
- `Rolling Upgrade` — deploy تدریجی version جدید روی nodeها، بدون توقف کل service.

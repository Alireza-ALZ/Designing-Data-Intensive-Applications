# Chapter 1 — Reliable, Scalable, and Maintainable Applications

## Summary

در این فصل، چند شیوهٔ بنیادی برای فکر کردن دربارهٔ `data-intensive application`ها را بررسی کردیم. این اصول در ادامهٔ کتاب راهنمای ما خواهند بود؛ جایی که وارد جزئیات فنی عمیق‌تری می‌شویم.

یک application برای مفید بودن باید requirementهای مختلفی را برآورده کند. بخشی از این requirementها `functional requirement` هستند؛ یعنی application باید چه کاری انجام دهد، مانند امکان ذخیره، بازیابی، جست‌وجو و پردازش داده به روش‌های مختلف. بخشی دیگر `nonfunctional requirement` هستند؛ یعنی ویژگی‌های کلی‌ای مانند security، reliability، compliance، scalability، compatibility و maintainability. در این فصل reliability، scalability و maintainability را با جزئیات بررسی کردیم.

Reliability یعنی سیستم‌ها حتی هنگام وقوع faultها به‌درستی کار کنند. faultها ممکن است در hardware رخ دهند - که معمولاً تصادفی و غیرهم‌بسته‌اند - در software رخ دهند - که bugهای آن معمولاً systematic و دشوار برای مدیریت‌اند - یا از طرف انسان‌ها ایجاد شوند؛ انسان‌ها نیز ناگزیر گاهی اشتباه می‌کنند. تکنیک‌های `fault tolerance` می‌توانند برخی انواع fault را از دید end user پنهان کنند.

Scalability یعنی داشتن strategyهایی برای خوب نگه داشتن performance، حتی وقتی load افزایش می‌یابد. برای بحث دربارهٔ scalability ابتدا باید روش‌هایی برای توصیف کمی load و performance داشته باشیم. در این فصل، `home timeline`های Twitter را به‌طور خلاصه به‌عنوان مثالی برای توصیف load بررسی کردیم و `response time percentile`ها را به‌عنوان روشی برای اندازه‌گیری performance دیدیم. در یک scalable system، می‌توان processing capacity را افزایش داد تا سیستم در load بالا نیز reliable باقی بماند.

Maintainability جنبه‌های زیادی دارد، اما در اصل دربارهٔ بهتر کردن زندگی تیم‌های engineering و operations است که باید با سیستم کار کنند. abstractionهای خوب می‌توانند complexity را کاهش دهند و تغییر دادن و سازگار کردن سیستم با use caseهای جدید را آسان‌تر کنند. Operability خوب یعنی visibility مناسبی نسبت به health سیستم داشته باشیم و راه‌های مؤثری برای مدیریت آن در اختیارمان باشد.

متأسفانه برای reliable، scalable یا maintainable کردن applicationها راه‌حل ساده‌ای وجود ندارد. بااین‌حال، patternها و techniqueهای مشخصی در انواع مختلف applicationها بارها تکرار می‌شوند. در فصل‌های بعد چند نمونه از data systemها را بررسی می‌کنیم و تحلیل خواهیم کرد که چگونه برای دستیابی به این هدف‌ها تلاش می‌کنند.

در بخش سوم کتاب، patternهای مربوط به سیستم‌هایی را بررسی خواهیم کرد که از چند component تشکیل شده‌اند و با یکدیگر کار می‌کنند؛ مانند سیستمی که در شکل ۱-۱ نشان داده شد.

## Key Terms

- `Data-Intensive Application` — applicationای که چالش اصلی آن مقدار، complexity یا سرعت تغییر داده است.
- `Functional Requirement` — requirement مربوط به کاری که application باید انجام دهد؛ مانند ذخیره، بازیابی، جست‌وجو یا پردازش داده.
- `Nonfunctional Requirement` — ویژگی کلی و کیفی سیستم؛ مانند security، reliability، scalability، compatibility یا maintainability.
- `Compliance` — رعایت الزام‌های قانونی، regulatory یا سیاست‌های سازمانی در رفتار سیستم.
- `Compatibility` — توانایی کار کردن سیستم با platformها، componentها یا interfaceهای دیگر.
- `Processing Capacity` — مقدار ظرفیت محاسباتی موجود برای پردازش load سیستم.
- `Reliability` — توانایی ادامهٔ عملکرد صحیح سیستم، حتی هنگام وقوع fault.
- `Scalability` — توانایی حفظ performance قابل‌قبول هنگام افزایش load.
- `Maintainability` — توانایی نگهداری، فهم، تغییر و توسعهٔ سیستم در طول زمان.

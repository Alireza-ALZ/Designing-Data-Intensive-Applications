# Chapter 6 — Partitioning

## Partitioning and Replication

Partitioning معمولاً با replication ترکیب می‌شود تا copyهای هر partition روی چند node ذخیره شوند. بنابراین، با اینکه هر record دقیقاً به یک partition تعلق دارد، ممکن است همان record برای Fault Tolerance روی چند node مختلف ذخیره شده باشد.

یک node ممکن است بیش از یک partition را نگه‌داری کند. اگر از model مربوط به leader–follower replication استفاده شود، ترکیب partitioning و replication می‌تواند شبیه شکل ۶-۱ باشد. leader هر partition به یک node اختصاص داده می‌شود و followerهای آن partition روی nodeهای دیگر قرار می‌گیرند. هر node می‌تواند برای بعضی partitionها leader و برای partitionهای دیگر follower باشد.

هر چیزی که در Chapter 5 دربارهٔ replication در databaseها گفتیم، دربارهٔ replication مربوط به partitionها نیز صدق می‌کند. انتخاب partitioning scheme تا حد زیادی مستقل از انتخاب replication scheme است؛ بنابراین برای ساده نگه‌داشتن بحث، در این chapter replication را نادیده می‌گیریم.

**شکل ۶-۱.** ترکیب replication و partitioning: هر node برای بعضی partitionها نقش leader و برای partitionهای دیگر نقش follower دارد.

## Key Terms

- `Partitioning` — تقسیم عمدی یک dataset بزرگ به چند بخش مستقل برای توزیع data و query load میان nodeها.
- `Sharding` — نام دیگری برای partitioning که در برخی databaseها و data systemها رایج است.
- `Partition` — بخشی از dataset که معمولاً هر record را در خود نگه می‌دارد و مانند یک database کوچک عمل می‌کند.
- `Shared-Nothing Cluster` — clusterای که در آن nodeها storage و processing مستقل دارند و برای scale کردن data و query load به‌کار می‌روند.
- `Query Throughput` — تعداد queryهایی که system در واحد زمان می‌تواند پردازش کند.
- `Query Load` — حجم queryها و workload پردازشی واردشده به database یا partitionها.
- `Partitioned Database` — databaseای که dataset آن میان چند partition و معمولاً چند node توزیع شده است.

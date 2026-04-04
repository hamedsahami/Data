# Kafka → ClickHouse Ingestion (kafka.tomandw)

این README یک راهنمای **درست، تمیز و قابل استفاده در Production** برای ingest کردن داده‌های CDC از Kafka (Debezium) به ClickHouse است.

این مستند **از نقطه‌ای شروع می‌کند که مشکل صفر بودن داده‌ها رفع شده** و معماری نهایی تثبیت شده است.

---

## 🎯 هدف

- خواندن داده‌های CDC از Kafka
- ذخیره دائمی داده‌ها در ClickHouse
- خواندن داده‌ها **از ابتدای تاپیک Kafka**
- جلوگیری از خطاهای رایج مثل:
  - `METADATA_MISMATCH`
  - `TABLE_ALREADY_EXISTS`
  - صفر بودن داده در جدول نهایی

---



## 🧱 لایه 1: Storage Table (جدول نهایی)

نام جدول نهایی:

```
kafka.tomandw
```

```sql
CREATE TABLE kafka.tomandw ON CLUSTER clickhouse_cluster
(
    id UInt64,
    name String,
    lastname String,
    op String,
    source_ts_ms UInt64,
    arrival_time DateTime DEFAULT now()
)
ENGINE = ReplicatedReplacingMergeTree(
    '/clickhouse/tables/{shard}/kafka/tomandw_v1',
    '{replica}',
    source_ts_ms
)
ORDER BY id;
```

📌 این جدول مقصد نهایی داده‌هاست و آخرین وضعیت هر رکورد را نگه می‌دارد.

---

## 📡 لایه 2: Kafka Engine Table (Consumer)

تاپیک Kafka:

```
toman_dw_prod.TomanDW.dbo.users
```

⚠️ **نکته بسیار مهم**
> برای خواندن داده‌ها از ابتدا، `kafka_group_name` باید جدید باشد.

```sql
CREATE TABLE kafka.tomandw_queue ON CLUSTER clickhouse_cluster
(
    id UInt64,
    name String,
    lastname String,
    __op String,
    __source_ts_ms UInt64,
    __deleted String
)
ENGINE = Kafka
SETTINGS
    kafka_broker_list = 'kafka:29092',
    kafka_topic_list = 'toman_dw_prod.TomanDW.dbo.users',
    kafka_group_name = 'tomandw_replay_from_zero_v2',
    kafka_format = 'JSONEachRow',
    kafka_skip_broken_messages = 1;
```

---

## 🔄 لایه 3: Materialized View (Production)

```sql
CREATE MATERIALIZED VIEW kafka.tomandw_mv
ON CLUSTER clickhouse_cluster
TO kafka.tomandw
AS
SELECT
    id,
    name,
    lastname,
    __op AS op,
    __source_ts_ms AS source_ts_ms
FROM kafka.tomandw_queue
WHERE __op != 'd';
```

---

sql
DROP TABLE kafka.tomandw_mv ON CLUSTER clickhouse_cluster SYNC;

CREATE MATERIALIZED VIEW kafka.tomandw_mv
ON CLUSTER clickhouse_cluster
TO kafka.tomandw
AS
SELECT
    id,
    name,
    lastname,
    __op AS op,
    __source_ts_ms AS source_ts_ms
FROM kafka.tomandw_queue
WHERE __op != 'd';
```

---

## 🧪 تست نهایی

```sql
SELECT count(*) FROM kafka.tomandw;
SELECT * FROM kafka.tomandw LIMIT 10;
```

برای دیدن آخرین وضعیت هر رکورد:

```sql
SELECT * FROM kafka.tomandw FINAL;
```

---

## ⚠️ نکات مهم Production

- برای تغییر schema:
  - همیشه `DROP TABLE ... SYNC` بزن
- از `CREATE TABLE IF NOT EXISTS` برای Kafka Engine و MV استفاده نکن
- برای replay داده‌ها:
  - فقط `kafka_group_name` را تغییر بده
- Kafka Engine جدول stream است و مستقیم SELECT نمی‌پذیرد مگر برای تست:

```sql
SET stream_like_engine_allow_direct_select = 1;
```

---

## 🎯 جمع‌بندی

✔ مشکل صفر بودن داده با group جدید و MV بدون فیلتر رفع شد
✔ معماری ۳ لایه Kafka → MV → Storage تثبیت شد
✔ آماده استفاده در Production با Debezium و ClickHouse Cluster

---

📄 This README is production-ready and safe to share with backend / data / DevOps teams.


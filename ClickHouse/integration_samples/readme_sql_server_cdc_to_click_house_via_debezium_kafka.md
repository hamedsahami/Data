# 🚀 SQL Server CDC to ClickHouse Pipeline via Debezium & Kafka

این پروژه شامل پیاده‌سازی یک **خط لوله داده (Real-time Data Pipeline)** برای انتقال تغییرات (Insert / Update / Delete) از دیتابیس **SQL Server** به **ClickHouse** با استفاده از **CDC (Change Data Capture)**، **Debezium** و **Apache Kafka** است.

هدف اصلی این معماری، دریافت بلادرنگ تغییرات دیتابیس عملیاتی و ذخیره‌سازی پایدار و تحلیلی آن‌ها در ClickHouse می‌باشد.

---

## 🧩 معماری سیستم

```text
SQL Server (CDC Enabled)
        │
        ▼
Debezium SQL Server Connector
        │
        ▼
Apache Kafka (Topics)
        │
        ▼
ClickHouse (Kafka Engine → Materialized View → Storage Table)
```

### اجزای اصلی

| Component | Description |
|---------|------------|
| **Source** | SQL Server با CDC فعال |
| **Connector** | Debezium Connector for SQL Server |
| **Message Broker** | Apache Kafka |
| **Sink / Target** | ClickHouse (معماری ۳ لایه) |

---

## 🏗 معماری ClickHouse (سه لایه)

### لایه ۱: Storage Table (لایه ذخیره‌سازی)

این جدول داده‌ها را به‌صورت دائمی ذخیره می‌کند و از موتور `ReplicatedMergeTree` برای پایداری و مقیاس‌پذیری در کلاستر استفاده می‌شود.

```sql
CREATE TABLE sqlserver.products_data ON CLUSTER clickhouse_cluster
(
    ProductID UInt64,
    ProductName String,
    Price String,           -- برای جلوگیری از خطا با داده‌های غیرعددی
    Stock Int32,
    CreatedAt Int64,
    op String,              -- نوع عملیات (c: create, u: update, r: read)
    ts_ms UInt64,           -- زمان رخداد در دیتابیس اصلی
    arrival_time DateTime DEFAULT now()
)
ENGINE = ReplicatedMergeTree('/clickhouse/tables/{shard}/products_data_v2', '{replica}')
ORDER BY (ProductID, ts_ms);
```

---

### لایه ۲: Kafka Engine Table (لایه شنونده)

این جدول به عنوان **Consumer** کافکا عمل می‌کند و پیام‌ها را مستقیماً از تاپیک Debezium دریافت می‌کند.

```sql
CREATE TABLE sqlserver.products_queue ON CLUSTER clickhouse_cluster
(
    ProductID UInt64,
    ProductName String,
    Price String,
    Stock Int32,
    CreatedAt Int64,
    __op String,
    __source_ts_ms UInt64,
    __deleted String
)
ENGINE = Kafka
SETTINGS
    kafka_broker_list = 'kafka:29092',
    kafka_topic_list = 'mssql-server.kafka.dbo.Products',
    kafka_group_name = 'final_sync_group_001',
    kafka_format = 'JSONEachRow',
    kafka_skip_broken_messages = 1;
```

📌 **نکته:**
- نام `kafka_group_name` تعیین می‌کند که دیتا از ابتدا (Earliest) یا ادامه مصرف شود.

---

### لایه ۳: Materialized View (واسط انتقال)

این لایه قلب سیستم است و داده‌ها را از Kafka Engine خوانده و به Storage Table تزریق می‌کند.

```sql
CREATE MATERIALIZED VIEW sqlserver.products_mv ON CLUSTER clickhouse_cluster
TO sqlserver.products_data AS
SELECT
    ProductID,
    ProductName,
    Price,
    Stock,
    CreatedAt,
    __op AS op,
    __source_ts_ms AS ts_ms
FROM sqlserver.products_queue
WHERE __deleted = 'false';
```

---

## ⚠️ چالش‌ها و راه‌حل‌ها (Troubleshooting)

| چالش | علت | راه‌حل |
|-----|-----|--------|
| Count = 0 | عدم تطابق حروف بزرگ و کوچک فیلدها | نام ستون‌ها دقیقاً مطابق JSON خروجی Debezium تنظیم شد (مثل `ProductID`) |
| Incompatible Columns | کش متادیتای قدیمی در ZooKeeper | تغییر مسیر `ReplicatedMergeTree` |
| خطای Price | وجود داده‌های رشته‌ای در فیلد عددی | تغییر نوع ستون به `String` |
| عدم دریافت دیتای قدیمی | Offset مصرف‌شده در Kafka | استفاده از `kafka_group_name` جدید |

---

## 💡 نکات کلیدی توسعه

### 🔤 Case Sensitivity
ClickHouse به **حساسیت حروف بزرگ و کوچک** در JSON حساس است. فیلدها باید دقیقاً مطابق خروجی Debezium باشند.

### 📸 Snapshot Data
برای دریافت دیتای قدیمی (Initial Snapshot):
- باید از یک `kafka_group_name` جدید استفاده شود

### 🔍 Direct Select از Kafka Table
برای تست مستقیم داده‌ها از جدول Kafka:

```sql
SET stream_like_engine_allow_direct_select = 1;
SELECT * FROM sqlserver.products_queue LIMIT 10;
```

---

## ✅ مزایای این معماری

- 🔄 انتقال بلادرنگ داده‌ها
- 📈 مقیاس‌پذیر و مقاوم در برابر خطا
- 🧠 مناسب برای تحلیل‌های OLAP
- 🧩 جداسازی کامل Source و Sink

---

## 📌 موارد قابل توسعه در آینده

- پیاده‌سازی Deduplication با `ReplacingMergeTree`
- پشتیبانی از Delete واقعی
- مانیتورینگ Lag کافکا
- Schema Evolution Handling

---

✍️ **Author**: Data Engineering Team

📦 **Stack**: SQL Server · Debezium · Kafka · ClickHouse


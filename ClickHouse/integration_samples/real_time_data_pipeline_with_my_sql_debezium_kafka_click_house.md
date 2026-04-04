# 🧩 Real-time Data Pipeline

این سند توضیح کامل یک **خط لوله داده‌ای بلادرنگ (Real-time Data Pipeline)** است که تغییرات MySQL را با استفاده از Debezium و Kafka در کمتر از چند ثانیه به ClickHouse منتقل می‌کند.

---

## 🔌 اتصال ClickHouse به شبکه Kafka (پیش‌نیاز)

برای اینکه ClickHouse بتواند Kafka را ببیند و از آن داده بخواند، لازم است هر دو سرویس روی **یک شبکه مشترک** (مثلاً Docker Network) قرار بگیرند.

در این پروژه، ClickHouse به‌صورت صریح به شبکه Kafka متصل شده است؛ به همین دلیل مقدار `kafka_broker_list` به‌صورت نام سرویس (`kafka:29092`) تنظیم شده و نه IP.

---

## 1️⃣ معماری کلی سیستم

جریان داده به شکل زیر است:

```
MySQL
  ↓
Debezium (CDC)
  ↓
Kafka Topic
  ↓
ClickHouse Kafka Engine
  ↓
Materialized View
  ↓
ClickHouse ReplicatedMergeTree (Final Table)
```

---

## 2️⃣ منبع داده — MySQL & Debezium

- هر تغییری که در MySQL اتفاق می‌افتد (Insert / Update / Delete)
- توسط **Debezium** شناسایی می‌شود (CDC)
- و به یک پیام **JSON** در Kafka تبدیل می‌شود

فیلدهای مهم:
- `__op` → نوع عملیات
- `__source_ts_ms` → زمان رخداد در MySQL

---

## 3️⃣ لایه ذخیره‌سازی دائمی — Final Table (ClickHouse)

این جدول مقصد نهایی و امن داده‌هاست.

```sql
CREATE TABLE kafka.users_data ON CLUSTER 'clickhouse_cluster'
(
    id UInt64,
    name String,
    email String,
    created_at String,
    op String,
    ts_ms UInt64,
    arrival_time DateTime DEFAULT now()
)
ENGINE = ReplicatedMergeTree('/clickhouse/tables/{shard}/users_data', '{replica}')
ORDER BY (id, ts_ms);
```

---

## 4️⃣ لایه دریافت از Kafka — Kafka Engine

این جدول فقط یک **Listener / Sensor** است:

- دیتا را ذخیره نمی‌کند
- فقط پیام‌ها را از Kafka می‌خواند

```sql
CREATE TABLE kafka.users_queue ON CLUSTER 'clickhouse_cluster'
(
    id UInt64,
    name String,
    email String,
    created_at String,
    __op String,
    __source_ts_ms UInt64,
    __deleted String
)
ENGINE = Kafka
SETTINGS kafka_broker_list = 'kafka:29092',
         kafka_topic_list = 'mysql-server.testdb.users',
         kafka_group_name = 'ch_final_consumer_group',
         kafka_format = 'JSONEachRow';
```

---

## 5️⃣ پمپ انتقال — Materialized View

قلب سیستم ❤️

```sql
CREATE MATERIALIZED VIEW kafka.users_mv ON CLUSTER 'clickhouse_cluster'
TO kafka.users_data AS
SELECT
    id,
    name,
    email,
    created_at,
    __op AS op,
    __source_ts_ms AS ts_ms
FROM kafka.users_queue
WHERE __deleted = 'false';
```

---

## 🔄 مسیر کامل حرکت داده (Data Journey)

1. کاربر در MySQL ثبت‌نام می‌کند
2. Debezium تغییر را شناسایی می‌کند
3. پیام به Kafka ارسال می‌شود
4. ClickHouse از Kafka می‌خواند
5. Materialized View داده را ذخیره می‌کند
6. ReplicatedMergeTree آن را Replicate می‌کند

---

## ✅ وضعیت نهایی سیستم

- Real-time (کمتر از ۵ ثانیه)
- High Availability
- مناسب Production

---

## 🧩 توضیح کانفیگ‌ها (Configuration Breakdown)

در این بخش، دقیقاً همان توضیحاتی که برای کانفیگ‌ها آماده شده، به‌صورت مرحله‌به‌مرحله و مفهومی آورده شده است.

---

## 1️⃣ لایه ذخیره‌سازی نهایی (Final Table)

این جدول روی **هارد دیسک هر ۳ نود** ذخیره می‌شود و وظیفه نگهداری امن و دائمی داده‌ها را دارد.

```sql
CREATE TABLE kafka.users_data ON CLUSTER 'clickhouse_cluster'
(
    id UInt64,                -- ۱. شناسه عددی (مثل ردیف در MySQL)
    name String,              -- ۲. نام کاربر
    email String,             -- ۳. ایمیل
    created_at String,        -- ۴. زمان ساخت در MySQL
    op String,                -- ۵. نوع عملیات (Insert=c, Update=u)
    ts_ms UInt64,             -- ۶. مهر زمانی میلی‌ثانیه‌ای از منبع
    arrival_time DateTime DEFAULT now() -- ۷. زمان ورود به ClickHouse
)
ENGINE = ReplicatedMergeTree('/clickhouse/tables/{shard}/users_data', '{replica}')
ORDER BY (id, ts_ms);
```

### توضیح خطوط کلیدی

- **ON CLUSTER**  
  به‌جای ساخت دستی جدول روی هر نود، این دستور به مدیر کلاستر گفته می‌شود تا جدول را روی هر ۳ نود بسازد.

- **ReplicatedMergeTree**  
  - `MergeTree`: داده‌ها مرتب ذخیره می‌شوند و در پس‌زمینه برای بهبود Performance ادغام می‌شوند.
  - `Replicated`: با کمک ClickHouse Keeper، هر دیتایی که یک نود دریافت کند به بقیه نودها هم منتقل می‌شود.

- **{shard} و {replica}**  
  آدرس‌های منحصربه‌فرد هر نود در Keeper برای جلوگیری از قاطی شدن داده‌ها.

- **ORDER BY (id, ts_ms)**  
  داده‌ها ابتدا بر اساس شناسه کاربر و سپس زمان مرتب می‌شوند؛ این کار Query روی داده‌های حجیم را بسیار سریع می‌کند.

---

## 2️⃣ لایه دریافت از کافکا (Kafka Engine)

این جدول **هیچ دیتایی ذخیره نمی‌کند** و فقط نقش یک شنودگر (Listener) متصل به Kafka را دارد.

```sql
CREATE TABLE kafka.users_queue ON CLUSTER 'clickhouse_cluster'
(
    id UInt64,
    name String,
    email String,
    created_at String,
    __op String,            -- نوع تغییر در Debezium
    __source_ts_ms UInt64,  -- زمان تغییر در Debezium
    __deleted String        -- آیا رکورد حذف شده؟
)
ENGINE = Kafka
SETTINGS kafka_broker_list = 'kafka:29092',
         kafka_topic_list = 'mysql-server.testdb.users',
         kafka_group_name = 'ch_final_group',
         kafka_format = 'JSONEachRow';
```

### توضیح خطوط کلیدی

- **ENGINE = Kafka**  
  این جدول مثل یک برنامه مصرف‌کننده (Consumer) عمل می‌کند، نه یک جدول ذخیره‌سازی.

- **kafka_group_name**  
  بسیار حیاتی است؛ Kafka با این نام به خاطر می‌سپارد ClickHouse تا کدام Offset پیام‌ها را خوانده است.

- **JSONEachRow**  
  چون Debezium پیام‌ها را به‌صورت JSON می‌فرستد، هر JSON یک ردیف در نظر گرفته می‌شود.

---

## 3️⃣ پل اتصال (Materialized View)

این بخش همان قسمت «جادویی» سیستم است که داده را از Kafka گرفته و به جدول نهایی منتقل می‌کند.

```sql
CREATE MATERIALIZED VIEW kafka.users_mv ON CLUSTER 'clickhouse_cluster'
TO kafka.users_data AS
SELECT 
    id, name, email, created_at,
    __op AS op,
    __source_ts_ms AS ts_ms
FROM kafka.users_queue
WHERE __deleted = 'false';
```

### توضیح خطوط کلیدی

- **MATERIALIZED VIEW**  
  برخلاف View معمولی، این یک Trigger واقعی است و با رسیدن هر پیام جدید فعال می‌شود.

- **TO kafka.users_data**  
  داده پردازش‌شده مستقیماً به جدول نهایی برای ذخیره دائمی ارسال می‌شود.

- **__op AS op**  
  نام فیلدهای Debezium به نام‌های تمیز و قابل Query تغییر داده می‌شوند.

---

## 4️⃣ دستورات مدیریتی و خطایابی

برای پاک‌سازی کامل این Pipeline و ساخت مجدد:

```sql
DROP TABLE IF EXISTS kafka.users_data ON CLUSTER 'clickhouse_cluster' SYNC;
```

- **SYNC**  
  ClickHouse را مجبور می‌کند تا قبل از بازگشت دستور، متادیتا را کاملاً از Keeper پاک کند؛ بدون این گزینه ممکن است در CREATE بعدی خطا بگیری.

---

### وضعیت فعلی سیستم

- دیتابیس کاملاً توزیع‌شده
- تحمل‌پذیر در برابر Down شدن نودها
- اتصال دائم به Kafka بدون وقفه

### قدم بعدی پیشنهادی

استفاده از **ReplacingMergeTree** برای نگه‌داشتن فقط آخرین نسخه هر رکورد (به‌جای نگه‌داشتن تمام Updateها).

---

### 🔹 تنظیمات جدول نهایی (ReplicatedMergeTree)

```sql
ON CLUSTER 'clickhouse_cluster'
```
- دستور را به‌جای یک نود، روی **تمام نودهای کلاستر** اجرا می‌کند
- تضمین می‌کند اسکیمای جدول روی همه نودها یکسان است

```sql
ENGINE = ReplicatedMergeTree('/clickhouse/tables/{shard}/users_data', '{replica}')
```
- موتور High Availability کلیک‌هاوس
- داده‌ها بین Replicaها Sync می‌شوند
- اگر یک نود Down شود، داده از دست نمی‌رود

```sql
{shard} / {replica}
```
- جایگزین خودکار توسط ClickHouse Keeper
- مشخص می‌کند هر نود:
  - عضو کدام Shard است
  - Replica چه نودی محسوب می‌شود

```sql
ORDER BY (id, ts_ms)
```
- ترتیب فیزیکی ذخیره داده روی دیسک
- باعث می‌شود نسخه‌های جدیدتر یک رکورد بعد از نسخه‌های قدیمی قرار بگیرند
- برای Queryهای Time-based و Versioning بسیار مهم است

---

### 🔹 تنظیمات Kafka Engine

```sql
ENGINE = Kafka
```
- این جدول **داده ذخیره نمی‌کند**
- فقط نقش Consumer را دارد

```sql
kafka_broker_list = 'kafka:29092'
```
- آدرس Kafka Broker
- استفاده از Service Name به‌جای IP باعث پایداری بیشتر می‌شود

```sql
kafka_topic_list = 'mysql-server.testdb.users'
```
- Topicای که Debezium تغییرات MySQL را در آن می‌نویسد

```sql
kafka_group_name = 'ch_final_consumer_group'
```
- مثل Bookmark عمل می‌کند
- Kafka یادش می‌ماند این Consumer تا کدام Offset پیام‌ها را خوانده
- باعث می‌شود بعد از Restart دوباره از اول نخواند

```sql
kafka_format = 'JSONEachRow'
```
- هر پیام Kafka یک JSON مستقل است
- ClickHouse فیلدهای JSON را با ستون‌ها Match می‌کند

---

### 🔹 تنظیمات Materialized View

```sql
TO kafka.users_data
```
- خروجی Materialized View مستقیماً داخل جدول نهایی نوشته می‌شود
- بدون نیاز به INSERT دستی

```sql
AS SELECT ... FROM kafka.users_queue
```
- داده را از Kafka Engine می‌خواند
- امکان Transform، Rename و Filter را می‌دهد

```sql
__op AS op
__source_ts_ms AS ts_ms
```
- فیلدهای Debezium به نام‌های ساده‌تر تبدیل می‌شوند
- مناسب Query و Analytics

```sql
WHERE __deleted = 'false'
```
- رکوردهای حذف‌شده (Delete) را فیلتر می‌کند
- باعث می‌شود فقط داده‌های فعال وارد جدول نهایی شوند

---

## 🧪 نمونه تست (How to Test)

در این بخش دقیقاً مشخص می‌کنیم **کجا بروی** و **چه دستوری بزنی** تا کل Pipeline را End-to-End تست کنی.

---

### 🟢 قدم 0: مطمئن شو سرویس‌ها بالا هستند

روی سیستمی که Docker Compose را اجرا کرده‌ای:

```bash
docker ps
```

باید حداقل این سرویس‌ها را ببینی:
- mysql
- kafka
- clickhouse-server
- debezium

---

### 🟢 قدم 1: ورود به MySQL و Insert داده تست

1. وارد کانتینر MySQL شو:

```bash
docker exec -it mysql mysql -u root -p
```

2. دیتابیس را انتخاب کن:

```sql
USE testdb;
```

3. یک رکورد تست Insert کن:

```sql
INSERT INTO users (name, email, created_at)
VALUES ('Ali', 'ali@test.com', NOW());
```

📌 **اینجا Debezium فعال می‌شود** و پیام را به Kafka می‌فرستد.

---

### 🟢 قدم 2: (اختیاری) دیدن پیام در Kafka

اگر می‌خواهی مطمئن شوی Kafka پیام را گرفته:

```bash
docker exec -it kafka kafka-console-consumer \
  --bootstrap-server kafka:29092 \
  --topic mysql-server.testdb.users \
  --from-beginning
```

باید یک JSON شبیه این ببینی:
- `__op : "c"`
- `__source_ts_ms`

---

### 🟢 قدم 3: ورود به ClickHouse و Query گرفتن

1. وارد ClickHouse شو:

```bash
docker exec -it clickhouse clickhouse-client
```

2. داده نهایی را ببین:

```sql
SELECT *
FROM kafka.users_data
ORDER BY ts_ms DESC
LIMIT 5;
```

✅ اگر رکورد را دیدی یعنی Pipeline درست کار می‌کند.

---

### 🟢 قدم 4: تست Update

1. دوباره در MySQL:

```sql
UPDATE users
SET email = 'ali_new@test.com'
WHERE name = 'Ali';
```

2. دوباره در ClickHouse:

```sql
SELECT id, email, op, ts_ms
FROM kafka.users_data
WHERE name = 'Ali'
ORDER BY ts_ms;
```

باید دو رکورد ببینی:
- `op = c` (insert)
- `op = u` (update)

---

### 🟢 قدم 5: بررسی Latency (اختیاری ولی حرفه‌ای)

```sql
SELECT
    id,
    ts_ms,
    arrival_time,
    arrival_time - toDateTime(ts_ms / 1000) AS latency_seconds
FROM kafka.users_data
ORDER BY ts_ms DESC
LIMIT 5;
```

Latency معمولاً زیر چند ثانیه است.

---

## ✅ نتیجه نهایی

اگر تمام مراحل بالا بدون خطا انجام شد:
- Debezium سالم است
- Kafka سالم است
- ClickHouse به Kafka وصل است
- Materialized View درست کار می‌کند

Pipeline شما **Production-Ready** است 🚀

Happy Streaming 🌊


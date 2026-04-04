# SQL Server to Kafka CDC (Debezium) Setup Guide

این مستند مراحل راه‌اندازی و عیب‌یابی کانکتور **Debezium** برای انتقال داده‌ها از دیتابیس **SQL Server** به **Kafka** را تشریح می‌کند.

---

## ۱. پیش‌نیازها در SQL Server

قبل از هر چیز، قابلیت **CDC** باید روی دیتابیس و جدول مورد نظر فعال باشد.

```sql
-- ۱. فعال‌سازی CDC در سطح دیتابیس
USE TomanDW;
EXEC sys.sp_cdc_enable_db;

-- ۲. فعال‌سازی CDC برای جدول کاربران
EXEC sys.sp_cdc_enable_table
    @source_schema = N'dbo',
    @source_name   = N'users',
    @role_name     = NULL; -- یا نام رول خاص برای دسترسی
```

---

## ۲. تنظیمات دسترسی کاربر (User Permissions)

کاربر `debezium` باید دسترسی‌های زیر را داشته باشد تا بتواند مرحله **Snapshot** و **Streaming** را با موفقیت انجام دهد:

```sql
USE TomanDW;

-- دسترسی برای خواندن داده‌ها
GRANT SELECT ON dbo.users TO debezium;

-- دسترسی حیاتی برای خواندن لاگ‌های تغییرات
GRANT SELECT ON SCHEMA::cdc TO debezium;

-- دسترسی برای مشاهده ساختار جداول
GRANT VIEW DEFINITION TO debezium;
```

---

## ۳. پیکربندی کانکتور (Connector JSON)

فایل تنظیمات کانکتور (`toman-dw-connector.json`) با بهینه‌سازی برای SQL Server:

```json
{
  "name": "toman-dw-connector",
  "config": {
    "connector.class": "io.debezium.connector.sqlserver.SqlServerConnector",
    "tasks.max": "1",
    "database.hostname": "172.26.1.14",
    "database.port": "1433",
    "database.user": "debezium",
    "database.password": "P@ssw0rd123",
    "database.names": "TomanDW",
    "topic.prefix": "toman_dw_prod",
    "table.include.list": "dbo.users",
    "snapshot.mode": "initial",
    "snapshot.isolation.mode": "read_committed",
    "snapshot.locking.mode": "none",
    "database.encrypt": "false",
    "schema.history.internal.kafka.bootstrap.servers": "kafka:29092",
    "schema.history.internal.kafka.topic": "schema-changes.toman.dw",
    "transforms": "unwrap",
    "transforms.unwrap.type": "io.debezium.transforms.ExtractNewRecordState"
  }
}
```

---

## ۴. دستورات مدیریتی (CLI)

### ارسال کانکتور به Kafka Connect

```bash
curl -i -X POST \
  -H "Accept:application/json" \
  -H "Content-Type:application/json" \
  http://localhost:8083/connectors/ \
  -d @toman-dw-connector.json
```

### بررسی وضعیت کانکتور

```bash
curl -s http://localhost:8083/connectors/toman-dw-connector/status | jq
```

### مشاهده لاگ‌های زنده (عیب‌یابی)

```bash
docker logs -f [CONTAINER_NAME] | grep -E "snapshot|matching|records"
```

---

## ۵. نکات عیب‌یابی (Troubleshooting)

### ❗ عدم نمایش تاپیک دیتا

Debezium تا زمانی که اولین رکورد را با موفقیت نخواند یا تغییری در جدول نبیند، تاپیک دیتا را ایجاد نمی‌کند.

---

### ❗ خطای `Unknown Topic`

اگر Kafka اجازه ساخت خودکار تاپیک را نمی‌دهد، تاپیک را به‌صورت دستی بسازید:

```
toman_dw_prod.TomanDW.dbo.users
```

---

### ❗ Snapshot با ۰ رکورد

اگر در لاگ‌ها پیام زیر را دیدید:

```
Finished exporting 0 records
```

در حالی که جدول داده دارد، حتماً دسترسی `SELECT` کاربر `debezium` را در SQL Server بررسی کنید.

---

## 📌 نکته پایانی

برای محیط‌های Production توصیه می‌شود:
- رمز عبور دیتابیس در **Secrets Manager** نگهداری شود
- دسترسی‌ها به حداقل مورد نیاز محدود شوند
- مانیتورینگ Kafka Connect و Debezium فعال باشد


# راهنمای جامع اتصال SQL Server به Kafka با Debezium

این پروژه تغییرات جدول **Products** در دیتابیس **SQL Server** را به‌صورت لحظه‌ای (Real-time) به یک **Topic** در **Apache Kafka** منتقل می‌کند. این کار با استفاده از **Debezium SQL Server Connector** و **Kafka Connect** انجام می‌شود.

---

## پیش‌نیازها

- Docker و Docker Compose
- SQL Server (در حال اجرا داخل کانتینر)
- Apache Kafka
- Kafka Connect (به همراه Debezium)
- Kafka UI (اختیاری برای مانیتورینگ)

---

## مرحله ۱: فعال‌سازی SQL Server Agent

> بدون SQL Server Agent، قابلیت **CDC** کار نخواهد کرد.

دستورات زیر را در ترمینال لینوکس اجرا کنید:

```bash
# ۱. فعال کردن SQL Server Agent در کانتینر
docker exec -it mssql_server /opt/mssql/bin/mssql-conf set sqlagent.enabled true

# ۲. ری‌استارت کانتینر برای اعمال تغییرات
docker restart mssql_server
```

---

## مرحله ۲: آماده‌سازی دیتابیس (SQL)

کدهای زیر را در کنسول SQL (مانند `sqlcmd` یا `DBeaver`) اجرا کنید:

```sql
-- ۱. ساخت دیتابیس
CREATE DATABASE kafka;
GO
USE kafka;
GO

-- ۲. فعال‌سازی CDC در سطح دیتابیس
EXEC sys.sp_cdc_enable_db;
GO

-- ۳. ساخت جدول محصولات (با پشتیبانی از فارسی)
CREATE TABLE Products (
    ProductID INT PRIMARY KEY IDENTITY(1,1),
    ProductName NVARCHAR(100) NOT NULL,
    Price DECIMAL(18, 2),
    Stock INT,
    CreatedAt DATETIME DEFAULT GETDATE()
);
GO

-- ۴. فعال‌سازی CDC برای جدول محصولات
EXEC sys.sp_cdc_enable_table
    @source_schema = N'dbo',
    @source_name = N'Products',
    @role_name = NULL,
    @supports_net_changes = 1;
GO
```

---

## مرحله ۳: ساخت فایل پیکربندی کانکتور

فایلی با نام `mssql-connector.json` ایجاد کرده و محتوای زیر را در آن قرار دهید:

```json
{
  "name": "mssql-connector",
  "config": {
    "connector.class": "io.debezium.connector.sqlserver.SqlServerConnector",
    "tasks.max": "1",
    "database.hostname": "mssql_server",
    "database.port": "1433",
    "database.user": "sa",
    "database.password": "YourStrongPassword123!",
    "database.names": "kafka",
    "database.encrypt": "false",
    "topic.prefix": "mssql-server",
    "table.include.list": "dbo.Products",
    "snapshot.mode": "initial",
    "schema.history.internal.kafka.bootstrap.servers": "kafka:29092",
    "schema.history.internal.kafka.topic": "schema-changes.mssql.products",
    "transforms": "unwrap",
    "transforms.unwrap.type": "io.debezium.transforms.ExtractNewRecordState",
    "transforms.unwrap.drop.tombstones": "false",
    "transforms.unwrap.delete.handling.mode": "rewrite",
    "transforms.unwrap.add.fields": "op,source.ts_ms",
    "value.converter": "org.apache.kafka.connect.json.JsonConverter",
    "value.converter.schemas.enable": "false",
    "value.converter.browser.encoding": "UTF-8"
  }
}
```

---

## مرحله ۴: ثبت کانکتور در Kafka Connect

دستورات زیر را در ترمینال اجرا کنید:

```bash
# حذف کانکتور قدیمی (در صورت وجود)
curl -X DELETE localhost:8083/connectors/mssql-connector

# ثبت کانکتور جدید
curl -i -X POST \
  -H "Accept:application/json" \
  -H "Content-Type:application/json" \
  localhost:8083/connectors/ \
  -d @mssql-connector.json
```

---

## مرحله ۵: تست و درج داده‌های انبوه

برای تست، ۱۰۰ ردیف داده به‌صورت خودکار وارد جدول کنید:

```bash
docker exec -it mssql_server /opt/mssql-tools18/bin/sqlcmd \
  -S localhost \
  -U sa \
  -P 'YourStrongPassword123!' \
  -C \
  -d kafka \
  -Q "
DECLARE @i INT = 1;
WHILE @i <= 100
BEGIN
    INSERT INTO Products (ProductName, Price, Stock)
    VALUES (N'محصول شماره ' + CAST(@i AS NVARCHAR(10)), RAND()*1000, @i);
    SET @i = @i + 1;
END;
"
```

پس از اجرای این دستور، پیام‌ها باید به‌صورت Real-time در Kafka منتشر شوند.

---

## مرحله ۶: مانیتورینگ

1. **بررسی وضعیت کانکتور**:
   ```bash
   curl -s localhost:8083/connectors/mssql-connector/status
   ```

2. **مشاهده پیام‌ها**:
   - ورود به Kafka UI:
     ```
     http://IP:8082
     ```

3. **نام تاپیک در کافکا**:
   ```
   mssql-server.kafka.dbo.Products
   ```

---

## 📌 نکات کلیدی

- **پورت 29092**: برای ارتباط داخلی بین کانتینرهای Kafka و Kafka Connect استفاده شده است.
- **حرف N در SQL**: برای ذخیره صحیح متون فارسی از `N'متن فارسی'` استفاده شده است.
- **Unwrap Transform**: برای تولید JSON تمیز و ساده (فقط شامل فیلدهای اصلی جدول) در Kafka.
- **CDC**: تمام تغییرات Insert / Update / Delete به‌صورت لحظه‌ای به Kafka ارسال می‌شوند.

---

## نتیجه

با این راهنما، شما یک Pipeline کامل **SQL Server → Debezium → Kafka** دارید که برای:

- Data Streaming
- Event-Driven Architecture
- Sync دیتابیس‌ها
- Real-time Analytics

کاملاً آماده استفاده در محیط Production است 🚀


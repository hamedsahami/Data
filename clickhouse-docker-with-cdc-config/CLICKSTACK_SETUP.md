# راهنمای نصب و استفاده از ClickStack (Grafana + ClickHouse)

ClickStack یک پلتفرم observability است که از Grafana با ClickHouse datasource برای نظارت و تحلیل داده‌های ClickHouse استفاده می‌کند.

## 📋 اطلاعات نصب

ClickStack در این کلاستر به صورت Grafana با ClickHouse datasource نصب شده است.

### مشخصات نصب

- **نوع**: Grafana با ClickHouse Datasource Plugin
- **پورت**: `3000`
- **URL**: `http://localhost:3000`
- **Username**: `admin`
- **Password**: `admin`
- **Datasource**: ClickHouse Cluster (via HAProxy)

## 🔌 اتصال به ClickHouse

ClickStack از طریق HAProxy به کلاستر ClickHouse متصل می‌شود:

- **Host**: `clickhouse-haproxy` (داخل شبکه Docker)
- **Port**: `8124` (پورت داخلی HAProxy)
- **Username**: `default`
- **Password**: `default`
- **Database**: `default`

## 🚀 دسترسی به Grafana

### 1. باز کردن رابط وب

مرورگر خود را باز کنید و به آدرس زیر بروید:

```
http://localhost:3000
```

### 2. ورود به سیستم

- **Username**: `admin`
- **Password**: `admin`

⚠️ **نکته امنیتی**: در اولین ورود، Grafana از شما می‌خواهد پسورد admin را تغییر دهید.

### 3. بررسی Datasource

پس از ورود:

1. به **Configuration** → **Data Sources** بروید
2. باید datasource با نام **"ClickHouse Cluster (via HAProxy)"** را ببینید
3. روی آن کلیک کنید و **"Test"** را بزنید
4. باید پیام **"Data source is working"** را ببینید

## 📊 استفاده از ClickStack

### ایجاد Dashboard

1. به **Dashboards** → **New Dashboard** بروید
2. **Add visualization** را انتخاب کنید
3. Datasource را **"ClickHouse Cluster (via HAProxy)"** انتخاب کنید
4. کوئری SQL خود را بنویسید

### مثال کوئری‌های مفید

#### بررسی وضعیت کلاستر

```sql
SELECT 
    hostName() as node,
    uptime() as uptime_seconds,
    version() as version
FROM cluster('clickhouse_cluster', system.one)
```

#### بررسی جداول

```sql
SELECT 
    database,
    name,
    engine,
    total_rows,
    total_bytes
FROM system.tables
WHERE database != 'system'
ORDER BY total_bytes DESC
```

#### بررسی استفاده از حافظه

```sql
-- استفاده از حافظه توسط کوئری‌های در حال اجرا
SELECT 
    hostName() as node,
    user,
    formatReadableSize(memory_usage) as memory_used,
    formatReadableSize(peak_memory_usage) as peak_memory_used
FROM cluster('clickhouse_cluster', system.processes)
WHERE query_id != ''
ORDER BY memory_usage DESC

-- یا بررسی حافظه کل سیستم
SELECT 
    hostName() as node,
    metric,
    formatReadableSize(value) as value_readable
FROM cluster('clickhouse_cluster', system.metrics)
WHERE metric IN ('MemoryTracking', 'MemoryAllocatorAllocated', 'MemoryAllocatorFree')
ORDER BY node, metric
```

#### بررسی کوئری‌های در حال اجرا

```sql
SELECT 
    hostName() as node,
    user,
    query,
    elapsed,
    read_rows,
    read_bytes
FROM cluster('clickhouse_cluster', system.processes)
WHERE query != ''
```

**نکته مهم:** برای استفاده از `cluster()` function، باید در فایل `clickhouse-nodes.xml` برای هر replica، `user` و `password` تعریف شده باشد. این تنظیمات در کانفیگ فعلی اضافه شده است.

## 🔧 تنظیمات Datasource

Datasource به صورت خودکار با تنظیمات زیر provision شده است:

```yaml
Name: ClickHouse Cluster (via HAProxy)
Type: ClickHouse
URL: http://clickhouse-haproxy:8124
Database: default
Username: default
Password: default (stored securely)
Port: 8124
Timeout: 30 seconds
```

### تغییر تنظیمات (در صورت نیاز)

1. به **Configuration** → **Data Sources** بروید
2. روی **"ClickHouse Cluster (via HAProxy)"** کلیک کنید
3. تنظیمات را تغییر دهید
4. **Save & Test** را بزنید

## 🧪 تست اتصال

### تست از طریق Grafana UI

1. به **Configuration** → **Data Sources** بروید
2. روی datasource کلیک کنید
3. دکمه **"Test"** را بزنید
4. باید پیام موفقیت را ببینید

### تست از طریق API

```powershell
# در Windows PowerShell
$cred = [Convert]::ToBase64String([Text.Encoding]::ASCII.GetBytes("admin:admin"))
Invoke-WebRequest -Uri "http://localhost:3000/api/datasources" -Headers @{Authorization = "Basic $cred"} -UseBasicParsing
```

### تست مستقیم از کانتینر

```bash
docker exec clickhouse-clickstack sh -c "wget -qO- 'http://clickhouse-haproxy:8124/?user=default&password=default&query=SELECT%20currentUser()'"
```

**خروجی مورد انتظار**: `default`

## 📈 Dashboard های پیشنهادی

### 1. Dashboard وضعیت کلاستر

```sql
-- تعداد نودها
SELECT count() as node_count 
FROM cluster('clickhouse_cluster', system.one)

-- وضعیت هر نود
SELECT 
    hostName() as node,
    uptime() as uptime_seconds,
    version() as version
FROM cluster('clickhouse_cluster', system.one)
```

### 2. Dashboard استفاده از منابع

```sql
-- استفاده از حافظه
SELECT 
    hostName() as node,
    formatReadableSize(sum(memory_usage)) as total_memory
FROM cluster('clickhouse_cluster', system.processes)
GROUP BY node

-- استفاده از دیسک
SELECT 
    hostName() as node,
    formatReadableSize(sum(bytes_on_disk)) as disk_usage
FROM cluster('clickhouse_cluster', system.parts)
GROUP BY node
```

### 3. Dashboard کوئری‌ها

```sql
-- کوئری‌های کند
SELECT 
    hostName() as node,
    user,
    query,
    elapsed,
    read_rows
FROM cluster('clickhouse_cluster', system.processes)
WHERE query != ''
ORDER BY elapsed DESC
LIMIT 10
```

## 🔍 عیب‌یابی

### مشکل: Datasource کار نمی‌کند

**بررسی‌ها:**

1. **بررسی اتصال شبکه:**
```bash
docker exec clickhouse-clickstack ping -c 3 clickhouse-haproxy
```

2. **بررسی دسترسی به ClickHouse:**
```bash
docker exec clickhouse-clickstack sh -c "wget -qO- 'http://clickhouse-haproxy:8124/ping'"
```

3. **بررسی لاگ Grafana:**
```bash
docker logs clickhouse-clickstack | grep -i "clickhouse\|datasource\|error"
```

### مشکل: Grafana راه‌اندازی نمی‌شود

```bash
# بررسی وضعیت
docker-compose ps clickstack

# بررسی لاگ
docker-compose logs clickstack

# Restart
docker-compose restart clickstack
```

### مشکل: Plugin نصب نشده

```bash
# بررسی لاگ نصب plugin
docker logs clickhouse-clickstack | grep -i "vertamedia-clickhouse"

# اگر نصب نشده، restart کنید
docker-compose restart clickstack
```

## 📝 دستورات مفید

### Restart ClickStack

```bash
docker-compose restart clickstack
```

### مشاهده لاگ‌ها

```bash
docker-compose logs -f clickstack
```

### دسترسی به shell Grafana

```bash
docker exec -it clickhouse-clickstack sh
```

### بررسی Datasource از API

```powershell
$cred = [Convert]::ToBase64String([Text.Encoding]::ASCII.GetBytes("admin:admin"))
$headers = @{Authorization = "Basic $cred"}
Invoke-RestMethod -Uri "http://localhost:3000/api/datasources" -Headers $headers
```

## 🔐 امنیت

⚠️ **توجه**: برای محیط production:

1. **تغییر پسورد admin Grafana** (در اولین ورود)
2. **استفاده از HTTPS** برای Grafana
3. **محدود کردن دسترسی شبکه** به Grafana
4. **استفاده از Authentication مناسب** (LDAP, OAuth, etc.)

## 📚 منابع

- [Grafana Documentation](https://grafana.com/docs/)
- [ClickHouse Grafana Plugin](https://grafana.com/grafana/plugins/vertamedia-clickhouse-datasource/)
- [ClickStack Documentation](https://clickhouse.com/docs/use-cases/observability/clickstack/overview)

---

**آخرین به‌روزرسانی:** 2025-11-16


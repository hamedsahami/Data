# ClickHouse Cluster with Docker Compose

این پروژه شامل یک کلاستر ClickHouse با 3 نود و 3 نود ClickHouse Keeper برای مدیریت هماهنگ سازی است.

## 📋 ساختار پروژه

### فایل‌های اصلی
- **docker-compose.yml**: فایل اصلی Docker Compose برای اجرای کلاستر
- **haproxy.cfg**: تنظیمات HAProxy برای load balancing
- **clickhouse-config.xml**: تنظیمات مشترک برای تمام نودهای ClickHouse (شبکه، Zookeeper، DDL)
- **clickhouse-nodes.xml**: تعریف کلاستر ClickHouse با نام `clickhouse_cluster`
- **clickhouse-users.xml**: تعریف یوزرها، profiles و quotas

### فایل‌های اختصاصی هر نود
- **macros.xml**: ماکروهای نود clickhouse-1 (shard=1, replica=clickhouse-1)
- **macros-2.xml**: ماکروهای نود clickhouse-2 (shard=1, replica=clickhouse-2)
- **macros-3.xml**: ماکروهای نود clickhouse-3 (shard=1, replica=clickhouse-3)
- **keeper-config-1.xml**: تنظیمات Keeper نود 1 (server_id=1)
- **keeper-config-2.xml**: تنظیمات Keeper نود 2 (server_id=2)
- **keeper-config-3.xml**: تنظیمات Keeper نود 3 (server_id=3)

## 🏗️ معماری کلاستر

### ساختار کلاستر
- **1 Shard** با **3 Replica**
- نام کلاستر: `clickhouse_cluster`
- تمام replicaها در یک shard قرار دارند (replication setup)

### نودهای ClickHouse
- **clickhouse-1**: Replica 1
- **clickhouse-2**: Replica 2
- **clickhouse-3**: Replica 3

### نودهای ClickHouse Keeper
- **keeper-1**: Server ID 1
- **keeper-2**: Server ID 2
- **keeper-3**: Server ID 3

## 🔧 تغییرات انجام شده و علت آن‌ها

### 1. اضافه کردن پسورد برای کاربر `default`

**فایل تغییر یافته:** `clickhouse-users.xml`

**تغییرات:**
- کاربر `default` حالا پسورد دارد: `default`
- پسورد به صورت SHA256 hash ذخیره شده: `37a8eec1ce19687d132fe29051dca629d164e2c4958ba141d5f4133a33f0688f`
- از `replace="true"` استفاده شده تا تعریف پیش‌فرض کانتینر override شود

**علت تغییر:**
- کاربر `default` در ClickHouse به صورت پیش‌فرض بدون پسورد است که یک خطر امنیتی محسوب می‌شود
- بدون پسورد، هر کسی می‌تواند با کاربر `default` وارد شود
- برای محیط production و حتی development، داشتن پسورد ضروری است

**کد تغییر یافته:**
```xml
<default replace="true">
    <password_sha256_hex>37a8eec1ce19687d132fe29051dca629d164e2c4958ba141d5f4133a33f0688f</password_sha256_hex>
    <networks>
        <ip>::/0</ip>
    </networks>
    <profile>default</profile>
    <access_management>1</access_management>
    <!-- ... -->
</default>
```

### 2. حذف Quota محدودیت‌ها

**فایل تغییر یافته:** `clickhouse-users.xml`

**تغییرات:**
- بخش `<quotas>` خالی شد (بدون محدودیت)
- تگ `<quota>default</quota>` از کاربران حذف شد

**علت تغییر:**
- Quota با `queries: 0` تنظیم شده بود که اجرای کوئری را مسدود می‌کرد
- کاربر می‌توانست وارد شود اما نمی‌توانست کوئری اجرا کند
- برای محیط development، حذف محدودیت‌ها مناسب‌تر است

**کد قبل از تغییر:**
```xml
<quotas>
    <default>
        <interval>
            <duration>3600</duration>
            <queries>0</queries>  <!-- این باعث مسدود شدن کوئری‌ها می‌شد -->
            <errors>0</errors>
            <result_rows>0</result_rows>
            <read_rows>0</read_rows>
            <execution_time>0</execution_time>
        </interval>
    </default>
</quotas>
```

**کد بعد از تغییر:**
```xml
<quotas>
    <!-- No quota limits - unlimited queries -->
</quotas>
```

## 🌐 پورت‌های استفاده شده

### HAProxy Load Balancer ✨
- **8080**: HTTP interface (load balanced) - برای HTTP requests و رابط وب 🌐
  - Map شده به پورت داخلی 8124
- **9004**: Native protocol (load balanced) - برای JDBC و اتصالات مستقیم 🔴
  - Map شده به پورت داخلی 9000
- **8404**: HAProxy Statistics Dashboard

### ClickHouse Nodes (دسترسی مستقیم - اختیاری)
- **clickhouse-1**: 
  - HTTP: `8123` → داخلی `8123`
  - Native: `9000` → داخلی `9000`
- **clickhouse-2**: 
  - HTTP: `8124` → داخلی `8123`
  - Native: `9001` → داخلی `9000`
- **clickhouse-3**: 
  - HTTP: `8125` → داخلی `8123`
  - Native: `9002` → داخلی `9000`

### ClickHouse Keeper Nodes
- **keeper-1**: 
  - REST API: `9181` → داخلی `9181`
  - Client: `9234` → داخلی `9234`
- **keeper-2**: 
  - REST API: `9182` → داخلی `9181`
  - Client: `9235` → داخلی `9234`
- **keeper-3**: 
  - REST API: `9183` → داخلی `9181`
  - Client: `9236` → داخلی `9234`

## 👥 یوزرهای پیش‌فرض

### Super Admin
- **Username**: `admin`
- **Password**: `password`
- **SHA256 Hash**: `5e884898da28047151d0e56f8dc6292773603d0d6aabbdd62a11ef721d1542d8`
- **Privileges**: دسترسی کامل به تمام پایگاه داده‌ها و مدیریت کاربران
- **Network Access**: از همه IP‌ها (`::/0`)

### Default User
- **Username**: `default`
- **Password**: `default` ✅ (اضافه شده برای امنیت)
- **SHA256 Hash**: `37a8eec1ce19687d132fe29051dca629d164e2c4958ba141d5f4133a33f0688f`
- **Privileges**: دسترسی کامل (بدون محدودیت quota)
- **Network Access**: از همه IP‌ها (`::/0`)

## 📊 Profiles و Quotas

### Profiles
- **default**: 
  - `max_memory_usage`: 10GB
  - `use_uncompressed_cache`: 0
  - `load_balancing`: random
- **readonly**: 
  - `readonly`: 1
  - `max_memory_usage`: 5GB

### Quotas
- **بدون محدودیت**: تمام quota‌ها حذف شده‌اند

## 🚀 نحوه راه‌اندازی

1. **اجرای کلاستر:**
```bash
docker-compose up -d
```

2. **بررسی وضعیت کانتینرها:**
```bash
docker-compose ps
```

3. **مشاهده لاگ‌ها:**
```bash
docker-compose logs -f clickhouse-1
```

4. **بررسی وضعیت کلاستر:**
```bash
docker exec clickhouse-1 clickhouse-client --user admin --password password --query "SELECT * FROM system.clusters"
```

## 🔌 نحوه اتصال

### از طریق رابط وب (Web UI)

**از طریق HAProxy (پورت 8080):**
```
http://localhost:8080/play?user=default&password=default
```

**از طریق نود مستقیم:**
```
http://localhost:8123/play?user=default&password=default
```

⚠️ **نکته مهم:** حتماً `user` و `password` را در URL اضافه کنید، در غیر این صورت خطای Authentication می‌گیرید.

### از طریق ClickHouse Client

```bash
# از طریق HAProxy (توصیه می‌شود)
clickhouse-client --host localhost --port 9004 --user default --password default

# یا از طریق نود مستقیم
clickhouse-client --host localhost --port 9000 --user default --password default
```

### از طریق HTTP API

```bash
# با Basic Authentication
curl 'http://localhost:8080/?query=SELECT 1' --user default:default

# یا با پارامترهای URL
curl 'http://localhost:8080/?user=default&password=default&query=SELECT 1'
```

### از طریق JDBC

```
URL: jdbc:clickhouse://localhost:9004/default
Username: default
Password: default
```

### مشاهده آمار HAProxy

```
http://localhost:8404/stats
```

## 📝 کار با کلاستر

### ایجاد جدول توزیع شده

```sql
-- ایجاد جدول در تمام نودها
CREATE TABLE IF NOT EXISTS default.test_table 
ON CLUSTER clickhouse_cluster
(
    id UInt32,
    name String,
    created_date Date
) ENGINE = MergeTree()
ORDER BY id;

-- درج داده
INSERT INTO default.test_table VALUES 
(1, 'Test 1', '2024-01-01'),
(2, 'Test 2', '2024-01-02');

-- خواندن داده
SELECT * FROM default.test_table;
```

### ایجاد جدول Replicated

```sql
-- ایجاد جدول replicated در تمام نودها
CREATE TABLE IF NOT EXISTS default.replicated_table 
ON CLUSTER clickhouse_cluster
(
    id UInt64,
    name String,
    created_at DateTime
) ENGINE = ReplicatedMergeTree('/clickhouse/tables/{shard}/replicated_table', '{replica}')
ORDER BY id;
```

### بررسی وضعیت کلاستر

```sql
-- لیست نودهای کلاستر
SELECT * FROM system.clusters WHERE cluster = 'clickhouse_cluster';

-- وضعیت replicas
SELECT * FROM system.replicas;

-- وضعیت Keeper
SELECT * FROM system.zookeeper WHERE path = '/clickhouse';

-- بررسی کاربران
SELECT name, auth_type, host_ip FROM system.users;

-- بررسی ماکروها
SELECT * FROM system.macros;
```

## 🔍 عیب‌یابی

### بررسی لاگ‌ها

```bash
# لاگ ClickHouse
docker-compose logs clickhouse-1
docker-compose logs clickhouse-2
docker-compose logs clickhouse-3

# لاگ Keeper
docker-compose logs keeper-1
docker-compose logs keeper-2
docker-compose logs keeper-3

# لاگ HAProxy
docker-compose logs clickhouse-haproxy
```

### بررسی وضعیت کاربران

```bash
# اتصال با کاربر admin
docker exec clickhouse-1 clickhouse-client --user admin --password password

# بررسی کاربران
SELECT name, auth_type, host_ip FROM system.users;
```

### بررسی وضعیت Keeper

```bash
# بررسی وضعیت Keeper
docker exec keeper-1 clickhouse-keeper-cli --host keeper-1 --port 9181 stat
```

### دسترسی به shell کانتینر

```bash
# ClickHouse
docker exec -it clickhouse-1 bash

# HAProxy
docker exec -it clickhouse-haproxy sh

# Keeper
docker exec -it keeper-1 bash
```

### متوقف کردن کلاستر

```bash
docker-compose down
```

### پاک کردن تمام داده‌ها و شروع مجدد

```bash
docker-compose down -v
docker-compose up -d
```

## ⚙️ تنظیمات کانفیگ

### clickhouse-config.xml
- **Network**: `listen_host: 0.0.0.0`, `http_port: 8123`, `tcp_port: 9000`
- **Zookeeper**: اتصال به 3 نود Keeper (keeper-1, keeper-2, keeper-3) روی پورت 9181
- **Distributed DDL**: مسیر `/clickhouse/task_queue/ddl`
- **MergeTree**: `max_suspicious_broken_parts: 5`
- **Logger**: سطح error، حداکثر 10 فایل 1GB

### clickhouse-nodes.xml
- **Cluster Name**: `clickhouse_cluster`
- **Structure**: 1 shard با 3 replica
- **Replicas**: clickhouse-1, clickhouse-2, clickhouse-3 (همه روی پورت 9000)

### haproxy.cfg
- **HTTP Frontend**: پورت 8124 (map شده به 8080)
- **Native Frontend**: پورت 9000 (map شده به 9004)
- **Stats Frontend**: پورت 8404
- **Load Balancing**: round-robin
- **Health Checks**: HTTP check برای `/ping` با interval 2s

### keeper-config
- **Raft Configuration**: 3 نود با server_id 1, 2, 3
- **Ports**: TCP 2181 (internal), Client 9234
- **Coordination Settings**: 
  - `operation_timeout_ms: 10000`
  - `session_timeout_ms: 30000`
  - `snapshot_distance: 1000`

## 🔐 تنظیمات امنیتی

⚠️ **توجه**: برای محیط production، حتماً:

1. **رمز عبور Super Admin را تغییر دهید**
   ```bash
   # تولید SHA256 هش جدید
   echo -n "your_new_password" | sha256sum
   
   # به‌روزرسانی در clickhouse-users.xml
   # سپس restart کانتینرها
   docker-compose restart
   ```

2. **رمز عبور کاربر default را تغییر دهید**
   - در حال حاضر پسورد `default` است که برای production مناسب نیست
   - می‌توانید از دستور بالا برای تولید hash جدید استفاده کنید

3. **دسترسی شبکه را محدود کنید**
   - در `clickhouse-users.xml` به جای `::/0` از IP‌های خاص استفاده کنید

4. **از SSL/TLS استفاده کنید**

5. **تنظیمات firewall را اعمال کنید**

## 📚 منابع

- [ClickHouse Documentation](https://clickhouse.com/docs)
- [ClickHouse Keeper](https://clickhouse.com/docs/en/guides/sre/keeper/clickhouse-keeper)
- [Docker Compose Documentation](https://docs.docker.com/compose/)
- [HAProxy Documentation](http://www.haproxy.org/#docs)

## 📋 خلاصه تغییرات

| فایل | تغییر | علت |
|------|-------|-----|
| `clickhouse-users.xml` | اضافه کردن پسورد برای کاربر `default` | امنیت - جلوگیری از دسترسی بدون پسورد |
| `clickhouse-users.xml` | حذف quota محدودیت‌ها | امکان اجرای کوئری‌ها |

## 📊 اطلاعات نسخه

- **ClickHouse Version**: 25.8.11.66
- **HAProxy Version**: 3.0-alpine
- **Docker Compose**: Version 3.8 (attribute obsolete)

---

**آخرین به‌روزرسانی:** 2025-11-16

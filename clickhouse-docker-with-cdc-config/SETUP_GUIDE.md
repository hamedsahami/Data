# راهنمای نصب و راه‌اندازی کلاستر ClickHouse

این راهنما به صورت گام‌به‌گام شما را در نصب و راه‌اندازی کلاستر ClickHouse راهنمایی می‌کند.

## 📋 پیش‌نیازها

قبل از شروع، مطمئن شوید که موارد زیر را دارید:

- ✅ **Docker** نسخه 20.10 یا بالاتر
- ✅ **Docker Compose** نسخه 1.29 یا بالاتر
- ✅ **حداقل 8GB RAM** (توصیه می‌شود 16GB)
- ✅ **حداقل 20GB فضای خالی** برای داده‌ها
- ✅ **دسترسی به اینترنت** برای دانلود image‌ها

### بررسی نصب Docker

```bash
# در Windows PowerShell یا Command Prompt
docker --version
docker-compose --version
```

اگر Docker نصب نیست، از [Docker Desktop](https://www.docker.com/products/docker-desktop) دانلود و نصب کنید.

---

## 🚀 مرحله 1: آماده‌سازی محیط

### 1.1 ایجاد پوشه پروژه

یک پوشه برای پروژه ایجاد کنید:

```bash
# در Windows
mkdir D:\clickhouse-cluster
cd D:\clickhouse-cluster
```

یا از هر مسیر دیگری که ترجیح می‌دهید استفاده کنید.

### 1.2 دانلود/کپی فایل‌های پروژه

تمام فایل‌های زیر را در پوشه پروژه قرار دهید:

```
clickhouse-cluster/
├── docker-compose.yml
├── haproxy.cfg
├── clickhouse-config.xml
├── clickhouse-nodes.xml
├── clickhouse-users.xml
├── macros.xml
├── macros-2.xml
├── macros-3.xml
├── keeper-config-1.xml
├── keeper-config-2.xml
└── keeper-config-3.xml
```

**نکته:** مطمئن شوید که همه فایل‌ها در یک پوشه هستند.

---

## 🔍 مرحله 2: بررسی فایل‌ها

قبل از راه‌اندازی، فایل‌های مهم را بررسی کنید:

### 2.1 بررسی docker-compose.yml

مطمئن شوید که فایل `docker-compose.yml` وجود دارد و شامل سرویس‌های زیر است:
- 3 نود ClickHouse (clickhouse-1, clickhouse-2, clickhouse-3)
- 3 نود Keeper (keeper-1, keeper-2, keeper-3)
- 1 نود HAProxy (clickhouse-haproxy)

### 2.2 بررسی فایل‌های کانفیگ

مطمئن شوید که فایل‌های زیر وجود دارند:
- ✅ `clickhouse-config.xml`
- ✅ `clickhouse-nodes.xml`
- ✅ `clickhouse-users.xml`
- ✅ `haproxy.cfg`
- ✅ `macros.xml`, `macros-2.xml`, `macros-3.xml`
- ✅ `keeper-config-1.xml`, `keeper-config-2.xml`, `keeper-config-3.xml`

---

## 🏃 مرحله 3: راه‌اندازی کلاستر

### 3.1 اجرای کلاستر

در پوشه پروژه، دستور زیر را اجرا کنید:

```bash
docker-compose up -d
```

این دستور:
- شبکه Docker را ایجاد می‌کند
- Volume‌ها را ایجاد می‌کند
- تمام کانتینرها را راه‌اندازی می‌کند

**زمان انتظار:** حدود 30-60 ثانیه برای دانلود image‌ها (در اولین بار)

### 3.2 بررسی وضعیت کانتینرها

پس از اجرا، وضعیت کانتینرها را بررسی کنید:

```bash
docker-compose ps
```

باید خروجی مشابه زیر را ببینید:

```
NAME                 STATUS          PORTS
clickhouse-1         Up X seconds    ...
clickhouse-2         Up X seconds    ...
clickhouse-3         Up X seconds    ...
clickhouse-haproxy   Up X seconds    ...
keeper-1             Up X seconds    ...
keeper-2             Up X seconds    ...
keeper-3             Up X seconds    ...
```

**⚠️ اگر کانتینری در حالت `Exited` است، به بخش عیب‌یابی مراجعه کنید.**

---

## ⏳ مرحله 4: صبر برای راه‌اندازی کامل

### 4.1 انتظار برای راه‌اندازی سرویس‌ها

کلاستر ClickHouse نیاز به زمان دارد تا کاملاً راه‌اندازی شود:

1. **Keeper نودها** (اولین): حدود 10-15 ثانیه
2. **ClickHouse نودها** (بعدی): حدود 20-30 ثانیه
3. **HAProxy** (آخرین): حدود 5 ثانیه

**توصیه:** 30-60 ثانیه صبر کنید تا همه سرویس‌ها آماده شوند.

### 4.2 بررسی لاگ‌ها

برای اطمینان از راه‌اندازی صحیح، لاگ‌ها را بررسی کنید:

```bash
# بررسی لاگ clickhouse-1
docker-compose logs clickhouse-1

# بررسی لاگ keeper-1
docker-compose logs keeper-1

# بررسی لاگ haproxy
docker-compose logs clickhouse-haproxy
```

**نکته:** اگر خطایی می‌بینید، به بخش عیب‌یابی مراجعه کنید.

---

## ✅ مرحله 5: بررسی و تست

### 5.1 تست اتصال به ClickHouse

با کاربر `default` و پسورد `default` تست کنید:

```bash
# تست اتصال مستقیم
docker exec clickhouse-1 clickhouse-client --user default --password default --query "SELECT currentUser(), version()"
```

**خروجی مورد انتظار:**
```
default    25.8.11.66
```

### 5.2 تست اتصال از طریق HAProxy

```bash
# در Windows PowerShell
Invoke-WebRequest -Uri "http://localhost:8080/?user=default&password=default&query=SELECT 1" -UseBasicParsing
```

**خروجی مورد انتظار:** `1`

### 5.3 بررسی وضعیت کلاستر

```bash
docker exec clickhouse-1 clickhouse-client --user admin --password password --query "SELECT * FROM system.clusters"
```

باید کلاستر `clickhouse_cluster` با 3 نود را ببینید.

### 5.4 بررسی وضعیت Keeper

```bash
docker exec clickhouse-1 clickhouse-client --user admin --password password --query "SELECT * FROM system.zookeeper WHERE path = '/clickhouse'"
```

---

## 🌐 مرحله 6: دسترسی به رابط وب

### 6.1 باز کردن رابط وب ClickHouse

مرورگر خود را باز کنید و به آدرس زیر بروید:

```
http://localhost:8080/play?user=default&password=default
```

**⚠️ مهم:** حتماً `user=default&password=default` را در URL اضافه کنید.

### 6.2 تست کوئری در رابط وب

در رابط وب، کوئری زیر را اجرا کنید:

```sql
SELECT currentUser(), version(), now()
```

باید نتیجه را ببینید.

---

## 📊 مرحله 7: بررسی آمار HAProxy

### 7.1 دسترسی به داشبورد HAProxy

مرورگر خود را باز کنید و به آدرس زیر بروید:

```
http://localhost:8404/stats
```

این داشبورد وضعیت نودها، ترافیک و health checks را نشان می‌دهد.

---

## 🎯 مرحله 8: تست کامل کلاستر

### 8.1 ایجاد جدول تست

```bash
docker exec clickhouse-1 clickhouse-client --user default --password default --query "
CREATE TABLE IF NOT EXISTS default.test_table 
ON CLUSTER clickhouse_cluster
(
    id UInt32,
    name String,
    created_date Date
) ENGINE = MergeTree()
ORDER BY id
"
```

### 8.2 درج داده

```bash
docker exec clickhouse-1 clickhouse-client --user default --password default --query "
INSERT INTO default.test_table VALUES 
(1, 'Test 1', '2024-01-01'),
(2, 'Test 2', '2024-01-02')
"
```

### 8.3 خواندن داده

```bash
docker exec clickhouse-1 clickhouse-client --user default --password default --query "
SELECT * FROM default.test_table
"
```

**خروجی مورد انتظار:**
```
1    Test 1    2024-01-01
2    Test 2    2024-01-02
```

---

## 🔧 عیب‌یابی

### مشکل 1: کانتینرها راه‌اندازی نمی‌شوند

**علت:** ممکن است پورت‌ها در حال استفاده باشند.

**راه حل:**
```bash
# بررسی پورت‌های استفاده شده
netstat -ano | findstr "8123 8124 8125 9000 9001 9002 8080"

# اگر پورتی در حال استفاده است، در docker-compose.yml تغییر دهید
```

### مشکل 2: خطای Authentication

**علت:** پسورد اشتباه یا کاربر وجود ندارد.

**راه حل:**
```bash
# بررسی کاربران
docker exec clickhouse-1 clickhouse-client --user admin --password password --query "SELECT name FROM system.users"

# اگر کاربر default وجود ندارد، کانتینرها را restart کنید
docker-compose restart
```

### مشکل 3: Keeper نودها راه‌اندازی نمی‌شوند

**علت:** مشکل در کانفیگ Keeper.

**راه حل:**
```bash
# بررسی لاگ Keeper
docker-compose logs keeper-1

# بررسی فایل‌های keeper-config
# مطمئن شوید که server_id ها متفاوت هستند (1, 2, 3)
```

### مشکل 4: HAProxy نمی‌تواند به ClickHouse متصل شود

**علت:** ClickHouse نودها هنوز راه‌اندازی نشده‌اند.

**راه حل:**
```bash
# صبر کنید تا ClickHouse نودها کاملاً راه‌اندازی شوند
docker-compose logs -f clickhouse-1

# سپس HAProxy را restart کنید
docker-compose restart clickhouse-haproxy
```

### مشکل 5: خطای "Connection refused"

**علت:** سرویس هنوز راه‌اندازی نشده است.

**راه حل:**
```bash
# بررسی وضعیت
docker-compose ps

# اگر کانتینری متوقف شده، لاگ آن را بررسی کنید
docker-compose logs [container-name]

# سپس restart کنید
docker-compose restart [container-name]
```

---

## 🛑 توقف کلاستر

برای توقف کلاستر:

```bash
docker-compose down
```

این دستور:
- تمام کانتینرها را متوقف می‌کند
- شبکه را حذف می‌کند
- **اما volume‌ها و داده‌ها را نگه می‌دارد**

---

## 🗑️ حذف کامل کلاستر

برای حذف کامل کلاستر و تمام داده‌ها:

```bash
docker-compose down -v
```

**⚠️ هشدار:** این دستور تمام داده‌ها را حذف می‌کند!

---

## 📝 دستورات مفید

### مشاهده لاگ‌ها

```bash
# لاگ همه سرویس‌ها
docker-compose logs -f

# لاگ یک سرویس خاص
docker-compose logs -f clickhouse-1

# آخرین 50 خط لاگ
docker-compose logs --tail 50 clickhouse-1
```

### Restart سرویس‌ها

```bash
# Restart همه سرویس‌ها
docker-compose restart

# Restart یک سرویس خاص
docker-compose restart clickhouse-1
```

### دسترسی به shell کانتینر

```bash
# ClickHouse
docker exec -it clickhouse-1 bash

# HAProxy
docker exec -it clickhouse-haproxy sh
```

### بررسی استفاده از منابع

```bash
docker stats
```

---

## ✅ چک‌لیست نهایی

پس از راه‌اندازی، موارد زیر را بررسی کنید:

- [ ] همه کانتینرها در حال اجرا هستند (`docker-compose ps`)
- [ ] می‌توانید با کاربر `default` و پسورد `default` متصل شوید
- [ ] رابط وب روی `http://localhost:8080/play` کار می‌کند
- [ ] می‌توانید جدول ایجاد کنید
- [ ] می‌توانید داده درج کنید
- [ ] می‌توانید داده بخوانید
- [ ] HAProxy stats روی `http://localhost:8404/stats` کار می‌کند
- [ ] کلاستر `clickhouse_cluster` در `system.clusters` وجود دارد

---

## 📞 اطلاعات اتصال

پس از راه‌اندازی موفق، می‌توانید از اطلاعات زیر استفاده کنید:

### رابط وب
- **URL**: `http://localhost:8080/play?user=default&password=default`
- **Username**: `default`
- **Password**: `default`

### ClickHouse Client
```bash
clickhouse-client --host localhost --port 9004 --user default --password default
```

### HTTP API
```
http://localhost:8080/?user=default&password=default&query=SELECT 1
```

### JDBC
```
URL: jdbc:clickhouse://localhost:9004/default
Username: default
Password: default
```

### Admin User
- **Username**: `admin`
- **Password**: `password`

---

## 🎉 تبریک!

اگر همه مراحل را با موفقیت انجام دادید، کلاستر ClickHouse شما آماده استفاده است!

برای اطلاعات بیشتر، به فایل `README.md` مراجعه کنید.

---

**آخرین به‌روزرسانی:** 2025-11-16


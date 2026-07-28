# Apache Airflow with Docker, MySQL, and Redis

This repository provides a Docker Compose environment for running **Apache Airflow** with:

- Apache Airflow 3.x
- MySQL 8 as the metadata database
- Redis as the Celery broker
- CeleryExecutor
- Persistent storage using Docker volumes and bind mounts

---

# Architecture

```text
                +----------------------+
                |   Airflow Webserver  |
                +----------+-----------+
                           |
                +----------v-----------+
                |   Airflow Scheduler  |
                +----------+-----------+
                           |
                +----------v-----------+
                |    Celery Workers    |
                +----------+-----------+
                           |
                    Redis Broker
                           |
                +----------v-----------+
                |       MySQL          |
                | Metadata Database    |
                +----------------------+
```

---

# Project Structure

```text
.
├── .env
├── docker-compose.yml
├── dags/
├── logs/
├── plugins/
├── config/
└── mysql/
    └── init/
```

| Directory | Description |
|-----------|-------------|
| `dags/` | Airflow DAG files |
| `logs/` | Airflow logs |
| `plugins/` | Custom Airflow plugins |
| `config/` | Airflow configuration files |
| `mysql/init/` | Optional MySQL initialization scripts |

---

# Prerequisites

- Docker Engine 24+
- Docker Compose v2+
- At least 4 GB RAM
- At least 2 CPU cores

Verify installation:

```bash
docker --version
docker compose version
```

> 💡 **نکته (فارسی)**
>
> قبل از اجرای پروژه مطمئن شوید Docker Engine و Docker Compose به درستی نصب شده‌اند.
>
> اگر Docker Daemon اجرا نشده باشد، هیچ‌کدام از سرویس‌ها اجرا نخواهند شد.

---

# Configuration

All configurable values are stored in the `.env` file.

Example:

```env
AIRFLOW_UID=50000

MYSQL_ROOT_PASSWORD=root123
MYSQL_DATABASE=airflow
MYSQL_USER=airflow
MYSQL_PASSWORD=airflow123

AIRFLOW_USER=admin
AIRFLOW_PASSWORD=admin
AIRFLOW_EMAIL=admin@example.com
```

Modify these values before deploying to production.

> 💡 **نکته (فارسی)**
>
> فایل `.env` محل نگهداری تنظیمات اصلی پروژه است.
>
> پیشنهاد می‌شود:
>
> - رمز عبور MySQL را تغییر دهید.
> - در محیط Production از رمزهای عبور قوی استفاده کنید.
> - فایل `.env` را در Git Commit نکنید.

---

# Create Required Directories

Before running the containers, create the required folders.

```bash
mkdir -p dags logs plugins config mysql/init
```

> 💡 **نکته (فارسی)**
>
> این پوشه‌ها به صورت Bind Mount به داخل کانتینر متصل می‌شوند؛ بنابراین هر تغییری در این پوشه‌ها مستقیماً در داخل کانتینر نیز اعمال خواهد شد.

---

# Initialize Airflow

Initialize the Airflow database and create the administrator account.

```bash
docker compose up airflow-init
```

This command only needs to be executed once.

> ⚠️ **نکته مهم (فارسی)**
>
> این دستور فقط در اولین اجرای پروژه لازم است.
>
> این سرویس:
>
> - دیتابیس Airflow را مقداردهی اولیه می‌کند.
> - Migrationها را اجرا می‌کند.
> - کاربر Administrator را ایجاد می‌کند.
>
> در اجرای‌های بعدی معمولاً نیازی به اجرای مجدد آن نیست مگر اینکه دیتابیس حذف شده باشد.

---

# Start the Environment

Run all services in detached mode.

```bash
docker compose up -d
```

Check running containers:

```bash
docker compose ps
```

> 💡 **نکته (فارسی)**
>
> اولین اجرای پروژه ممکن است چند دقیقه زمان ببرد زیرا Docker تصاویر مورد نیاز را دانلود می‌کند.
>
> اگر یکی از سرویس‌ها در وضعیت **Restarting** قرار گرفت، ابتدا لاگ همان سرویس را بررسی کنید.

---

# Stop the Environment

```bash
docker compose down
```

---

# Restart

```bash
docker compose restart
```

---

# View Logs

View logs for all services:

```bash
docker compose logs -f
```

Scheduler logs:

```bash
docker compose logs -f airflow-scheduler
```

Worker logs:

```bash
docker compose logs -f airflow-worker
```

Webserver logs:

```bash
docker compose logs -f airflow-webserver
```

MySQL logs:

```bash
docker compose logs -f mysql
```

---

# Access Airflow

Open your browser:

```
http://localhost:8080
```

Default credentials:

| Username | Password |
|----------|----------|
| admin | admin |

> 💡 **نکته (فارسی)**
>
> اگر صفحه Airflow باز نشد:
>
> - بررسی کنید پورت 8080 توسط برنامه دیگری اشغال نشده باشد.
> - وضعیت کانتینرها را بررسی کنید.
> - لاگ Webserver را مشاهده کنید:
>
> ```bash
> docker compose logs airflow-webserver
> ```

---

# Persistent Storage

The following data is persisted between container restarts.

| Resource | Persistence |
|----------|-------------|
| DAGs | `./dags` |
| Plugins | `./plugins` |
| Logs | `./logs` |
| Config | `./config` |
| MySQL Database | Docker Volume (`mysql-data`) |
| Redis Data | Docker Volume (`redis-data`) |

Deleting containers will **not** remove your data.

To remove all persistent data:

```bash
docker compose down -v
```

> 📌 **نکته مهم (فارسی)**
>
> فایل‌های DAG را کافی است داخل پوشه `dags/` قرار دهید.
>
> Airflow به صورت خودکار DAGهای جدید را شناسایی می‌کند و معمولاً نیازی به Restart کردن سرویس‌ها نیست.

> ⚠️ **هشدار**
>
> اجرای دستور زیر:
>
> ```bash
> docker compose down -v
> ```
>
> باعث حذف کامل اطلاعات Metadata، کاربران، Connectionها، Variableها و دیتابیس MySQL خواهد شد.

---

# Useful Commands

Update containers:

```bash
docker compose pull
docker compose up -d
```

List running containers:

```bash
docker compose ps
```

Open a shell inside the Webserver:

```bash
docker compose exec airflow-webserver bash
```

Run Airflow CLI:

```bash
docker compose exec airflow-webserver airflow version
```

List DAGs:

```bash
docker compose exec airflow-webserver airflow dags list
```

---

# Troubleshooting (راهنمای عیب‌یابی)

### مشاهده وضعیت سرویس‌ها

```bash
docker compose ps
```

---

### مشاهده همه لاگ‌ها

```bash
docker compose logs -f
```

---

### مشاهده لاگ Webserver

```bash
docker compose logs airflow-webserver
```

---

### مشاهده لاگ Scheduler

```bash
docker compose logs airflow-scheduler
```

---

### مشاهده لاگ Worker

```bash
docker compose logs airflow-worker
```

---

### مشاهده لاگ MySQL

```bash
docker compose logs mysql
```

---

### ورود به داخل کانتینر

```bash
docker compose exec airflow-webserver bash
```

---

### بررسی نسخه Airflow

```bash
airflow version
```

---

# Services

| Service | Port |
|----------|------|
| Airflow UI | 8080 |
| MySQL | 3306 |
| Redis | 6379 |

---

# Best Practices (نکات پیشنهادی)

- فایل‌های DAG را فقط داخل پوشه `dags/` قرار دهید.
- از Commit کردن فایل `.env` خودداری کنید.
- از دیتابیس MySQL به صورت منظم Backup تهیه کنید.
- برای نصب کتابخانه‌های Python از یک Docker Image سفارشی استفاده کنید.
- لاگ‌های قدیمی را به صورت دوره‌ای پاکسازی کنید.
- در محیط Production از رمزهای عبور پیش‌فرض استفاده نکنید.
- قبل از ارتقاء نسخه Airflow، از Metadata Database نسخه پشتیبان تهیه کنید.
- در محیط Production استفاده از Reverse Proxy مانند Nginx یا Traefik توصیه می‌شود.
- مقدار `FERNET_KEY` را در Production حتماً مقداردهی کنید.

---

# Reset Airflow

برای بازنشانی کامل محیط:

```bash
docker compose down -v
docker compose up airflow-init
docker compose up -d
```

> ⚠️ **توجه**
>
> این عملیات تمام اطلاعات Airflow را حذف کرده و محیط را از ابتدا ایجاد می‌کند.

---

# License

This project is provided as a starting point for Apache Airflow deployments using Docker Compose. Feel free to customize it according to your project requirements.
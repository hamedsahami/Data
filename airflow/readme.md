# Apache Airflow with Docker, MySQL, and Redis

This repository provides a Docker Compose environment for running **Apache Airflow** with:

- Apache Airflow 3.x
- MySQL 8 as the metadata database
- Redis as the Celery broker
- CeleryExecutor
- Persistent storage using Docker volumes and bind mounts

---

# Architecture

```
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

```
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

---

# Create Required Directories

Before running the containers, create the required folders.

```bash
mkdir -p dags logs plugins config mysql/init
```

---

# Initialize Airflow

Initialize the Airflow database and create the administrator account.

```bash
docker compose up airflow-init
```

This command only needs to be executed once.

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

> **Warning**
>
> This command permanently deletes the MySQL database and Redis data.

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

Open a shell inside the Airflow Webserver:

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

# Reset Airflow

Stop containers:

```bash
docker compose down -v
```

Remove local folders (optional):

```bash
rm -rf logs/*
```

Initialize again:

```bash
docker compose up airflow-init
docker compose up -d
```

---

# Services

| Service | Port |
|----------|------|
| Airflow UI | 8080 |
| MySQL | 3306 |
| Redis | 6379 |

---

# Production Recommendations

For production deployments consider:

- Using Docker Secrets instead of plain-text passwords
- Configuring HTTPS with a reverse proxy (Nginx or Traefik)
- Using a generated Fernet key instead of an empty one
- Regular MySQL backups
- Monitoring with Prometheus and Grafana
- Log aggregation (ELK, Loki, or OpenSearch)
- Building a custom Airflow image with required Python packages

---

# License

This project is provided as a starting point for Apache Airflow deployments using Docker Compose. Feel free to modify it to meet your project's requirements.
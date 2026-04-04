# Kafka Cluster with Zookeeper, KSQLDB and Kafka UI (Docker Compose)

This project sets up a **3-node Apache Kafka cluster** همراه با ابزارهای جانبی زیر با استفاده از Docker Compose:

* Zookeeper
* Kafka (3 Brokers)
* Kafka UI
* KSQLDB Server + CLI

---

## 🧱 Architecture Overview

```
                +-------------------+
                |     Kafka UI      |
                |    (Port 8080)    |
                +---------+---------+
                          |
        +-----------------+-----------------+
        |        Kafka Cluster (3 Nodes)    |
        |   kafka1   kafka2   kafka3        |
        +--------+--------+--------+--------+
                 |        |        |
                 +--------+--------+
                          |
                    +-----+------+
                    | Zookeeper  |
                    |  Port 2181 |
                    +------------+

                +-------------------+
                |   KSQLDB Server   |
                |    (Port 8088)    |
                +-------------------+
```

---

## 🚀 How to Run

```bash
docker-compose up -d
```

برای توقف:

```bash
docker-compose down
```

---

## 🔧 Services Breakdown

### 🟢 Zookeeper

```yaml
image: confluentinc/cp-zookeeper:6.0.0
```

**وظیفه:** مدیریت متادیتای Kafka Cluster

#### مهم‌ترین تنظیمات:

* `ZOOKEEPER_CLIENT_PORT=2181`
  پورتی که Kafka Brokerها برای اتصال به Zookeeper استفاده می‌کنن.

* `ZOOKEEPER_TICK_TIME=2000`
  زمان پایه برای heartbeat بین نودها (میلی‌ثانیه).

---

### 🔵 Kafka Brokers (kafka1, kafka2, kafka3)

هر ۳ نود Kafka تقریباً تنظیمات مشابه دارن و فقط در `BROKER_ID` و پورت متفاوت هستن.

#### 📌 تنظیمات کلیدی:

##### 1. Broker Identity

```yaml
KAFKA_BROKER_ID: 1 | 2 | 3
```

شناسه یکتا برای هر broker در کلاستر.

---

##### 2. اتصال به Zookeeper

```yaml
KAFKA_ZOOKEEPER_CONNECT: "zookeeper:2181"
```

Kafka برای مدیریت cluster به Zookeeper متصل می‌شود.

---

##### 3. Listener Configuration (خیلی مهم ⚠️)

```yaml
KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
```

تعریف پروتکل برای listenerها.

```yaml
KAFKA_ADVERTISED_LISTENERS:
  PLAINTEXT://kafka1:9092,
  PLAINTEXT_HOST://localhost:29092
```

این بخش تعیین می‌کند که:

* داخل Docker: از `kafka1:9092` استفاده شود
* خارج از Docker (لوکال): از `localhost:29092`

📌 اگر این اشتباه تنظیم شود → کلاینت‌ها نمی‌توانند به Kafka وصل شوند.

---

##### 4. Replication Settings (خیلی مهم برای Production ⚠️)

```yaml
KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 3
```

Replication برای topicهای داخلی Kafka (مثل consumer offsets)

---

```yaml
KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 3
```

برای transactional producerها

---

```yaml
KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 2
```

حداقل replica لازم برای commit شدن transaction

---

```yaml
KAFKA_MIN_INSYNC_REPLICAS: 2
```

حداقل replica در sync برای accept کردن write

📌 این یعنی:

* اگر کمتر از 2 broker در sync باشند → write fail می‌شود
* باعث افزایش durability می‌شود

---

### 🌐 Kafka UI

```yaml
image: provectuslabs/kafka-ui:latest
```

**آدرس دسترسی:**

```
http://localhost:8080
```

#### تنظیمات:

```yaml
KAFKA_CLUSTERS_0_NAME: local
```

نام cluster در UI

```yaml
KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS:
  kafka1:9092,kafka2:9092,kafka3:9092
```

لیست brokerها برای اتصال UI به Kafka

---

### ⚡ KSQLDB Server

```yaml
image: confluentinc/cp-ksqldb-server:latest
```

**آدرس:**

```
http://localhost:8088
```

#### تنظیمات مهم:

```yaml
KSQL_BOOTSTRAP_SERVERS:
  kafka1:9092,kafka2:9092,kafka3:9092
```

اتصال به Kafka cluster

---

```yaml
KSQL_LISTENERS: http://0.0.0.0:8088
```

باز کردن API روی همه اینترفیس‌ها

---

```yaml
KSQL_CACHE_MAX_BYTES_BUFFERING: 0
```

غیرفعال کردن cache برای:

* کاهش latency
* مناسب برای development

---

### 💻 KSQLDB CLI

```yaml
image: confluentinc/cp-ksqldb-cli:latest
```

برای اجرای interactive query

#### اتصال به CLI:

```bash
docker exec -it ksqldb-cli ksql http://ksqldb-server:8088
```

---

## 🔌 Ports Summary

| Service       | Port  |
| ------------- | ----- |
| Zookeeper     | 2181  |
| Kafka1        | 29092 |
| Kafka2        | 29093 |
| Kafka3        | 29094 |
| Kafka UI      | 8080  |
| KSQLDB Server | 8088  |

---

## 🧪 Example: Connect to Kafka

### از داخل Docker:

```bash
kafka-console-producer --broker-list kafka1:9092 --topic test
```

### از خارج Docker:

```bash
kafka-console-producer --broker-list localhost:29092 --topic test
```

---

## ⚠️ Important Notes

* این setup برای **development و testing** مناسب است، نه production کامل
* در production:

  * از **KRaft mode (بدون Zookeeper)** استفاده کنید
  * SSL / SASL فعال کنید
  * مانیتورینگ (Prometheus + Grafana) اضافه کنید
* حتماً `replication factor` را با تعداد brokerها هماهنگ نگه دارید

---

## 📌 Tips

* اگر connection مشکل داشت:

  * اول `advertised.listeners` را چک کن
* اگر topic ساخته نمی‌شود:

  * replication factor > تعداد brokerها نباشد
* اگر latency زیاد بود:

  * cache در ksqldb را بررسی کن

---

## 📄 License

MIT

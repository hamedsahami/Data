# Kafka Cluster with KSQLDB and Kafka UI (Docker Compose)

This project sets up a **3-node Apache Kafka cluster** همراه با ابزارهای زیر:

* Zookeeper
* Kafka (3 Brokers)
* Kafka UI
* KSQLDB Server + CLI

📌 این نسخه **Kafka Connect ندارد** و برای سناریوهای streaming ساده‌تر مناسب است.

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

Stop:

```bash
docker-compose down
```

---

## 🔧 Services Breakdown

---

### 🟢 Zookeeper

```yaml
image: confluentinc/cp-zookeeper:6.0.0
```

#### تنظیمات مهم:

```yaml
ZOOKEEPER_CLIENT_PORT: 2181
```

پورتی که Kafka به آن متصل می‌شود.

```yaml
ZOOKEEPER_TICK_TIME: 2000
```

interval پایه برای heartbeat.

---

### 🔵 Kafka Brokers (kafka1, kafka2, kafka3)

هر broker دارای تنظیمات مشابه است.

---

### 📌 تنظیمات کلیدی Kafka

#### 1. Broker ID

```yaml
KAFKA_BROKER_ID: 1 | 2 | 3
```

شناسه یکتا برای هر نود.

---

#### 2. اتصال به Zookeeper

```yaml
KAFKA_ZOOKEEPER_CONNECT: "zookeeper:2181"
```

---

#### 3. Listener Configuration ⚠️ (خیلی مهم)

```yaml
KAFKA_LISTENER_SECURITY_PROTOCOL_MAP:
  PLAINTEXT:PLAINTEXT,
  PLAINTEXT_HOST:PLAINTEXT
```

```yaml
KAFKA_ADVERTISED_LISTENERS:
  PLAINTEXT://kafka1:9092,
  PLAINTEXT_HOST://localhost:29092
```

📌 توضیح:

| نوع اتصال   | آدرس            |
| ----------- | --------------- |
| داخل Docker | kafka1:9092     |
| خارج Docker | localhost:29092 |

اگر این اشتباه باشد → clientها وصل نمی‌شوند.

---

#### 4. Replication Settings

```yaml
KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 3
```

برای internal topicها

---

```yaml
KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 3
```

برای transactional data

---

```yaml
KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 2
KAFKA_MIN_INSYNC_REPLICAS: 2
```

📌 معنی:

* حداقل 2 replica باید sync باشند
* در غیر این صورت write fail می‌شود
* افزایش durability

---

### 🌐 Kafka UI

```yaml
image: provectuslabs/kafka-ui:latest
```

📍 آدرس:

```
http://localhost:8080
```

---

#### تنظیمات:

```yaml
KAFKA_CLUSTERS_0_NAME: local
```

نام cluster

```yaml
KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS:
  kafka1:9092,kafka2:9092,kafka3:9092
```

اتصال به brokerها

---

### ⚡ KSQLDB Server

```yaml
image: confluentinc/cp-ksqldb-server:latest
```

📍 آدرس:

```
http://localhost:8088
```

---

#### تنظیمات:

```yaml
KSQL_BOOTSTRAP_SERVERS:
  kafka1:9092,kafka2:9092,kafka3:9092
```

اتصال به Kafka

---

```yaml
KSQL_LISTENERS: http://0.0.0.0:8088
```

باز کردن API روی همه اینترفیس‌ها

---

```yaml
KSQL_CACHE_MAX_BYTES_BUFFERING: 0
```

📌 نتیجه:

* کاهش latency
* مناسب برای development
* بدون buffering

---

### 💻 KSQLDB CLI

```yaml
image: confluentinc/cp-ksqldb-cli:latest
```

برای اجرای query:

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

## 🧪 Example Usage

### Create Topic

```bash
kafka-topics --create \
  --bootstrap-server localhost:29092 \
  --replication-factor 3 \
  --partitions 1 \
  --topic test
```

---

### Produce Message

```bash
kafka-console-producer \
  --broker-list localhost:29092 \
  --topic test
```

---

### Consume Message

```bash
kafka-console-consumer \
  --bootstrap-server localhost:29092 \
  --topic test \
  --from-beginning
```

---

## ⚠️ Important Notes

### 1. بدون Kafka Connect

در این setup:

❌ نمی‌توان:

* connector ساخت
* به DB وصل شد (MySQL, Postgres, etc.)

✅ فقط:

* streaming داخل Kafka
* پردازش با KSQLDB

---

### 2. Replication Alignment

چون:

* 3 broker داریم
* replication = 3

📌 کاملاً High Availability داریم

---

### 3. Common Issues

#### مشکل اتصال

→ بررسی کن:

```
advertised.listeners
```

#### مشکل write

→ بررسی کن:

```
min.insync.replicas
```

---

## 📌 When to Use This Setup

این setup مناسب است برای:

* یادگیری Kafka
* تست KSQLDB
* stream processing ساده
* development

---

## 🚀 Next Step

برای سناریوهای واقعی‌تر:

* اضافه کردن Kafka Connect
* اضافه کردن Schema Registry
* استفاده از Avro / Protobuf
* مهاجرت به KRaft

---

## 📄 License

MIT

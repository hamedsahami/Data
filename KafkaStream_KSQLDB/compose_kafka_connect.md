# Kafka Cluster with Kafka Connect, KSQLDB and Kafka UI (Docker Compose)

This project provides a **production-like Kafka ecosystem** using Docker Compose, including:

* Zookeeper
* Kafka Cluster (3 Brokers)
* Kafka Connect (Distributed Mode)
* Kafka UI (with Connect integration)
* KSQLDB Server + CLI

---

## 🧱 Architecture Overview

```
                    +----------------------+
                    |      Kafka UI        |
                    |      (Port 8080)     |
                    +----------+-----------+
                               |
        +----------------------+----------------------+
        |           Kafka Cluster (3 Brokers)         |
        |     kafka1        kafka2        kafka3      |
        +--------+------------+------------+----------+
                 |            |            |
                 +------------+------------+
                              |
                        +-----+------+
                        |  Zookeeper  |
                        +------------+

                +-------------------------+
                |     Kafka Connect       |
                |      (Port 8083)        |
                +-----------+-------------+
                            |
                    +-------+--------+
                    |    KSQLDB      |
                    |   (Port 8088)  |
                    +----------------+
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

بدون تغییر نسبت به نسخه قبل.

#### نکات:

* مدیریت cluster metadata
* Kafka بدون آن (در این setup) کار نمی‌کند

---

### 🔵 Kafka Brokers (3 Nodes)

علاوه بر تنظیمات قبلی، اینجا یک تغییر مهم داریم:

```yaml
KAFKA_DEFAULT_REPLICATION_FACTOR: 3
```

#### 📌 اهمیت این تنظیم:

* مقدار پیش‌فرض replication برای topicهای جدید
* اگر هنگام create topic مقدار ندی → این استفاده می‌شود
* با 3 broker → کاملاً مناسب (High Availability)

---

### 🔌 Kafka Connect (Distributed Mode)

```yaml
image: confluentinc/cp-kafka-connect:6.0.0
```

**آدرس API:**

```
http://localhost:8083
```

---

### 📌 تنظیمات بسیار مهم Kafka Connect:

#### 1. اتصال به Kafka

```yaml
CONNECT_BOOTSTRAP_SERVERS:
  kafka1:9092,kafka2:9092,kafka3:9092
```

---

#### 2. Cluster Mode

```yaml
CONNECT_GROUP_ID: "connect-cluster"
```

* تعیین‌کننده cluster برای connect workers
* در حالت distributed → چند worker می‌توانند join شوند

---

#### 3. Internal Topics (خیلی مهم ⚠️)

```yaml
CONNECT_CONFIG_STORAGE_TOPIC: "connect-configs"
CONNECT_OFFSET_STORAGE_TOPIC: "connect-offsets"
CONNECT_STATUS_STORAGE_TOPIC: "connect-status"
```

این topicها برای موارد زیر هستند:

| Topic   | کاربرد                    |
| ------- | ------------------------- |
| config  | نگهداری تنظیمات connector |
| offsets | ذخیره offsetها            |
| status  | وضعیت connector           |

---

#### 4. Replication برای Connect

```yaml
CONNECT_*_REPLICATION_FACTOR: 3
```

📌 تضمین:

* Fault tolerance
* عدم از دست رفتن state

---

#### 5. Converterها

```yaml
CONNECT_KEY_CONVERTER: JsonConverter
CONNECT_VALUE_CONVERTER: JsonConverter
```

* تبدیل داده بین Kafka و Connector
* JSON → مناسب برای development
* در production معمولاً Avro + Schema Registry

---

#### 6. REST Config

```yaml
CONNECT_REST_PORT: 8083
CONNECT_REST_ADVERTISED_HOST_NAME: "kafka-connect"
```

برای دسترسی سایر سرویس‌ها (مثل KSQLDB)

---

### 🌐 Kafka UI (with Connect Integration)

```yaml
KAFKA_CLUSTERS_0_KAFKACONNECT_0_ADDRESS:
  http://kafka-connect:8083
```

📌 قابلیت‌های جدید:

* مشاهده Connectorها
* ایجاد / حذف Connector
* بررسی status

---

### ⚡ KSQLDB Server (Enhanced)

```yaml
KSQL_KSQL_CONNECT_URL: http://kafka-connect:8083
```

#### 📌 این تنظیم چی کار می‌کند؟

* اتصال KSQLDB به Kafka Connect
* امکان:

  * ساخت connector از داخل KSQL
  * اجرای sink/source مستقیم

---

```yaml
KSQL_KSQL_STREAMS_REPLICATION_FACTOR: 3
```

* replication برای state storeهای KSQLDB
* افزایش resilience

---

### 💻 KSQLDB CLI

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
| Kafka Connect | 8083  |
| Kafka UI      | 8080  |
| KSQLDB Server | 8088  |

---

## 🧪 Example: Create Connector

```bash
curl -X POST http://localhost:8083/connectors \
  -H "Content-Type: application/json" \
  -d '{
    "name": "test-connector",
    "config": {
      "connector.class": "FileStreamSource",
      "tasks.max": "1",
      "file": "/tmp/test.txt",
      "topic": "test-topic"
    }
  }'
```

---

## ⚠️ Important Notes

### 1. Replication Consistency

تمام این‌ها باید هماهنگ باشند:

* Kafka brokers: 3
* replication factor: 3
* min ISR: 2

---

### 2. Internal Topics (خیلی مهم)

Kafka Connect بدون این topicها کار نمی‌کند:

* connect-configs
* connect-offsets
* connect-status

اگر auto-create خاموش باشد → باید دستی بسازی

---

### 3. Listener Misconfiguration

بزرگ‌ترین دلیل مشکل اتصال:

* `advertised.listeners` اشتباه

---

### 4. Development vs Production

این setup:

✅ مناسب:

* تست
* یادگیری
* development

❌ مناسب نیست:

* production واقعی بدون تغییرات

---

## 📌 Production Improvements

برای production پیشنهاد می‌شود:

* استفاده از **KRaft (بدون Zookeeper)**
* فعال‌سازی:

  * SSL
  * SASL Auth
* اضافه کردن:

  * Schema Registry
  * Monitoring (Prometheus + Grafana)
* استفاده از Avro/Protobuf به جای JSON

---

## 📄 License

MIT

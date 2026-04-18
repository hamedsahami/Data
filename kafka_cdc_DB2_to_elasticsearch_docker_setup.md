# Kafka CDC Pipeline: DB2 → Debezium → Kafka → Elasticsearch

This guide provides a complete Docker Compose setup to practice a real-time CDC pipeline using:
- IBM DB2 (source database)
- Debezium (CDC connector for DB2)
- Kafka + Zookeeper
- Kafka Connect
- Elasticsearch
- Kibana
- Kafka UI
- Debezium UI

---

# 🧱 Architecture

DB2 → Debezium → Kafka → Kafka Connect Sink → Elasticsearch

---

# 📦 docker-compose.yml

```yaml
version: '3.8'

services:
  zookeeper:
    image: confluentinc/cp-zookeeper:7.5.0
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181

  kafka:
    image: confluentinc/cp-kafka:7.5.0
    depends_on:
      - zookeeper
    ports:
      - "9092:9092"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1

  db2:
    image: ibmcom/db2:11.5.8.0
    privileged: true
    ports:
      - "50000:50000"
    environment:
      LICENSE: accept
      DB2INST1_PASSWORD: password
      DBNAME: testdb
    volumes:
      - db2_data:/database

  connect:
    image: debezium/connect:2.5
    ports:
      - "8083:8083"
    depends_on:
      - kafka
      - db2
    environment:
      BOOTSTRAP_SERVERS: kafka:9092
      GROUP_ID: 1
      CONFIG_STORAGE_TOPIC: connect_configs
      OFFSET_STORAGE_TOPIC: connect_offsets
      STATUS_STORAGE_TOPIC: connect_statuses

  debezium-ui:
    image: debezium/debezium-ui:latest
    ports:
      - "8080:8080"
    environment:
      KAFKA_CONNECT_URIS: http://connect:8083
    depends_on:
      - connect

  kafka-ui:
    image: provectuslabs/kafka-ui:latest
    ports:
      - "8085:8080"
    environment:
      KAFKA_CLUSTERS_0_NAME: local
      KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS: kafka:9092
      KAFKA_CLUSTERS_0_ZOOKEEPER: zookeeper:2181

  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.12.0
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
    ports:
      - "9200:9200"

  kibana:
    image: docker.elastic.co/kibana/kibana:8.12.0
    depends_on:
      - elasticsearch
    ports:
      - "5601:5601"

volumes:
  db2_data:
```

---

# 🌐 UIs Available

- Kafka UI: http://localhost:8085
- Debezium UI: http://localhost:8080
- Kibana: http://localhost:5601

---

# ▶️ Start the stack

```bash
docker-compose up -d
```

---

# 🧩 Step 1: Create DB2 Table

Connect to DB2 container:

```bash
docker exec -it <db2_container> bash
su - db2inst1
db2 connect to testdb
```

Create table:

```sql
CREATE TABLE users (
  id INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  name VARCHAR(255),
  email VARCHAR(255)
);

INSERT INTO users (name, email) VALUES ('Ali', 'ali@test.com');
```

---

# 🔌 Step 2: Register Debezium DB2 Connector

```bash
curl -X POST http://localhost:8083/connectors \
-H "Content-Type: application/json" \
-d '{
  "name": "db2-connector",
  "config": {
    "connector.class": "io.debezium.connector.db2.Db2Connector",
    "database.hostname": "db2",
    "database.port": "50000",
    "database.user": "db2inst1",
    "database.password": "password",
    "database.dbname": "testdb",
    "database.server.name": "db2server1",
    "table.include.list": "DB2INST1.USERS",
    "database.history.kafka.bootstrap.servers": "kafka:9092",
    "database.history.kafka.topic": "schema-changes.db2"
  }
}'
```

---

# 🔍 Step 3: Verify Kafka Topic

```bash
docker exec -it <kafka_container> kafka-topics --bootstrap-server localhost:9092 --list
```

Expected:

```
db2server1.DB2INST1.USERS
```

---

# 📤 Step 4: Add Elasticsearch Sink Connector

```bash
curl -X POST http://localhost:8083/connectors \
-H "Content-Type: application/json" \
-d '{
  "name": "es-sink",
  "config": {
    "connector.class": "io.confluent.connect.elasticsearch.ElasticsearchSinkConnector",
    "topics": "db2server1.DB2INST1.USERS",
    "connection.url": "http://elasticsearch:9200",
    "key.ignore": "true",
    "schema.ignore": "true"
  }
}'
```

---

# 🧪 Step 5: Test the Pipeline

```sql
INSERT INTO users (name, email) VALUES ('Sara', 'sara@test.com');
```

---

# 🔎 Step 6: Verify in Elasticsearch

```bash
curl http://localhost:9200/db2server1.DB2INST1.USERS/_search?pretty
```

---

# 📊 Kibana

Open http://localhost:5601

Index pattern:

```
db2server1.DB2INST1.USERS
```

---

# ⚠️ Notes

- DB2 setup may take a few minutes on first startup
- Debezium DB2 uses transaction logs (no polling)
- Schema names are uppercase by default in DB2

---

# ✅ Summary

You now have a CDC pipeline:

DB2 → Kafka → Elasticsearch (real-time)

---


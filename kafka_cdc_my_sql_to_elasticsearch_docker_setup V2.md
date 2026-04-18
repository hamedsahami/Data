# Kafka CDC Pipeline: MySQL → Debezium → Kafka → Elasticsearch

This guide provides a complete Docker Compose setup to practice a real-time CDC pipeline using:
- MySQL (source database)
- Debezium (CDC connector)
- Kafka + Zookeeper
- Kafka Connect
- Elasticsearch
- Kibana (optional UI)

---

# 🧱 Architecture

MySQL → Debezium → Kafka → Kafka Connect Sink → Elasticsearch

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

  mysql:
    image: mysql:8
    ports:
      - "3306:3306"
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: testdb
    command: --server-id=1 --log-bin=mysql-bin --binlog-format=row

  connect:
    image: debezium/connect:2.5
    ports:
      - "8083:8083"
    depends_on:
      - kafka
      - mysql
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
```

---

# 🌐 UIs Available

After starting the stack, you can access:

- Kafka UI: http://localhost:8085
- Debezium UI: http://localhost:8080
- Kibana: http://localhost:5601

---

# ▶️ Start the stack

```bash
docker-compose up -d
```

Check services:

```bash
docker ps
```

---

# 🧩 Step 1: Create MySQL Table

```sql
CREATE TABLE testdb.users (
  id INT PRIMARY KEY AUTO_INCREMENT,
  name VARCHAR(255),
  email VARCHAR(255)
);

INSERT INTO testdb.users (name, email) VALUES ('Ali', 'ali@test.com');
```

---

# 🔌 Step 2: Register Debezium Connector

```bash
curl -X POST http://localhost:8083/connectors \
-H "Content-Type: application/json" \
-d '{
  "name": "mysql-connector",
  "config": {
    "connector.class": "io.debezium.connector.mysql.MySqlConnector",
    "database.hostname": "mysql",
    "database.port": "3306",
    "database.user": "root",
    "database.password": "root",
    "database.server.id": "1",
    "database.server.name": "dbserver1",
    "database.include.list": "testdb",
    "table.include.list": "testdb.users",
    "database.history.kafka.bootstrap.servers": "kafka:9092",
    "database.history.kafka.topic": "schema-changes.testdb"
  }
}'
```

---

# 🔍 Step 3: Verify Kafka Topic

List topics:

```bash
docker exec -it <kafka_container> kafka-topics --bootstrap-server localhost:9092 --list
```

You should see:

```
dbserver1.testdb.users
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
    "topics": "dbserver1.testdb.users",
    "connection.url": "http://elasticsearch:9200",
    "key.ignore": "true",
    "schema.ignore": "true"
  }
}'
```

---

# 🧪 Step 5: Test the Pipeline

Insert new data:

```sql
INSERT INTO testdb.users (name, email) VALUES ('Sara', 'sara@test.com');
```

---

# 🔎 Step 6: Verify in Elasticsearch

```bash
curl http://localhost:9200/dbserver1.testdb.users/_search?pretty
```

You should see indexed documents.

---

# 📊 Optional: Kibana

Open:

http://localhost:5601

Create index pattern:

```
dbserver1.testdb.users
```

---

# ⚠️ Notes

- First run includes snapshot of existing data
- Updates & deletes are also captured
- Ensure MySQL binlog is enabled (already configured)

---

# 🚀 Next Improvements

- Add Schema Registry (Avro)
- Add Kafka UI (e.g., AKHQ)
- Tune Elasticsearch mappings

---

# ✅ Summary

You now have a full CDC pipeline running locally with Docker:

MySQL → Kafka → Elasticsearch (real-time)

---


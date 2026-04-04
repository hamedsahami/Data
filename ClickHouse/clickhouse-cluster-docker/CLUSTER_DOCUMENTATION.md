# ClickHouse Cluster Documentation

## 🏗️ Architecture Overview

This ClickHouse cluster consists of:
- **3 ClickHouse Nodes** (clickhouse-1, clickhouse-2, clickhouse-3)
- **3 ClickHouse Keeper Nodes** (keeper-1, keeper-2, keeper-3)
- **1 HAProxy Load Balancer** (haproxy)

## 📊 Cluster Configuration

### Cluster Definition
```xml
<clickhouse_cluster>
    <shard>
        <replica>
            <host>clickhouse-1</host>
            <port>9000</port>
        </replica>
        <replica>
            <host>clickhouse-2</host>
            <port>9000</port>
        </replica>
        <replica>
            <host>clickhouse-3</host>
            <port>9000</port>
        </replica>
    </shard>
</clickhouse_cluster>
```

### Macros Configuration
Each node has specific macros defined:

**clickhouse-1:**
```xml
<macros>
    <shard>1</shard>
    <replica>clickhouse-1</replica>
</macros>
```

**clickhouse-2:**
```xml
<macros>
    <shard>1</shard>
    <replica>clickhouse-2</replica>
</macros>
```

**clickhouse-3:**
```xml
<macros>
    <shard>1</shard>
    <replica>clickhouse-3</replica>
</macros>
```

## 🌐 Network Configuration

### Ports
- **HAProxy HTTP**: `80` → ClickHouse HTTP (8123)
- **HAProxy Native**: `9004` → ClickHouse Native (9000)
- **HAProxy Stats**: `8404` → HAProxy Statistics
- **Direct ClickHouse**: `8123`, `8124`, `8125` (HTTP) and `9000`, `9001`, `9002` (Native)

### Host Mappings
```yaml
extra_hosts:
  - "keeper-1:172.18.0.3"
  - "keeper-2:172.18.0.2"
  - "keeper-3:172.18.0.4"
  - "clickhouse-2:172.18.0.6"
  - "clickhouse-3:172.18.0.7"
```

## 👥 User Management

### Default Users
- **admin**: Full access (password: `password`)
- **readonly**: Read-only access (password: `readonly`)
- **default**: Basic user (password: `default`)

### User Creation Scripts
- `create-user.sh` (Linux/Mac)
- `create-user.ps1` (Windows PowerShell)

## 📋 Database Schemas

### Available Databases
- `default` - Default database
- `my_database` - Custom database for testing
- `test_db` - Test database
- `system` - System database
- `INFORMATION_SCHEMA` - Information schema

### Sample Tables

#### 1. Simple MergeTree Table
```sql
CREATE TABLE default.test_table_final 
ON CLUSTER clickhouse_cluster
(
    id UInt32,
    name String,
    created_date Date
) ENGINE = MergeTree()
ORDER BY id;
```

#### 2. ReplicatedMergeTree Table
```sql
CREATE TABLE my_database.Test
ON CLUSTER clickhouse_cluster
(
    id UInt64,
    name String,
    email String,
    created_at DateTime
)
ENGINE = ReplicatedMergeTree('/clickhouse/tables/{shard}/Test', '{replica}')
ORDER BY id;
```

#### 3. Customers Table (Replicated)
```sql
CREATE TABLE my_database.customers
ON CLUSTER clickhouse_cluster
(
    id UInt64,
    name String,
    email String,
    created_at DateTime
)
ENGINE = ReplicatedMergeTree('/clickhouse/tables/{shard}/customers', '{replica}')
ORDER BY id;
```

## 🔧 Connection Information

### HTTP Connection
```
URL: http://localhost/
Username: admin
Password: password
```

### JDBC Connection
```
URL: jdbc:clickhouse://localhost:9004/default
Username: admin
Password: password
```

### Native Protocol
```
Host: localhost
Port: 9004
Username: admin
Password: password
```

## 📝 Common SQL Commands

### Database Operations
```sql
-- Create database
CREATE DATABASE my_database;

-- Show databases
SHOW DATABASES;

-- Use database
USE my_database;
```

### Table Operations
```sql
-- Create table with ON CLUSTER
CREATE TABLE my_table ON CLUSTER clickhouse_cluster
(
    id UInt64,
    name String
) ENGINE = MergeTree()
ORDER BY id;

-- Create replicated table
CREATE TABLE my_replicated_table ON CLUSTER clickhouse_cluster
(
    id UInt64,
    name String
) ENGINE = ReplicatedMergeTree('/clickhouse/tables/{shard}/my_table', '{replica}')
ORDER BY id;

-- Insert data
INSERT INTO my_table VALUES (1, 'Test');

-- Select data
SELECT * FROM my_table;

-- Drop table
DROP TABLE IF EXISTS my_table ON CLUSTER clickhouse_cluster;
```

### System Queries
```sql
-- Check cluster status
SELECT * FROM system.clusters;

-- Check macros
SELECT * FROM system.macros;

-- Check tables
SELECT * FROM system.tables;

-- Check replicas
SELECT * FROM system.replicas;
```

## 🚀 Management Scripts

### Docker Commands
```bash
# Start cluster
docker-compose up -d

# Stop cluster
docker-compose down

# Restart specific service
docker-compose restart clickhouse-1

# View logs
docker logs clickhouse-1

# Execute SQL
docker exec clickhouse-1 clickhouse-client --query "SELECT 1"
```

### User Management
```bash
# Create user (Linux/Mac)
./create-user.sh

# Create user (Windows)
./create-user.ps1
```

## 📊 Monitoring

### HAProxy Statistics
- URL: http://localhost:8404/stats
- Username: admin
- Password: admin

### ClickHouse System Tables
```sql
-- Query statistics
SELECT * FROM system.query_log;

-- Server metrics
SELECT * FROM system.metrics;

-- Memory usage
SELECT * FROM system.processes;
```

## 🔍 Troubleshooting

### Common Issues

1. **Connection Refused**
   - Check if containers are running: `docker ps`
   - Verify port mappings in docker-compose.yml

2. **Macro Not Found**
   - Ensure macros.xml files are properly mounted
   - Restart ClickHouse containers

3. **Replica Already Exists**
   - Drop existing table first
   - Use different path in ReplicatedMergeTree

4. **Database Not Found**
   - Create database on all nodes manually
   - Or use `ON CLUSTER` for database creation

### Health Checks
```sql
-- Test connection
SELECT 'Connection OK' AS status;

-- Check cluster health
SELECT * FROM system.clusters WHERE cluster = 'clickhouse_cluster';

-- Verify replication
SELECT * FROM system.replicas;
```

## 📁 File Structure

```
clickhouseazar/
├── docker-compose.yml          # Main Docker Compose configuration
├── clickhouse-config.xml       # ClickHouse server configuration
├── clickhouse-users.xml        # User definitions
├── clickhouse-nodes.xml        # Cluster node definitions
├── keeper-config-1.xml         # Keeper node 1 configuration
├── keeper-config-2.xml         # Keeper node 2 configuration
├── keeper-config-3.xml         # Keeper node 3 configuration
├── macros.xml                   # Macros for clickhouse-1
├── macros-2.xml                # Macros for clickhouse-2
├── macros-3.xml                # Macros for clickhouse-3
├── haproxy.cfg                 # HAProxy configuration
├── create-user.sh              # User creation script (Linux/Mac)
├── create-user.ps1             # User creation script (Windows)
├── manage-users.sql            # SQL user management commands
└── README.md                   # Basic documentation
```

## 🎯 Best Practices

1. **Always use `ON CLUSTER`** for DDL operations
2. **Use ReplicatedMergeTree** for data replication
3. **Monitor HAProxy statistics** for load balancing
4. **Backup configurations** before changes
5. **Test queries** on single nodes before cluster operations

## 📞 Support

For issues or questions:
1. Check container logs: `docker logs <container_name>`
2. Verify configuration files
3. Test individual components
4. Check network connectivity

---

**Cluster Status**: ✅ **OPERATIONAL**
**Last Updated**: 2025-10-28
**Version**: ClickHouse 25.9.3.48

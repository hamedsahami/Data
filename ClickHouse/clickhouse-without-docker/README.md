# راهنمای نصب ClickHouse به صورت کلاستر

این راهنما نحوه نصب ClickHouse به صورت کلاستر **بدون استفاده از داکر** با ClickHouse Keeper روی **سه نود**، **یک شارد** و **دو رپلیکا** را شرح می‌دهد.

---

## نصب ClickHouse روی هر سه نود

روی هر سرور (با IPهای 192.168.42.128، 192.168.42.129، 192.168.42.130) آخرین نسخه LTS را نصب کنید. راهنمای نصب در [مستندات رسمی](https://clickhouse.com/docs/install) موجود است.

---

طبق مستندات رسمی ClickHouse، برای سناریوی شما، تنظیمات هر نود به صورت زیر است:

## نود ۱ (192.168.42.130)

### تنظیم ClickHouse Server (در فایل‌های جداگانه در `/etc/clickhouse-server/config.d/`)

**cluster.xml**
```xml
<clickhouse>
    <remote_servers>
        <cluster_1S_2R>
            <shard>
                <internal_replication>true</internal_replication>
                <replica>
                    <host>192.168.42.130</host>
                    <port>9001</port>
                </replica>
                <replica>
                    <host>192.168.42.129</host>
                    <port>9001</port>
                </replica>
            </shard>
        </cluster_1S_2R>
    </remote_servers>
</clickhouse>
```

**network.xml**
```xml
<clickhouse>
    <listen_host>0.0.0.0</listen_host>
    <http_port>8124</http_port>
    <tcp_port>9001</tcp_port>
</clickhouse>
```

**ddl.xml**
```xml
<clickhouse>
    <distributed_ddl>
        <path>/clickhouse/task_queue/ddl</path>
    </distributed_ddl>
</clickhouse>
```

**macros.xml**
```xml
<clickhouse>
    <macros>
        <shard>01</shard>
        <replica>01</replica>
        <cluster>cluster_1S_2R</cluster>
    </macros>
</clickhouse>
```

**zookeeper.xml**
```xml
<clickhouse>
    <zookeeper>
        <node>
            <host>192.168.42.130</host>
            <port>9181</port>
        </node>
        <node>
            <host>192.168.42.129</host>
            <port>9181</port>
        </node>
        <node>
            <host>192.168.42.128</host>
            <port>9181</port>
        </node>
    </zookeeper>
</clickhouse>
```

**/etc/clickhouse-keeper/keeper_config.xml**
```xml
<clickhouse replace="true">
    <logger>
        <level>information</level>
        <log>/var/log/clickhouse-keeper/clickhouse-keeper.log</log>
        <errorlog>/var/log/clickhouse-keeper/clickhouse-keeper.err.log</errorlog>
        <size>1000M</size>
        <count>3</count>
    </logger>
    <listen_host>0.0.0.0</listen_host>
    <keeper_server>
        <tcp_port>9181</tcp_port>
        <server_id>1</server_id>
        <log_storage_path>/var/lib/clickhouse/coordination/log</log_storage_path>
        <snapshot_storage_path>/var/lib/clickhouse/coordination/snapshots</snapshot_storage_path>
        <coordination_settings>
            <operation_timeout_ms>10000</operation_timeout_ms>
            <session_timeout_ms>30000</session_timeout_ms>
            <raft_logs_level>information</raft_logs_level>
        </coordination_settings>
        <raft_configuration>
            <server>
                <id>1</id>
                <hostname>192.168.42.130</hostname>
                <port>9234</port>
            </server>
            <server>
                <id>2</id>
                <hostname>192.168.42.129</hostname>
                <port>9234</port>
            </server>
            <server>
                <id>3</id>
                <hostname>192.168.42.128</hostname>
                <port>9234</port>
            </server>
        </raft_configuration>
    </keeper_server>
</clickhouse>
```

## نود ۲ (192.168.42.129)

### تنظیمات مشابه نود ۱ با تغییر شماره رپلیکا

**cluster.xml, network.xml, ddl.xml, zookeeper.xml** همانند نود ۱ است.

**macros.xml**
```xml
<clickhouse>
    <macros>
        <shard>01</shard>
        <replica>02</replica>
        <cluster>cluster_1S_2R</cluster>
    </macros>
</clickhouse>
```

**/etc/clickhouse-keeper/keeper_config.xml**
```xml
<clickhouse replace="true">
    <logger>
        <level>information</level>
        <log>/var/log/clickhouse-keeper/clickhouse-keeper.log</log>
        <errorlog>/var/log/clickhouse-keeper/clickhouse-keeper.err.log</errorlog>
        <size>1000M</size>
        <count>3</count>
    </logger>
    <listen_host>0.0.0.0</listen_host>
    <keeper_server>
        <tcp_port>9181</tcp_port>
        <server_id>2</server_id>
        <log_storage_path>/var/lib/clickhouse/coordination/log</log_storage_path>
        <snapshot_storage_path>/var/lib/clickhouse/coordination/snapshots</snapshot_storage_path>
        <coordination_settings>
            <operation_timeout_ms>10000</operation_timeout_ms>
            <session_timeout_ms>30000</session_timeout_ms>
            <raft_logs_level>information</raft_logs_level>
        </coordination_settings>
        <raft_configuration>
            <server>
                <id>1</id>
                <hostname>192.168.42.130</hostname>
                <port>9234</port>
            </server>
            <server>
                <id>2</id>
                <hostname>192.168.42.129</hostname>
                <port>9234</port>
            </server>
            <server>
                <id>3</id>
                <hostname>192.168.42.128</hostname>
                <port>9234</port>
            </server>
        </raft_configuration>
    </keeper_server>
</clickhouse>
```

## تعریف سوپرادمین

**فایل**: `/etc/clickhouse-server/users.d/admin.xml` روی نود ۱ و ۲
```xml
<clickhouse>
    <users>
        <admin>
            <password_sha256_hex>رمز هش شده</password_sha256_hex>
            <profile>default</profile>
            <quota>default</quota>
            <networks>
                <ip>0.0.0.0/0</ip>
                <ip>::/0</ip>
            </networks>
            <access_management>1</access_management>
            <named_collection_control>1</named_collection_control>
            <show_named_collections>1</show_named_collections>
            <show_named_collections_secrets>1</show_named_collections_secrets>
        </admin>
    </users>
</clickhouse>
```

## نود ۳ (192.168.42.128)

این نود فقط برای ClickHouse Keeper استفاده می‌شود.

**/etc/clickhouse-keeper/keeper_config.xml**
```xml
<clickhouse replace="true">
    <logger>
        <level>information</level>
        <log>/var/log/clickhouse-keeper/clickhouse-keeper.log</log>
        <errorlog>/var/log/clickhouse-keeper/clickhouse-keeper.err.log</errorlog>
        <size>1000M</size>
        <count>3</count>
    </logger>
    <listen_host>0.0.0.0</listen_host>
    <keeper_server>
        <tcp_port>9181</tcp_port>
        <server_id>3</server_id>
        <log_storage_path>/var/lib/clickhouse/coordination/log</log_storage_path>
        <snapshot_storage_path>/var/lib/clickhouse/coordination/snapshots</snapshot_storage_path>
        <coordination_settings>
            <operation_timeout_ms>10000</operation_timeout_ms>
            <session_timeout_ms>30000</session_timeout_ms>
            <raft_logs_level>information</raft_logs_level>
        </coordination_settings>
        <raft_configuration>
            <server>
                <id>1</id>
                <hostname>192.168.42.130</hostname>
                <port>9234</port>
            </server>
            <server>
                <id>2</id>
                <hostname>192.168.42.129</hostname>
                <port>9234</port>
            </server>
            <server>
                <id>3</id>
                <hostname>192.168.42.128</hostname>
                <port>9234</port>
            </server>
        </raft_configuration>
    </keeper_server>
</clickhouse>
```

## باز کردن پورت‌ها روی همه نودها
```bash
sudo ss -tulpn | grep 8124
```

## نصب HAProxy برای Load Balancing

### 1. اضافه کردن repository رسمی
```bash
sudo apt update
sudo apt install -y software-properties-common ca-certificates lsb-release
sudo add-apt-repository ppa:vbernat/haproxy-3.0
```

### 2. نصب HAProxy
```bash
sudo apt update
sudo apt install -y haproxy=3.0.*
```

### 3. بررسی نسخه
```bash
haproxy -v
```

### 4. کانفیگ پیشنهادی HAProxy
```haproxy
global
    log /dev/log local0
    daemon
    maxconn 2048

defaults
    log     global
    mode    tcp
    option  tcplog
    timeout connect 5s
    timeout client  1m
    timeout server  1m

frontend clickhouse_tcp
    bind *:19000
    default_backend clickhouse_replicas

backend clickhouse_replicas
    balance roundrobin
    option tcp-check
    server node1 192.168.42.130:9009 check
    server node2 192.168.42.129:9009 check
    server node3 192.168.42.128:9009 check

frontend clickhouse_http
    bind *:18123
    default_backend clickhouse_http_replicas

backend clickhouse_http_replicas
    balance roundrobin
    option tcp-check
    server node1 192.168.42.130:8124 check
    server node2 192.168.42.129:8124 check
    server node3 192.168.42.128:8124 check
```

### 5. باز کردن پورت‌ها در firewall
```bash
sudo ufw allow 19000/tcp
sudo ufw allow 18123/tcp
```

### 6. ریستارت HAProxy
```bash
sudo systemctl restart haproxy
sudo systemctl status haproxy
```

### 7. تست اتصال
- SQL:
```bash
clickhouse-client --host <HAProxy_IP> --port 19000
```
- HTTP/UI:
```bash
curl http://<HAProxy_IP>:18123
```


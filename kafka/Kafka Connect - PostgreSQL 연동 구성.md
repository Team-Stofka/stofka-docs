# 🔗 Kafka Connect - PostgreSQL 연동 구성 문서

---

## 📁 목차

- [Kafka Connect란?](notion://www.notion.so/1bd6f5c64d4581a5870ad3e6f52ddd19?v=1bd6f5c64d4581fe9b45000c1474ae45&p=1be6f5c64d45804a8d2ced0d4c0a7796&pm=s#kafka-connect%EB%9E%80)
- [시스템 구성도](notion://www.notion.so/1bd6f5c64d4581a5870ad3e6f52ddd19?v=1bd6f5c64d4581fe9b45000c1474ae45&p=1be6f5c64d45804a8d2ced0d4c0a7796&pm=s#%EC%8B%9C%EC%8A%A4%ED%85%9C-%EA%B5%AC%EC%84%B1%EB%8F%84)
- [사전 준비 사항](notion://www.notion.so/1bd6f5c64d4581a5870ad3e6f52ddd19?v=1bd6f5c64d4581fe9b45000c1474ae45&p=1be6f5c64d45804a8d2ced0d4c0a7796&pm=s#%EC%82%AC%EC%A0%84-%EC%A4%80%EB%B9%84-%EC%82%AC%ED%95%AD)
- [PostgreSQL 테이블 구조](notion://www.notion.so/1bd6f5c64d4581a5870ad3e6f52ddd19?v=1bd6f5c64d4581fe9b45000c1474ae45&p=1be6f5c64d45804a8d2ced0d4c0a7796&pm=s#postgresql-%ED%85%8C%EC%9D%B4%EB%B8%94-%EA%B5%AC%EC%A1%B0)
- [Kafka Connect 설정](notion://www.notion.so/1bd6f5c64d4581a5870ad3e6f52ddd19?v=1bd6f5c64d4581fe9b45000c1474ae45&p=1be6f5c64d45804a8d2ced0d4c0a7796&pm=s#kafka-connect-%EC%84%A4%EC%A0%95)
- [PostgreSQL Sink Connector 등록](notion://www.notion.so/1bd6f5c64d4581a5870ad3e6f52ddd19?v=1bd6f5c64d4581fe9b45000c1474ae45&p=1be6f5c64d45804a8d2ced0d4c0a7796&pm=s#postgresql-sink-connector-%EB%93%B1%EB%A1%9D)
- [Kafka에 JSON 데이터 전송하기](notion://www.notion.so/1bd6f5c64d4581a5870ad3e6f52ddd19?v=1bd6f5c64d4581fe9b45000c1474ae45&p=1be6f5c64d45804a8d2ced0d4c0a7796&pm=s#kafka%EC%97%90-json-%EB%8D%B0%EC%9D%B4%ED%84%B0-%EC%A0%84%EC%86%A1%ED%95%98%EA%B8%B0)
- [전체 동작 구조 및 이유](notion://www.notion.so/1bd6f5c64d4581a5870ad3e6f52ddd19?v=1bd6f5c64d4581fe9b45000c1474ae45&p=1be6f5c64d45804a8d2ced0d4c0a7796&pm=s#%EC%A0%84%EC%B2%B4-%EB%8F%99%EC%9E%91-%EA%B5%AC%EC%A1%B0-%EB%B0%8F-%EC%9D%B4%EC%9C%A0)
- [확인 및 디버깅](notion://www.notion.so/1bd6f5c64d4581a5870ad3e6f52ddd19?v=1bd6f5c64d4581fe9b45000c1474ae45&p=1be6f5c64d45804a8d2ced0d4c0a7796&pm=s#%ED%99%95%EC%9D%B8-%EB%B0%8F-%EB%94%94%EB%B2%84%EA%B9%85)

---

## 🧠 Kafka Connect란?

> Kafka Connect는 Kafka와 외부 시스템(PostgreSQL, Elasticsearch 등)을 손쉽게 연결해주는 데이터 파이프라인 프레임워크다.

- Source Connector → 외부 시스템 → Kafka
- Sink Connector → Kafka → 외부 시스템

---

## 🖥️ 시스템 구성도

```
┌─────────────┐      Kafka Topic       ┌────────────────────┐
│ Data Source │ ─────────────────────▶ │   Kafka Broker(s)  │
└─────────────┘                        └────────────────────┘
                                           │
                                           ▼
                               ┌─────────────────────────┐
                               │     Kafka Connect       │
                               └─────────────────────────┘
                                           │
                                           ▼
                             ┌────────────────────────────┐
                             │      PostgreSQL DB         │
                             └────────────────────────────┘

```

---

## ✅ 사전 준비 사항

| 항목                 | 값                    |
| -------------------- | --------------------- |
| Kafka 브로커         | 172.16.10.101 ~ 103   |
| Kafka Connect 서버   | 172.16.10.111         |
| PostgreSQL 서버      | 172.16.10.51          |
| PostgreSQL 접속 정보 | postgres / postgres   |
| Kafka Connect Port   | 8083                  |
| JDBC Driver 설치     | postgresql-42.7.1.jar |

---

## 🗃️ PostgreSQL 테이블 구조

**테이블명**: `public.market_candle`

**Primary Key**: `code`, `timestamp`

| 컬럼명                  | 타입    |
| ----------------------- | ------- |
| type                    | text    |
| code                    | text    |
| timestamp               | int8    |
| opening_price           | numeric |
| high_price              | numeric |
| low_price               | numeric |
| trade_price             | numeric |
| candle_acc_trade_volume | numeric |
| candle_acc_trade_price  | numeric |
| stream_type             | text    |

**테이블 생성 쿼리**

```sql
CREATE TABLE candle (
		type TEXT,
    code TEXT NOT NULL,
    timestamp BIGINT,
    opening_price NUMERIC,
    high_price NUMERIC,
    low_price NUMERIC,
    trade_price NUMERIC,
    candle_acc_trade_volume NUMERIC,
    candle_acc_trade_price NUMERIC,
    stream_type TEXT,
    PRIMARY KEY (code, timestamp)
);
```

---

## ⚙️ Kafka Connect 설정

### 🔌 plugin path 설정 및 JDBC 드라이버 설치

```bash
mkdir -p /home/ubuntu/kafka_2.13-3.6.2/plugins
wget <https://jdbc.postgresql.org/download/postgresql-42.7.1.jar>
cp postgresql-42.7.1.jar /home/ubuntu/kafka_2.13-3.6.2/plugins/
```

### ⚙️ connect-distributed.properties 설정

```
bootstrap.servers=172.16.10.101:9092,172.16.10.102:9092,172.16.10.103:9092
group.id=connect-cluster

key.converter=org.apache.kafka.connect.json.JsonConverter
value.converter=org.apache.kafka.connect.json.JsonConverter
key.converter.schemas.enable=true
value.converter.schemas.enable=true

offset.storage.topic=connect-offsets
offset.storage.replication.factor=2

config.storage.topic=connect-configs
config.storage.replication.factor=2

status.storage.topic=connect-status
status.storage.replication.factor=2

offset.flush.interval.ms=10000

listeners=HTTP://0.0.0.0:8083
rest.advertised.host.name=172.16.10.111
rest.advertised.port=8083

plugin.path=/home/ubuntu/kafka_2.13-3.6.2/plugins
log4j.logger.org.apache.kafka.connect=DEBUG
log4j.logger.org.apache.kafka.connect.transforms=DEBUG
```

---

## 🔌 PostgreSQL Sink Connector 등록

### candle-sink-connector.json

```json
{
  "name": "candle-sink-connector",
  "config": {
    "connector.class": "io.confluent.connect.jdbc.JdbcSinkConnector",
    "tasks.max": "1",
    "topics": "market_candle",
    "connection.url": "jdbc:postgresql://172.16.10.51:5432/postgres",
    "connection.user": "postgres",
    "connection.password": "postgres",
    "table.name": "public.market_candle",
    "insert.mode": "upsert",
    "pk.mode": "record_value",
    "pk.fields": "code,timestamp",
    "auto.create": "false",
    "auto.evolve": "true",
    "delete.enabled": "false",
    "key.converter": "org.apache.kafka.connect.json.JsonConverter",
    "key.converter.schemas.enable": "true",
    "value.converter": "org.apache.kafka.connect.json.JsonConverter",
    "value.converter.schemas.enable": "true"
  }
}
```

### 등록 명령어

```bash
curl -X POST <http://localhost:8083/connectors> \\
  -H "Content-Type: application/json" \\
  -d @candle-sink-connector.json
```

---

## 📤 Kafka에 JSON 데이터 전송하기

### 전송 예시

```bash
echo '{
  "schema": {
    "type": "struct",
    "fields": [
      {"field": "type", "type": "string"},
      {"field": "code", "type": "string"},
      {"field": "opening_price", "type": "double"},
      {"field": "high_price", "type": "double"},
      {"field": "low_price", "type": "double"},
      {"field": "trade_price", "type": "double"},
      {"field": "candle_acc_trade_volume", "type": "double"},
      {"field": "candle_acc_trade_price", "type": "double"},
      {"field": "timestamp", "type": "int64"},
      {"field": "stream_type", "type": "string"}
    ],
    "optional": false,
    "name": "market_candle"
  },
  "payload": {
    "type": "candle",
    "code": "BTC-USD",
    "opening_price": 45000.0,
    "high_price": 46000.0,
    "low_price": 44000.0,
    "trade_price": 45500.0,
    "candle_acc_trade_volume": 1234.5,
    "candle_acc_trade_price": 56000000.0,
    "timestamp": 1617090000,
    "stream_type": "realtime"
  }
}' | /home/ubuntu/kafka_2.13-3.6.2/bin/kafka-console-producer.sh --broker-list 172.16.10.101:9092 --topic market_candle
```

### ✔️ 이렇게 구성한 이유

- `pk.fields`를 `code`, `timestamp`로 지정하여 **중복 없이** 업서트 가능
- `value.converter.schemas.enable=true` → Kafka Connect가 JSON 구조를 이해하고 스키마 기반으로 매핑
- `insert.mode=upsert` → 동일 키가 있을 경우 update, 없으면 insert

---

## 🔄 전체 동작 구조 및 이유

- 실시간 시장 캔들 데이터가 Kafka Topic에 적재됨
- Kafka Connect Sink Connector가 데이터를 PostgreSQL로 **upsert**
- 실시간 데이터의 **중복 방지**, **빠른 반영**, **정확한 갱신**이 가능

---

## 🧪 확인 및 디버깅

### Connector 상태 확인

```bash
curl <http://localhost:8083/connectors/candle-sink-connector/status>
```

![candle-sink-connector 정상 RUNNUNG 확인 가능](attachment:7e46d656-873d-44dd-b98b-d644ebe25deb:image.png)

candle-sink-connector 정상 RUNNUNG 확인 가능

### PostgreSQL 테이블 확인

```sql
SELECT * FROM public.market_candle ORDER BY timestamp DESC LIMIT 5;
```

![위와 같이 데이터가 들어온 것을 확인 가능](attachment:b8eadab9-7ea1-40cf-bdb1-8e365af09eb1:image.png)

위와 같이 데이터가 들어온 것을 확인 가능

### Kafka Connect 로그 확인

```bash
tail -f /home/ubuntu/kafka_2.13-3.6.2/logs/connect.log
```

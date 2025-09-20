# ✅ Kafka 실시간 모니터링 구축 요약 (Prometheus + Grafana)

---

## 📌 1. Kafka에 JMX Exporter Agent 설정

### ✅ 목적

Kafka 브로커의 내부 메트릭(JMX)을 Prometheus가 수집할 수 있도록 **HTTP로 노출**.

### ✅ 작업 내용

### 1-1. JMX Exporter JAR 다운로드

```bash
wget https://repo1.maven.org/maven2/io/prometheus/jmx/jmx_prometheus_javaagent/0.20.0/jmx_prometheus_javaagent-0.20.0.jar -P /opt/kafka/libs/

```

### 1-2. `jmx-exporter.yml` 설정 파일 생성

```bash
sudo mkdir /opt/kafka/config
sudo nano /opt/kafka/config/jmx-exporter.yml
```

```yaml
startDelaySeconds: 0
lowercaseOutputName: true
lowercaseOutputLabelNames: true
rules:
  # Broker Topic Metrics (메시지 수, 처리량 등)
  - pattern: "kafka.server<type=BrokerTopicMetrics, name=(.+)><>Value"
    name: kafka_server_brokertopicmetrics_$1
    type: GAUGE

  # Kafka Controller 상태
  - pattern: "kafka.controller<type=(.+), name=(.+)><>Value"
    name: kafka_controller_$1_$2
    type: GAUGE

  # Kafka ReplicaManager (언더 리플리케이션 파티션 등)
  - pattern: "kafka.server<type=ReplicaManager, name=(.+)><>Value"
    name: kafka_server_replica_$1
    type: GAUGE

  # Kafka Network 처리량
  - pattern: "kafka.network<type=(.+), name=(.+)><>Value"
    name: kafka_network_$1_$2
    type: GAUGE

  # Kafka Request Handler
  - pattern: "kafka.request<type=(.+), name=(.+), request=(.+)><>Value"
    name: kafka_request_$1_$2_$3
    type: GAUGE

  # Kafka Log Flush, Log Metrics
  - pattern: "kafka.log<type=(.+), name=(.+)><>Value"
    name: kafka_log_$1_$2
    type: GAUGE

  # Kafka Fetcher (replica fetcher 등)
  - pattern: "kafka.server<type=FetcherLagMetrics, clientId=(.+), topic=(.+), partition=(.+)><>Value"
    name: kafka_fetcher_lag_$2_$3_partition_$4
    type: GAUGE

  # JVM 메트릭 (GC, Memory 등)
  - pattern: "java.lang<type=GarbageCollector, name=(.+)><>CollectionCount"
    name: jvm_gc_$1_collection_count
    type: COUNTER
  - pattern: "java.lang<type=GarbageCollector, name=(.+)><>CollectionTime"
    name: jvm_gc_$1_collection_time
    type: COUNTER
  - pattern: "java.lang<type=MemoryPool, name=(.+)><>Usage"
    name: jvm_memorypool_$1_usage
    type: GAUGE
  - pattern: "java.lang<type=Memory, name=(.+)><>HeapMemoryUsage"
    name: jvm_memory_$1_heap
    type: GAUGE

  # 쓰레드 수
  - pattern: "java.lang<type=Threading><>ThreadCount"
    name: jvm_thread_count
    type: GAUGE
```

### 1-3. Kafka 실행 스크립트에 JVM 옵션 추가 (`kafka-server-start.sh`)

```bash
if [ "x$KAFKA_JMX_OPTS" = "x" ]; then
  export KAFKA_JMX_OPTS="-javaagent:/opt/kafka/libs/jmx_prometheus_javaagent-0.20.0.jar=7071:/opt/kafka/config/jmx-exporter.yml"
fi
```

`kafka-1 → 7071`

### 1-4. Kafka 브로커 재시작

```bash
lsof -i :9092
kill -9 <Kafka PID>
nohup bin/kafka-server-start.sh config/kraft/server.properties > kafka.log 2>&1 &
```

### 1-5. JMX Exporter 확인

```bash
curl http://localhost:7071/metrics
```

---

## 📌 2. Prometheus 설정

### ✅ 목적

Kafka 브로커에서 노출된 메트릭을 **주기적으로 수집**.

### ✅ 작업 내용

### 2-1. 디렉토리 생성 및 설정 파일 작성

```bash
mkdir -p ~/prometheus
cd ~/prometheus
nano prometheus.yml
```

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: "kafka-exporter"
    static_configs:
      - targets:
          - "172.16.10.101:9308"
          - "172.16.10.102:9308"
          - "172.16.10.103:9308"

  # Kafka JMX Exporter (JVM 메트릭)
  - job_name: "kafka-jmx"
    static_configs:
      - targets:
          ["172.16.10.101:7071", "172.16.10.102:7071", "172.16.10.103:7071"]

  - job_name: "node-exporter"
    static_configs:
      - targets:
          - 172.16.10.151:9100 # Master (kubernetes)
          - 172.16.10.161:9100 # Worker 1 (argo)
          - 172.16.10.202:9100 # Worker 2 (was-producer)
          - 172.16.10.201:9100 # Worker 3 (was-sse)
          - 172.16.10.231:9100 # Worker 4 (web)

  - job_name: "postgres-exporter"
    static_configs:
      - targets: ["172.16.10.51:9187"]

  - job_name: "redis-exporter"
    static_configs:
      - targets: ["172.16.10.61:9121"]
```

### 2-2. Prometheus 실행 (Docker)

```bash
docker run -d \
  --name prometheus \
  -p 9090:9090 \
  -v ~/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml \
  prom/prometheus
```

### 2-3. Prometheus UI 접속 확인

브라우저 → `http://<서버IP>:9090`

→ 상단 메뉴 `Status → Targets` → 브로커 포트 상태 `✓` 확인

---

## 📌 3. Grafana 설정

### ✅ 목적

Prometheus가 수집한 Kafka 메트릭을 **시각화 대시보드**로 보여줌

### ✅ 작업 내용

### 3-1. Grafana 실행 (Docker)

```bash
docker run -d \
  --name=grafana \
  -p 3000:3000 \
  grafana/grafana

```

### 3-2. 웹 UI 접속 및 로그인

- 브라우저 접속: `http://<서버IP>:3000`
- ID / PW: `admin` / `admin` → 비밀번호 변경
  - ID : admin
  - PW : Grafana1!

### 3-3. Prometheus 데이터 소스 추가

- `⚙️ Settings → Data Sources → Add data source`
- `Prometheus` 선택
- URL: `http://<서버IP>:9090` 입력 → `Save & Test`

### 3-4. Kafka 대시보드 Import

- `+ → Import`
- Dashboard ID 입력: `7589` (Kafka Overview)
- Prometheus 선택 → `Import`

---

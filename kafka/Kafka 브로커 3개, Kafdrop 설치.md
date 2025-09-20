# 🧱 Kafka 브로커 3개 (KRaft) 구성과 Kafdrop 설치

---

## 📁 목차

- Kafka 구성 개요
- 시스템 구성도
- 사전 준비 사항
- 설치 및 환경 설정
- KRaft 포맷 및 브로커 실행
- Kafdrop 설치 (UI)
- Kafdrop 접속

---

## 🧠 Kafka 구성 개요

> KRaft 모드로 Kafka 브로커 3개 구성

- 각 브로커는 **서로 다른 VM**에 설치
- **KRaft 모드 (Zookeeper 미사용)**

---

## 🖥️ 시스템 구성도

```
┌───────────────┐        ┌───────────────┐       ┌───────────────┐
│   Broker 1    │        │   Broker 2    │       │   Broker 3    │
│ 172.16.10.101 │◀────▶│ 172.16.10.102 │◀────▶│ 172.16.10.103 │
└───────────────┘        └───────────────┘       └───────────────┘

```

---

## ✅ 사전 준비 사항

| 항목          | 값                        |
| ------------- | ------------------------- |
| Kafka 버전    | 3.6.2                     |
| 설치 경로     | `/opt/kafka`              |
| 각 브로커 IP  | 172.16.10.101 / 102 / 103 |
| KRaft 포트    | 9092                      |
| 컨트롤러 포트 | 9093                      |
| Kafka 사용자  | ubuntu                    |
| Java          | 11 이상 버전 설치         |

---

## ⚙️ 설치 및 환경 설정

### 1️⃣ Kafka 압축 해제

```bash
wget https://downloads.apache.org/kafka/3.6.2/kafka_2.13-3.6.2.tgz
tar -xvzf kafka_2.13-3.6.2.tgz
mv kafka_2.13-3.6.2 /opt/kafka
```

### 2️⃣ 각 브로커에 맞는 `server.properties` 설정

### 📄 `/opt/kafka/config/kraft/server.properties`

```
process.roles=broker,controller
node.id=1                          # 브로커마다 고유 번호 (1, 2, 3)
controller.quorum.voters=1@172.16.10.101:9093,2@172.16.10.102:9093,3@172.16.10.103:9093

listeners=PLAINTEXT://172.16.10.101:9092,CONTROLLER://172.16.10.101:9093
advertised.listeners=PLAINTEXT://172.16.10.101:9092
listener.security.protocol.map=CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT

log.dirs=/tmp/kraft-combined-logs

```

- `node.id`, `IP`, `log.dirs`는 브로커마다 다르게 설정해야 함

---

## 🧱 KRaft 포맷 및 브로커 실행

### 1️⃣ 브로커 1 KRaft 랜덤 UUID 포맷

```bash
/opt/kafka/bin/kafka-storage.sh format -t $(/opt/kafka/bin/kafka-storage.sh random-uuid) -c /opt/kafka/config/kraft/server.properties
```

### 2️⃣ 생성된 UUID 확인

```bash
cat /tmp/kraft-combined-logs/meta.properties
```

![로그.png](attachment:97e39059-1b90-47e0-9f59-e7c2477897e5:로그.png)

- [cluster.id](http://cluster.id) 메모

### 3️⃣ 브로커 2, 3 KRaft 지정 UUID 포맷

```bash
/opt/kafka/bin/kafka-storage.sh format -t {cluster.id}
-c /opt/kafka/config/kraft/server.properties
```

- 메모해둔 cluster.id를 적어서 포맷 진행

---

### 4️⃣ 모든 브로커 실행

```bash
/opt/kafka/bin/kafka-server-start.sh /opt/kafka/config/kraft/server.properties
 nohup bin/kafka-server-start.sh config/kraft/server.properties > kafka.log 2>&1 &
```

- 모든 브로커를 해당 명령어로 실행

---

## 🧰 Kafdrop 설치 (UI)

### 1️⃣ Kafdrop 3.31.0 JAR 다운로드

```bash
wget https://github.com/obsidiandynamics/kafdrop/releases/download/3.31.0/kafdrop-3.31.0.jar -O kafdrop.jar
```

### 2️⃣ Kafdrop 실행

```bash
java -jar kafdrop.jar \
  --kafka.brokerConnect=172.16.10.101:9092,172.16.10.102:9092,172.16.10.103:9092 \
  --server.port=9000
```

- 백그라운드 실행 예:

```bash
nohup java -jar kafdrop.jar \
  --kafka.brokerConnect=172.16.10.101:9092,172.16.10.102:9092,172.16.10.103:9092 \
  --server.port=9000 > kafdrop.log 2>&1 &
```

---

## 🌐 Kafdrop 접속

```
http://<Kafdrop 실행한 VM IP>:9000
```

![image.png](attachment:feb7576a-df4d-4064-b980-e58cd05fe7e9:image.png)

- 정상 접속 및 브로커 구동 확인 완료!

---

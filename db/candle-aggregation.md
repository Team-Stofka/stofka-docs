## 📘 PostgreSQL 다중 구간 캔들 생성 및 자동 실행

### 🏗️ **기본 개요**

- **목적**: 실시간 `초 단위` 데이터 기반으로 다양한 시간 구간의 캔들 생성
- **데이터베이스**: PostgreSQL
- **언어**: PL/pgSQL
- **특징**: 계층적(하향식) 캔들 생성 구조 + 중복 방지 + 성능 최적화 고려

---

### 🔁 **구간별 계층 구조**

각 상위 구간은 불필요한 연산을 최소화하기 위해 **이미 처리된 하위 구간 테이블을 참조**해 생성

| 상위 캔들              | 하위 참조 테이블           |
| ---------------------- | -------------------------- |
| 1분봉 (`candle_1m`)    | 초캔들 (Tick/Trade 데이터) |
| 3분봉 (`candle_3m`)    | 1분봉                      |
| 5분봉 (`candle_5m`)    | 1분봉                      |
| 10분봉 (`candle_10m`)  | 5분봉                      |
| 15분봉 (`candle_15m`)  | 5분봉                      |
| 30분봉 (`candle_30m`)  | 15분봉                     |
| 1시간봉 (`candle_60m`) | 30분봉                     |
| 4시간봉 (`candle_4h`)  | 1시간봉                    |
| 1일봉 (`candle_1d`)    | 4시간봉                    |

---

### 💡 **왜 이렇게 만들었는가? (최적화 이유)**

- **데이터 중복 처리 비용 최소화**
  → 동일 구간에서 여러 번 집계하는 것을 방지하기 위해 상위 구간은 **이미 집계된 하위 구간 데이터만 참조**
- **성능 최적화**
  → 초캔들로 모든 구간을 만들면 과도한 비용 발생 → 이를 피하기 위해 **계층 구조 + 최근 생성된 시간 이후의 데이터만 생성**
- **재사용성 강화**
  → 하위 구간 결과를 기반으로 생성 → 상위 구간 로직 간결화 및 일관성 확보
- **유지 보수 용이성**
  → 특정 구간에서 오류 발생 시 영향 범위가 명확 → **디버깅 편의성 향상**

---

### ⚙️ **캔들 생성 로직 요약**

1. **타임 버킷 계산**
   - 각 구간별 기준 시각으로 `timestamp`를 버킷팅
   - 예: 30분 → `timestamp`를 30분 단위로 내림
     ```sql
     date_trunc('minute', candle_time)
       - INTERVAL '1 second' * (extract(minute FROM candle_time)::int % 30) * 60

     ```
2. **가장 최근 생성된 시간 이후만 처리**

   - `WITH` 절로 **마지막 생성된 캔들의 시간**을 조회
   - 이후부터의 데이터만 대상으로 집계
   - ✅ **처리 대상 최소화 → 처리 시간 감소**

   ```sql
   WITH last_thirty_minute AS (
       SELECT COALESCE(MAX(candle_time), '1970-01-01 00:00:00') AS last_time
       FROM thirty_minute_candles
   )
   SELECT ...
   FROM fifteen_minute_candles, last_thirty_minute
   WHERE candle_time > last_thirty_minute.last_time

   ```

3. **집계 방식**
   - `opening_price`: 구간 내 가장 이른 시점 가격 (`ROW_NUMBER() = 1`)
   - `high_price`: 최고가
   - `low_price`: 최저가
   - `trade_price`: 구간 마지막 시점 가격 (`ROW_NUMBER() DESC`)
   - `candle_acc_trade_volume`: 거래량 총합
4. **중복 삽입 방지**
   - `ON CONFLICT DO NOTHING` 사용
5. **계층 구조 적용**
   - 상위 캔들 생성 시 하위 구간 테이블을 참조

---

### 🔧 1**분봉 생성 로직, 3분봉 생성 로직**

```sql
1분봉

WITH last_minute AS (
	    SELECT COALESCE(MAX(candle_time), '1970-01-01 00:00:00') AS last_time
	    FROM minute_candles
	),
	raw_data AS (
	    SELECT
	        code,
	        date_trunc('minute', to_timestamp(timestamp / 1000.0)) AS minute_time,
	        opening_price,
	        high_price,
	        low_price,
	        trade_price,
	        candle_acc_trade_volume,
	        candle_acc_trade_price,
	        timestamp,
	        ROW_NUMBER() OVER (
	            PARTITION BY code, date_trunc('minute', to_timestamp(timestamp / 1000.0))
	            ORDER BY timestamp ASC
	        ) AS row_num_first,
	        ROW_NUMBER() OVER (
	            PARTITION BY code, date_trunc('minute', to_timestamp(timestamp / 1000.0))
	            ORDER BY timestamp DESC
	        ) AS row_num_last
	    FROM market_candle, last_minute
	    WHERE to_timestamp(timestamp / 1000.0) > last_minute.last_time
	),
	aggregated_data AS (
	    SELECT
	        code,
	        minute_time as candle_time,
	        MAX(high_price) AS high_price,
	        MIN(low_price) AS low_price,
	        SUM(candle_acc_trade_volume) AS candle_acc_trade_volume,
	        SUM(candle_acc_trade_price) AS candle_acc_trade_price,
	        (SELECT opening_price FROM raw_data WHERE raw_data.code = d.code AND raw_data.minute_time = d.minute_time AND row_num_first = 1) AS opening_price,
	        (SELECT trade_price FROM raw_data WHERE raw_data.code = d.code AND raw_data.minute_time = d.minute_time AND row_num_last = 1) AS trade_price
	    FROM raw_data d
	    GROUP BY code, minute_time
	)
	INSERT INTO minute_candles (code, candle_time, high_price, low_price, candle_acc_trade_volume, candle_acc_trade_price, opening_price, trade_price)
	SELECT * FROM aggregated_data
	ON CONFLICT (code, candle_time) DO NOTHING;
```

```sql
3분봉

WITH last_three_minute AS (
	    SELECT COALESCE(MAX(candle_time), '1970-01-01 00:00:00') AS last_time
	    FROM three_minute_candles
	),
	raw_data AS (
	    SELECT
	        code,
	        date_trunc('minute', candle_time)
	            - INTERVAL '1 second' * (extract(minute from candle_time)::int % 3) * 60 AS bucket_time,
	        candle_time,
	        opening_price,
	        high_price,
	        low_price,
	        trade_price,
	        candle_acc_trade_volume,
	        candle_acc_trade_price,
	        ROW_NUMBER() OVER (
	            PARTITION BY code, date_trunc('minute', candle_time)
	                         - INTERVAL '1 second' * (extract(minute from candle_time)::int % 3) * 60
	            ORDER BY candle_time ASC
	        ) AS row_num_first,
	        ROW_NUMBER() OVER (
	            PARTITION BY code, date_trunc('minute', candle_time)
	                         - INTERVAL '1 second' * (extract(minute from candle_time)::int % 3) * 60
	            ORDER BY candle_time DESC
	        ) AS row_num_last
	    FROM minute_candles, last_three_minute
	    WHERE candle_time > last_three_minute.last_time
	),
	aggregated_data AS (
	    SELECT
	        code,
	        bucket_time AS candle_time,
	        MAX(high_price) AS high_price,
	        MIN(low_price) AS low_price,
	        SUM(candle_acc_trade_volume) AS candle_acc_trade_volume,
	        SUM(candle_acc_trade_price) AS candle_acc_trade_price,
	        (SELECT opening_price FROM raw_data WHERE raw_data.code = d.code AND raw_data.bucket_time = d.bucket_time AND row_num_first = 1) AS opening_price,
	        (SELECT trade_price FROM raw_data WHERE raw_data.code = d.code AND raw_data.bucket_time = d.bucket_time AND row_num_last = 1) AS trade_price
	    FROM raw_data d
	    GROUP BY code, bucket_time
	)
	INSERT INTO minute_candles (code, candle_time, high_price, low_price, candle_acc_trade_volume, candle_acc_trade_price, opening_price, trade_price)
	SELECT * FROM aggregated_data
	ON CONFLICT (code, candle_time) DO NOTHING;
```

---

## 📘 pg_cro

---

### 📌 1. `pg_cron` 설치 및 자동 실행 설정

### 🔧 설치

```bash
sudo apt install postgresql-15-cron
```

### 🔧 PostgreSQL 설정 (`postgresql.conf`) 수정

```
shared_preload_libraries = 'pg_cron'
```

- 설정 반영을 위해 PostgreSQL 재시작 필요:

```bash
sudo systemctl restart postgresql
```

### 🔧 `pg_cron` 확장 설치 (DB 내에서)

```sql
CREATE EXTENSION IF NOT EXISTS pg_cron;
```

### 📂 기본 설치 위치

- `pg_cron` 관련 실행은 기본적으로 **`cron.job` 테이블을 통해 관리됨** (시스템 cron이 아님)
- 작업은 **`postgresql.conf`에서 설정한 DB 인스턴스 내부**에서 주기적으로 실행됨

---

### 📌 2. `pg_hba.conf` 수정 (PostgreSQL 접근 허용)

pg_cron은 내부적으로 PostgreSQL에서 **localhost 접근을 시도**함.

기본적으로 다음과 같은 항목이 필요함:

### 📄 `/etc/postgresql/15/main/pg_hba.conf` 에서 다음 항목 확인 및 수정:

```
# TYPE     DATABASE        USER            ADDRESS                 METHOD
host       all             all             127.0.0.1/32            trust
```

- `postgres` 유저가 `localhost`에서 접근 가능하도록 설정 필요
- `trust`로 설정 시 패스워드 없이 접근 허용됨 (**운영 환경에선 암호화 사용 필수**)

수정 후 PostgreSQL 재시작:

```bash
sudo systemctl restart postgresql
```

---

### 📌 3. 캔들 생성 함수 및 메인 함수 등록

### ⛏️ 캔들 생성 함수 계층 구조

- 1분봉: `generate_minute_candles()`
- 3분봉: `generate_three_minute_candles()`
- 5분봉: `generate_five_minute_candles()`
- 10분봉: `generate_ten_minute_candles()`
- 15분봉: `generate_fifteen_minute_candles()`
- 30분봉: `generate_thirty_minute_candles()`
- 1시간봉: `generate_hour_candles()`
- 4시간봉: `generate_four_hour_candles()`
- 1일봉: `generate_day_candles()`

### 🧠 전체 캔들 생성 메인 함수

```sql
CREATE OR REPLACE FUNCTION generate_all_candles() RETURNS void AS $$
BEGIN
    PERFORM generate_minute_candles();
    PERFORM generate_three_minute_candles();
    PERFORM generate_five_minute_candles();
    PERFORM generate_ten_minute_candles();
    PERFORM generate_fifteen_minute_candles();
    PERFORM generate_thirty_minute_candles();
    PERFORM generate_hour_candles();
    PERFORM generate_four_hour_candles();
    PERFORM generate_day_candles();
END;
$$ LANGUAGE plpgsql;
```

---

### 📌 4. `pg_cron`으로 1분마다 메인 함수 실행 등록

```sql
SELECT cron.schedule(
  'generate candles every 1min',
  '* * * * *',
  $$SELECT generate_all_candles();$$
);
```

- `* * * *` → 매 분마다 실행
- 등록된 작업은 `cron.job` 테이블에서 확인 가능

```sql
SELECT * FROM cron.job;
```

---

### ✅ 최종 요약

| 항목                | 설명                                         |
| ------------------- | -------------------------------------------- |
| ✅ pg_cron 설치     | PostgreSQL 확장으로 작업 예약 가능           |
| ✅ pg_hba.conf 수정 | `localhost` → `postgres` 유저 접근 허용 필요 |
| ✅ 함수 등록        | 각 구간별 생성 함수 + 메인 함수              |
| ✅ 스케줄 설정      | `pg_cron`으로 1분마다 메인 함수 실행         |

---

### 🧠 **성능 및 확장 고려사항**

| 항목                                            | 설명                                                   |
| ----------------------------------------------- | ------------------------------------------------------ |
| ✅ **변경된 이후만 처리**                       | `MAX(candle_time)` 이후만 조회 → 처리량 최소화         |
| ✅ **계층적 집계**                              | 상위 구간은 하위 구간 테이블 참조                      |
| ✅ **중복 제거**                                | `ON CONFLICT` 활용                                     |
| ✅ **`ROW_NUMBER()` 대신 `FIRST_VALUE()` 고려** | 성능 개선                                              |
| ✅ **파티셔닝**                                 | 월 단위 파티션 → 대용량에 대비                         |
| ✅ **정합성 검증**                              | 상위 구간 생성 시 필요한 하위 구간 개수 일치 여부 체크 |

---

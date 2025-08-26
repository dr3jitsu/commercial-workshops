# Apache Flink Cheatsheet on Confluent Cloud

<div align="center">
  <img src="images/confluent.png" width="50%" height="50%">
</div>

## Overview
This repository provides a **cheatsheet for working with Apache Flink on Confluent Cloud**. It includes practical examples for commonly used **Flink SQL operations**, ranging from basic data ingestion to advanced stream processing techniques.

---

## 🔑 Key Topics Covered
- Table Creation
- Inserting & Selecting Data
- Format Conversions (JSON ↔ AVRO, CSV → JSON/AVRO)
- Handling Nested JSON
- Deduplication
- Data Enrichment
- Data Transformation
- Filtering
- Aggregation
- Serialization
- Denormalization
- Windowing

---

## ⚡ Flink SQL Cheatsheet

### 1. Create Table and Insert Records
```sql
CREATE TABLE sensor_events (
  sensor_id STRING,
  temperature DOUBLE,
  ts TIMESTAMP(3),
  WATERMARK FOR ts AS ts - INTERVAL '5' SECOND
);
```

```sql
INSERT INTO sensor_events VALUES
  ('sensor-1', 24.5, TIMESTAMP '2025-08-26 14:00:00'),
  ('sensor-2', 30.2, TIMESTAMP '2025-08-26 14:01:00'),
  ('sensor-3', 28.9, TIMESTAMP '2025-08-26 14:02:00'),
  ('sensor-1', 24.5, CURRENT_TIMESTAMP),
  ('sensor-2', 30.2, CURRENT_TIMESTAMP),
  ('sensor-3', 28.9, CURRENT_TIMESTAMP);

```

### 2. Add Primary Key & Insert Data using Select 
```sql
CREATE TABLE sensor_events_rekeyed (
  sensor_id STRING NOT NULL PRIMARY KEY NOT ENFORCED,
  temperature DOUBLE,
  ts TIMESTAMP_LTZ(3) METADATA FROM 'timestamp',
  WATERMARK FOR ts AS ts - INTERVAL '5' SECOND
) 
WITH (
  'changelog.mode' = 'upsert'
);
```
```sql
INSERT INTO sensor_events_rekeyed
SELECT
  sensor_id,
  temperature,
  ts
FROM sensor_events;
```

```sql
CREATE TABLE sensor_output AS
SELECT
  sensor_id,
  AVG(temperature) AS avg_temp
FROM sensor_events_rekeyed
GROUP BY sensor_id;
```
### 3. Format Conversions
#### JSON → AVRO
```sql
CREATE TABLE sensor_events_avro (
  sensor_id STRING,
  temperature DOUBLE,
  ts TIMESTAMP(3)
) WITH (
  'value.format' = 'avro-registry'
);

INSERT INTO sensor_events_avro
SELECT sensor_id, temperature,ts FROM sensor_events;
```

### 4. Handling Nested JSON
```sql
CREATE TABLE `NestedTopic` (
  `id` STRING,
  `nested_JSON` ARRAY< ROW< `type` STRING NOT NULL, `value` STRING NOT NULL > NOT NULL > 
)
DISTRIBUTED BY HASH(`id`)
INTO 2 BUCKETS WITH ('changelog.mode' = 'retract', 
  'connector' = 'confluent', 
  'key.format' = 'avro-registry', 
  'value.format' = 'avro-registry')
```
```sql
INSERT INTO NestedTopic VALUES
(
  'id-1',
  ARRAY[
    ROW('temperature', '25.3'),
    ROW('humidity', '60')
  ]
),
(
  'id-2',
  ARRAY[
    ROW('vibration', 'high'),
    ROW('fuel_rate', '12.5')
  ]
),
(
  'id-3',
  ARRAY[
    ROW('temperature', '29.8'),
    ROW('pressure', '1013'),
    ROW('status', 'active')
  ]
),
(
  'id-4',
  ARRAY[
    ROW('temperature', '18.4'),
    ROW('humidity', '75'),
    ROW('battery', 'low')
  ]
),
(
  'id-5',
  ARRAY[
    ROW('vibration', 'medium'),
    ROW('fuel_rate', '9.3'),
    ROW('status', 'idle')
  ]
),
(
  'id-6',
  ARRAY[
    ROW('temperature', '32.1'),
    ROW('humidity', '55'),
    ROW('pressure', '1009'),
    ROW('alert', 'overheat')
  ]
);

```
```sql
SELECT
   t.id,
   t.nested_JSON
FROM `NestedTopic` t
WHERE JSON_QUERY(JSON_STRING(t.nested_JSON), '$[*].type') LIKE '%vibration%'
```
### 5. Deduplication
```sql
INSERT INTO sensor_events VALUES
  ('sensor-1', 24.5, TIMESTAMP '2025-08-26 14:00:00'),
  ('sensor-2', 30.2, TIMESTAMP '2025-08-26 14:01:00'),
  ('sensor-3', 28.9, TIMESTAMP '2025-08-26 14:02:00'),
  -- duplicates
  ('sensor-1', 24.5, TIMESTAMP '2025-08-26 14:00:00'),
  ('sensor-2', 30.2, TIMESTAMP '2025-08-26 14:01:00'),
  ('sensor-3', 28.9, TIMESTAMP '2025-08-26 14:02:00'),
  ('sensor-2', 30.2, TIMESTAMP '2025-08-26 14:01:00');
```
```sql
SELECT *
FROM (
  SELECT *,
    ROW_NUMBER() OVER (PARTITION BY sensor_id ORDER BY ts DESC) AS rownum
  FROM sensor_events
)
WHERE rownum = 1;
```
```sql
SELECT DISTINCT sensor_id, temperature, ts
FROM sensor_events_rekeyed;

```
```sql
SELECT sensor_id, temperature, ts
FROM (
  SELECT *,
         ROW_NUMBER() OVER (
           PARTITION BY sensor_id, temperature, ts
           ORDER BY ts DESC
         ) AS rownum
  FROM sensor_events_rekeyed
)
WHERE rownum = 1;
```

```sql
CREATE TABLE sensor_events_deduped AS
SELECT sensor_id, temperature, ts
FROM (
  SELECT *,
    ROW_NUMBER() OVER (PARTITION BY sensor_id ORDER BY ts DESC) AS rownum
  FROM sensor_events_rekeyed
)
WHERE rownum = 1;
```

### 6. Data Enrichment (Join with Reference Data)

```sql
CREATE TABLE sensor_metadata (
  sensor_id STRING PRIMARY KEY NOT ENFORCED,
  location STRING
) WITH (
  'connector' = 'confluent' 
);

```

```sql
INSERT INTO sensor_metadata VALUES
  ('sensor-1', 'Jakarta'),
  ('sensor-2', 'Bandung'),
  ('sensor-3', 'Surabaya');

```

```sql
SELECT e.sensor_id, e.temperature, d.location
FROM sensor_events e
LEFT JOIN sensor_metadata d
ON e.sensor_id = d.sensor_id;
```

### 7. Data Transformation
```sql
SELECT sensor_id,
       temperature * 1.8 + 32 AS temperature_f
FROM sensor_events;
```

### 8. Filtering
```sql
SELECT *
FROM sensor_events
WHERE temperature > 30;
```

### 9. Aggregation
```sql
SELECT sensor_id,
       COUNT(*) AS event_count,
       AVG(temperature) AS avg_temp
FROM sensor_events
GROUP BY sensor_id;
```

### 10. Serialization
```sql
CREATE TABLE serialized_avro (
  sensor_id STRING,
  temperature DOUBLE
) WITH (
  'connector' = 'kafka',
  'topic' = 'serialized_topic',
  'format' = 'avro'
);
```

### 11. Denormalization
```sql
SELECT sensor_id,
       MAX(temperature) AS max_temp,
       MIN(temperature) AS min_temp,
       AVG(temperature) AS avg_temp
FROM sensor_events
GROUP BY sensor_id;
```

### 12. Windowing
#### Tumbling Window
```sql
SELECT
  sensor_id,
  TUMBLE_START(ts, INTERVAL '1' MINUTE) AS window_start,
  AVG(temperature) AS avg_temp
FROM sensor_events
GROUP BY
  sensor_id,
  TUMBLE(ts, INTERVAL '1' MINUTE);
```

#### Hopping Window
```sql
SELECT
  sensor_id,
  HOP_START(ts, INTERVAL '5' MINUTE, INTERVAL '1' HOUR) AS window_start,
  COUNT(*) AS cnt
FROM sensor_events
GROUP BY
  sensor_id,
  HOP(ts, INTERVAL '5' MINUTE, INTERVAL '1' HOUR);
```

#### Sliding Window
```sql
SELECT
  sensor_id,
  SESSION_START(ts, INTERVAL '10' MINUTE) AS session_start,
  COUNT(*) AS cnt
FROM sensor_events
GROUP BY
  sensor_id,
  SESSION(ts, INTERVAL '10' MINUTE);
```

---

## 🚀 Usage
Clone this repository and open `cheatsheet.md` for ready-to-use Flink SQL commands on Confluent Cloud.

```bash
git clone https://github.com/<your-repo>/flink-cheatsheet.git
cd flink-cheatsheet
open cheatsheet.md
```

---

## 📚 References
- [Confluent Cloud Documentation](https://docs.confluent.io/)
- [Apache Flink SQL Reference](https://nightlies.apache.org/flink/flink-docs-release-1.18/docs/dev/table/sql/overview/)
- [Flink on Confluent Cloud](https://www.confluent.io/)

---

✨ Happy Streaming with Flink & Confluent Cloud!


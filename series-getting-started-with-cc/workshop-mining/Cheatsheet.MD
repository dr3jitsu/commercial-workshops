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
  ('sensor-3', 28.9, TIMESTAMP '2025-08-26 14:02:00');
```

### 2. Insert & Select Data
```sql
INSERT INTO sensor_output
SELECT sensor_id, AVG(temperature) AS avg_temp
FROM sensor_events
GROUP BY sensor_id;
```

### 3. Format Conversions
#### JSON → AVRO
```sql
CREATE TABLE avro_sink (
  sensor_id STRING,
  temperature DOUBLE
) WITH (
  'connector' = 'kafka',
  'topic' = 'avro_topic',
  'format' = 'avro'
);

INSERT INTO avro_sink
SELECT sensor_id, temperature FROM sensor_events;
```

#### CSV → JSON
```sql
CREATE TABLE csv_source (
  sensor_id STRING,
  temperature DOUBLE
) WITH (
  'connector' = 'kafka',
  'topic' = 'csv_topic',
  'format' = 'csv'
);

CREATE TABLE json_sink (
  sensor_id STRING,
  temperature DOUBLE
) WITH (
  'connector' = 'kafka',
  'topic' = 'json_topic',
  'format' = 'json'
);

INSERT INTO json_sink
SELECT * FROM csv_source;
```

### 4. Handling Nested JSON
```sql
SELECT
  nested.field1,
  nested.field2,
  nested.inner.field3
FROM nested_json_table;
```

### 5. Deduplication
```sql
SELECT *
FROM (
  SELECT *,
    ROW_NUMBER() OVER (PARTITION BY sensor_id ORDER BY ts DESC) AS rownum
  FROM sensor_events
)
WHERE rownum = 1;
```

### 6. Data Enrichment (Join with Reference Data)
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


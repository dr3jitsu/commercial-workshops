# Workshop: Build an Agentic AI Data Streaming Pipeline with Confluent Data Streaming Platform

This workshop is a **hands-on lab** where you’ll learn how to combine real-time data streaming with agentic AI (LLMs) to make automated, intelligent decisions.

We’ll walk step-by-step through building a **streaming pipeline using Confluent Cloud (Kafka + Flink)**, where live data is processed, enriched, and then passed to an AI model that evaluates it and returns actionable results.

Instead of just moving data around, you’ll see how modern platforms can react to events in real time—for example, automatically deciding whether a mortgage application should be APPROVED or DENIED based on streaming credit, payment, and application data.

Goal: Show how to design and run an intelligent, event-driven decision system by combining Kafka, Flink, and LLMs in Confluent Cloud.

---

## Table of Contents
1. [Objectives](#objectives)
2. [Prerequisites](#prerequisites)
3. [Step 1 — Create/Prepare Your Confluent Cloud Account](#step-1--createprepare-your-confluent-cloud-account)
4. [Step 2 — Install Confluent Cloud CLI](#step-2--install-confluent-cloud-cli)
5. [Step 3 — Create Environment, Kafka Cluster, and Flink Compute Pool](#step-3--create-environment-kafka-cluster-and-flink-compute-pool)
6. [Step 4 — Create API Keys](#step-4--create-api-keys)
7. [Step 5 — Create Kafka Topics](#step-5--create-kafka-topics)
8. [Step 6 — Create Datagen Source Connectors (2x)](#step-6--create-datagen-source-connectors-2x)
   - [6.1 SensorData Connector](#61-sensordata-connector)
   - [6.2 ProductionStream Connector](#62-productionstream-connector)
9. [Step 7 — Flink Stream Processing](#step-7--flink-stream-processing)
   - [7.1 Open Flink SQL Workspace](#71-open-flink-sql-workspace)
   - [7.2 Create and Populate Reference Tables (mines and equipment)](#72-create-and-populate-reference-tables-mines-and-equipment)
   - [7.3 Rekey and Prepare Streaming Topics (sensordata and productionstream)](#73-rekey-and-prepare-streaming-topics-sensordata-and-productionstream)
   - [7.4 Enrich Sensor Data with Equipment and Mine Metadata](#74-enrich-sensor-data-with-equipment-and-mine-metadata)
   - [7.5 Aggregate Sensor Data with Tumbling Windows](#75-aggregate-sensor-data-with-tumbling-windows)
   - [7.6 Verify Topics and Explore Streaming Data](#76-verify-topics-and-explore-streaming-data)
   - [7.7 Enrich & Aggregate: Equipment Performance](#77-enrich-&-aggregate:-equipment-performance)
10. [Step 8 — Flink Agentic AI (LLM Inference)](#step-8--flink-agentic-ai-llm-inference)
    - [8.1 Prepare LLM Access (Google AI)](#81-prepare-llm-access-google-ai)
    - [8.2 Create Model Connection via Confluent CLI](#82-create-model-connection-via-confluent-cli)
    - [8.3 Create AI Model in Flink SQL](#83-create-ai-model-in-flink-sql)
    - [8.4 Invoke the Model](#84-invoke-the-model)
    - [8.5 Optional: 2nd Agent for Credit Score](#85-optional-2nd-agent-for-credit-score)
11. [Verification & Expected Results](#verification--expected-results)
12. [Cleanup](#cleanup)
13. [Notes & Tips](#notes--tips)


---

## Objectives
Design and implement a **real-time streaming pipeline** where **Flink** processes and enriches mortgage application data, and an **LLM** automatically outputs approval decisions. You will:
- Generate sample data with **Datagen Connectors**.
- Build Flink SQL transformations (rekey, dedupe, aggregate, join).
- Connect Flink to **Google AI** and run an **Agent** that returns **APPROVE/DENY** plus reasoning.

---

## Prerequisites
- A **Confluent Cloud** account.
- **Confluent CLI** installed.
- Access to **Google AI (Gemini) API** (you may use the provided sample endpoint/key for the lab or your own).
- A modern browser.

---

## Step 1 — Create/Prepare Your Confluent Cloud Account
1. Sign up and log in to **Confluent Cloud**.
2. Open **Billing & payment** (menu at top-right).
3. Under **Payment details & contacts**, enter promo code **`CONF`** to delay entering a credit card for 30 days (for lab purposes).

> You can return to the billing UI later to add a payment method when needed.

---

## Step 2 — Install Confluent Cloud CLI
Install the CLI for your OS using the official guide:  
https://docs.confluent.io/confluent-cli/current/install.html

---

## Step 3 — Create Environment, Kafka Cluster, and Flink Compute Pool
1. In Confluent Cloud, click **+ Add Environment** → enter a name → **Create**.
2. Inside the environment, click **Create cluster**.
3. Choose **Basic Cluster**, and set:
   - **Cloud:** AWS
   - **Region:** **Singapore (asia-southeast1)**
   - **Availability:** **Single Zone**
   - Name your cluster → **Launch cluster**.
4. At the environment level, create a Flink compute pool : **Flink** → **Create Compute Pool**:
   - **Cloud Provider:** AWS
   - **Region:** **Singapore (asia-southeast1)** (must match the Kafka cluster)
   - **Pool name:** choose a name
   - **Max CFU:** `10`
   - **Create**.

---

## Step 4 — Create API Keys
1. Go to your **Kafka Cluster** → **API Keys** → **Add Key**.
2. **Select account for API Key:** choose **My Account**.
3. **Next** → **Download** your **API Key** and **Secret**. **Save them securely**; you’ll need them for connectors.

---

## Step 5 — Create Kafka Topics
Create 2 topics (Partitions: `1` for each):

- `sensordata`
- `productionstream`

**UI path:** Cluster → **Topics** → **+ Add Topic** → fill in details → **Create with defaults**.
Skip the data contract creation step

---

## Step 6 — Create Datagen Source Connectors (2x)
You’ll create a **Datagen Source** connector per topic to continuously generate realistic sample data.

**UI path:** Cluster → **Connectors** → **+ Add Connector** → **Sample Data – Datagen Source** → configure.

> For all connectors below:
> - **Kafka Credentials:** *Use existing API key* → supply your **API Key** and **Secret** (from Step 4).
> - **Output Record Value Format:** `JSON_SR`.
> - **Select a Schema:** **Provide your own schema** (paste from below).
> - **Advanced Configuration → Max interval between messages (ms):** `10000`.
> - **Tasks:** `1` (default is fine).
> - **Name** each connector appropriately.

### 6.1 SensorData Connector
- **Topic Selection:** `sensordata`
- **Name:** `sensordata`

**Schema:**
```json
{
  "type": "record",
  "name": "sensordata",
  "namespace": "mining.demo",
  "fields": [
    {
      "name": "sensordata_id",
      "type": {
        "type": "int",
        "arg.properties": {
         "range": {
            "min": 0,
            "max": 1000000000,
            "step": 1
          }
        }
      }
    },
    {
      "name": "equipment_id",
      "type": {
        "type": "string",
        "arg.properties": {
          "options": [
            "101", "102", "103",
            "201", "202",
            "301", "302",
            "401",
            "501", "502"
          ]
        }
      }
    },
    {
      "name": "temperature_c",
      "type": {
        "type": "float",
        "arg.properties": {
          "range": { "min": -20.0, "max": 120.0 }
        }
      }
    },
    {
      "name": "vibration_level",
      "type": {
        "type": "float",
        "arg.properties": {
          "range": { "min": 0.0, "max": 1.0 }
        }
      }
    },
    {
      "name": "fuel_rate_lph",
      "type": {
        "type": "float",
        "arg.properties": {
          "range": { "min": 0.0, "max": 500.0 }
        }
      }
    }
  ]
}

```

### 6.2 ProductionStream Connector
- **Topic Selection:** `productionstream`
- **Name:** `productionstream`

**Schema:**
```json
{
  "type": "record",
  "name": "productionstream",
  "namespace": "mining.demo",
  "fields": [
    {
      "name": "production_id",
      "type": {
        "type": "int",
        "arg.properties": {
          "range": {
            "min": 0,
            "max": 1000000000,
            "step": 1
          }
        }
      }
    },
    {
      "name": "equipment_id",
      "type": {
        "type": "string",
        "arg.properties": {
          "options": [
            "101", "102", "103",
            "201", "202",
            "301", "302",
            "401",
            "501", "502"
          ]
        }
      }
    },
    {
      "name": "ore_tons",
      "type": {
        "type": "float",
        "arg.properties": {
          "range": { "min": 0.1, "max": 50.0 }
        }
      }
    },
    {
      "name": "mineral_grade_pct",
      "type": {
        "type": "float",
        "arg.properties": {
          "range": { "min": 0.1, "max": 100.0 }
        }
      }
    }
  ]
}

```
Once all three connectors are running, you should see data flowing into the topics.

---

## Step 7 — Flink Stream Processing

### 7.1 Open Flink SQL Workspace
1. **Environments** → choose your environment.
2. **Flink** → choose your **compute pool** → **Open SQL workspace**.
3. Top-right of the SQL workspace:
   - **Catalog:** your **Environment**
   - **Database:** your **Kafka Cluster**
4. Put **each query in a separate query tab/box** (click the **+** icon per query).

### 7.2 **Create and Populate Reference Tables (mines and equipment)**

Before processing real-time streams, we need reference data that describes mines and their equipment. These tables provide contextual metadata that will later be used to enrich raw sensor and production streams.

The mines table stores information about mining sites (name, location, mineral type, and production capacity).

The equipment table stores details about the mining equipment assigned to each mine (type, model, and operational status).

This reference data ensures that when we join streaming events, each record can be enriched with descriptive mine and equipment information.


1. Create table mines in query tab.
```sql
CREATE TABLE mines (
  mine_id INT,
  mine_name STRING,
  location STRING,
  mineral_type STRING,
  capacity_tons INT,
  PRIMARY KEY (mine_id) NOT ENFORCED
);
```

2. Insert Records to table mines
```sql
INSERT INTO mines VALUES
  (1, 'Grasberg Mine', 'Papua, ID', 'Copper/Gold', 2000000),
  (2, 'Escondida', 'Chile', 'Copper', 1800000),
  (3, 'Carajás Mine', 'Pará, Brazil', 'Iron Ore', 1500000),
  (4, 'Khewra Mine', 'Punjab, Pakistan', 'Rock Salt', 800000),
  (5, 'Kiruna Mine', 'Kiruna, Sweden', 'Iron Ore', 1000000);
```
3. Create table equipment
```sql
CREATE TABLE equipment (
  equipment_id INT,
  mine_id INT,
  eq_type STRING,
  eq_model STRING,
  status STRING,
  PRIMARY KEY (equipment_id) NOT ENFORCED
); 
```

4. Insert Records to table equipment
```sql
INSERT INTO equipment VALUES
  (101, 1, 'Haul Truck', 'CAT 793F', 'Active'),
  (102, 1, 'Excavator', 'Hitachi EX5600', 'Active'),
  (103, 1, 'Crusher', 'Metso C200', 'Active'),
  (201, 2, 'Drill Rig', 'Sandvik D45', 'Active'),
  (202, 2, 'Haul Truck', 'Komatsu 930E', 'Active'),
  (301, 3, 'Conveyor Belt', 'ThyssenKrupp', 'Active'),
  (302, 3, 'Excavator', 'Liebherr R9800', 'Active'),
  (401, 4, 'Loader', 'CAT 992K', 'Active'),
  (501, 5, 'Drill Rig', 'Atlas Copco DML', 'Active'),
  (502, 5, 'Haul Truck', 'Volvo R100E', 'Active');
```
You’ll notice the new topics have been created.

### 7.3 **Rekey and Prepare Streaming Topics (sensordata and productionstream)**

Raw Kafka topics (sensordata and productionstream) need to be transformed into Flink-managed tables with proper primary keys, event-time processing, and watermarks. This step ensures that downstream operations like joins, aggregations, and windowed analysis work correctly.

We will:

Rekey the sensordata stream by using sensordata_id as the primary key, while extracting event time from the row metadata.

Rekey the productionstream stream by using production_id as the primary key, also with event-time extraction.

This prepares both streams for accurate time-based processing in Flink SQL.

1. Create the sensordata_rekeyed table

```sql
CREATE TABLE sensordata_rekeyed (
  sensordata_id INT NOT NULL PRIMARY KEY NOT ENFORCED,
  equipment_id STRING,
  temperature_c FLOAT,
  vibration_level FLOAT,
  fuel_rate_lph FLOAT,
  event_time TIMESTAMP_LTZ(3) METADATA FROM 'timestamp',
  WATERMARK FOR event_time AS event_time - INTERVAL '5' SECOND
) DISTRIBUTED BY (sensordata_id) INTO 1 BUCKETS
WITH (
  'changelog.mode' = 'upsert'
);
```
2. Populate sensordata_rekeyed from the raw topic

```sql
INSERT INTO sensordata_rekeyed
SELECT
  sensordata_id,
  equipment_id,
  temperature_c,
  vibration_level,
  fuel_rate_lph,
  $rowtime as event_time
FROM sensordata;
```

3. Create the productionstream_rekeyed table

```sql
CREATE TABLE productionstream_rekeyed (
  production_id INT NOT NULL PRIMARY KEY NOT ENFORCED,
  equipment_id STRING,
  ore_tons FLOAT,
  mineral_grade_pct FLOAT,
  event_time TIMESTAMP_LTZ(3) METADATA FROM 'timestamp'
)
DISTRIBUTED BY (production_id) INTO 1 BUCKETS
WITH (
  'changelog.mode' = 'upsert'
);
```
4. Populate productionstream_rekeyed from the raw topic
```sql
INSERT INTO productionstream_rekeyed
SELECT
  production_id,
  equipment_id,
  ore_tons,
  mineral_grade_pct,
  $rowtime as event_time
FROM productionstream;
```
Both sensordata_rekeyed and productionstream_rekeyed tables are now keyed, time-aware streams that support windowing, aggregation, and enrichment in the next steps.

### 7.4 **Enrich Sensor Data with Equipment and Mine Metadata**

Now that the raw sensor data (sensordata_rekeyed) has been rekeyed and prepared with event time, we can enrich it with contextual information from the equipment and mines reference tables.

This step creates a denormalized SensorEnriched table, which includes:

Sensor readings (temperature, vibration, fuel rate).

Derived metrics like temperature in Fahrenheit and a vibration alert flag.

Equipment metadata (type, model, status).

Mine metadata (name, location, mineral type, production capacity).

By joining streams with static reference data, downstream analytics and monitoring queries can leverage both real-time values and descriptive context.

1. Create the SensorEnriched table

```sql
CREATE TABLE SensorEnriched (
    sensordata_id INT NOT NULL PRIMARY KEY NOT ENFORCED,
    equipment_id INT,
    equipment_type STRING,
    equipment_model STRING,
    equipment_status STRING,
    mine_id INT,
    mine_name STRING,
    location STRING,
    mineral_type STRING,
    capacity_tons DOUBLE,
    temperature_c FLOAT,
    temperature_f FLOAT,
    vibration_level FLOAT,
    vibration_alert INT,
    fuel_rate_lph FLOAT,
    event_time TIMESTAMP_LTZ(3) METADATA FROM 'timestamp',
    WATERMARK FOR event_time AS event_time - INTERVAL '5' SECOND
)
DISTRIBUTED BY (sensordata_id) INTO 1 BUCKETS
WITH (
  'changelog.mode' = 'upsert'
);

```
2. Populate SensorEnriched with joined and derived data

```sql
INSERT INTO SensorEnriched
SELECT
    s.sensordata_id,
    CAST(s.equipment_id AS INT),
    e.eq_type AS equipment_type,
    e.eq_model AS equipment_model,
    e.status AS equipment_status,
    e.mine_id,
    m.mine_name,
    m.location,
    m.mineral_type,
    m.capacity_tons,
    s.temperature_c,
    s.temperature_c * 9 / 5 + 32 AS temperature_f,
    s.vibration_level,
    CASE 
        WHEN s.vibration_level > 0.05 THEN 1 
        ELSE 0 
    END AS vibration_alert,
    s.fuel_rate_lph,
    s.`event_time`
FROM sensordata_rekeyed s
LEFT JOIN equipment e
    ON CAST(s.equipment_id AS INT) = e.equipment_id
LEFT JOIN mines m
    ON e.mine_id = m.mine_id;
```
### 7.5 **Aggregate Sensor Data with Tumbling Windows**

Once the sensor data has been enriched with equipment and mine context, the next step is to summarize the readings over fixed time intervals. This makes it easier to monitor equipment health and performance trends in real time.

We use a 5-minute tumbling window to aggregate data for each piece of equipment. Within each window, we compute:

Counts: number of sensor readings.

Averages: temperature, vibration, fuel consumption.

Sums: total fuel consumption, number of vibration alerts.

Window boundaries: human-readable start and end times of the aggregation window.

This results in a time-series summary table (SensorEnriched_agg) that can be directly used for dashboards, monitoring, or anomaly detection.

1. Create the SensorEnriched_agg table with windowed aggregates

```sql
CREATE TABLE SensorEnriched_agg AS
SELECT
  CAST(equipment_id AS INT) AS equipment_id,
  equipment_type,
  equipment_model,
  equipment_status,
  mine_id,
  mine_name,
  location,
  mineral_type,
  capacity_tons,
  DATE_FORMAT(window_start, 'yyyy-MM-dd HH:mm:ss') AS window_start_str,
  DATE_FORMAT(window_end, 'yyyy-MM-dd HH:mm:ss') AS window_end_str,
  COUNT(*) AS reading_count,
  AVG(temperature_c) AS avg_temperature_c,
  AVG(temperature_f) AS avg_temperature_f,
  AVG(vibration_level) AS avg_vibration_level,
  SUM(vibration_alert) AS vibration_alert_count,
  SUM(fuel_rate_lph) AS sum_fuel_lph,
  AVG(fuel_rate_lph) AS avg_fuel_rate_lph
FROM TABLE(
    TUMBLE(
        TABLE SensorEnriched,
        DESCRIPTOR(event_time),
        INTERVAL '5' MINUTES
    )
)
GROUP BY
  CAST(equipment_id AS INT),
  equipment_type,
  equipment_model,
  equipment_status,
  mine_id,
  mine_name,
  location,
  mineral_type,
  capacity_tons,
  window_start,
  window_end;

```

### 7.6 **Verify Topics and Explore Streaming Data**

Before proceeding with more complex joins or AI integrations, it is important to validate that the topics and tables are being populated correctly. In this step, we will:

Check that the rekeyed and enriched tables exist and are active.

Explore sample records from each table to confirm that data is flowing as expected.

Validate that enrichment and aggregations are producing meaningful values.

1. List available topics in Flink SQL workspace

```sql
SHOW TABLES;
```

2. Explore data in the raw rekeyed streams

```sql
-- Sensor data after rekeying
SELECT * FROM sensordata_rekeyed LIMIT 10;

-- Production stream after rekeying
SELECT * FROM productionstream_rekeyed LIMIT 10;
```

3. Explore enriched sensor data

```sql
SELECT * FROM SensorEnriched LIMIT 10;

```
4. Explore aggregated summaries

```sql
SELECT * FROM SensorEnriched_agg LIMIT 10;

```
### 7.7 Enrich & Aggregate: Equipment Performance

In this step, we create an aggregated EquipmentPerformance table that combines productionstream, sensordata, and static reference data (equipment and mines) to compute real-time performance metrics per equipment.

Like we did for sensor aggregation, we’ll use a 5-minute tumbling window to calculate metrics such as ore mined, average grade, fuel efficiency, and vibration alerts for each piece of equipment.

This provides a time-bucketed performance summary that can be used for dashboards, monitoring, or feeding downstream AI agents.

1. Create the EquipmentPerformance table with windowed aggregates

```sql
CREATE TABLE EquipmentPerformance AS
SELECT
    e.equipment_id,
    COALESCE(e.eq_type, 'UNKNOWN') AS equipment_type,
    e.eq_model AS equipment_model,
    e.status AS equipment_status,
    e.mine_id,
    m.mine_name,
    m.location,
    m.mineral_type,
    m.capacity_tons,

    -- Window boundaries
    DATE_FORMAT(window_start, 'yyyy-MM-dd HH:mm:ss') AS window_start_str,
    DATE_FORMAT(window_end, 'yyyy-MM-dd HH:mm:ss') AS window_end_str,

    -- Production metrics
    SUM(p.ore_tons) AS total_ore_tons,
    AVG(p.mineral_grade_pct) AS avg_mineral_grade,
    COUNT(p.production_id) AS production_events,

    -- Sensor metrics
    AVG(s.fuel_rate_lph) AS avg_fuel_rate_lph,
    AVG(s.temperature_c) AS avg_temperature_c,
    AVG(s.vibration_level) AS avg_vibration_level,
    SUM(CASE WHEN s.vibration_level > 0.05 THEN 1 ELSE 0 END) AS vibration_alert_count,

    -- Calculated efficiency (ore per unit of fuel)
    SUM(p.ore_tons) / NULLIF(SUM(s.fuel_rate_lph), 0) AS fuel_efficiency

FROM TABLE(
    TUMBLE(
        TABLE sensordata_rekeyed s
        LEFT JOIN productionstream_rekeyed p
          ON CAST(s.equipment_id AS INT) = CAST(p.equipment_id AS INT)
        LEFT JOIN equipment e
          ON CAST(s.equipment_id AS INT) = e.equipment_id
        LEFT JOIN mines m
          ON e.mine_id = m.mine_id,
        DESCRIPTOR(s.event_time),
        INTERVAL '5' MINUTES
    )
)
GROUP BY
    e.equipment_id,
    e.eq_type,
    e.eq_model,
    e.status,
    e.mine_id,
    m.mine_name,
    m.location,
    m.mineral_type,
    m.capacity_tons,
    window_start,
    window_end;

```

2. Verify EquipmentPerformance data

```sql
SELECT * FROM EquipmentPerformance LIMIT 10;

```
---

## Step 8 — Flink Agentic AI (LLM Inference)

### 8.1 Prepare LLM Access (Google AI)
You have two options:

- **Option A (Lab sample):**
  - **API Key:** `AIzaSyBidz11vgKLz_1RMgtZ0FDG1x`
  - **Gemini Endpoint:** `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash:generateContent`

- **Option B (Use your own):**
  1. Go to https://aistudio.google.com → **Get API Key** → create or copy your key.
  2. Note the **API Endpoint** (the URL before `?key=GEMINI_API_KEY`), e.g.  
     `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash:generateContent`

> **Security Tip:** Treat API keys as secrets. Prefer environment variables or secret managers in real projects.

### 8.2 Create Model Connection via Confluent CLI
1. Login to Confluent:
   ```bash
   confluent login
   ```
2. Ensure you’re on the correct environment:
   ```bash
   confluent environment list
   confluent environment use <env-id>
   ```
3. Create the **Flink connection** (replace placeholders with your values where needed):
   ```bash
   confluent flink connection create mortgageagent-connection --cloud GCP 
   --region <your flink region id> --environment <your Confluent environment id>      
   --type googleai      
   --endpoint https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash:generateContent      
   --api-key <your Google AI API key>
   ```

> Ensure the **endpoint** matches what you noted in 8.1.

### 8.3 Create AI Model in Flink SQL
Create a model with inputs/outputs and a system prompt describing the decision logic.

```sql
CREATE MODEL AgentMortgageModel
INPUT (
  `details` VARCHAR(2147483647)
)
OUTPUT (
  `decisionreasoning` VARCHAR(2147483647)
)
WITH (
  'googleai.connection' = 'mortgageagent-connection',

  'googleai.system_prompt' = '
  Your task is to evaluate mortgage applications and provide a final decision: APPROVE or DENY.  

  Input data will include:
  - application_id
  - customer_email
  - mortgage_type
  - mortgage_value
  - credit score
  - newest payment month year
  - credit limit

  Evaluation logic is as follows:
  1. Credit Score (Highest Priority): Higher is better; a score of 80 or higher is a strong positive, 70 or lower is a red flag.  
  2. Mortgage Value: Lower is better; a value of 500,000,000 or less is a positive factor.  
  3. Mortgage Type: floating_rate is more favorable than fixed_rate.  
  4. Credit Limit (Lowest Priority): Higher is better; a limit of 500,000,000 or more is a positive factor.  

  The final output must be in the format:
  application id : {application id}.  
  decision : {Approve / Deny}.  
  reasoning : {your reasoning}.  
  ',

  'provider' = 'googleai',

  'task' = 'text_generation'
);

```

### 8.4 Invoke the Model
Invoke the model against the joined feed and return the decision + reasoning.

```sql
SELECT application_id, decisionreasoning
FROM mortgage_submission_feed,
LATERAL TABLE(ML_PREDICT('AgentMortgageModel', CONCAT(
  'Application ID: ', CAST(application_id AS STRING),
  ', Mortgage Type: ', mortgage_type,
  ', Mortgage Value: IDR', CAST(mortgage_value AS STRING),
  ', Credit Score: ', CAST(credit_score AS STRING),
  ', Latest Payment: ', CAST(newest_payment_month_year AS STRING),
  ', Credit Limit: ', CAST(credit_limit AS STRING)
)));
```

### 8.5 Optional: 2nd Agent for Credit Score
Create a **CreditScoreAgent** that proposes a new credit score based on latest payment recency and usage.

**Model:**
```sql
CREATE MODEL CreditScoreAgent
INPUT (
  `details` VARCHAR(2147483647)
)
OUTPUT (
  `newcreditscore` VARCHAR(2147483647)
)
WITH (
  'googleai.connection' = 'mortgageagent-connection',

  'googleai.system_prompt' = '
  Your task is to propose a new credit score for the customer; whether new or stay as it is.  

  This agent will take a user’s current data and propose an updated credit score.  

  It follows these rules:
  1. If the latest payment month date is within the last 6 months and the credit usage percentage is below 50%, increase the current credit score by 10.  
  2. If the latest payment month date is older than 6 months ago but the credit usage percentage is still below 50%, increase the current credit score by 5.  
  3. In all other scenarios (e.g., credit usage is 50% or more), the credit score remains unchanged.  

  The agent will need the following inputs:
  - User Email (for identification)  
  - Latest Payment Month Date  
  - Current Credit Score  
  - Credit Usage Percentage  

  The agent will output a single integer value: Proposed Credit Score
  ',

  'provider' = 'googleai',

  'task' = 'text_generation'
);

```

**Invoke:**
```sql
SELECT customer_email, newcreditscore
FROM enriched_topic_credit_payment_history,
LATERAL TABLE(ML_PREDICT('CreditScoreAgent', CONCAT(
  'Credit Score: ', CAST(credit_score AS STRING),
  ', Latest Payment Month Date: ', newest_payment_month_year,
  ', Credit Usage Percentage: ', CAST(credit_usage_percentage AS STRING),
  ', User Email: ', customer_email
)));
```

---

## Verification & Expected Results
- **Topics** show incoming records for all three sources.
- **`credit_score_rekeyed`, `mortgage_application_rekeyed`, `payment_history_rekeyed`** contain keyed data.
- **`credit_score_deduped`** keeps only the latest row per `customer_email`.
- **`enriched_topic_credit_payment_history`** shows aggregated payments and credit metrics with `newest_payment_month_year` as `YYYY-MM`.
- **`mortgage_submission_feed`** shows the combined application + profile data.
- **Model invocation** returns rows with **`application_id`** and **`decisionreasoning`** such as:
  ```
  application id : 123. decision : APPROVE. reasoning : ...
  ```

---

## Cleanup
- **Pause/stop connectors** to avoid unnecessary usage.
- **Drop** Flink tables created for the lab if you are done.
- Optionally **delete topics** and/or **delete the environment** in Confluent Cloud.

---

## Notes & Tips
- Keep Kafka and Flink **cloud/region** aligned (e.g., **GCP / asia-southeast2**) to avoid cross-region latency or unsupported configs.
- If data doesn’t appear in Flink tables, verify **topic names**, **schemas**, and **connector status**.
- If CTAS or joins fail, ensure **PKs** are set as shown and **changelog.mode** is compatible (`upsert` vs `append`).
- Never commit **API keys** to version control. Use **secrets managers** in production.


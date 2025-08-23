# Workshop: Build an Agentic AI Data Streaming Pipeline with Confluent Data Streaming Platform

This README is a step‑by‑step lab guide to build a **real‑time streaming pipeline with agentic AI** for automated **mortgage application** processing using **Confluent Cloud (Kafka + Flink)** and an LLM connection (Google AI / Gemini). It is organized so you can follow each step from account setup through model inference.

> **What you’ll build**
>
> - Kafka topics that simulate mortgage-related data (credit score, mortgage applications, and payment history).
> - Flink SQL pipelines to **rekey**, **deduplicate**, **aggregate**, and **join** those streams.
> - An **Agentic AI** integration with Flink to make **APPROVE/DENY** decisions for mortgage applications.
>
> **Goal:** demonstrate intelligent, event-driven decision-making with modern data platforms.

---

## Table of Contents
1. [Objectives](#objectives)
2. [Prerequisites](#prerequisites)
3. [Step 1 — Create/Prepare Your Confluent Cloud Account](#step-1--createprepare-your-confluent-cloud-account)
4. [Step 2 — Install Confluent Cloud CLI](#step-2--install-confluent-cloud-cli)
5. [Step 3 — Create Environment, Kafka Cluster, and Flink Compute Pool](#step-3--create-environment-kafka-cluster-and-flink-compute-pool)
6. [Step 4 — Create API Keys](#step-4--create-api-keys)
7. [Step 5 — Create Kafka Topics](#step-5--create-kafka-topics)
8. [Step 6 — Create Datagen Source Connectors (3x)](#step-6--create-datagen-source-connectors-3x)
9. [Step 7 — Flink Stream Processing](#step-7--flink-stream-processing)
   - [7.1 Open Flink SQL Workspace](#71-open-flink-sql-workspace)
   - [7.2 Rekey topics (create PK) + insert data](#72-rekey-topics-create-pk--insert-data)
   - [7.3 CTAS Example: Deduplicate Credit Score](#73-ctas-example-deduplicate-credit-score)
   - [7.4 Enrich: Payments + Credit Score](#74-enrich-payments--credit-score)
   - [7.5 Join With Mortgage Applications](#75-join-with-mortgage-applications)
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
      "name": "timestamp",
      "type": "string",
      "logicalType": "timestamp-millis"
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
      "name": "timestamp",
      "type": "string",
      "logicalType": "timestamp-millis"
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
5. Create table mines in query tab.
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

6. Insert Records to table mines
```sql
INSERT INTO mines VALUES
  (1, 'Grasberg Mine', 'Papua, ID', 'Copper/Gold', 2000000),
  (2, 'Escondida', 'Chile', 'Copper', 1800000),
  (3, 'Carajás Mine', 'Pará, Brazil', 'Iron Ore', 1500000),
  (4, 'Khewra Mine', 'Punjab, Pakistan', 'Rock Salt', 800000),
  (5, 'Kiruna Mine', 'Kiruna, Sweden', 'Iron Ore', 1000000);
```
7. Create table equipment
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

8. Insert Records to table equipment
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


### 7.2 **Join Table:** Equipment and Mine Join for Unified Metadata
Table **Equipment** and join with **Mines**.
This query creates a table combining equipment details with mine information using an inner join, producing a unified view of equipment and their associated mines. It ensures each equipment record is enriched with mine-specific attributes like name, location, and mineral type.


```sql
CREATE TABLE equipmentwithmines (
  equipment_id INT PRIMARY KEY NOT ENFORCED,
  mine_id INT,
  mine_name STRING,
  location STRING,
  mineral_type STRING,
  capacity_tons BIGINT,
  eq_type STRING,
  eq_model STRING,
  status STRING
)DISTRIBUTED BY (equipment_id) INTO 1 BUCKETS
WITH ('changelog.mode' = 'upsert');
```
```sql
INSERT INTO equipmentwithmines
SELECT
  e.equipment_id,
  e.mine_id,
  m.mine_name,
  m.location,
  m.mineral_type,
  m.capacity_tons,
  e.eq_type,
  e.eq_model,
  e.status
FROM equipment e
INNER JOIN mines m
  ON e.mine_id = m.mine_id;
```

### 7.3 **Join Table:** Sensor Data Enrichment with Equipment and Mine Join
This query creates an enriched sensor data table by joining raw sensor readings with equipment and mine metadata. 
It also adds derived fields like temperature in Fahrenheit and a vibration alert flag for easier monitoring and analysis

```sql
CREATE TABLE SensorEnriched AS
SELECT
    s.sensordata_id,
    s.equipment_id,
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
    s.fuel_rate_lph
FROM `sensordata` s
LEFT JOIN equipment e
    ON CAST(s.equipment_id AS INT) = e.equipment_id
LEFT JOIN mines m
    ON e.mine_id = m.mine_id;
```
```sql
ALTER TABLE SensorEnriched SET ('changelog.mode' = 'append');
```
### 7.4 **Aggregation:** Sensor Data Enriched Aggregation with Tumbling Windows
This query aggregates sensor readings per equipment in 5-minute tumbling windows, calculating averages, counts, and alerts for metrics like temperature, vibration, and fuel usage. It produces time-windowed summaries for monitoring equipment performance.
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
  window_start,
  window_end,
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
        DESCRIPTOR($rowtime),
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
### 7.4 Enrich: CTAS Equipment Performance

This Flink SQL query creates a denormalized, aggregated table EquipmentPerformance that combines data from equipment, mines, sensordata, and productionstream. It calculates per-equipment metrics, including total ore mined, average mineral grade, fuel efficiency, average sensor readings, and vibration alert counts. The query uses joins to include mine information, handles nullable columns with COALESCE, and aggregates the streaming data using SUM, AVG, and COUNT. The resulting table provides a real-time overview of equipment performance for monitoring, analytics, or AI applications.
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

    -- Aggregated metrics
    SUM(p.ore_tons) AS total_ore_tons,
    AVG(p.mineral_grade_pct) AS avg_mineral_grade,
    COUNT(p.production_id) AS production_events,
    AVG(s.fuel_rate_lph) AS avg_fuel_rate_lph,

    -- Calculated column: fuel efficiency (ore_tons / fuel_rate)
    SUM(p.ore_tons) / NULLIF(SUM(s.fuel_rate_lph),0) AS fuel_efficiency,

    -- Sensor metrics
    AVG(s.temperature_c) AS avg_temperature_c,
    AVG(s.vibration_level) AS avg_vibration_level,
    SUM(CASE WHEN s.vibration_level > 0.05 THEN 1 ELSE 0 END) AS vibration_alert_count

FROM equipment e
JOIN mines m
    ON e.mine_id = m.mine_id
LEFT JOIN sensordata s
    ON CAST(s.equipment_id AS INT) = e.equipment_id
LEFT JOIN productionstream p
    ON CAST(p.equipment_id AS INT) = e.equipment_id

-- Optional: group by equipment to aggregate metrics
GROUP BY
    e.equipment_id,
    e.eq_type,
    e.eq_model,
    e.status,
    e.mine_id,
    m.mine_name,
    m.location,
    m.mineral_type,
    m.capacity_tons;

```

XXXXXXXXX

### 7.5 Join With Mortgage Applications
Create a **feed** that combines mortgage applications with the enriched customer profile.

```sql
CREATE TABLE mortgage_submission_feed (
  application_id INT NOT NULL PRIMARY KEY NOT ENFORCED,
  customer_email STRING,
  mortgage_type STRING,
  mortgage_value INT,
  credit_score INT,
  newest_payment_month_year STRING,
  credit_limit INT
) WITH ('changelog.mode' = 'upsert');
```

```sql
INSERT INTO mortgage_submission_feed
SELECT
  ma.application_id,
  ma.customer_email,
  ma.mortgage_type,
  ma.mortgage_value,
  etcph.credit_score,
  etcph.newest_payment_month_year,
  etcph.credit_limit_idr
FROM mortgage_application_rekeyed ma
LEFT JOIN enriched_topic_credit_payment_history etcph
  ON ma.customer_email = etcph.customer_email;
```

Set the table to **append**:
```sql
ALTER TABLE `mortgage_submission_feed` SET ('changelog.mode'='append');
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


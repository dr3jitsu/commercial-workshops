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
   - **Cloud:** Google Cloud
   - **Region:** **Jakarta (asia-southeast2)**
   - **Availability:** **Single Zone**
   - Name your cluster → **Launch cluster**.
4. Create a Flink compute pool: **Flink** → **Create Compute Pool**:
   - **Cloud Provider:** GCP
   - **Region:** **Jakarta (asia-southeast2)** (must match the Kafka cluster)
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
Create three topics (Partitions: `1` for each):

- `Credit_Score`
- `Mortgage_Application`
- `Payment_History`

**UI path:** Cluster → **Topics** → **+ Add Topic** → fill in details → **Create with defaults**.

---

## Step 6 — Create Datagen Source Connectors (3x)
You’ll create a **Datagen Source** connector per topic to continuously generate realistic sample data.

**UI path:** Cluster → **Connectors** → **+ Add Connector** → **Sample Data – Datagen Source** → configure.

> For all connectors below:
> - **Kafka Credentials:** *Use existing API key* → supply your **API Key** and **Secret** (from Step 4).
> - **Output Record Value Format:** `JSON_SR`.
> - **Select a Schema:** **Provide your own schema** (paste from below).
> - **Advanced Configuration → Max interval between messages (ms):** `10000`.
> - **Tasks:** `1` (default is fine).
> - **Name** each connector appropriately.

### 6.1 Credit_Score Connector
- **Topic Selection:** `Credit_Score`
- **Name:** `credit_score`

**Schema:**
```json
{
  "type": "record",
  "name": "CreditScore",
  "namespace": "demo",
  "fields": [
    {
      "name": "customer_email",
      "type": {
        "type": "string",
        "arg.properties": {
          "options": [
            "alex.jose@gmail.com",
            "james.joe@gmail.com",
            "john.doe@gmail.com",
            "lisa.kudrow@gmail.com",
            "jeniffer.aniston@gmail.com",
            "ross.geller@gmail.com",
            "joey.tribbiani@gmail.com",
            "courtney.cox@gmail.com"
          ]
        }
      }
    },
    {
      "name": "income_type",
      "type": {
        "type": "string",
        "arg.properties": {
          "options": [
            "Regular/Monthly",
            "Quarterly",
            "Annual",
            "Infrequent"
          ]
        }
      }
    },
    {
      "name": "credit_score",
      "type": {
        "type": "int",
        "arg.properties": {
          "range": { "min": 50, "max": 100 }
        }
      }
    },
    {
      "name": "credit_usage_percentage",
      "type": {
        "type": "int",
        "arg.properties": {
          "range": { "min": 0, "max": 100 }
        }
      }
    },
    {
      "name": "credit_limit_idr",
      "type": {
        "type": "int",
        "arg.properties": {
          "range": {
            "min": 100000000,
            "max": 1000000000,
            "step": 1000000
          }
        }
      }
    }
  ]
}
```

### 6.2 Mortgage_Application Connector
- **Topic Selection:** `Mortgage_Application`
- **Name:** `mortgage_application`

**Schema:**
```json
{
  "type": "record",
  "name": "MortgageApplication",
  "namespace": "demo",
  "fields": [
    {
      "name": "application_id",
      "type": {
        "type": "int",
        "arg.properties": {
          "iteration": { "start": 100, "max": 150 }
        }
      }
    },
    {
      "name": "customer_email",
      "type": {
        "type": "string",
        "arg.properties": {
          "options": [
            "alex.jose@gmail.com",
            "james.joe@gmail.com",
            "john.doe@gmail.com",
            "lisa.kudrow@gmail.com",
            "jeniffer.aniston@gmail.com",
            "ross.geller@gmail.com",
            "joey.tribbiani@gmail.com",
            "courtney.cox@gmail.com"
          ]
        }
      }
    },
    {
      "name": "mortgage_value",
      "type": {
        "type": "int",
        "arg.properties": {
          "range": {
            "min": 100000000,
            "max": 1000000000,
            "step": 1000000
          }
        }
      }
    },
    {
      "name": "mortgage_type",
      "type": {
        "type": "string",
        "arg.properties": {
          "options": ["fixed_rate", "adjustable_rate"]
        }
      }
    }
  ]
}
```

### 6.3 Payment_History Connector
- **Topic Selection:** `Payment_History`
- **Name:** `payment_history`

**Schema:**
```json
{
  "type": "record",
  "name": "PaymentHistory",
  "namespace": "demo",
  "fields": [
    {
      "name": "customer_email",
      "type": {
        "type": "string",
        "arg.properties": {
          "options": [
            "alex.jose@gmail.com",
            "james.joe@gmail.com",
            "john.doe@gmail.com",
            "lisa.kudrow@gmail.com",
            "jeniffer.aniston@gmail.com",
            "ross.geller@gmail.com",
            "joey.tribbiani@gmail.com",
            "courtney.cox@gmail.com"
          ]
        }
      }
    },
    {
      "name": "payment_month_year",
      "type": {
        "type": "long",
        "arg.properties": {
          "range": {
            "min": 1578614400000,
            "step": 2626560000,
            "max": 1744243200000
          }
        }
      }
    },
    {
      "name": "amount_idr",
      "type": {
        "type": "long",
        "arg.properties": {
          "range": {
            "min": 1000000,
            "step": 100000,
            "max": 100000000
          }
        }
      }
    },
    {
      "name": "application_id",
      "type": {
        "type": "int",
        "arg.properties": {
          "range": { "min": 50, "max": 100 }
        }
      }
    }
  ]
}
```

Once all three connectors are running, you should see data flowing into the three topics.

---

## Step 7 — Flink Stream Processing

### 7.1 Open Flink SQL Workspace
1. **Environments** → choose your environment.
2. **Flink** → choose your **compute pool** → **Open SQL workspace**.
3. Top-right of the SQL workspace:
   - **Catalog:** your **Environment**
   - **Database:** your **Kafka Cluster**
4. Put **each query in a separate query tab/box** (click the **+** icon per query).

### 7.2 Rekey topics (create PK) + insert data
We’ll rekey each topic to establish a primary key for joins (Datagen didn’t set PKs).

**Credit Score:**
```sql
CREATE TABLE credit_score_rekeyed (
  customer_email STRING NOT NULL PRIMARY KEY NOT ENFORCED,
  income_type STRING,
  credit_score INT,
  credit_usage_percentage INT,
  credit_limit_idr INT
) DISTRIBUTED BY (customer_email) INTO 1 BUCKETS
WITH ('changelog.mode' = 'upsert');

INSERT INTO credit_score_rekeyed
SELECT
  customer_email,
  income_type,
  credit_score,
  credit_usage_percentage,
  credit_limit_idr
FROM `credit_score`;
```

**Mortgage Application:**
```sql
CREATE TABLE mortgage_application_rekeyed (
  application_id INT NOT NULL PRIMARY KEY NOT ENFORCED,
  customer_email STRING,
  mortgage_value INT,
  mortgage_type STRING
) DISTRIBUTED BY (application_id) INTO 1 BUCKETS
WITH ('changelog.mode' = 'upsert');

INSERT INTO mortgage_application_rekeyed
SELECT
  application_id,
  customer_email,
  mortgage_value,
  mortgage_type
FROM mortgage_application;
```

**Payment History:**
```sql
CREATE TABLE payment_history_rekeyed (
  customer_email STRING NOT NULL PRIMARY KEY NOT ENFORCED,
  payment_month_year BIGINT,
  amount_idr INT,
  application_id INT
) DISTRIBUTED BY (customer_email) INTO 1 BUCKETS
WITH ('changelog.mode' = 'upsert');

INSERT INTO payment_history_rekeyed
SELECT
  customer_email,
  payment_month_year,
  amount_idr,
  application_id
FROM payment_history;
```

### 7.3 CTAS Example: Deduplicate Credit Score
Use **CTAS** to create and populate a deduplicated table in one query (retain latest per email).

```sql
CREATE TABLE credit_score_deduped (
  customer_email STRING PRIMARY KEY NOT ENFORCED,
  income_type STRING,
  credit_score INT,
  credit_usage_percentage INT,
  credit_limit_idr INT
) DISTRIBUTED BY HASH(customer_email)
WITH ('changelog.mode' = 'upsert')
AS
SELECT customer_email, income_type, credit_score, credit_usage_percentage, credit_limit_idr
FROM (
  SELECT *,
         ROW_NUMBER() OVER (
           PARTITION BY customer_email
           ORDER BY $rowtime DESC
         ) AS rownum
  FROM credit_score_rekeyed
)
WHERE rownum = 1;
```

### 7.4 Enrich: Payments + Credit Score
Aggregate **Payment History** and join with the deduped **Credit Score**.

1. Change the changelog mode of `payment_history_rekeyed` to **append**:
```sql
ALTER TABLE payment_history_rekeyed SET ('changelog.mode' = 'append');
```

2. Create **`enriched_topic_credit_payment_history`** (CTAS):  
   - Counts and sums payments per user  
   - Converts latest payment epoch to `yyyy-MM` string  
   - Left joins credit score details
```sql
CREATE TABLE enriched_topic_credit_payment_history (
  customer_email STRING NOT NULL PRIMARY KEY NOT ENFORCED,
  payment_count BIGINT,
  total_payment BIGINT,
  newest_payment_month_year STRING,
  income_type STRING,
  credit_score INT,
  credit_usage_percentage INT,
  credit_limit_idr INT
) WITH ('changelog.mode' = 'upsert')
AS
WITH payment_agg AS (
  SELECT
    customer_email,
    COUNT(*) AS payment_count,
    SUM(amount_idr) AS total_payment,
    MAX(payment_month_year) AS latest_payment_epoch
  FROM payment_history_rekeyed
  GROUP BY customer_email
),
enriched_payment AS (
  SELECT
    customer_email,
    payment_count,
    total_payment,
    DATE_FORMAT(TO_TIMESTAMP_LTZ(latest_payment_epoch, 3), 'yyyy-MM') AS newest_payment_month_year
  FROM payment_agg
)
SELECT
  ep.customer_email,
  ep.payment_count,
  ep.total_payment,
  ep.newest_payment_month_year,
  cs.income_type,
  cs.credit_score,
  cs.credit_usage_percentage,
  cs.credit_limit_idr
FROM enriched_payment ep
LEFT JOIN credit_score_deduped cs
ON ep.customer_email = cs.customer_email;
```

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
INPUT (`details` VARCHAR(2147483647))
OUTPUT (`decisionreasoning` VARCHAR(2147483647))
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
INPUT (`details` VARCHAR(2147483647))
OUTPUT (`newcreditscore` VARCHAR(2147483647))
WITH (
  'googleai.connection' = 'mortgageagent-connection',
  'googleai.system_prompt' = 'Your task is to propose new credit score for the customer; whether new or stay as it is. This agent will take a user current data and propose an updated credit score. It follows these rules; If the latest payment month date is within the last 6 months and the credit usage percentage is below 50%, increase the current credit score by 10. If the latest payment month date is older than 6 months ago but the credit usage percentage is still below 50%, increase the current credit score by 5. In all other scenarios (e.g., credit usage is 50% or more), the credit score remains unchanged.The agent will need the following inputs:User Email (for identification),Latest Payment Month Date, Current Credit Score, and Credit Usage Percentage. The agent will output a single integer value: Proposed Credit Score',
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

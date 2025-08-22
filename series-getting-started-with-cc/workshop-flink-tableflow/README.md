<div align="center" padding=25px>
    <img src="images/confluent.png" width=50% height=50%>
</div>

# <div align="center">Build Agentic AI Data Streaming Pipeline using Confluent Data Streaming Platform</div>
## <div align="center">Lab Guide</div>
<br>

## **Agenda**
1. [Objective](#objective)
2. [Lab Account Preparation](#step-1)
3. [Environment, Cluster and Flink Setup](#step-2)
4. [Connectors and Topics Creation](#step-3)
5. [Flink Stream Processing](#step-4)
6. [Flink Agentic AI](#step-5)

***

## <a name="objective"></a>Objective

In this lab, participants will design and implement a real-time streaming pipeline with agentic AI
for automated mortgage application processing. The core goal is to demonstrate how modern
data platforms can be used for intelligent, event-driven decision-making. We will learn how to
build a real-time data pipeline using Confluent Kafka and Flink to process and enrich mortgage
application data from multiple sources, and then use an LLM to automatically determine
application approval.

***

## <a name="step-1"></a>Lab Account Preparation

1. Create a Confluent Cloud Account.
   - Sign up for a Confluent Cloud account [here](https://www.confluent.io/confluent-cloud/tryfree/).
   - Once you have signed up and logged in, click on the menu icon at the upper right hand corner,
     click on “Billing & payment”, then enter promo code **CONFLUENTDEV1** under “Payment details & contacts”
     to delay entering a credit card for 30 days.

<div align="center" padding=25px>
    <img src="images/billing.png" width=75% height=75%>
</div>

2. Install Confluent Cloud CLI based on your OS:
   [Install CLI](https://docs.confluent.io/confluent-cli/current/install.html)

***

## <a name="step-2"></a>Environment, Cluster and Flink Setup

An environment contains clusters and its deployed components such as Apache Flink, Connectors,
ksqlDB, and Schema Registry.

1. Click **+ Add Environment** → enter Environment Name → **Create**.
2. Click **Create Cluster** → choose **Basic Cluster** and follow:
   - Cloud: Google Cloud
   - Region: Jakarta (asia-southeast2)
   - Availability: Single Zone
   - Then, click **Launch Cluster**.

<div align="center">
    <img src="images/create-cluster.png" width=75% height=75%>
</div>

3. Create **Flink Compute Pool**:
   - Cloud: GCP
   - Region: Jakarta (asia-southeast2)
   - Pool Name: your choice
   - Max CFU: 10
   - Click **Create**.

<div align="center">
    <img src="images/create-flink-pool.png" width=75% height=75%>
</div>

4. Create API Key:
   - Go to Cluster → **API Keys** → **Add Key**
   - Select *My Account*
   - Download/save API Key and Secret

***

## <a name="step-3"></a>Connectors and Topics Creation

1. Create 3 Topics:
   - `Credit_Score` (Partitions: 1)
   - `Mortgage_Application` (Partitions: 1)
   - `Payment_History` (Partitions: 1)

<div align="center">
    <img src="images/create-topic.png" width=75% height=75%>
</div>

2. Create Datagen Source Connectors to simulate data.

**Credit_Score schema:**
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
          "range": {
            "min": 50,
            "max": 100
          }
        }
      }
    },
{
      "name": "credit_usage_percentage",
      "type": {
        "type": "int",
        "arg.properties": {
          "range": {
            "min": 0,
            "max": 100
          }
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

**Mortgage_Application schema:**
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
          "iteration": {
            "start" : 100,
"max" : 150
          }
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
          "options": [
            "fixed_rate",
            "adjustable_rate"
          ]
        }
      }
    }
  ]
}


```

**Payment_History schema:**
```json
{
  "type": "record",
  "name": "PaymentHistory",
  "namespace": "demo",
  "fields": [
    {"name": "customer_email", "type": "string"},
    {"name": "payment_month_year", "type": "long"},
    {"name": "amount_idr", "type": "long"},
    {"name": "application_id", "type": "int"}
  ]
}
```

***

## <a name="step-4"></a>Flink Stream Processing

Kafka topics and schemas are always in sync with our Flink cluster.

1. Open **Flink SQL Workspace**.
   - Catalog = Environment
   - Database = Cluster

2. Rekey topics with Primary Keys:

```sql
CREATE TABLE credit_score_rekeyed (
  customer_email STRING NOT NULL PRIMARY KEY NOT ENFORCED,
  income_type STRING,
  credit_score INT,
  credit_usage_percentage INT,
  credit_limit_idr INT
) WITH ('changelog.mode' = 'upsert');

INSERT INTO credit_score_rekeyed
SELECT customer_email, income_type, credit_score, credit_usage_percentage, credit_limit_idr
FROM credit_score;
```

... (same steps for mortgage_application_rekeyed and payment_history_rekeyed).

3. Deduplicate and Enrich Data:

```sql
CREATE TABLE credit_score_deduped (
  customer_email STRING PRIMARY KEY NOT ENFORCED,
  income_type STRING,
  credit_score INT,
  credit_usage_percentage INT,
  credit_limit_idr INT
) WITH ('changelog.mode' = 'upsert')
AS
SELECT customer_email, income_type, credit_score, credit_usage_percentage, credit_limit_idr
FROM (
  SELECT *, ROW_NUMBER() OVER (PARTITION BY customer_email ORDER BY $rowtime DESC) AS rownum
  FROM credit_score_rekeyed
)
WHERE rownum = 1;
```

... then join with `payment_history_rekeyed` and `mortgage_application_rekeyed`.

***

## <a name="step-5"></a>Flink Agentic AI

Confluent Cloud Flink supports integration with AI providers like Google AI, OpenAI, Vertex AI.

1. Create model connection via CLI:
```bash
confluent flink connection create mortgageagent-connection --cloud GCP --region <your-region> --environment <your-env-id> --type googleai --endpoint https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash:generateContent --api-key <your-google-api-key>
```

2. Create AI Model in Flink SQL:
```sql
CREATE MODEL AgentMortgageModel
INPUT (`details` VARCHAR)
OUTPUT (`decisionreasoning` VARCHAR)
WITH (
  'googleai.connection' = 'mortgageagent-connection',
  'googleai.system_prompt' = 'Your task is to evaluate mortgage applications and provide a final decision...',
  'provider' = 'googleai',
  'task' = 'text_generation'
);
```

3. Invoke AI Model:
```sql
SELECT application_id, decisionreasoning
FROM mortgage_submission_feed,
LATERAL TABLE(ML_PREDICT('AgentMortgageModel', CONCAT(
  'Application ID: ', CAST(application_id AS STRING),
  ', Mortgage Type: ', mortgage_type,
  ', Mortgage Value: ', CAST(mortgage_value AS STRING),
  ', Credit Score: ', CAST(credit_score AS STRING),
  ', Latest Payment: ', CAST(newest_payment_month_year AS STRING),
  ', Credit Limit: ', CAST(credit_limit AS STRING)
)));
```

4. (Optional) Create second agent for credit score reevaluation.

***

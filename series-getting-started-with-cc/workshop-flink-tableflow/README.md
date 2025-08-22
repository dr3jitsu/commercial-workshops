<div align="center">
  <img src="images/confluent.png" alt="Confluent" width="50%" />
</div>

# Enable Real‑Time Data Transformations & Agentic AI Stream Processing on Confluent Cloud (Workshop README)

This README converts the original workshop guide into a clean, step‑by‑step format that’s easy to follow and ready to use. Follow the numbered steps exactly as written.

---

## 0) Objective

Design and implement a **real‑time streaming pipeline** with **agentic AI** for automated mortgage application processing. You will:
- Build a pipeline using **Confluent Kafka** and **Flink** to process and enrich mortgage data from multiple sources.
- Use an **LLM** to automatically determine application approval.

> **Final Architecture**: Refer to the diagram in the original guide (place your architecture image at `images/architecture.png` if desired).

---

## 1) Lab Account Preparation

1. **Create a Confluent Cloud Account**
   1.1. Sign up at <https://confluent.cloud> and log in.  
   1.2. Open the **menu (top‑right)** → **Billing & payment**.  
   1.3. Under **Payment details & contacts**, enter promo code **`CONFLUENTDEV1`** to delay entering a credit card for 30 days.

2. **Install the Confluent Cloud CLI**
   2.1. Install based on your OS: <https://docs.confluent.io/confluent-cli/current/install.html>.

---

## 2) Environment, Cluster & Flink Setup

3. **Create an Environment**
   3.1. Click **+ Add Environment**.  
   3.2. Enter an **Environment Name** → **Create**.

4. **Create a Kafka Cluster (Basic)**
   4.1. In your new environment, click **Create Cluster**.  
   4.2. Choose **Basic**.
   4.3. Set:
   - **Cloud**: Google Cloud (GCP)  
   - **Region**: **Jakarta (`asia-southeast2`)**  
   - **Availability**: **Single Zone**  
   4.4. Name your cluster → **Launch Cluster**.

5. **Create a Flink Compute Pool**
   5.1. Go to your **Environment** → **Flink** → **Create Compute Pool**.  
   5.2. Set:
   - **Cloud Provider**: GCP  
   - **Region**: **Jakarta (`asia-southeast2`)** (must match your Kafka cluster)  
   5.3. Enter a **Flink Pool name** and set **Max CFU = 10**.  
   5.4. Click **Create**.

6. **Create an API Key for the Kafka Cluster**
   6.1. Go to your **Cluster** → **API Keys** → **Add Key**.  
   6.2. In **Create Key** wizard, choose **Select account for API Key = My Account**.  
   6.3. Click **Next** → **Download your API Key & Secret** → store securely (used by Connect).

---

## 3) Topics & Datagen Connectors

### 3.1 Create Topics

7. **Create Three Topics**
   7.1. In your **Cluster**, open **Topics** → **+ Add Topic**.  
   7.2. Create the following (use defaults unless specified):

   - **Topic**: `Credit_Score` — **Partitions**: `1`  
   - **Topic**: `Mortgage_Application` — **Partitions**: `1`  
   - **Topic**: `Payment_History` — **Partitions**: `1`

### 3.2 Create Datagen Source Connectors

> **Tip:** Confluent’s **Datagen Source** generates synthetic data—no database required.

8. **Create `credit_score` Connector**
   8.1. Go to **Connectors** → **+ Add Connector** → choose **Sample Data – Datagen Source**.  
   8.2. In **Launch Sample Data** → open **Additional Configuration** and set:  
   - **Topic Selection**: `Credit_Score`  
   - **Kafka Credentials**: **Use Existing API Key** (enter the key & secret you created)  
   - **Configuration**: **`JSON_SR`** for **Output Record Value Format**  
   - **Select a Schema**: **Provide your own schema**, then paste:
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
             "range": { "min": 100000000, "max": 1000000000, "step": 1000000 }
           }
         }
       }
     ]
   }
   ```
   8.3. Open **Advanced Configuration** → set **Max interval between messages (ms) = 10000**.  
   8.4. Click **Next** → **Name**: `credit_score` → **Number of tasks**: `1` → **Launch**.  
   8.5. Confirm that topic **`Credit_Score`** is receiving data.

9. **Create `mortgage_application` Connector**
   9.1. Repeat step **8**, targeting **Topic: `Mortgage_Application`**, with schema:
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
             "range": { "min": 100000000, "max": 1000000000, "step": 1000000 }
           }
         }
       },
       {
         "name": "mortgage_type",
         "type": {
           "type": "string",
           "arg.properties": {
             "options": [ "fixed_rate", "adjustable_rate" ]
           }
         }
       }
     ]
   }
   ```

10. **Create `payment_history` Connector**
    10.1. Repeat step **8**, targeting **Topic: `Payment_History`**, with schema:
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
              "range": { "min": 1578614400000, "step": 2626560000, "max": 1744243200000 }
            }
          }
        },
        {
          "name": "amount_idr",
          "type": {
            "type": "long",
            "arg.properties": {
              "range": { "min": 1000000, "step": 100000, "max": 100000000 }
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
    10.2. Confirm all three topics are receiving data.

---

## 4) Flink Stream Processing

> **Note:** Kafka topics and schemas are synchronized with Flink. Topics appear as tables in Flink, and vice versa.

11. **Open Flink SQL Workspace**
    11.1. In Confluent Cloud, go to **Environments** → select your environment.  
    11.2. Click **Flink** → select the compute pool created earlier → **Open SQL workspace**.  
    11.3. Set **Catalog = Your Environment** and **Database = Your Kafka Cluster**.  
    11.4. Use **separate query tabs** (click the `+` button) for each SQL below.

### 4.1 Rekey Source Topics (to enable joins)

12. **Rekey `Credit_Score` to `credit_score_rekeyed`**
    ```sql
    CREATE TABLE credit_score_rekeyed (
      customer_email STRING NOT NULL PRIMARY KEY NOT ENFORCED,
      income_type STRING,
      credit_score INT,
      credit_usage_percentage INT,
      credit_limit_idr INT
    ) DISTRIBUTED BY (customer_email) INTO 1 BUCKETS
    WITH ('changelog.mode' = 'upsert');
    ```
    ```sql
    INSERT INTO credit_score_rekeyed
    SELECT
      customer_email,
      income_type,
      credit_score,
      credit_usage_percentage,
      credit_limit_idr
    FROM `credit_score`;
    ```

13. **Rekey `Mortgage_Application` to `mortgage_application_rekeyed`**
    ```sql
    CREATE TABLE mortgage_application_rekeyed (
      application_id INT NOT NULL PRIMARY KEY NOT ENFORCED,
      customer_email STRING,
      mortgage_value INT,
      mortgage_type STRING
    ) DISTRIBUTED BY (application_id) INTO 1 BUCKETS
    WITH ('changelog.mode' = 'upsert');
    ```
    ```sql
    INSERT INTO mortgage_application_rekeyed
    SELECT
      application_id,
      customer_email,
      mortgage_value,
      mortgage_type
    FROM mortgage_application;
    ```

14. **Rekey `Payment_History` to `payment_history_rekeyed`**
    ```sql
    CREATE TABLE payment_history_rekeyed (
      customer_email STRING NOT NULL PRIMARY KEY NOT ENFORCED,
      payment_month_year BIGINT,
      amount_idr INT,
      application_id INT
    ) DISTRIBUTED BY (customer_email) INTO 1 BUCKETS
    WITH ('changelog.mode' = 'upsert');
    ```
    ```sql
    INSERT INTO payment_history_rekeyed
    SELECT
      customer_email,
      payment_month_year,
      amount_idr,
      application_id
    FROM payment_history;
    ```

### 4.2 Deduplicate Credit Score with CTAS

15. **Create `credit_score_deduped` using CTAS (keep latest per customer)**  
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

### 4.3 Aggregate Payments & Join with Credit Score (CTAS)

16. **Switch `payment_history_rekeyed` to append mode**
    ```sql
    ALTER TABLE payment_history_rekeyed SET ('changelog.mode' = 'append');
    ```

17. **Create `enriched_topic_credit_payment_history`**
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

> You should now see a new topic `enriched_topic_credit_payment_history` with enriched data.  
> The field `newest_payment_month_year` is formatted as `YYYY-MM` (converted from UNIX epoch).

### 4.4 Join Mortgage Applications with Enriched Customer Data

18. **Create `mortgage_submission_feed` (upsert)**  
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

19. **Insert joined data**
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

20. **Switch to append mode for consumption**
    ```sql
    ALTER TABLE `mortgage_submission_feed` SET ('changelog.mode'='append');
    ```

---

## 5) Flink Agentic AI

Flink in Confluent Cloud supports connections to **AWS Bedrock, SageMaker, Azure OpenAI, Azure ML, Google AI, OpenAI,** and **Vertex AI**. This lab uses **Google AI**.

> **Lab API (example):**
> - **API Key**: `AIzaSyBidz11vgKLz_1RMgtZ0FDG1x23SmrikgM`  
> - **Endpoint**: `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash:generateContent`  
> *(Replace with your own for real projects.)*

### 5.1 Create the Model Connection via Confluent CLI

21. **Install & login (if not already)**
    ```bash
    confluent login
    ```

22. **Select the correct environment**
    ```bash
    confluent environment list
    confluent environment use <env-id>
    ```

23. **Create the connection (Google AI)**
    ```bash
    confluent flink connection create mortgageagent-connection       --cloud GCP       --region <your flink region id>       --environment <your Confluent environment id>       --type googleai       --endpoint https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash:generateContent       --api-key <your Google AI API key>
    ```

> Ensure the endpoint matches the one provided by Google AI Studio.  
> Get your own key at <https://aistudio.google.com/> (Get API Key).

### 5.2 Create the AI Model in Flink SQL

24. **Create the mortgage decision agent**
    ```sql
    CREATE MODEL AgentMortgageModel
    INPUT (`details` VARCHAR(2147483647))
    OUTPUT (`decisionreasoning` VARCHAR(2147483647))
    WITH (
      'googleai.connection' = 'mortgageagent-connection',
      'googleai.system_prompt' = 'Your task is to evaluate mortgage applications and provide a final decision: APPROVE or DENY. Input data will include application_id, customer_email, mortgage_type, mortgage_value,credit score, newest payment month year, and credit limit. Evaluation logic is as follows: 1. Credit Score (Highest Priority): Higher is better; a score of 80 or higher is a strong positive, 70 or lower is a red flag. 2. Mortgage Value: Lower is better; a value of 500,000,000 or less is a positive factor. 3. Mortgage Type: floating_rate is more favorable than fixed_rate. 4. Credit Limit (Lowest Priority): Higher is better; a limit of 500,000,000 or more is a positive factor. The final output must be in the format: application id : {application id}. decision : {Approve / Deny}. reasoning : {your reasoning}.',
      'provider' = 'googleai',
      'task' = 'text_generation'
    );
    ```

### 5.3 Invoke the Model

25. **Score applications from `mortgage_submission_feed`**
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

### 5.4 (Optional) Create a Credit Score Re‑evaluation Agent

26. **Create the second agent**
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

27. **Invoke the second agent**
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

## 6) Notes & Tips

- Replace the **example Google AI API key** with your own for real environments.  
- Keep your **Confluent API keys** and **LLM keys** secure (never commit to public repos).  
- For screenshots and architecture diagrams, add files under `images/` and update links in this README.

---

## 7) Cleanup (Optional)

28. **Tear down lab resources to avoid charges**
   - Delete **connectors**, **topics**, **Flink tables**, **compute pool**, and **cluster** you created.
   - Remove any **API keys** no longer needed.

# Real-Time Banking Risk Monitoring & Promotions with Flink SQL, Tableflow, and DuckDB

---

## 1\. Workshop Overview

In this hands-on workshop you will build a complete real-time banking analytics pipeline using Flink SQL on Confluent Cloud. Starting from synthetic card and ATM transactions, you will design streaming tables, detect risky customer behavior (large withdrawals, FX exposure, merchant failures), and enrich events with customer 360 and city/merchant discount campaigns.

By the end of the core phases, you will have implemented end-to-end risk monitoring and promotional use cases fully in Flink SQL. In the optional advanced phases, you will persist Flink outputs into Iceberg tables via Tableflow, analyze them with DuckDB.

### 1.1. Scenario

You are a **Solutions/Data Engineer** at a **retail bank**. The bank wants to:

- Monitor **card/ATM transactions** in real time.  
- Detect **risky customer behavior**:  
  - Large / frequent **cash withdrawals**.  
  - High **FX exposure** for foreign-currency transactions.  
  - **Merchants** experiencing repeated **failed payments**.  
- Enrich transactions with:  
  - **Customer profiles** (Customer 360).  
  - **City \+ merchant discount campaigns**.  
- (Advanced) Make this data available to:  
  - **Tableflow** (as Iceberg tables).  
  - **DuckDB** (for analytics).

All streaming data is created with the **Flink `faker` connector**, so you can run the full workshop **inside Confluent Cloud Flink SQL** without any external sources.

---

### 1.2. Learning Path

You will go through 5 phases:

1. **Phase 1 – Streaming Data with Faker (Flink SQL)**  
   Build streaming tables:  
     
   - `card_transactions`  
   - `retail_customers`  
   - `city_discounts`

   

2. **Phase 2 – Risk Detection (Flink SQL)**  
     
   - Global KPI (GROUP BY).  
   - Rolling 1-hour risk (window `OVER`).  
   - Persisted risk table (`flagged_transactions`).  
   - Alert tables (`fx_risk_alerts`, `withdrawal_risk_alerts`).  
   - Merchant failure KPI (TUMBLE).

   

3. **Phase 3 – Enrichment & Patterns (Flink SQL)**  
     
   - Apply discounts (interval join) → `discounted_transactions`.  
   - Customer 360 (temporal join) → `card_transactions_with_customer`.  
   - Failure pattern detection (`MATCH_RECOGNIZE`) → `merchant_failure_patterns`.

   

4. **Phase 4 – Flink → Tableflow → DuckDB**

- Enable Tableflow → Iceberg tables.  
  - Query with DuckDB.


---

### 1.3. Prerequisites

**Required for Phases 1–3:**

- You have registered a \*\*Confluent Cloud\*\* account (trial or paid) at https://confluent.cloud.  
- You have created a \*\*Basic Kafka cluster\*\* in Confluent Cloud (AWS) in your chosen \*\*region Jakarta\*\*.  
- You have created a \*\*Flink compute pool\*\* in \*\*the same cloud and region\*\* as the Basic Kafka cluster.  
- You have a \*\*Flink SQL workspace\*\* attached to that compute pool.  
- Access to a **Confluent Cloud Flink SQL** workspace.

---

## 2\. Phase 1 – Streaming Data with Faker (Flink SQL)

**Goal:** Create 3 core streaming tables using **`connector = 'faker'`**:

- `card_transactions`  
- `retail_customers`  
- `city_discounts`

All SQL in this phase is run in the **Flink SQL workspace**.

---

### 2.1. Step 1 – Card Transactions (`card_transactions`)

#### 2.1.1. Objective

Create a **continuous stream of card/ATM transactions** to be used as the main fact table for risk and promotions.

#### 2.1.2. Create faker source table: `card_transactions_raw`

```sql
CREATE TABLE card_transactions_raw (
  txn_id           STRING NOT NULL,
  account_number   STRING,
  `timestamp`      TIMESTAMP(3) WITH LOCAL TIME ZONE,
  amount           DECIMAL(10, 2),
  currency         STRING,
  merchant         STRING,
  location         STRING,
  status           STRING,
  transaction_type STRING,
  CONSTRAINT PK_card_txn PRIMARY KEY (txn_id) NOT ENFORCED
)
DISTRIBUTED BY HASH(txn_id) INTO 6 BUCKETS
WITH (
  'connector'      = 'faker',
  'changelog.mode' = 'append',

  'fields.account_number.expression' = 'ACC#{Number.numberBetween ''1000000'',''1005000''}',
  'fields.amount.expression'         = '#{NUMBER.numberBetween ''10'',''1000''}',
  'fields.currency.expression'       = '#{Options.option ''USD'',''EUR'',''INR'',''GBP'',''JPY''}',

  'fields.location.expression' = '#{Options.option ''New York'',''Los Angeles'',''Chicago'',''Charlotte '',''San Francisco'',''Indianapolis'',''Seattle'',''Denver'',''Washington'',''Boston'',''El Paso'',''Nashville'',''Detroit'',''Oklahoma City'',''Portland'',''Las Vegas'',''Memphis'',''Louisville'',''Baltimore''}',

  'fields.merchant.expression' = '#{Options.option ''Walmart Inc.'', ''Amazon.com Inc.'', ''CVS Health'', ''Costco Wholesale Corporation'', ''Schwarz Group'', ''McKesson Corporation'', ''McDonalds Corporation'', ''Starbucks Corporation'', ''Cencora'', ''The Home Depot Inc.'', ''Yum! Brands'', ''The Kroger Co.'', ''Aldi Group'', ''Walgreens Boots Alliance'', ''Cardinal Health'', ''Subway'', ''JD.com Inc.'', ''Target Corporation'', ''Ahold Delhaize'', ''Lowe Companies Inc.''}',

  'fields.transaction_type.expression' = '#{Options.option ''payment'',''payment'', ''payment'' ,''refund'', ''withdrawal''}',
  'fields.status.expression'          = '#{Options.option ''Successful'',''Successful'', ''Failed'' }',
  'fields.txn_id.expression'          = '#{IdNumber.valid}',

  'rows-per-second' = '20'
);
```

**Function explanations**

- `connector = 'faker'`: generates synthetic rows continuously.  
- `changelog.mode = 'append'`: every generated row is an **insert**, no updates.  
- `DISTRIBUTED BY HASH(txn_id)`: spreads rows across buckets by transaction ID (parallelism).  
- `CONSTRAINT PK_card_txn PRIMARY KEY NOT ENFORCED`:  
  - Declares a logical PK, but Flink does **not enforce uniqueness**.

**Check data**

```sql
SELECT * FROM card_transactions_raw LIMIT 5;
```

**Example 5 rows (card\_transactions\_raw)**

| txn\_id | account\_number | timestamp | amount | currency | merchant | location | status | transaction\_type |
| :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- |
| TXN-001 | ACC1000123 | 2024-01-01 10:00:10 | 250.00 | USD | Walmart Inc. | New York | Successful | payment |
| TXN-002 | ACC1000456 | 2024-01-01 10:00:15 | 80.00 | USD | Walmart Inc. | New York | Failed | payment |
| TXN-003 | ACC1000123 | 2024-01-01 10:01:30 | 500.00 | EUR | Amazon.com Inc. | Seattle | Successful | withdrawal |
| TXN-004 | ACC1000456 | 2024-01-01 10:02:45 | 120.00 | GBP | Starbucks Corp. | Boston | Successful | payment |
| TXN-005 | ACC1000789 | 2024-01-01 10:03:10 | 900.00 | USD | Costco Whole. | Denver | Successful | withdrawal |

---

#### 2.1.3. Create logical table with watermark: `card_transactions`

```sql
CREATE TABLE card_transactions (
  txn_id           STRING NOT NULL,
  account_number   STRING,
  `timestamp`      TIMESTAMP(3) WITH LOCAL TIME ZONE,
  amount           DECIMAL(10, 2),
  currency         STRING,
  merchant         STRING,
  location         STRING,
  status           STRING,
  transaction_type STRING,
  WATERMARK FOR `timestamp` AS `timestamp` - INTERVAL '5' SECOND,
  PRIMARY KEY (txn_id) NOT ENFORCED
)
WITH ('changelog.mode' = 'append')
AS
SELECT * FROM card_transactions_raw;
```

**Function explanations**

- `WATERMARK FOR timestamp AS timestamp - INTERVAL '5' SECOND`:  
  - Tells Flink how far **behind** it can consider event time to be “on time”.  
  - Used for **event-time windows** like `TUMBLE` and `HOP`.  
- `PRIMARY KEY (txn_id) NOT ENFORCED`:  
  - Logical PK; helps with **upsert** sinks or queries but not enforced for uniqueness.

**Why this step**

- You separate **physical ingestion** (faker) from a **clean logical view** with event-time semantics.  
- All further queries will use **`card_transactions`**.

---

### 2.2. Step 2 – Retail Customers (`retail_customers`)

#### 2.2.1. Objective

Create a stream of **customer master data** and a versioned table suitable for **temporal joins**.

#### 2.2.2. Create faker source: `retail_customers_raw`

```sql
CREATE TABLE retail_customers_raw (
  account_number STRING NOT NULL,
  customer_name  STRING,
  email          STRING,
  phone_number   STRING,
  date_of_birth  TIMESTAMP(3),
  city           STRING,
  created_at     TIMESTAMP(3) WITH LOCAL TIME ZONE
)
DISTRIBUTED INTO 6 BUCKETS
WITH (
  'connector'      = 'faker',
  'changelog.mode' = 'append',

  'fields.account_number.expression' = 'ACC#{Number.numberBetween ''1000000'',''1005000''}',
  'fields.customer_name.expression'  = '#{Name.fullName}',
  'fields.email.expression'          = '#{Internet.emailAddress}',
  'fields.phone_number.expression'   = '#{PhoneNumber.cellPhone}',
  'fields.date_of_birth.expression'  = '#{date.birthday ''18'',''50''}',
  'fields.city.expression'           = '#{Address.city}',

  'rows-per-second' = '5'
);
```

**Check**

```sql
SELECT * FROM retail_customers_raw LIMIT 5;
```

**Example 5 rows**

| account\_number | customer\_name | email | phone\_number | date\_of\_birth | city | created\_at |
| :---- | :---- | :---- | :---- | :---- | :---- | :---- |
| ACC1000123 | Alice Johnson | [alice.j@example.com](mailto:alice.j@example.com) | \+1-555-111-1111 | 1990-05-10 00:00:00 | New York | 2024-01-01 09:55:00 |
| ACC1000456 | Bob Smith | [bob.smith@example.com](mailto:bob.smith@example.com) | \+1-555-222-2222 | 1985-02-20 00:00:00 | Seattle | 2024-01-01 09:58:00 |
| ACC1000789 | Carol Martinez | [carol.m@example.com](mailto:carol.m@example.com) | \+1-555-333-3333 | 1992-07-15 00:00:00 | Boston | 2024-01-01 10:00:00 |
| ACC1000456 | Bob A. Smith | [bob.a.smith@example.com](mailto:bob.a.smith@example.com) | \+1-555-222-3333 | 1985-02-20 00:00:00 | Seattle | 2024-01-01 10:05:00 |
| ACC1000999 | Daniel Lee | [daniel.lee@example.com](mailto:daniel.lee@example.com) | \+1-555-444-4444 | 1988-11-30 00:00:00 | Chicago | 2024-01-01 10:07:00 |

---

#### 2.2.3. Create versioned table: `retail_customers`

```sql
CREATE TABLE retail_customers (
  account_number STRING NOT NULL,
  customer_name  STRING,
  email          STRING,
  phone_number   STRING,
  date_of_birth  TIMESTAMP(3),
  city           STRING,
  created_at     TIMESTAMP(3) WITH LOCAL TIME ZONE,
  WATERMARK FOR created_at AS created_at - INTERVAL '5' SECONDS,
  PRIMARY KEY (account_number) NOT ENFORCED
)
WITH ('changelog.mode' = 'upsert')
AS
SELECT * FROM retail_customers_raw;
```

**Function explanations**

- `changelog.mode = 'upsert'`:  
  - Flink will treat rows as **updates keyed by `account_number`**.  
  - Only the latest row per key is kept in the table state.  
- This makes `retail_customers` behave like a **slowly changing dimension**.

**Example: updates for ACC1000456**

Source (`retail_customers_raw`):

| account\_number | customer\_name | email | created\_at |
| :---- | :---- | :---- | :---- |
| ACC1000456 | Bob Smith | [bob.smith@example.com](mailto:bob.smith@example.com) | 2024-01-01 09:58:00 |
| ACC1000456 | Bob A. Smith | [bob.a.smith@example.com](mailto:bob.a.smith@example.com) | 2024-01-01 10:05:00 |

`retail_customers` now holds only:

| account\_number | customer\_name | email | created\_at |
| :---- | :---- | :---- | :---- |
| ACC1000456 | Bob A. Smith | [bob.a.smith@example.com](mailto:bob.a.smith@example.com) | 2024-01-01 10:05:00 |

---

### 2.3. Step 3 – City/Merchant Discounts (`city_discounts`)

#### 2.3.1. Objective

Simulate **discount campaigns** for specific cities and merchants.

#### 2.3.2. Create faker source: `city_discounts_raw`

```sql
CREATE TABLE city_discounts_raw (
  city                  STRING,
  merchant_name         STRING,
  min_transaction_value DECIMAL(10, 2),
  discount_amount       DECIMAL(10, 2),
  `timestamp`           TIMESTAMP(3) WITH LOCAL TIME ZONE
)
DISTRIBUTED INTO 6 BUCKETS
WITH (
  'connector'      = 'faker',
  'changelog.mode' = 'append',

  'fields.city.expression' = '#{Options.option ''New York'',''Los Angeles'',''Chicago'',''Charlotte '',''San Francisco'',''Indianapolis'',''Seattle'',''Denver'',''Washington'',''Boston'',''El Paso'',''Nashville'',''Detroit'',''Oklahoma City'',''Portland'',''Las Vegas'',''Memphis'',''Louisville'',''Baltimore''}',

  'fields.merchant_name.expression' = '#{Options.option ''Walmart Inc.'', ''Amazon.com Inc.'', ''CVS Health'', ''Costco Wholesale Corporation'', ''Schwarz Group'', ''McKesson Corporation'', ''McDonalds Corporation'', ''Starbucks Corporation'', ''Cencora'', ''The Home Depot Inc.'', ''Yum! Brands'', ''The Kroger Co.'', ''Aldi Group'', ''Walgreens Boots Alliance'', ''Cardinal Health'', ''Subway'', ''JD.com Inc.'', ''Target Corporation'', ''Ahold Delhaize'', ''Lowe Companies Inc.''}',

  'fields.min_transaction_value.expression' = '#{NUMBER.numberBetween ''500'',''1000''}',
  'fields.discount_amount.expression'       = '#{NUMBER.numberBetween ''2'',''20''}',

  'rows-per-second' = '5'
);
```

Check:

```sql
SELECT * FROM city_discounts_raw LIMIT 5;
```

**Example 5 rows**

| city | merchant\_name | min\_transaction\_value | discount\_amount | timestamp |
| :---- | :---- | :---- | :---- | :---- |
| New York | Walmart Inc. | 750.00 | 10.00 | 2024-01-01 10:00:00 |
| New York | Walmart Inc. | 900.00 | 15.00 | 2024-01-01 10:30:00 |
| Seattle | Amazon.com Inc. | 600.00 | 5.00 | 2024-01-01 10:05:00 |
| Boston | Starbucks Corp. | 400.00 | 8.00 | 2024-01-01 10:10:00 |
| Denver | Costco Whole. | 850.00 | 12.00 | 2024-01-01 10:20:00 |

---

#### 2.3.3. Create logical discounts table: `city_discounts`

```sql
CREATE TABLE city_discounts (
  city                  STRING,
  merchant_name         STRING,
  min_transaction_value DECIMAL(10, 2),
  discount_amount       DECIMAL(10, 2),
  `timestamp`           TIMESTAMP(3) WITH LOCAL TIME ZONE,
  WATERMARK FOR `timestamp` AS `timestamp` - INTERVAL '5' SECONDS
)
WITH ('changelog.mode' = 'append')
AS
SELECT * FROM city_discounts_raw;
```

**Function explanation**

- Watermark lets us do **interval joins** based on event time (e.g. discounts active at/around a transaction time).

---

## 3\. Phase 2 – Risk Detection (Flink SQL)

**Goal:** Implement effective **risk monitoring** for withdrawals and FX exposure, and generate alerts/KPIs.

You will:

1. Global KPI (GROUP BY).  
2. Rolling 1-hour withdrawal risk.  
3. Rolling FX exposure.  
4. Persist `flagged_transactions`.  
5. Route alerts.  
6. Compute merchant failure KPIs.

---

### 3.1. Step 4 – Global Withdrawal KPI (GROUP BY)

```sql
SELECT
  account_number,
  transaction_type,
  SUM(amount) AS total_withdrawn
FROM card_transactions
WHERE transaction_type = 'withdrawal'
  AND status = 'Successful'
GROUP BY account_number, transaction_type
HAVING SUM(amount) > 5000;
```

**Function explanations**

- `GROUP BY`: groups rows by `account_number` and `transaction_type`.  
- `SUM(amount)`: sums the `amount` within each group.  
- `HAVING`: filters **grouped results** (like WHERE for aggregated rows).

**Example input (withdrawals)**

| txn\_id | account\_number | timestamp | amount | status | transaction\_type |
| :---- | :---- | :---- | :---- | :---- | :---- |
| W1 | ACC1000123 | 10:00 | 2000 | Successful | withdrawal |
| W2 | ACC1000123 | 10:20 | 4000 | Successful | withdrawal |
| W3 | ACC1000123 | 10:40 | 1000 | Failed | withdrawal |
| W4 | ACC1000456 | 10:05 | 1500 | Successful | withdrawal |
| W5 | ACC1000456 | 10:35 | 4500 | Successful | withdrawal |

**Example output**

| account\_number | transaction\_type | total\_withdrawn |
| :---- | :---- | :---- |
| ACC1000123 | withdrawal | 6000.00 |
| ACC1000456 | withdrawal | 6000.00 |

**Why this query**

- Gives a **global, lifetime KPI** per account.  
- It does **not** consider **time windows**; for streaming alerts we need rolling windows (next step).

---

### 3.2. Step 5 – Rolling Withdrawal Risk (OVER Window)

```sql
SELECT
  txn_id,
  account_number,
  currency,
  transaction_type,
  amount,
  merchant,
  location,
  SUM(amount) OVER w AS total_withdrawal_last_hour,
  CASE WHEN SUM(amount) OVER w > 10000 THEN 'YES'
       ELSE 'NO'
  END AS WITHDRAW_FLAG
FROM card_transactions
WHERE transaction_type = 'withdrawal'
  AND status = 'Successful'
WINDOW w AS (
  PARTITION BY account_number, transaction_type
  ORDER BY `timestamp` ASC
  RANGE BETWEEN INTERVAL '1' HOUR PRECEDING AND CURRENT ROW
);
```

**Function explanations**

- `SUM(amount) OVER w`:  
  - A **window function** that computes a sum across a set of rows defined by window `w`.  
- `WINDOW w AS (...)`:  
  - `PARTITION BY account_number, transaction_type`:  
    - Keeps independent windows per account & type.  
  - `ORDER BY timestamp ASC`:  
    - Processes events in time order.  
  - `RANGE BETWEEN INTERVAL '1' HOUR PRECEDING AND CURRENT ROW`:  
    - The window covers events in the last 1 hour up to the current event.

**Example input (ACC1000123)**

| txn\_id | timestamp | amount | status | transaction\_type |
| :---- | :---- | :---- | :---- | :---- |
| W1 | 10:00:00 | 4000 | Successful | withdrawal |
| W2 | 10:20:00 | 3000 | Successful | withdrawal |
| W3 | 10:40:00 | 5000 | Successful | withdrawal |
| W4 | 11:30:00 | 2000 | Successful | withdrawal |
| W5 | 12:00:00 | 1000 | Successful | withdrawal |

**Example output**

| txn\_id | timestamp | amount | total\_withdrawal\_last\_hour | WITHDRAW\_FLAG (limit 10k) |
| :---- | :---- | :---- | :---- | :---- |
| W1 | 10:00 | 4000 | 4000 | NO |
| W2 | 10:20 | 3000 | 7000 (W1+W2) | NO |
| W3 | 10:40 | 5000 | 12000 (W1+W2+W3) | YES |
| W4 | 11:30 | 2000 | 2000 | NO |
| W5 | 12:00 | 1000 | 3000 (W4+W5) | NO |

**Why this query**

- Computes a **time-bounded risk metric** for each event.  
- Different from GROUP BY: you get a **row per transaction** with a rolling sum, suitable for **real-time alerts**.

---

### 3.3. Step 6 – Rolling FX Exposure

```sql
WITH fx_exposure AS (
  SELECT
    txn_id,
    account_number,
    currency,
    amount,
    SUM(amount) OVER w AS total_fx_amount,
    CASE WHEN SUM(amount) OVER w > 3000 THEN 'YES'
         ELSE 'NO'
    END AS FX_FLAG
  FROM card_transactions
  WHERE transaction_type <> 'refund'
    AND status = 'Successful'
  WINDOW w AS (
    PARTITION BY account_number, currency
    ORDER BY `timestamp` ASC
    RANGE BETWEEN INTERVAL '1' HOUR PRECEDING AND CURRENT ROW
  )
)
SELECT * FROM fx_exposure;
```

**Function explanations**

- Same window function pattern as Step 5, but:  
  - `PARTITION BY account_number, currency`:  
    - Windows per account and per currency.  
  - Excludes `refund` so we only measure exposure from outgoing spend.

**Example input (ACC1000456, EUR)**

| txn\_id | timestamp | amount | currency | transaction\_type | status |
| :---- | :---- | :---- | :---- | :---- | :---- |
| F1 | 10:10 | 1500 | EUR | payment | Successful |
| F2 | 10:30 | 1000 | EUR | payment | Successful |
| F3 | 10:40 | 800 | EUR | payment | Successful |
| F4 | 11:15 | 2000 | EUR | payment | Successful |
| F5 | 11:40 | 1000 | EUR | refund | Successful |

**Example output (FX limit 3000\)**

| txn\_id | amount | total\_fx\_amount | FX\_FLAG |
| :---- | :---- | :---- | :---- |
| F1 | 1500 | 1500 | NO |
| F2 | 1000 | 2500 | NO |
| F3 | 800 | 3300 | YES |
| F4 | 2000 | 2000 | NO |
| F5 | 1000 | – (excluded) | – |

**Why this query**

- Adds additional risk dimension: **per-currency FX exposure** within a moving window.

---

### 3.4. Step 7 – Persist Risks in `flagged_transactions`

#### 3.4.1. Create table

```sql
CREATE TABLE flagged_transactions (
  txn_id           STRING,
  account_number   STRING,
  currency         STRING,
  transaction_type STRING,
  amount           DECIMAL(10, 2),
  merchant         STRING,
  location         STRING,
  `timestamp`      TIMESTAMP(3) WITH LOCAL TIME ZONE,
  withdrawal_total DECIMAL(10, 2),
  withdraw_flag    STRING,
  forex_total      DECIMAL(10, 2),
  fx_flag          STRING,
  PRIMARY KEY (txn_id) NOT ENFORCED
);
```

#### 3.4.2. Insert from rolling queries

```sql
INSERT INTO flagged_transactions
WITH flagged_withdrawal_tx AS (
  SELECT
    txn_id,
    account_number,
    currency,
    transaction_type,
    amount,
    merchant,
    location,
    `timestamp`,
    SUM(amount) OVER w AS total_withdrawal,
    CASE WHEN SUM(amount) OVER w > 5000 THEN 'YES'
         ELSE 'NO'
    END AS WITHDRAW_FLAG
  FROM card_transactions
  WHERE transaction_type = 'withdrawal'
    AND status = 'Successful'
  WINDOW w AS (
    PARTITION BY account_number, transaction_type
    ORDER BY `timestamp` ASC
    RANGE BETWEEN INTERVAL '1' HOUR PRECEDING AND CURRENT ROW
  )
),
flagged_currency_tx AS (
  SELECT
    txn_id,
    account_number,
    currency,
    SUM(amount) OVER w AS total_fx,
    CASE WHEN SUM(amount) OVER w > 3000 THEN 'YES'
         ELSE 'NO'
    END AS FX_FLAG
  FROM card_transactions
  WHERE transaction_type <> 'refund'
    AND status = 'Successful'
  WINDOW w AS (
    PARTITION BY account_number, currency
    ORDER BY `timestamp` ASC
    RANGE BETWEEN INTERVAL '1' HOUR PRECEDING AND CURRENT ROW
  )
)
SELECT
  A.txn_id,
  A.account_number,
  A.currency,
  A.transaction_type,
  A.amount,
  A.merchant,
  A.location,
  A.`timestamp`,
  B.total_withdrawal      AS withdrawal_total,
  COALESCE(B.WITHDRAW_FLAG, 'NO') AS withdraw_flag,
  C.total_fx              AS forex_total,
  C.FX_FLAG               AS fx_flag
FROM card_transactions A
LEFT JOIN flagged_withdrawal_tx B
  ON A.txn_id = B.txn_id AND A.account_number = B.account_number
LEFT JOIN flagged_currency_tx C
  ON A.txn_id = C.txn_id AND A.account_number = C.account_number
WHERE COALESCE(B.WITHDRAW_FLAG, 'NO') = 'YES'
   OR C.FX_FLAG = 'YES';
```

Full SQL already shown above; here’s the concept:

- Two CTEs:  
  - `flagged_withdrawal_tx` – rolling withdrawal totals \+ flags.  
  - `flagged_currency_tx` – rolling FX totals \+ flags.  
- Join them back to `card_transactions`.  
- Filter to rows where at least one of the flags is `YES`.

**Example input combination**  
(From previous steps, assuming these rows had flags)

| txn\_id | account\_number | amount | currency | withdrawal\_total | WITHDRAW\_FLAG | total\_fx | FX\_FLAG |
| :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- |
| W2 | ACC1000123 | 3000 | USD | 7000 | YES | NULL | NULL |
| W3 | ACC1000123 | 5000 | USD | 12000 | YES | NULL | NULL |
| F3 | ACC1000456 | 800 | EUR | NULL | NO | 3300 | YES |
| F4 | ACC1000456 | 2000 | EUR | NULL | NO | 2000 | NO |
| P1 | ACC1000789 | 100 | USD | NULL | NO | 100 | NO |

**Example output (`flagged_transactions`)**

| txn\_id | account\_number | amount | currency | withdrawal\_total | withdraw\_flag | forex\_total | fx\_flag |
| :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- |
| W2 | ACC1000123 | 3000 | USD | 7000 | YES | NULL | NULL |
| W3 | ACC1000123 | 5000 | USD | 12000 | YES | NULL | NULL |
| F3 | ACC1000456 | 800 | EUR | NULL | NO | 3300 | YES |

**Why this step**

- Gives a **single “risk fact” table** where all risky events are centralized.

---

### 3.5. Step 8 – Alert Routing (Statement Set)

```sql
CREATE TABLE fx_risk_alerts (
  txn_id     STRING,
  account_id STRING,
  message    STRING,
  PRIMARY KEY (txn_id) NOT ENFORCED
);
```

### 

```sql

CREATE TABLE withdrawal_risk_alerts (
  txn_id     STRING,
  account_id STRING,
  message    STRING,
  PRIMARY KEY (txn_id) NOT ENFORCED
);
```

```sql
EXECUTE STATEMENT SET
BEGIN
  INSERT INTO fx_risk_alerts
  SELECT
    txn_id,
    account_number,
    CONCAT(
      'Dear customer ', account_number,
      ', your foreign currency transactions have exceeded the configured threshold. ',
      'Current total: ', CAST(forex_total AS STRING), ' ', currency, '.'
    ) AS message
  FROM flagged_transactions
  WHERE fx_flag = 'YES';

  INSERT INTO withdrawal_risk_alerts
  SELECT
    txn_id,
    account_number,
    CONCAT(
      'Dear customer ', account_number,
      ', your cash withdrawals have exceeded the configured threshold in the past hour. ',
      'Current total: ', CAST(withdrawal_total AS STRING), '.'
    ) AS message
  FROM flagged_transactions
  WHERE withdraw_flag = 'YES';
END;
```

**Function explanations**

- `EXECUTE STATEMENT SET ... BEGIN ... END`:  
  - Runs multiple `INSERT` statements as **one Flink job**.  
  - Shared sources; Flink optimizes processing of multiple sinks.

**Example outputs**

`withdrawal_risk_alerts`:

| txn\_id | account\_id | message (shortened) |
| :---- | :---- | :---- |
| W2 | ACC1000123 | Dear customer ACC1000123, your cash withdrawals have exceeded… (7000). |
| W3 | ACC1000123 | Dear customer ACC1000123, your cash withdrawals have exceeded… (12000). |
| Z1 | ACC1000888 | Dear customer ACC1000888, your cash withdrawals have exceeded… (9000). |

`fx_risk_alerts`:

| txn\_id | account\_id | message (shortened) |
| :---- | :---- | :---- |
| F3 | ACC1000456 | Dear customer ACC1000456, your foreign currency transactions have… (3300) |
| Z1 | ACC1000888 | Dear customer ACC1000888, your foreign currency transactions have… (5000) |

---

### 3.6. Step 9 – Merchant Failure KPI (TUMBLE)

```sql
SELECT
  window_start,
  window_end,
  window_time,
  merchant,
  transaction_type,
  COUNT(*)    AS total_tx_failed,
  SUM(amount) AS total_amt_failed
FROM TUMBLE(
       TABLE card_transactions,
       DESCRIPTOR(`timestamp`),
       INTERVAL '10' MINUTES
     )
WHERE transaction_type = 'payment'
  AND status = 'Failed'
GROUP BY
  window_start,
  window_end,
  window_time,
  merchant,
  transaction_type;
```

**Function explanations**

- `TUMBLE(TABLE t, DESCRIPTOR(column), INTERVAL ...)`:  
  - Creates **fixed, non-overlapping** time windows.  
  - `window_start`, `window_end`, `window_time` are generated window columns.  
- Window size: **10 minutes**.

**Example input**

| txn\_id | timestamp | merchant | amount | transaction\_type | status |
| :---- | :---- | :---- | :---- | :---- | :---- |
| P1 | 10:01 | Walmart Inc. | 100 | payment | Failed |
| P2 | 10:02 | Walmart Inc. | 200 | payment | Failed |
| P3 | 10:04 | Walmart Inc. | 300 | payment | Successful |
| P4 | 10:05 | Amazon.com Inc. | 50 | payment | Failed |
| P5 | 10:11 | Walmart Inc. | 400 | payment | Failed |

**Example output**

| window\_start | window\_end | merchant | transaction\_type | total\_tx\_failed | total\_amt\_failed |
| :---- | :---- | :---- | :---- | :---- | :---- |
| 10:00 | 10:10 | Walmart Inc. | payment | 2 | 300 |
| 10:00 | 10:10 | Amazon.com Inc. | payment | 1 | 50 |
| 10:10 | 10:20 | Walmart Inc. | payment | 1 | 400 |

---

## 4\. Phase 3 – Enrichment & Patterns

**Goal:** Add value via **discount application**, **Customer 360**, and **pattern detection**.

You build:

- `discounted_transactions`  
- `card_transactions_with_customer`  
- `merchant_failure_patterns`

---

### 4.1. Step 10 – Discounted Transactions (Interval Join)

#### 4.1.1. Create table

```sql
CREATE TABLE discounted_transactions (
  txn_id          STRING,
  account_number  STRING,
  original_amount DECIMAL(10, 2),
  final_amount    DECIMAL(10, 2),
  currency        STRING,
  merchant        STRING,
  city            STRING,
  discount_amount DECIMAL(10, 2),
  discount_source STRING,
  `timestamp`     TIMESTAMP(3) WITH LOCAL TIME ZONE,
  PRIMARY KEY (txn_id) NOT ENFORCED
);
```

#### 4.1.2. Insert via interval join

```sql
INSERT INTO discounted_transactions
SELECT
  trx.txn_id,
  trx.account_number,
  trx.amount           AS original_amount,
  CASE WHEN trx.amount >= disc.min_transaction_value
       THEN trx.amount - disc.discount_amount
       ELSE trx.amount
  END                  AS final_amount,
  trx.currency,
  trx.merchant,
  trx.location         AS city,
  disc.discount_amount,
  'CITY_CAMPAIGN'      AS discount_source,
  trx.`timestamp`
FROM card_transactions AS trx
LEFT JOIN city_discounts AS disc
  ON trx.location = disc.city
WHERE trx.`timestamp` BETWEEN disc.`timestamp` - INTERVAL '10' MINUTE
                           AND disc.`timestamp`
  AND trx.transaction_type <> 'refund';
```

**Function explanations**

- `LEFT JOIN ... ON trx.location = disc.city`:  
  - Joins transactions with discounts in the same city.  
- `WHERE trx.timestamp BETWEEN disc.timestamp - 10 MINUTE AND disc.timestamp`:  
  - Enforces a **temporal relationship** (discount not older than 10min).  
- `CASE WHEN ... THEN ... ELSE ... END`:  
  - Applies discount only if `amount >= min_transaction_value`.

**Example input**

`card_transactions`:

| txn\_id | timestamp | account\_number | amount | merchant | location | transaction\_type | status |
| :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- |
| D1 | 10:04 | ACC1000123 | 800 | Walmart Inc. | New York | payment | Successful |
| D2 | 10:20 | ACC1000123 | 600 | Walmart Inc. | New York | payment | Successful |
| D3 | 10:35 | ACC1000456 | 500 | Amazon.com Inc. | Seattle | payment | Successful |
| D4 | 10:50 | ACC1000789 | 900 | Costco Whole. | Denver | payment | Successful |
| D5 | 10:55 | ACC1000789 | 450 | Starbucks Corp. | Boston | refund | Successful |

`city_discounts`:

| city | merchant\_name | min\_transaction\_value | discount\_amount | timestamp |
| :---- | :---- | :---- | :---- | :---- |
| New York | Walmart Inc. | 750 | 10 | 10:05 |
| New York | Walmart Inc. | 900 | 15 | 10:30 |
| Seattle | Amazon.com Inc. | 600 | 5 | 10:37 |
| Denver | Costco Whole. | 850 | 12 | 10:52 |
| Boston | Starbucks Corp. | 400 | 8 | 10:57 |

**Example output**

| txn\_id | original\_amount | final\_amount | discount\_amount | city | merchant |
| :---- | :---- | :---- | :---- | :---- | :---- |
| D1 | 800 | 790 | 10 | New York | Walmart Inc. |
| D2 | 600 | 600 | NULL | New York | Walmart Inc. |
| D3 | 500 | 500 | NULL | Seattle | Amazon.com Inc. |
| D4 | 900 | 888 | 12 | Denver | Costco Whole. |

---

### 4.2. Step 11 – Customer 360 (`card_transactions_with_customer`)

```sql
SET 'sql.tables.scan.watermark-alignment.max-allowed-drift' = '0';
SET 'sql.tables.scan.idle-timeout' = '5s';

CREATE VIEW card_transactions_with_customer AS
SELECT
  t.txn_id,
  t.account_number,
  t.`timestamp`,
  t.amount,
  t.currency,
  t.merchant,
  t.location,
  t.status,
  t.transaction_type,
  c.customer_name,
  c.city AS customer_city
FROM card_transactions t
LEFT JOIN retail_customers
FOR SYSTEM_TIME AS OF t.`timestamp` AS c
ON t.account_number = c.account_number;
```

**Function explanations**

- `FOR SYSTEM_TIME AS OF t.timestamp`:  
  - Temporal join; uses the version of `retail_customers` that was valid at `t.timestamp`.  
- This ensures you get **historically correct** customer data.

**Example input**

`retail_customers` for ACC1000123:

| account\_number | customer\_name | city | created\_at |
| :---- | :---- | :---- | :---- |
| ACC1000123 | Alice Johnson | Boston | 2023-01-01 00:00:00 |
| ACC1000123 | Alice J. Smith | New York | 2024-01-01 09:30:00 |
| ACC1000123 | Alice Smith | Chicago | 2024-01-01 11:00:00 |

`card_transactions`:

| txn\_id | account\_number | timestamp | amount | merchant | location |
| :---- | :---- | :---- | :---- | :---- | :---- |
| C1 | ACC1000123 | 2024-01-01 09:45:00 | 200 | Walmart Inc. | New York |
| C2 | ACC1000123 | 2024-01-01 10:30:00 | 300 | Amazon.com Inc. | New York |
| C3 | ACC1000123 | 2024-01-01 11:15:00 | 150 | Starbucks Corp. | Chicago |

**Example output**

| txn\_id | timestamp | amount | merchant | customer\_name | customer\_city |
| :---- | :---- | :---- | :---- | :---- | :---- |
| C1 | 2024-01-01 09:45:00 | 200 | Walmart Inc. | Alice J. Smith | New York |
| C2 | 2024-01-01 10:30:00 | 300 | Amazon.com Inc. | Alice J. Smith | New York |
| C3 | 2024-01-01 11:15:00 | 150 | Starbucks Corp. | Alice Smith | Chicago |

---

### 4.3. Step 12 – Failure Pattern Detection (`MATCH_RECOGNIZE`)

#### 4.3.1. Create pattern output table

```sql
CREATE TABLE merchant_failure_patterns (
  merchant                 STRING,
  location                 STRING,
  amount_list              ARRAY<DECIMAL(10, 2)>,
  txn_list                 ARRAY<STRING>,
  total_failed_amount      DECIMAL(10, 2),
  time_series              ARRAY<TIMESTAMP(3) WITH LOCAL TIME ZONE>,
  first_failure_timestamp  TIMESTAMP(3) WITH LOCAL TIME ZONE,
  last_failure_timestamp   TIMESTAMP(3) WITH LOCAL TIME ZONE
);
```

#### 4.3.2. Insert via `MATCH_RECOGNIZE`

```sql
INSERT INTO merchant_failure_patterns
SELECT
  merchant,
  location,
  amount_list,
  txn_list,
  total_transaction_value,
  time_series,
  start_tstamp,
  end_tstamp
FROM card_transactions
MATCH_RECOGNIZE (
  PARTITION BY merchant, location
  ORDER BY `timestamp`
  MEASURES
    ARRAY_AGG(failed_event.amount)      AS amount_list,
    ARRAY_AGG(failed_event.txn_id)      AS txn_list,
    SUM(failed_event.amount)            AS total_transaction_value,
    ARRAY_AGG(failed_event.`timestamp`) AS time_series,
    FIRST(failed_event.`timestamp`)     AS start_tstamp,
    LAST(failed_event.`timestamp`)      AS end_tstamp
  ONE ROW PER MATCH
  AFTER MATCH SKIP PAST LAST ROW
  PATTERN (failed_event{3})
  DEFINE
    failed_event AS failed_event.status = 'Failed'
) MR;
```

**Function explanations**

- `MATCH_RECOGNIZE`:  
  - Allows **pattern matching over ordered events**.  
- `PARTITION BY merchant, location`:  
  - Analyze each merchant/location separately.  
- `PATTERN (failed_event{3,})`:  
  - Look for **3 or more consecutive failed\_event rows**.  
- `DEFINE failed_event AS failed_event.status = 'Failed'`:  
  - Defines a failed event as one whose `status = 'Failed'`.

**Example input (Walmart Inc., New York)**

| txn\_id | timestamp | status | amount | merchant | location |
| :---- | :---- | :---- | :---- | :---- | :---- |
| F1 | 10:00:00 | Failed | 50 | Walmart Inc. | New York |
| F2 | 10:01:30 | Failed | 60 | Walmart Inc. | New York |
| F3 | 10:02:00 | Failed | 70 | Walmart Inc. | New York |
| F4 | 10:03:00 | Successful | 100 | Walmart Inc. | New York |
| F5 | 10:04:00 | Failed | 30 | Walmart Inc. | New York |
| F6 | 10:05:00 | Failed | 40 | Walmart Inc. | New York |
| F7 | 10:06:00 | Failed | 50 | Walmart Inc. | New York |

**Example output**

| merchant | location | amount\_list | txn\_list | total\_failed\_amount | first\_failure\_timestamp | last\_failure\_timestamp |
| :---- | :---- | :---- | :---- | :---- | :---- | :---- |
| Walmart Inc. | New York | \[50, 60, 70\] | \[F1,F2,F3\] | 180 | 10:00:00 | 10:02:00 |
| Walmart Inc. | New York | \[30, 40, 50\] | \[F5,F6,F7\] | 120 | 10:04:00 | 10:06:00 |

---

## 5\. Phase 4 (Optional) – Flink → Tableflow → DuckDB (Detailed)

**Advanced** – Requires Confluent Cloud Kafka \+ Tableflow and local DuckDB.

**Goal:** Make Flink outputs (**risk** and **discounts**) available as **Iceberg tables** that you can query with DuckDB.

### 5.1. Step 13.3 – Enable Tableflow in Confluent Cloud

In Confluent Cloud UI:

1. Go to your **Kafka cluster**.  
2. Click **Tableflow**.  
3. For topic **`flagged_transactions`**:  
   - Click **Enable Tableflow**.  
   - Use Confluent-managed storage.  
4. For topic **`discounted_transactions`**:  
   - Click **Enable Tableflow**.  
5. Wait a few minutes.

Now Tableflow maintains **Iceberg tables** with the same names.

---

### 5.2. Step 13.4 – Query Tableflow with DuckDB

#### 5.4.1. Install DuckDB and extensions

```shell
curl https://install.duckdb.org | sh
duckdb

echo 'export PATH="/Users/ahartono/.duckdb/cli/latest":$PATH' >> ~/.zshrc
source ~/.zshrc

duckdb
```

In DuckDB:

```sql
INSTALL httpfs;
LOAD httpfs;

INSTALL iceberg;
LOAD iceberg;
```

#### 5.2.2. Attach Tableflow catalog

1. From Confluent Cloud Tableflow UI:  
     
   - Get the **REST Catalog Endpoint**, e.g.:

```
https://tableflow.us-east-2.aws.confluent.cloud/iceberg/catalog/organizations/{ORG_ID}/environments/{ENV_ID}
```

   - Create an **API key** (CLIENT\_ID / CLIENT\_SECRET).  
   - Note your **Cluster ID**, e.g. `lkc-abc123`.

   

2. In DuckDB:

```sql
CREATE SECRET iceberg_secret (
  TYPE ICEBERG,
  CLIENT_ID     '<YOUR_TABLEFLOW_API_KEY>',
  CLIENT_SECRET '<YOUR_TABLEFLOW_API_SECRET>',
  ENDPOINT      'https://tableflow.<REGION>.aws.confluent.cloud/iceberg/catalog/organizations/<ORG_ID>/environments/<ENV_ID>',
  OAUTH2_SCOPE  'catalog'
);
```

```sql
ATTACH 'warehouse' AS iceberg_catalog (
  TYPE iceberg,
  SECRET iceberg_secret,
  ENDPOINT 'https://tableflow.<REGION>.aws.confluent.cloud/iceberg/catalog/organizations/<ORG_ID>/environments/<ENV_ID>'
);

```

     
   Example

```sql
CREATE SECRET iceberg_secret (
  TYPE ICEBERG,
  CLIENT_ID     'C43WHKQT65FVRCE3',
  CLIENT_SECRET 'cfltk1DXlRxl7bWfo/1qj6pROQBmpVzNTJmKrYFhH6CqupgKmpl0wvOWU1I0iCkw',
  ENDPOINT      'https://tableflow.ap-southeast-3.aws.confluent.cloud/iceberg/catalog/organizations/eb2976d2-949c-4e74-88ef-848321cca158/environments/env-30209w',
  OAUTH2_SCOPE  'catalog'
);

```

```sql
ATTACH 'warehouse' AS iceberg_catalog (
  TYPE iceberg,
  SECRET iceberg_secret,
  ENDPOINT 'https://tableflow.ap-southeast-3.aws.confluent.cloud/iceberg/catalog/organizations/eb2976d2-949c-4e74-88ef-848321cca158/environments/env-30209w'
);
```

#### 

#### 5.2.3. Query the tables

```sql
USE iceberg_catalog."lkc-90ry25";  -- your cluster ID

SHOW TABLES;.mode box
```

You should see:

- `"flagged_transactions"`  
- `"discounted_transactions"`

Now query:

```sql
SELECT *
FROM iceberg_catalog."lkc-90ry25"."flagged_transactions"
LIMIT 5;
```

```sql
SELECT *txn_id,CAST("$$raw-value" AS VARCHAR) FROM iceberg_catalog."lkc-90ry25"."flagged_transactions"
LIMIT 5;

```

**Example output**

| txn\_id | account\_number | currency | amount | merchant | location | withdrawal\_total | withdraw\_flag | forex\_total | fx\_flag |
| :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- |
| W2 | ACC1000123 | USD | 3000 | Walmart Inc. | New York | 7000 | YES | NULL | NULL |
| W3 | ACC1000123 | USD | 5000 | Walmart Inc. | New York | 12000 | YES | NULL | NULL |
| F3 | ACC1000456 | EUR | 800 | Amazon.com Inc. | Seattle | NULL | NO | 3300 | YES |
| Z1 | ACC1000888 | GBP | 2500 | Target Corp. | Chicago | 9000 | YES | 5000 | YES |
| ... | ... | ... | ... | ... | ... | ... | ... | ... | ... |

---

---

## 6\. Final Checklist for Trainees

1. **Phase 1 – Flink \+ Faker**  
     
   - [ ] `card_transactions` working.  
   - [ ] `retail_customers` working.  
   - [ ] `city_discounts` working.

   

2. **Phase 2 – Risk Detection**  
     
   - [ ] Global withdrawal KPI query runs.  
   - [ ] Rolling withdrawal risk query (WITHDRAW\_FLAG) runs.  
   - [ ] Rolling FX exposure query (FX\_FLAG) runs.  
   - [ ] `flagged_transactions` table populated.  
   - [ ] `fx_risk_alerts` and `withdrawal_risk_alerts` populated.  
   - [ ] Merchant failure KPI query (TUMBLE) runs.

   

3. **Phase 3 – Enrichment & Patterns**  
     
   - [ ] `discounted_transactions` table populated.  
   - [ ] `card_transactions_with_customer` view returns joined data.  
   - [ ] `merchant_failure_patterns` contains detected patterns.

   

4. **Phase 4 – Optional: Tableflow \+ DuckDB**  
     
   - [ ] Kafka topics `bank.flagged-transactions`, `bank.discounted-transactions` receive data.  
   - [ ] Tableflow enabled on both topics.  
   - [ ] DuckDB connected to Tableflow and queries succeed.

   

Following this tutorial from beginning to end will take you from **basic Flink streaming** to a **modern data stack** for real-time banking risk and promotions.  

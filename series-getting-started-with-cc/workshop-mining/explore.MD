flowchart LR
    subgraph Source["Data Sources"]
        D1["Datagen Connector: SensorData"]
        D2["Datagen Connector: ProductionStream"]
    end

    subgraph Kafka["Confluent Cloud Kafka Cluster"]
        T1["Topic: sensordata"]
        T2["Topic: productionstream"]
        T3["Topic: sensordata_rekeyed"]
        T4["Topic: productionstream_rekeyed"]
        T5["Enriched + Aggregated Tables (SensorEnriched, SensorEnriched_agg, EquipmentPerformance)"]
    end

    subgraph Flink["Confluent Cloud Flink SQL"]
        F1["Transformations (Rekey, Join, Aggregate)"]
        F2["Agentic AI Invocation (ML_PREDICT)"]
    end

    subgraph AI["Gemini AI (Google AI)"]
        A1["EquipmentAgentModel"]
        A2["(Optional) MaintenanceAdvisorAgent"]
    end

    subgraph Results["Output & Decisions"]
        R1["Equipment Health: HEALTHY / WARNING / CRITICAL"]
        R2["(Optional) Maintenance Recommendations"]
    end

    %% Connections
    D1 --> T1
    D2 --> T2

    T1 --> F1
    T2 --> F1
    F1 --> T3
    F1 --> T4
    F1 --> T5

    T3 --> F1
    T4 --> F1

    T5 --> F2
    F2 --> A1
    F2 -.-> A2

    A1 --> R1
    A2 --> R2

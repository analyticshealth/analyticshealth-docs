# Architecture Diagrams

Sólido = implementado. Tracejado = Phase 4 (planejado).

---

## Ingestion Flow

``` mermaid
flowchart TB
    classDef done fill:#1a6b5a,stroke:#2dd4bf,color:#fff
    classDef planned fill:#374151,stroke:#6b7280,color:#9ca3af,stroke-dasharray:5 5
    classDef external fill:#1e3a5f,stroke:#3b82f6,color:#93c5fd
    classDef storage fill:#3b1f5e,stroke:#a855f7,color:#d8b4fe

    EB(["EventBridge\n06:00 UTC daily"]):::external
    CLI(["Initial Load\nlocal script"]):::external
    CLIENT(["Browser / App"]):::external
    SCALE(["Scale Photo"]):::external

    subgraph api_layer["API Layer — CDK analyticshealth-infra-api"]
        APIGW["API Gateway REST\nCognito authorizer"]:::done
    end

    subgraph sfn_layer["Orchestration — CDK analyticshealth-infra-ingestion"]
        SFN["Step Functions\nstandard workflow"]:::done
        DLQ[("SQS DLQ\n14-day retention")]:::done
        CW["CloudWatch Alarms\nLambda errors · DLQ depth · SFN failures"]:::done
        SFN -->|failure| DLQ
        DLQ --- CW
    end

    subgraph lambdas["Ingestion Lambdas — SAM analyticshealth-ingestion-prod"]
        FG["fetch_garmin\npython3.12 · arm64\nPowertools structured logs"]:::done
        OCR["ocr_weight\npython3.12 · arm64\nTextract pipeline"]:::done
        MI["manual_ingest\npython3.12 · arm64\nJWT user_id extraction"]:::done
        CONSOL["consolidator\nstub — Phase 4"]:::planned
    end

    subgraph aws_services["AWS Services"]
        SM["Secrets Manager\nanalyticshealth/garmin/{user_id}"]:::done
        TEXTRACT["Amazon Textract\nDetectDocumentText"]:::done
        GARMIN["Garmin Connect API"]:::external
    end

    subgraph storage_layer["Storage — CDK analyticshealth-infra-storage"]
        S3[("S3 Data Lake\nraw/ · temp-uploads/\nprocessed/ · knowledge-base/")]:::storage
        DDB[("DynamoDB\ningestion_control\nusers · sessions")]:::storage
        KMS["KMS CMK\nanalyticshealth-storage"]:::done
    end

    EB -->|"trigger"| SFN
    SFN -->|"invoke · 3x retry\nexponential backoff"| FG
    FG --> SM
    FG --> GARMIN
    FG -->|"raw/garmin/{user_id}/"| S3
    FG --> DDB

    CLIENT --> APIGW
    APIGW -->|"Cognito JWT\nuser_id from claims"| MI
    MI -->|"raw/manual/{user_id}/"| S3
    MI --> DDB

    SCALE -->|"PUT temp-uploads/"| S3
    S3 -.->|"S3 event\n(trigger pendente)"| OCR
    OCR --> TEXTRACT
    OCR -->|"raw/weight/{user_id}/"| S3
    OCR --> DDB

    CLI -->|"one-time\nfull history"| S3
    CLI --> DDB

    KMS -. "encrypts" .-> S3
    KMS -. "encrypts" .-> DDB
```

---

## RAG + Chat Flow (Phase 4)

``` mermaid
flowchart LR
    classDef done fill:#1a6b5a,stroke:#2dd4bf,color:#fff
    classDef planned fill:#374151,stroke:#6b7280,color:#9ca3af,stroke-dasharray:5 5
    classDef storage fill:#3b1f5e,stroke:#a855f7,color:#d8b4fe
    classDef external fill:#1e3a5f,stroke:#3b82f6,color:#93c5fd

    S3[("S3\nraw/")]:::storage
    KB_S3[("S3\nknowledge-base/\n{user_id}/")]:::storage
    DDB_SESS[("DynamoDB\nsessions\nTTL 90d")]:::storage

    CONSOL["consolidator Lambda\naggregate 6 data types\nper user per year"]:::planned
    BKB["Bedrock Knowledge Base\nTitan v2 embeddings\nmetadata filter: user_id"]:::planned
    PG[("RDS pgvector\nPostgreSQL db.t4g.micro\nto be re-provisioned")]:::planned

    CLIENT(["Browser / App"]):::external
    APIGW["API Gateway\nCognito authorizer"]:::done
    CHAT["Chat Lambda\nFunction URL streaming"]:::planned
    CLAUDE["Claude 3.5 Sonnet\nAmazon Bedrock"]:::planned

    S3 -->|"scheduled or\non raw/ PUT"| CONSOL
    CONSOL -->|"consolidated docs"| KB_S3
    KB_S3 -->|"KB sync job"| BKB
    BKB -->|"store embeddings"| PG

    CLIENT --> APIGW
    APIGW -->|"JWT · user_id"| CHAT
    CHAT -->|"Retrieve\nmetadata_filter: {user_id}"| BKB
    BKB -->|"top-k chunks"| CHAT
    CHAT -->|"prompt + context"| CLAUDE
    CLAUDE -->|"streamed response"| CHAT
    CHAT --> DDB_SESS
```

---

## CDK Stack Dependency Order

``` mermaid
flowchart LR
    classDef done fill:#1a6b5a,stroke:#2dd4bf,color:#fff
    classDef planned fill:#374151,stroke:#6b7280,color:#9ca3af,stroke-dasharray:5 5

    OIDC["analyticshealth-infra-oidc\nGitHub OIDC roles\n4 repos · least-privilege"]:::done
    STORAGE["analyticshealth-infra-storage\nS3 · DynamoDB · VPC · KMS"]:::done
    API["analyticshealth-infra-api\nAPI Gateway · Cognito"]:::done
    INGESTION["analyticshealth-infra-ingestion\nStep Functions · EventBridge\nSQS DLQ · CloudWatch Alarms"]:::done
    AI["analyticshealth-infra-ai\nBedrock KB · Agent\nPhase 4 placeholder"]:::planned

    SAM["SAM: analyticshealth-ingestion-prod\nfetch_garmin · ocr_weight\nmanual_ingest · consolidator"]:::done

    OIDC --> STORAGE
    STORAGE --> API
    STORAGE --> INGESTION
    STORAGE --> AI
    SAM -->|"exports Lambda ARNs\nvia CloudFormation"| INGESTION
```

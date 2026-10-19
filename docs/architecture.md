# SEW Drive Monitoring — Architecture Document

> Serverless IoT pipeline for real-time telemetry collection, anomaly detection, and automated alerting across ~500 AC motors.

---

## 1. System Overview

The system monitors approximately 500 SEW-Eurodrive MOVIDRIVE motors installed across 10 electrical cabinets at an industrial facility. Telemetry is collected via Cisco IoT modules using Modbus TCP, converted to MQTT, and streamed to AWS IoT Core over TLS 1.2.

All compute, storage, and alerting components are managed via an AWS SAM template, with an estimated operational cost of ~$0.13 per motor per month (~$67/month total).

| Parameter | Value |
|---|---|
| **Total cabinets** | ~10 |
| **Motors per cabinet** | ~50 (average) |
| **Total motors** | ~500 |
| **Features per motor** | 13–16 (temperature, current, voltage, speed, torque, etc.) |
| **Polling interval** | Every 5 seconds |
| **Edge protocol** | Modbus TCP over local Ethernet (192.168.1.x) |
| **Cloud protocol** | MQTT over TLS 1.2 (port 8883) |

---

## 2. Architecture Layers

The architecture is organized into four logical layers: **Ingestion**, **Processing**, **Data Storage**, and **Alerting & Monitoring**.

No VPC is provisioned — all services communicate over AWS-managed public endpoints with IAM and resource-based policies controlling access. The runtime is Python 3.12 on arm64 Lambda functions.

### 2.1 Data Flow

```
Cisco Modules (10 cabinets, each with ~50 motors)
        |
        | MQTT over TLS (1 batched message per cabinet every 5s)
        v
AWS IoT Core (MQTT broker, X.509 authentication)
        |
        | IoT Rule: forward to SQS
        | Error Action: fallback to S3
        v
Amazon SQS (buffer queue, batch delivery, DLQ)
        |
        | Batch of 10 messages → Lambda
        v
AWS Lambda (single function)
        |
        +--→ DynamoDB (live data, 7-day TTL, dashboard)
        |
        +--→ S3 Parquet (long-term archive)
        |
        +--→ Threshold check (in-memory)
                |
                | anomaly detected? (not already alerted today)
                v
        Amazon SNS → Lambda → Slack #sew-alerts
```

### 2.2 MQTT Message Format

Each module publishes a single batched message every 5 seconds:

```json
{
  "cabinet": "CABINET-XX",
  "timestamp": "2026-10-05T14:32:15.000Z",
  "motors": [
    {
      "id": "MOTOR-001",
      "ip": "192.168.1.71",
      "motor_temperature": 85.2,
      "drive_temperature": 42.1,
      "output_current": 12.3,
      "output_voltage": 400,
      "motor_speed": 1450,
      "motor_torque": 28.5,
      "dc_bus_voltage": 650,
      "active_power": 4.2,
      "energy_consumed": 12450.5,
      "operating_hours": 8760,
      "status_word": 0,
      "error_code": 0
    }
  ]
}
```

---

## 3. Volume Estimates

| Metric | Value |
|---|---|
| Motor records per second | 100 (500 motors / 5s) |
| MQTT messages per second | 2 (10 cabinets / 5s, batched) |
| Records per day | 8,640,000 |
| Records per month | 259,200,000 |
| Average record size | ~500 bytes (JSON, 16 features) |
| Data per day | ~4 GB (raw), ~400 MB (Parquet) |
| Data per month | ~120 GB (raw), ~12 GB (Parquet) |
| DynamoDB storage (7-day window) | ~28 GB (bounded, auto-cleanup via TTL) |

---

## 4. AWS Services Configuration

All resources are provisioned by the SAM template (`infrastructure/template.yaml`). Resource names follow the pattern `{service}-{environment}` and are parameterized for multi-environment deployment.

| Service | Resource Name | Key Configuration |
|---|---|---|
| **AWS IoT Core** | SEWTelemetryRule_{env} | X.509 auth; SQL topic rule; sew/SITE-XXX/+/telemetry |
| **SQS Queue** | sew-telemetry-queue-{env} | Visibility 120s, Batch 10, Max receive 3, Retention 4 days, SSE |
| **SQS DLQ** | sew-telemetry-dlq-{env} | Retention 14 days, SSE, CloudWatch alarm on depth > 0 |
| **Lambda** | sew-ingest-and-alert-{env} | Python 3.12, arm64, 512 MB, 60s timeout, Concurrency 10 |
| **Lambda** | sew-slack-notify-{env} | Python 3.12, arm64, 128 MB, 10s timeout, SNS trigger |
| **DynamoDB** | SEW-Motors-{env} | PAY_PER_REQUEST, PK+SK, GSI: ByMotor, TTL 7d, PITR, SSE |
| **DynamoDB** | SEW-Config-{env} | PAY_PER_REQUEST, threshold config, SSE |
| **DynamoDB** | SEW-AlertState-{env} | PAY_PER_REQUEST, TTL 48h, anti-spam state, SSE |
| **S3** | sew-data-{env}-{acct} | AES256, public access blocked, Lifecycle: IA@30d, Glacier@90d |
| **SNS** | sew-alerts-{env} | KMS encrypted |
| **Secrets Manager** | sew-slack-webhook | Slack webhook URL |
| **CloudWatch** | 5 alarms | DLQ depth, Queue age >60s, Lambda errors/throttles, IoT failures |

---

## 5. Security Controls

| Control | Implementation |
|---|---|
| **Transport Encryption** | TLS 1.2 mandatory on all MQTT (port 8883) and HTTPS connections |
| **Device Authentication** | X.509 client certificates per Cisco IoT module |
| **Encryption at Rest** | DynamoDB SSE, S3 AES-256, SNS KMS |
| **Secrets Management** | Slack webhook in Secrets Manager with scoped IAM policy |
| **IAM Least Privilege** | SAM-generated scoped policies per Lambda |
| **Network** | No VPC; S3 public access fully blocked |
| **Data Retention** | DynamoDB TTL 7d (motors), 48h (alerts); S3 error fallback 30d |

---

## 6. Cost Breakdown

Based on 500 motors, ~2 messages/second (batched), thresholds checked on every message.

| Service | Usage Basis | Est. Monthly |
|---|---|---|
| AWS IoT Core | ~2 msg/s ingestion | $4–6 |
| SQS | ~172K msgs/day | < $1 |
| Lambda (ingest) | 512 MB arm64, 10 concurrency | $8–12 |
| Lambda (notify) | 128 MB arm64, low invocations | < $1 |
| DynamoDB (Motors) | PAY_PER_REQUEST, TTL 7d | $20–25 |
| DynamoDB (Config + Alert) | Low traffic, cached reads | $2–3 |
| S3 | Parquet + lifecycle tiering | $5–8 |
| SNS + Secrets | Low volume | < $2 |
| CloudWatch | 5 alarms + logs | $3–5 |
| **Total** | | **~$67/month** |

### Key Optimizations

- **PAY_PER_REQUEST DynamoDB** eliminates provisioned capacity costs
- **arm64 Lambda** delivers ~20% better price-performance vs x86
- **Cabinet-level batching** reduces IoT Core costs by 90% (2 msg/s vs 100 msg/s)
- **In-memory threshold engine** avoids $38K/year CloudWatch Custom Metrics cost
- **S3 lifecycle tiering** automatically moves old data to cheaper storage classes

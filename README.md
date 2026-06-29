# AquaSenseCloud — IoT Data Pipeline on AWS

End-to-end serverless data pipeline deployed on AWS for automated processing of environmental sensor data from sea buoys in the Mar Menor (Murcia, Spain). The system ingests raw CSV data, computes monthly metrics, detects anomalies, and exposes results through a REST API — all orchestrated via Infrastructure as Code.

**Course:** Infraestructura para la Computación de Altas Prestaciones  
**Degree:** Grado en Ciencia e Ingeniería de Datos — Universidad de Murcia / UPCT  
**Authors:** Álvaro Espejo Martínez · Javier Hernández Rosique

---

## Architecture Overview

The solution is deployed inside a custom **VPC** spanning two availability zones, with public and private subnets, NAT Gateway, and **VPC Gateway Endpoints** for S3 and DynamoDB (cost and latency optimization — traffic stays within AWS's private network).

```
CSV upload → S3 Bucket
                └─► Lambda 1 (ingestion & normalization) → DynamoDB proy-Datos
                                                                └─► DynamoDB Streams
                                                                        ├─► Lambda 2 (metrics computation) → DynamoDB proy-Resultados
                                                                        └─► Lambda 3 (anomaly detection) → SNS → Email alert

                                                    proy-Resultados → Flask API (ECS Cluster + ALB) → Client
```

**Infrastructure deployed (26 resources across 3 CloudFormation stacks):**

| Layer | Services |
|---|---|
| Networking | VPC, 2 public + 2 private subnets, Internet Gateway, NAT Gateway, S3/DynamoDB VPC Endpoints |
| Compute | ECS Cluster (EC2), Auto Scaling Group (4–6 instances), Application Load Balancer |
| Serverless pipeline | 3 Lambda functions + DynamoDB Streams event triggers |
| Storage | S3 (raw ingestion), DynamoDB proy-Datos (raw), DynamoDB proy-Resultados (processed) |
| Notifications | SNS Topic + Email Subscription |
| Observability | CloudWatch Dashboard (Lambda invocations + ECS CPU/memory) |
| Container registry | ECR Repository (Flask Docker image) |

---

## Pipeline Detail

### Lambda 1 — Ingestion
Triggered automatically on S3 file upload. Reads the CSV, normalizes the date field into `YearMonth` (partition key) and `Day` (sort key) for efficient DynamoDB queries, and batch-writes records to `proy-Datos` to avoid dropping Streams events.

### Lambda 2 — Metrics Computation
Triggered by DynamoDB Streams on every insert/update. Computes monthly metrics incrementally (only reprocesses affected months):
- Monthly mean temperature
- Standard deviation across weekly samples
- Difference between current month's max temperature and previous month's max

Results are written to `proy-Resultados`.

### Lambda 3 — Anomaly Detection
Triggered by the same DynamoDB Streams. Scans each batch (up to 100 records) for weekly deviations exceeding the threshold of 0.5. If anomalies are detected, groups affected dates into a single SNS notification to avoid alert flooding.

---

## Flask REST API

Deployed as a Docker container on the ECS cluster, behind an Application Load Balancer. Serves pre-computed metrics from `proy-Resultados` with efficient DynamoDB key queries.

**Endpoints:**

| Route | Description |
|---|---|
| `GET /temp?month=M&year=YYYY` | Monthly mean temperature |
| `GET /sd?month=M&year=YYYY` | Standard deviation of weekly samples |
| `GET /maxdiff?month=M&year=YYYY` | Difference between current and previous month's max temperature |

Each response includes the queried date and the private IP of the EC2 instance that served the request (useful for verifying load balancer distribution).

---

## Deployment

The infrastructure is split into 3 CloudFormation stacks deployed in order:

```bash
# 1. Network layer (VPC, subnets, gateways, endpoints)
aws cloudformation deploy --template-file proy-icap-red.yaml --stack-name proy-icap-red

# 2. Compute layer (ECS cluster, ALB, ECR, Docker build, CloudWatch dashboard)
aws cloudformation deploy --template-file proy-icap-server.yaml \
  --stack-name proy-icap-server \
  --parameter-overrides NetworkStackName=proy-icap-red

# 3. Pipeline layer (S3, DynamoDB, Lambdas, SNS)
aws cloudformation deploy --template-file proy-icap-pipeline.yaml --stack-name proy-icap-pipeline
```

**After deployment:**
1. Manually set the S3 bucket as the trigger for Lambda 1 (console limitation)
2. Accept the SNS email subscription
3. Upload a CSV file to the S3 bucket to trigger the full pipeline

> **Note:** The S3 bucket name is hardcoded in the CloudFormation template. If deploying to a different AWS account, update `BucketName` in `proy-icap-pipeline.yaml` (S3 bucket names are globally unique).

---

## Tech Stack

| Category | Technologies |
|---|---|
| Cloud & IaC | AWS (S3, Lambda, DynamoDB, ECS, ECR, ALB, SNS, CloudWatch, VPC), CloudFormation |
| Language | Python 3.12 (Lambdas), Python 3.9 (Flask API) |
| Container | Docker, Amazon ECR |
| Libraries | boto3, Flask, ec2-metadata |

---

## Contributors

- **Álvaro Espejo Martínez** — Lambda 1 & 2 (ingestion, normalization, metrics), full IaC automation (CloudFormation templates for VPC, ECS, pipeline, dashboard)  
  [![GitHub](https://img.shields.io/badge/GitHub-181717?style=for-the-badge&logo=github&logoColor=white)](https://github.com/espejomartinezalvaro-spec)

- **Javier Hernández Rosique** — Flask API, DynamoDB integration, Lambda 3 & pipeline automation  
  [![GitHub](https://img.shields.io/badge/GitHub-181717?style=for-the-badge&logo=github&logoColor=white)](https://github.com/javierhernandezrosique)

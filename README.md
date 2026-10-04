# URL Shortener - Serverless REST API

A production-ready, serverless URL shortener built with **Python 3.13**, **AWS Lambda**, **Amazon API Gateway**, and **Amazon DynamoDB**. Create short links, track real-time click statistics, and execute low-latency redirects—all with zero server management.

[![Runtime architecture: client, API Gateway, Lambda and DynamoDB](docs/screenshots/architecture.png)](https://riya6389.github.io/url-shortener/architecture.html)

### ▶️ [Explore the Interactive Architecture Diagram](https://riya6389.github.io/url-shortener/architecture.html)

---

## 📌 Project Overview

This project provides a robust, highly scalable URL shortening service designed around cloud-native and serverless best practices:
- **Zero Server Maintenance:** Fully managed infrastructure using AWS Lambda and Amazon API Gateway.
- **High Concurrency & Low Latency:** On-demand DynamoDB single-table design with sub-10ms response times.
- **Cost Efficient:** Pay-per-use billing that scales to zero when idle, fitting comfortably within the AWS Free Tier.
- **Enterprise-Grade Reliability:** Automated multi-stage CI/CD deployments, point-in-time recovery (PITR), data retention policies, and CloudWatch alarm notifications.

---

## ⚙️ Architecture

```
┌─────────────┐      ┌──────────────────────────┐      ┌─────────────┐      ┌──────────────┐
│   Client    │─────▶│       API Gateway        │─────▶│   Lambda    │─────▶│  DynamoDB    │
│ (Browser /  │◀─────│  (REST API, Usage Plans, │◀─────│ (Python 3.13│◀─────│  (NoSQL      │
│  curl / app)│      │  Throttling & TLS)       │      │  Router)    │      │   Storage)   │
└─────────────┘      └──────────────────────────┘      └─────────────┘      └──────────────┘
                                                              │
                                                              ▼
                                               CloudWatch Logs + AWS X-Ray Tracing
                                               CloudWatch Alarms ──▶ SNS Alerts ──▶ Email
```

### Request Flow

1. **Client Request:** A client sends an HTTP request to the API Gateway endpoint (or optional regional custom domain).
2. **API Gateway Controls:**
   - Evaluates API key validity on creation requests (`POST /links`).
   - Applies usage plan quotas and stage throttling to guard against abuse.
   - Forwards authorized requests to the Lambda function.
3. **Lambda Execution (`app.py`):**
   - **`POST /links`:** Validates destination URL, generates a cryptographically secure 6-character short code, and commits to DynamoDB with collision retries.
   - **`GET /{code}`:** Verifies existence, atomically increments the visit counter, and returns an uncacheable `302 Found` redirect.
   - **`GET /links/{code}/stats`:** Fetches link metadata, click count, and creation timestamp.
4. **DynamoDB Persistence:**
   - Stores link mappings with `short_code` as the partition key.
   - Atomic conditional writes ensure strict data integrity.

---

## 🚀 Key Features

- **Cryptographically Secure Short Codes:** Uses Python's `secrets` module to generate 6-character random alphanumeric identifiers ($62^6 \approx 56.8\text{ billion}$ combinations), eliminating sequential prediction risks.
- **Atomic Collision Handling:** Employs DynamoDB `ConditionExpression='attribute_not_exists(short_code)'` with an automated 5-attempt retry loop to handle code collisions safely.
- **Atomic Click Tracking:** Leverages DynamoDB `UpdateItem` with `ADD click_count :inc` for atomic counter increments without concurrency race conditions.
- **Accurate Analytics via Temporary Redirects:** Emits HTTP `302 Found` redirects instead of cached `301 Moved Permanently`, ensuring repeat visits hit the backend and are recorded accurately.
- **Safe Route Resolution:** Redirect updates check `ConditionExpression='attribute_exists(short_code)'`, preventing ghost record insertion and correctly returning `404 Not Found` for nonexistent codes.
- **Strict Request Validation:** Enforces URL presence, HTTP/HTTPS scheme verification via `urllib.parse`, and an upper length threshold of 2,048 characters.
- **Tiered Security & Rate Limiting:** API Gateway Usage Plans throttle link creation (5 req/s, burst 10, 1,000/day per API key) while public redirect and stats routes handle up to 50 req/s with a burst of 100.
- **Least-Privilege IAM Security:** The Lambda execution role policy permits only `dynamodb:GetItem`, `dynamodb:PutItem`, and `dynamodb:UpdateItem` against the single dedicated table.
- **Data Protection & Disaster Recovery:** The DynamoDB table is provisioned with `PAY_PER_REQUEST` billing, `PointInTimeRecoverySpecification` enabled, and `DeletionProtectionEnabled: true` with `DeletionPolicy: Retain`.
- **Full Observability & Alerting:** End-to-end AWS X-Ray distributed tracing, custom CloudWatch log groups with retention lifecycle, and automated alarms for 5xx errors, Lambda errors, throttles, and latency anomalies.
- **Automated CI/CD Pipeline:** GitHub Actions workflow utilizing OIDC for passwordless AWS authentication, orchestrating unit tests, staging deployments, integration tests with automated data cleanup, and production promotion with smoke tests.

---

## 🛠️ Tech Stack

| Layer | Technology | Purpose |
|---|---|---|
| **Compute** | [AWS Lambda](https://aws.amazon.com/lambda/) (Python 3.13) | Serverless application logic and request routing |
| **API Management** | [Amazon API Gateway](https://aws.amazon.com/api-gateway/) | REST routing, usage plans, API key validation, and throttling |
| **Database** | [Amazon DynamoDB](https://aws.amazon.com/dynamodb/) | Low-latency NoSQL storage, atomic counters, PITR backup |
| **Infrastructure as Code** | [AWS SAM](https://aws.amazon.com/serverless/sam/) / CloudFormation | Declarative resource templates and configuration |
| **Observability** | [Amazon CloudWatch](https://aws.amazon.com/cloudwatch/) & [AWS X-Ray](https://aws.amazon.com/xray/) | Centralized logs, error metrics, latency alarms, and distributed tracing |
| **Alerting** | [Amazon SNS](https://aws.amazon.com/sns/) | Instant email notifications triggered by CloudWatch alarms |
| **Configuration** | [AWS SSM Parameter Store](https://docs.aws.amazon.com/systems-manager/) | Out-of-repo configuration management for alert recipients |
| **CI/CD** | [GitHub Actions](https://github.com/features/actions) + AWS OIDC | Automated testing, staging deployment, and production rollout |
| **Testing** | [pytest](https://pytest.org/) | Unit test suites, integration test harness, and smoke tests |

---

## 📂 Project Structure

```
url-shortener/
├── url_shortener/                          # Lambda application source code
│   ├── __init__.py
│   ├── app.py                              # Lambda entrypoint & HTTP router
│   ├── shortener.py                        # Validation & short code generation logic
│   └── requirements.txt                    # Runtime dependencies (boto3)
├── tests/                                  # Automated test suites
│   ├── __init__.py
│   ├── unit/                               # Isolated unit tests (mocked DynamoDB)
│   │   ├── __init__.py
│   │   ├── test_shortener.py               # Short code generator & validation tests
│   │   └── test_handler.py                 # Router, redirects, stats & error tests
│   ├── integration/                        # End-to-end integration tests (real API)
│   │   ├── __init__.py
│   │   └── test_api.py                     # API tests with automated test data cleanup
│   └── smoke/                              # Read-only post-deployment sanity tests
│       ├── __init__.py
│       └── test_smoke.py                   # Health and API key validation tests
├── events/                                 # Mock API Gateway events for local testing
│   ├── create_link.json
│   ├── get_redirect.json
│   └── get_stats.json
├── docs/                                   # Architectural documentation & assets
│   ├── architecture.html                   # Interactive architecture visualizer
│   ├── architecture.json                   # Diagram definition
│   └── screenshots/                        # Architectural diagrams
├── .github/workflows/
│   └── deploy.yml                          # Continuous integration & deployment pipeline
├── URL-Shortener.postman_collection.json   # Ready-to-import Postman test collection
├── POSTMAN_GUIDE.md                        # Step-by-step Postman collection guide
├── CUSTOM_DOMAIN_SETUP.md                  # Custom domain, ACM certificate & DNS instructions
├── DEPLOYMENT.md                           # Detailed AWS setup, IAM roles & deployment guide
├── template.yaml                           # AWS SAM infrastructure definition
├── samconfig.toml                          # Staging and production deployment parameters
├── pytest.ini                              # Pytest test configuration
├── .gitignore                              # Git exclusion rules (secrets, venv, caches)
└── README.md                               # Project documentation
```

---

## 📡 API Specification

### Base URL
```
https://<api-id>.execute-api.<region>.amazonaws.com/Prod
# Or your custom domain if configured: https://url-shortener.yourdomain.com
```

### 1. Create Short Link
Creates a new short URL for a destination target.

- **Method / Path:** `POST /links`
- **Authentication:** `x-api-key: YOUR_API_KEY` (Required)
- **Request Headers:** `Content-Type: application/json`
- **Request Body:**
  ```json
  {
    "url": "https://github.com/Riya6389/url-shortener"
  }
  ```

**Sample Request:**
```bash
curl -X POST https://<api-endpoint>/links \
  -H "Content-Type: application/json" \
  -H "x-api-key: YOUR_API_KEY" \
  -d '{"url": "https://github.com/Riya6389/url-shortener"}'
```

**Success Response (`201 Created`):**
```json
{
  "short_code": "a3X9mK",
  "short_url": "https://<api-endpoint>/a3X9mK",
  "original_url": "https://github.com/Riya6389/url-shortener"
}
```

---

### 2. Redirect to Original URL
Resolves the short code, increments the visit count, and redirects.

- **Method / Path:** `GET /{code}`
- **Authentication:** None (Public)

**Sample Request:**
```bash
curl -i https://<api-endpoint>/a3X9mK
```

**Success Response (`302 Found`):**
```http
HTTP/1.1 302 Found
Location: https://github.com/Riya6389/url-shortener
Access-Control-Allow-Origin: *
```

---

### 3. Get Link Statistics
Retrieves click metrics and creation timestamps for a short code.

- **Method / Path:** `GET /links/{code}/stats`
- **Authentication:** None (Public)

**Sample Request:**
```bash
curl https://<api-endpoint>/links/a3X9mK/stats
```

**Success Response (`200 OK`):**
```json
{
  "short_code": "a3X9mK",
  "original_url": "https://github.com/Riya6389/url-shortener",
  "click_count": 1,
  "created_at": "2026-10-04T14:30:00.123456"
}
```

---

### Error Responses

| Status Code | Meaning | Example Payload |
|---|---|---|
| `400 Bad Request` | Missing/empty payload, non-string URL, or invalid protocol | `{"error": "Missing required field: 'url'"}` |
| `403 Forbidden` | Missing or invalid `x-api-key` header | `{"message": "Forbidden"}` |
| `404 Not Found` | Short code does not exist | `{"error": "Short link not found"}` |
| `429 Too Many Requests` | Rate limit or daily quota breached | `{"message": "Too Many Requests"}` |
| `500 Internal Server Error` | Unexpected server or database exception | `{"error": "Internal server error"}` |

---

## 💻 Setup & Run Steps

### 1. Prerequisites
- **Python 3.12+** (Python 3.13 recommended)
- **AWS CLI v2** configured with necessary permissions (`aws configure`)
- **AWS SAM CLI** installed ([Install Guide](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/install-sam-cli.html))
- **Docker** (optional, for local containerized builds and emulation)

---

### 2. Local Environment Setup

1. **Clone the repository:**
   ```bash
   git clone https://github.com/Riya6389/url-shortener.git
   cd url-shortener
   ```

2. **Create and activate a virtual environment:**
   ```bash
   # Windows (PowerShell)
   python -m venv .venv
   .\.venv\Scripts\Activate.ps1

   # Linux / macOS
   python3 -m venv .venv
   source .venv/bin/activate
   ```

3. **Install dependencies:**
   ```bash
   pip install -r url_shortener/requirements.txt
   pip install pytest requests
   ```

---

### 3. Running Unit Tests

Unit tests run completely offline without requiring active AWS credentials:

```bash
# Run all unit tests
pytest tests/unit/ -v

# Run with short traceback
pytest tests/unit/ -v --tb=short
```

---

### 4. Local API Emulation with SAM

You can invoke Lambda functions directly or run a local API Gateway emulator:

```bash
# Direct function invocation with a mock event
sam local invoke UrlShortenerFunction -e events/create_link.json

# Start a local HTTP server on port 3000
sam local start-api

# In another terminal:
curl -X POST http://127.0.0.1:3000/links \
  -H "Content-Type: application/json" \
  -d '{"url": "https://example.com"}'
```

---

### 5. Deployment to AWS

#### One-Time Setup: Alert Email
Store your email in AWS Systems Manager (SSM) Parameter Store to receive CloudWatch alert notifications:
```bash
aws ssm put-parameter \
  --name /url-shortener/alert-email \
  --type String \
  --value you@example.com \
  --region us-east-1
```

#### Build and Deploy:
```bash
# 1. Build the SAM template
sam build

# 2. Deploy to staging
sam deploy --config-env staging

# 3. Deploy to production
sam deploy --config-env prod
```

#### Retrieve Stack Outputs:
```bash
# View endpoints and table details
sam list stack-outputs --stack-name url-shortener-prod

# Fetch the generated API Key value
KEY_ID=$(aws cloudformation describe-stacks --stack-name url-shortener-prod \
  --query "Stacks[0].Outputs[?OutputKey=='ApiKeyId'].OutputValue" --output text)

aws apigateway get-api-key --api-key "$KEY_ID" --include-value --query value --output text
```

---

## 🧪 Postman Collection

A pre-configured Postman collection is included in the root directory:
1. Open Postman and click **Import**.
2. Select [`URL-Shortener.postman_collection.json`](URL-Shortener.postman_collection.json).
3. Set your collection variables:
   - `base_url`: Your deployed API endpoint
   - `api_key`: Your API key for link creation
4. Refer to [POSTMAN_GUIDE.md](POSTMAN_GUIDE.md) for full instructions.

---

## 🔒 Security Best Practices Implemented

- **IAM Scoping:** Zero wildcards on data modifications; the Lambda role can only touch the specific stack table.
- **DDoS & Scraping Mitigation:** API Gateway usage plans restrict creation endpoints, while global throttling limits redirect surges.
- **SSM Secret Encapsulation:** Operational alert emails are resolved at deploy-time from SSM and never stored in code.
- **Browser Protection:** HTTP `302` headers prevent aggressive caching from corrupting visit tallies.
- **Git Hygiene:** Strict `.gitignore` configurations prevent `.env`, credentials, AWS tokens, and build artifacts from entering version control.

---

## 👤 Author

**Riya Agrahari**
- GitHub: [@Riya6389](https://github.com/Riya6389)

---

## 📄 License

This project is licensed under the [MIT License](LICENSE).

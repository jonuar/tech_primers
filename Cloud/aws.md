# AWS Cheatsheet

## Mental Model

AWS is organized around **services** (compute, storage, networking, ML, etc.) connected by **IAM permissions**. The mental model: every resource is an object, every action requires a permission, and everything is regional by default. For AI/backend work, you'll live in a subset: Lambda + API Gateway (serverless), EC2/ECS (containers), S3 (object storage), ECR (Docker images), CloudWatch (logs/metrics), and SageMaker (ML).

---

## Install & Minimal Setup

```bash
# Install AWS CLI v2
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip && sudo ./aws/install

# Configure credentials
aws configure
# Prompts: Access Key ID, Secret Access Key, region (e.g. us-east-1), output format (json)

# Verify
aws sts get-caller-identity

# Install boto3 (Python SDK)
pip install boto3
```

---

## Core Concepts

### IAM — Identity & Access Management
The permission layer for everything. Master this before anything else.

- **User** — a person or CI/CD system
- **Role** — an identity assumed by a service (Lambda, EC2, SageMaker). Always prefer roles over embedding keys.
- **Policy** — JSON document that grants or denies actions on resources
- **Principle of least privilege** — only grant the exact permissions needed

```json
// Example policy: allow a Lambda to read from a specific S3 bucket
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": ["s3:GetObject", "s3:ListBucket"],
    "Resource": [
      "arn:aws:s3:::my-bucket",
      "arn:aws:s3:::my-bucket/*"
    ]
  }]
}
```

### S3 — Object Storage
Infinitely scalable file storage. Think of it as a key-value store for files.

```bash
# CLI basics
aws s3 ls s3://my-bucket
aws s3 cp local-file.txt s3://my-bucket/path/file.txt
aws s3 sync ./local-dir s3://my-bucket/prefix/
aws s3 rm s3://my-bucket/path/file.txt
```

```python
import boto3

s3 = boto3.client("s3")

# Upload
s3.upload_file("local.txt", "my-bucket", "remote/key.txt")

# Download
s3.download_file("my-bucket", "remote/key.txt", "local.txt")

# Read directly into memory
response = s3.get_object(Bucket="my-bucket", Key="data.json")
content = response["Body"].read().decode("utf-8")

# Generate a presigned URL (temporary access)
url = s3.generate_presigned_url(
    "get_object",
    Params={"Bucket": "my-bucket", "Key": "file.txt"},
    ExpiresIn=3600  # seconds
)
```

### Lambda — Serverless Functions
Stateless functions that run on demand. Billed per invocation + duration. Cold starts are a real concern for latency-sensitive APIs.

```python
# Handler signature — event varies by trigger source
def handler(event, context):
    body = event.get("body", "")          # from API Gateway
    records = event.get("Records", [])    # from SQS / S3

    return {
        "statusCode": 200,
        "headers": {"Content-Type": "application/json"},
        "body": json.dumps({"message": "ok"})
    }
```

```bash
# Deploy with zip (minimal)
zip function.zip handler.py
aws lambda update-function-code \
  --function-name my-function \
  --zip-file fileb://function.zip
```

### API Gateway
HTTP layer in front of Lambda or any backend. Two flavors:
- **REST API** — full-featured, more config
- **HTTP API** — simpler, cheaper, faster (prefer this for most cases)

Key concepts: routes, integrations, stages, authorizers (JWT / Lambda).

### EC2 & ECS — Compute

```bash
# EC2: SSH into instance
ssh -i "key.pem" ec2-user@<public-ip>

# ECS: deploy a new task definition revision
aws ecs update-service \
  --cluster my-cluster \
  --service my-service \
  --force-new-deployment
```

### ECR — Container Registry
Push Docker images here to deploy to ECS, Lambda (container), or SageMaker.

```bash
# Authenticate Docker to ECR
aws ecr get-login-password --region us-east-1 \
  | docker login --username AWS --password-stdin \
    <account-id>.dkr.ecr.us-east-1.amazonaws.com

# Tag and push
docker build -t my-app .
docker tag my-app:latest <account-id>.dkr.ecr.us-east-1.amazonaws.com/my-app:latest
docker push <account-id>.dkr.ecr.us-east-1.amazonaws.com/my-app:latest
```

### CloudWatch — Logs & Metrics

```bash
# Tail Lambda logs in real time
aws logs tail /aws/lambda/my-function --follow

# Query logs with Insights
aws logs start-query \
  --log-group-name /aws/lambda/my-function \
  --start-time $(date -d "1 hour ago" +%s) \
  --end-time $(date +%s) \
  --query-string 'fields @timestamp, @message | filter @message like /ERROR/'
```

```python
import boto3

cloudwatch = boto3.client("cloudwatch")

# Publish a custom metric
cloudwatch.put_metric_data(
    Namespace="MyApp",
    MetricData=[{
        "MetricName": "InferenceLatency",
        "Value": 142.5,
        "Unit": "Milliseconds"
    }]
)
```

### SageMaker — ML Platform (Key Concepts)

| Concept | What it does |
|---|---|
| **Training Job** | Runs a training script on managed compute |
| **Model** | Artifact registered from a training job |
| **Endpoint** | Deployed, real-time inference server |
| **Batch Transform** | Offline inference on large datasets |
| **Pipeline** | Automated ML workflow (train → eval → deploy) |
| **Model Registry** | Versioned store for approved models |
| **Studio** | Web IDE for the whole ML lifecycle |

```python
import sagemaker
from sagemaker.sklearn import SKLearn

session = sagemaker.Session()
role = "arn:aws:iam::<account>:role/SageMakerRole"

# Launch a training job
estimator = SKLearn(
    entry_point="train.py",
    role=role,
    instance_type="ml.m5.xlarge",
    framework_version="1.2-1",
    sagemaker_session=session
)
estimator.fit({"train": "s3://my-bucket/data/"})

# Deploy as real-time endpoint
predictor = estimator.deploy(
    initial_instance_count=1,
    instance_type="ml.t2.medium"
)

# Invoke
result = predictor.predict([[1.0, 2.0, 3.0]])
```

---

## Most-Used Patterns

### Environment Variables in Lambda
```python
import os
DB_URL = os.environ["DB_URL"]  # set in Lambda config, not hardcoded
```

### Secrets Manager
```python
import boto3, json

client = boto3.client("secretsmanager", region_name="us-east-1")
secret = json.loads(
    client.get_secret_value(SecretId="prod/myapp/db")["SecretString"]
)
```

### SQS Trigger for Lambda (async decoupling)
```python
def handler(event, context):
    for record in event["Records"]:
        message = json.loads(record["body"])
        process(message)
        # Successful return = SQS deletes the message automatically
```

---

## Gotchas

- **Lambda cold starts** — first invocation after inactivity spins up a container (~100ms–1s+). Provisioned concurrency eliminates this at extra cost.
- **Lambda timeout** — default 3s, max 15min. AI inference often needs 30–60s; set timeout explicitly.
- **IAM role vs user keys** — never hardcode `AWS_ACCESS_KEY_ID` in code. Attach a role to the service.
- **S3 eventual consistency** — listing after a write may not reflect immediately in edge cases. For critical flows, read by exact key.
- **Region mismatch** — resources must be in the same region to communicate cheaply. Lambda calling RDS in another region adds latency and cost.
- **SageMaker endpoints cost money while idle** — always delete endpoints after testing unless they're in production.

---

## Quick Links

- [AWS CLI Command Reference](https://awscli.amazonaws.com/v2/documentation/api/latest/index.html)
- [boto3 docs](https://boto3.amazonaws.com/v1/documentation/api/latest/index.html)
- [SageMaker Python SDK](https://sagemaker.readthedocs.io)
- [AWS IAM Policy Simulator](https://policysim.aws.amazon.com) — test permissions before deploying
- [LocalStack](https://localstack.cloud) — run AWS services locally for dev/testing

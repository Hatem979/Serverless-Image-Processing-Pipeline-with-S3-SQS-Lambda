# Serverless Image Processing Pipeline

> **Event-driven, resilient, and fully serverless image-processing architecture built with AWS managed services.**

![AWS](https://img.shields.io/badge/AWS-Cloud-orange)
![Architecture](https://img.shields.io/badge/Architecture-Serverless-blue)
![S3](https://img.shields.io/badge/Amazon%20S3-Storage-green)
![Lambda](https://img.shields.io/badge/AWS%20Lambda-Compute-purple)
![SQS](https://img.shields.io/badge/Amazon%20SQS-Messaging-red)
![Step Functions](https://img.shields.io/badge/Step%20Functions-Orchestration-blue)

---

## Overview

This project implements a **serverless image-processing pipeline on AWS**.

Users upload images securely to an Amazon S3 source bucket using **pre-signed URLs**. S3 generates an event that is sent to an **Amazon SQS queue**, decoupling image uploads from the processing layer.

An AWS Lambda function consumes messages from SQS and starts an **AWS Step Functions** workflow responsible for processing the image and updating its metadata.

Processed images are stored in a separate S3 destination bucket and delivered globally through **Amazon CloudFront**.

Image metadata and processing status are stored in **Amazon DynamoDB**, while **Amazon SNS** is used for processing notifications. Failed messages can be isolated using an **SQS Dead-Letter Queue (DLQ)**.

---

# Architecture

![Serverless Image Processing Pipeline Architecture](./architecture/architecture-diagram.png)

### Architecture Flow

```text
User
 │
 ▼
API Gateway
 │
 ▼
Lambda
 │
 │ Generate Pre-signed URL
 ▼
S3 Source Bucket
 │
 │ ObjectCreated Event
 ▼
SQS Processing Queue
 │
 ▼
Lambda Queue Processor
 │
 ▼
Step Functions
 │
 ├── Process Image
 │     ├── Download from S3
 │     ├── Resize
 │     ├── Upload to Destination S3
 │     └── Update DynamoDB
 │
 └── Send SNS Notification
       
S3 Destination Bucket
 │
 ▼
CloudFront
 │
 ▼
User

Failed Messages
 │
 ▼
SQS Dead-Letter Queue
```

---

# Why This Architecture?

A synchronous image-processing application can force users to wait for processing and can become difficult to scale when many images arrive simultaneously.

This architecture separates **uploading** from **processing**.

```text
Upload Rate
     │
     ▼
    S3
     │
     ▼
    SQS
     │
     ▼
Processing Rate
```

SQS acts as a buffer between the upload and processing layers.

This provides:

* Loose coupling
* Asynchronous processing
* Automatic retries
* Failure isolation
* Better resilience
* Independent scaling
* Improved user experience

---

# End-to-End Workflow

## 1. Request Upload URL

The client sends a request to API Gateway.

```text
User
 ↓
API Gateway
 ↓
Lambda
```

Lambda generates a temporary **S3 pre-signed URL**.

The client uses this URL to upload the image directly to S3.

### Why pre-signed URLs?

* Keeps the S3 bucket private.
* Provides temporary upload access.
* Avoids sending image data through Lambda.
* Reduces unnecessary API Gateway and Lambda traffic.
* Separates API handling from object transfer.

---

## 2. Upload Image to S3

The image is uploaded directly to the source S3 bucket.

```text
User
 │
 │ PUT image
 ▼
S3 Source Bucket
```

Example:

```text
source-bucket/
└── uploads/
    ├── image-001.jpg
    ├── image-002.jpg
    └── image-003.jpg
```

The source bucket is designed to remain private.

---

## 3. S3 Event → SQS

When an image is uploaded, S3 generates an `ObjectCreated` event.

```text
S3
 │
 │ ObjectCreated
 ▼
SQS Processing Queue
```

SQS provides the asynchronous messaging layer between S3 and the processing system.

### Why SQS?

Instead of processing every upload immediately, messages are placed in a durable queue.

This allows the processing layer to consume messages independently from the upload rate.

---

## 4. SQS → Lambda

Lambda polls the SQS queue and processes incoming messages.

```text
SQS
 │
 ▼
Lambda
```

The Lambda function retrieves the uploaded object's information and starts the corresponding Step Functions execution.

This separates:

```text
Message Consumption
        ≠
Workflow Orchestration
```

---

# 5. Step Functions Workflow

AWS Step Functions orchestrates the image-processing workflow.

The current workflow consists of two main states:

```text
Start
 │
 ▼
ProcessImage
 │
 ├── Download Image
 ├── Resize Image
 ├── Upload Processed Image
 └── Update DynamoDB
 │
 ▼
SendSuccessNotification
 │
 ▼
SNS
 │
 ▼
End
```

The workflow is defined using Amazon States Language (JSON).

---

# Implementation

## AWS Lambda — Image Processing Function

The Lambda function performs the core image-processing logic.

### Responsibilities

1. Parse the image location from the Step Functions input.
2. Download the source image from S3.
3. Store the temporary file in Lambda `/tmp`.
4. Resize the image using Pillow.
5. Upload the processed image to the destination S3 bucket.
6. Update the image status and processed object key in DynamoDB.
7. Return the processing result to Step Functions.

### Lambda Implementation

```python
import json
import boto3
import os
from PIL import Image
import io

s3 = boto3.client('s3')
dynamodb = boto3.resource('dynamodb')

DEST_BUCKET = 'my-destination-images-bucket-859816906510-us-east-1-an'
TABLE_NAME = 'ImageMetadata'


def lambda_handler(event, context):
    print("Received event:", json.dumps(event))

    if 'bucket' in event:
        bucket = event['bucket']
        key = event['key']

    elif 'Payload' in event and 'bucket' in event['Payload']:
        bucket = event['Payload']['bucket']
        key = event['Payload']['key']

    else:
        raise ValueError(
            f"Could not parse bucket and key from event: {event}"
        )

    # 1. Download image from Source S3 Bucket to /tmp
    download_path = f"/tmp/{os.path.basename(key)}"

    s3.download_file(
        bucket,
        key,
        download_path
    )

    # 2. Image processing - Resize
    with Image.open(download_path) as img:

        img.thumbnail((800, 800))

        buffer = io.BytesIO()

        img_format = img.format if img.format else 'JPEG'

        img.save(
            buffer,
            format=img_format
        )

        buffer.seek(0)

        # 3. Upload image to Destination Bucket
        dest_key = f"processed-{os.path.basename(key)}"

        s3.put_object(
            Bucket=DEST_BUCKET,
            Key=dest_key,
            Body=buffer,
            ContentType=f"image/{img_format.lower()}"
        )

    # 4. Update image status in DynamoDB
    table = dynamodb.Table(TABLE_NAME)

    table.update_item(
        Key={'ImageId': key},
        UpdateExpression=(
            "SET #s = :completed, "
            "ProcessedKey = :pk"
        ),
        ExpressionAttributeNames={
            '#s': 'Status'
        },
        ExpressionAttributeValues={
            ':completed': 'COMPLETED',
            ':pk': dest_key
        }
    )

    return {
        'statusCode': 200,
        'body': json.dumps({
            'message': 'Image processed successfully',
            'processedKey': dest_key
        })
    }
```

### Lambda Processing Flow

```text
Step Functions Input
        │
        ▼
Parse Bucket + Key
        │
        ▼
Download Image
        │
        ▼
/tmp
        │
        ▼
Resize with Pillow
        │
        ▼
Upload to Destination S3
        │
        ▼
Update DynamoDB
        │
        ▼
Return Result
```

> **Note:** The current implementation performs image resizing. The architecture can be extended with a watermarking step using Pillow or another image-processing library.

---

# AWS Step Functions — Workflow Definition

The Step Functions state machine orchestrates the Lambda processing function and then publishes a success notification through SNS.

### Workflow Definition

```json
{
  "Comment": "Serverless Image Processing Workflow with SNS",
  "StartAt": "ProcessImage",
  "States": {
    "ProcessImage": {
      "Type": "Task",
      "Resource": "arn:aws:states:::lambda:invoke",
      "Parameters": {
        "FunctionName": "arn:aws:lambda:us-east-1:859816906510:function:ProcessImageFunction:$LATEST",
        "Payload.$": "$"
      },
      "Next": "SendSuccessNotification"
    },
    "SendSuccessNotification": {
      "Type": "Task",
      "Resource": "arn:aws:states:::sns:publish",
      "Parameters": {
        "TopicArn": "YOUR_SNS_TOPIC_ARN",
        "Message": {
          "Input.$": "$",
          "Status": "Success - Image successfully processed and delivered to CloudFront"
        }
      },
      "End": true
    }
  }
}
```

### Workflow Logic

```text
                    ┌──────────────────┐
                    │   Start          │
                    └────────┬─────────┘
                             │
                             ▼
                    ┌──────────────────┐
                    │  ProcessImage    │
                    │     Lambda       │
                    └────────┬─────────┘
                             │
                    Image Processing
                             │
                             ▼
                    ┌──────────────────┐
                    │ SendSuccess      │
                    │ Notification     │
                    │      SNS         │
                    └────────┬─────────┘
                             │
                             ▼
                            End
```

---

# Important Configuration

The Step Functions definition contains environment-specific AWS identifiers.

Before publishing the code publicly, replace or parameterize:

```text
FunctionName
TopicArn
Bucket Names
Table Names
AWS Account IDs
AWS Region
```

For example:

```json
"TopicArn": "YOUR_SNS_TOPIC_ARN"
```

should be replaced during deployment with the actual SNS topic ARN.

For a production-style implementation, these values should preferably be managed through **Infrastructure as Code, environment variables, or deployment parameters** rather than hard-coded directly into application code.

---

# S3 Destination

Processed images are stored in the destination S3 bucket.

Example:

```text
destination-bucket/
└── processed/
    ├── processed-image-001.jpg
    ├── processed-image-002.jpg
    └── processed-image-003.jpg
```

The separation between source and destination buckets allows different:

* Access policies
* Lifecycle policies
* Retention strategies
* Storage classes

---

# DynamoDB Metadata

Amazon DynamoDB stores image metadata and processing status.

Example item:

```json
{
  "ImageId": "uploads/image-001.jpg",
  "Status": "COMPLETED",
  "ProcessedKey": "processed-image-001.jpg"
}
```

Possible states:

```text
UPLOADED
PROCESSING
COMPLETED
FAILED
```

---

# SNS Notifications

Amazon SNS is used to notify subscribers after successful processing.

```text
Step Functions
      │
      ▼
SNS Topic
      │
      ▼
Subscribers
```

The current workflow sends a success notification after the image-processing Lambda completes successfully.

---

# CloudFront

Amazon CloudFront provides global delivery for processed images.

```text
User
 │
 ▼
CloudFront
 │
 ▼
S3 Destination Bucket
```

CloudFront provides:

* Edge caching
* Lower latency
* Global content delivery
* Reduced repeated origin requests

---

# Failure Handling

The architecture uses SQS retries and a Dead-Letter Queue.

```text
SQS
 │
 ▼
Lambda
 │
 ├── Success → Step Functions
 │
 └── Failure
       │
       ▼
     Retry
       │
       ▼
     Retry
       │
       ▼
     Retry
       │
       ▼
     SQS DLQ
```

This prevents permanently failing messages from remaining in the main queue indefinitely.

---

# Security

Security is based on AWS managed controls and the **Principle of Least Privilege**.

### S3

* Block Public Access
* Private buckets
* Bucket policies
* Encryption
* Pre-signed URLs

### IAM

Each component should receive only the permissions required for its role.

Example:

```text
Upload Lambda
 └── Generate S3 pre-signed URLs

Queue Processor
 ├── Consume SQS messages
 └── Start Step Functions executions

Image Processing Lambda
 ├── Read source S3 objects
 ├── Write destination S3 objects
 └── Update DynamoDB

Step Functions
 └── Invoke Lambda + Publish SNS
```

---

# Scalability

The architecture separates the upload rate from the processing rate.

```text
High Upload Volume
       │
       ▼
      S3
       │
       ▼
      SQS
       │
       ▼
Lambda Consumers
       │
       ▼
Step Functions
```

SQS acts as a buffer during traffic spikes.

Lambda and other managed AWS services can scale without manually provisioning application servers.

---

# Observability

The system can be monitored using Amazon CloudWatch and service-specific monitoring.

### Lambda

* Invocations
* Errors
* Duration
* Throttles
* Concurrency

### SQS

* Messages sent
* Messages received
* Messages visible
* Message age

### Step Functions

* Successful executions
* Failed executions
* Execution duration
* Execution history

### DynamoDB

* Read/write activity
* Consumed capacity
* Throttling

---

# Testing

## Successful Processing

```text
Request Pre-signed URL
        ↓
Upload Image
        ↓
S3 Source Bucket
        ↓
S3 Event
        ↓
SQS
        ↓
Lambda
        ↓
Step Functions
        ↓
Resize Image
        ↓
Destination S3
        ↓
DynamoDB = COMPLETED
        ↓
SNS Notification
        ↓
CloudFront
```

### Validation Checklist

* [ ] Image appears in source S3 bucket.
* [ ] S3 generates the expected event.
* [ ] SQS receives the message.
* [ ] Lambda receives the message.
* [ ] Step Functions execution starts.
* [ ] Image is resized successfully.
* [ ] Processed image appears in destination S3.
* [ ] DynamoDB status becomes `COMPLETED`.
* [ ] SNS notification is published.
* [ ] Image can be delivered through CloudFront.

---

# Failure Testing

An invalid or unsupported image can be used to test failure handling.

Expected behavior:

```text
S3
 ↓
SQS
 ↓
Lambda
 ↓
Processing Failure
 ↓
Retry
 ↓
Retry
 ↓
Retry
 ↓
SQS DLQ
```

This validates the resilience and failure-isolation mechanisms of the architecture.

---

# AWS Services

| AWS Service            | Purpose                              |
| ---------------------- | ------------------------------------ |
| **Amazon S3**          | Source and destination image storage |
| **Amazon SQS**         | Asynchronous messaging and buffering |
| **Amazon SQS DLQ**     | Failed-message isolation             |
| **AWS Lambda**         | Serverless image processing          |
| **Lambda Layers**      | Image-processing dependencies        |
| **AWS Step Functions** | Workflow orchestration               |
| **Amazon API Gateway** | Upload API / pre-signed URL endpoint |
| **Amazon DynamoDB**    | Image metadata and processing status |
| **Amazon SNS**         | Processing notifications             |
| **Amazon CloudFront**  | Global content delivery              |
| **AWS IAM**            | Access control                       |
| **Amazon CloudWatch**  | Monitoring and logging               |

---

# Architecture Principles Demonstrated

* Event-driven architecture
* Serverless architecture
* Asynchronous processing
* Loose coupling
* Queue-based buffering
* Retry mechanisms
* Dead-Letter Queues
* Workflow orchestration
* Object storage
* CDN and edge caching
* Pre-signed URLs
* Least-privilege IAM
* Lifecycle management
* Failure isolation
* Horizontal scalability

---

# Screenshots

The following screenshots document key parts of the AWS implementation.

## 1. S3 Event Notification

Demonstrates the connection between the source S3 bucket and the SQS processing queue.

![S3 Event Notification](./screenshots/s3-event.png)

---

## 2. SQS and Dead-Letter Queue

Demonstrates the processing queue, retry configuration, and DLQ.

![SQS and DLQ](./screenshots/sqs-dlq.png)

---

## 3. Step Functions Workflow

Demonstrates the deployed image-processing workflow.

![Step Functions Workflow](./screenshots/step-functions.png)

---

## 4. Successful Processing Result

Demonstrates that an image successfully passed through the processing pipeline and was stored in the destination bucket.

![Successful Processing](./screenshots/processing-result.png)

---

# Project Structure

```text
serverless-image-processing/
│
├── README.md
│
├── architecture/
│   └── architecture-diagram.png
│
├── screenshots/
│   ├── s3-event.png
│   ├── sqs-dlq.png
│   ├── step-functions.png
│   └── processing-result.png
│
├── lambda/
│   ├── upload-url-generator/
│   └── queue-processor/
│
├── step-functions/
│   └── workflow.json
│
└── infrastructure/
    └── README.md
```

---

# Cost Considerations

The project is primarily built using AWS serverless and managed services.

For development and testing, the workload should be kept small and AWS Free Tier usage should be monitored.

Cost-control practices include:

* Use small test images.
* Limit test executions.
* Monitor Lambda usage.
* Monitor Step Functions state transitions.
* Monitor S3 storage.
* Apply S3 lifecycle policies.
* Avoid unnecessary CloudFront traffic.
* Delete resources that are no longer required.
* Monitor AWS Billing and Free Tier usage.

> AWS Free Tier limits and eligibility can change over time. Always verify the current AWS pricing and Free Tier terms before deployment.

---

# Future Improvements

Possible improvements include:

* Add an actual watermarking step.
* Split image processing into multiple Lambda functions.
* Add Step Functions retry/catch policies.
* Add failure notifications through SNS.
* Implement Amazon Cognito authentication.
* Add Terraform Infrastructure as Code.
* Add CI/CD using GitHub Actions.
* Add CloudWatch alarms and dashboards.
* Add automated testing.
* Support multiple thumbnail sizes.
* Add image format conversion.
* Add S3 versioning and advanced lifecycle policies.

---

# Key Learning Outcomes

This project demonstrates practical experience with:

### AWS Services

* Amazon S3
* Amazon SQS
* AWS Lambda
* AWS Step Functions
* Amazon DynamoDB
* Amazon SNS
* Amazon API Gateway
* Amazon CloudFront
* AWS IAM
* Amazon CloudWatch

### Cloud Architecture

* Event-driven architecture
* Serverless architecture
* Asynchronous processing
* Decoupled systems
* Fault tolerance
* Retry mechanisms
* Failure isolation
* Workflow orchestration
* Global content delivery
* Least-privilege security

### Engineering Skills

* Python
* Boto3
* JSON / Amazon States Language
* AWS SDK integration
* Image processing with Pillow
* Serverless application design

---

# Conclusion

This project demonstrates a complete **event-driven serverless image-processing pipeline on AWS**.

The architecture separates uploading, messaging, processing, storage, metadata management, notification, and content delivery into independent components.

The main architectural pattern is:

```text
Upload
  ↓
Event
  ↓
Queue
  ↓
Process
  ↓
Store
  ↓
Notify
  ↓
Deliver
```

This design provides:

**Scalability + Resilience + Decoupling + Security + Serverless Operations + Global Content Delivery**

---

## Author

**Hatem Nasser Fathey Alsayed**

Cloud & Infrastructure Engineering Portfolio

---

## Disclaimer

This project is intended for educational and portfolio purposes. AWS service availability, pricing, and Free Tier limits may change over time.

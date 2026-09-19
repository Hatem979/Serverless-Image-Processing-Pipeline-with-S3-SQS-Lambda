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

An AWS Lambda function consumes messages from SQS and starts an **AWS Step Functions** workflow responsible for validating, resizing, watermarking, extracting metadata, and storing the processed image.

Processed images are stored in a separate S3 destination bucket and delivered globally through **Amazon CloudFront**.

Image metadata and processing status are stored in **Amazon DynamoDB**, while **Amazon SNS** is used for processing notifications. Failed messages are isolated using an **SQS Dead-Letter Queue (DLQ)**.

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
 ├── Validate
 ├── Resize
 ├── Watermark
 ├── Extract Metadata
 ├── Store Result
 └── Update Status
        │
        ├──────────────► DynamoDB
        │
        └──────────────► SNS
                             
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
* Reduces unnecessary API Gateway/Lambda traffic.
* Separates authentication/API handling from object transfer.

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

## 5. Step Functions Workflow

AWS Step Functions orchestrates the image-processing workflow.

```text
Start
 │
 ▼
Validate Image
 │
 ▼
Resize Image
 │
 ▼
Watermark Image
 │
 ▼
Extract Metadata
 │
 ▼
Store Processed Image
 │
 ▼
Update DynamoDB
 │
 ▼
Success
```

Step Functions provides:

* Workflow orchestration
* Retry handling
* Error handling
* Conditional logic
* Execution history
* Visual workflow monitoring
* Separation between processing steps

---

## 6. Image Processing

AWS Lambda performs the image-processing operations.

### Resize

Creates a smaller version of the original image.

```text
4000 × 3000
     ↓
800 × 600
```

### Watermark

Adds a predefined watermark to the processed image.

### Metadata Extraction

Extracts information such as:

* Image dimensions
* Format
* File size
* Processing timestamp

---

## 7. Lambda Layers

Image-processing libraries such as **Pillow** or **Sharp** can be packaged using Lambda Layers.

```text
Lambda
│
├── Application Code
│
└── Lambda Layer
      └── Image Processing Dependencies
```

This keeps application code separate from external dependencies and allows reusable dependency packages.

---

## 8. Store Processed Image

Processed images are stored in a separate destination S3 bucket.

```text
Source Bucket
     │
     ▼
Processing Pipeline
     │
     ▼
Destination Bucket
```

Example:

```text
destination-bucket/
└── processed/
    ├── image-001-thumbnail.jpg
    ├── image-001-watermarked.jpg
    └── image-002-thumbnail.jpg
```

Separating source and destination storage makes it easier to apply different access controls, lifecycle policies, and retention strategies.

---

# Failure Handling

## SQS Retry + Dead-Letter Queue

Processing failures are handled using SQS retry behavior and a Dead-Letter Queue.

```text
SQS Main Queue
      │
      ▼
   Lambda
      │
      │ Failure
      ▼
    Retry
      │
      ▼
    Retry
      │
      ▼
    Retry
      │
      │ maxReceiveCount exceeded
      ▼
    SQS DLQ
```

Possible failure scenarios include:

* Corrupted image
* Unsupported image format
* Lambda processing error
* Missing S3 object
* Invalid input
* Unexpected application error

The DLQ prevents permanently failing messages from being retried indefinitely.

---

# Metadata Management

Amazon DynamoDB stores image metadata and processing status.

Example item:

```json
{
  "imageId": "image-001",
  "fileName": "photo.jpg",
  "uploadTime": "2026-09-18T18:00:00Z",
  "width": 4000,
  "height": 3000,
  "status": "COMPLETED",
  "processedKey": "processed/image-001.jpg"
}
```

Possible states:

```text
UPLOADED
PROCESSING
COMPLETED
FAILED
```

This provides a persistent record of the processing lifecycle.

---

# Notifications

Amazon SNS is used to publish processing results.

```text
Processing
    │
    ├── SUCCESS ──► SNS
    │
    └── FAILURE ──► SNS
```

Example:

```text
Image Processing Completed

Image: image-001.jpg
Status: COMPLETED
Output: processed/image-001.jpg
```

---

# Global Content Delivery

Amazon CloudFront is used to deliver processed images globally.

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
* Global distribution
* Reduced repeated requests to the origin

The S3 destination bucket is kept private and access is controlled through the CloudFront architecture.

---

# Security

Security is based on AWS managed controls and the **Principle of Least Privilege**.

### S3

* Block Public Access
* Private buckets
* Bucket policies
* Encryption at rest
* Pre-signed URLs for temporary access

### IAM

Each component receives only the permissions required for its role.

Example:

```text
Upload Lambda
 └── Generate S3 pre-signed upload URLs

Queue Processor
 ├── Consume SQS messages
 └── Start Step Functions executions

Processing Workflow
 ├── Read source S3 objects
 ├── Write destination S3 objects
 └── Update DynamoDB

Notification Component
 └── Publish to SNS
```

---

# S3 Lifecycle Management

Lifecycle policies can automatically manage object storage over time.

Example:

```text
S3 Standard
     │
     │ After X days
     ▼
Infrequent Access
     │
     │ After Y days
     ▼
Expiration
```

Different lifecycle rules can be applied to:

* Original images
* Processed images
* Temporary objects

This helps optimize long-term storage costs.

---

# Scalability

The architecture is designed to handle variable workloads using managed AWS services.

A sudden increase in uploads does not require the processing layer to immediately process every image.

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

The queue absorbs temporary traffic spikes while the processing layer consumes messages asynchronously.

---

# Observability

The system can be monitored using Amazon CloudWatch and the native monitoring capabilities of the AWS services.

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

CloudWatch Logs can be used to investigate application and processing errors.

---

# Testing

## Successful Processing

The expected successful flow is:

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
Image Processing
        ↓
S3 Destination Bucket
        ↓
DynamoDB = COMPLETED
        ↓
SNS Notification
        ↓
CloudFront
```

### Validation

The following should be verified:

* Image appears in the source bucket.
* SQS receives the event.
* Lambda receives the message.
* Step Functions execution succeeds.
* Processed image appears in the destination bucket.
* DynamoDB status becomes `COMPLETED`.
* SNS sends the expected notification.
* Processed image is accessible through CloudFront.

---

## Failure Testing

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

This validates the retry and failure-isolation design.

---

# AWS Services

| AWS Service            | Role                                    |
| ---------------------- | --------------------------------------- |
| **Amazon S3**          | Source and destination object storage   |
| **Amazon SQS**         | Asynchronous message queue              |
| **Amazon SQS DLQ**     | Failed-message isolation                |
| **AWS Lambda**         | Serverless compute and image processing |
| **Lambda Layers**      | Image-processing dependencies           |
| **AWS Step Functions** | Workflow orchestration                  |
| **Amazon API Gateway** | Upload API / pre-signed URL endpoint    |
| **Amazon DynamoDB**    | Image metadata and processing status    |
| **Amazon SNS**         | Success/failure notifications           |
| **Amazon CloudFront**  | Global content delivery                 |
| **AWS IAM**            | Identity and access management          |
| **Amazon CloudWatch**  | Monitoring and logging                  |

---

# Architecture Principles Demonstrated

This project demonstrates practical implementation of:

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

## S3 Event Notification

Demonstrates the connection between the source S3 bucket and the SQS processing queue.

![S3 Event Notification](./screenshots/s3-event.png)

---

## SQS and Dead-Letter Queue

Demonstrates the processing queue, retry configuration, and DLQ.

![SQS and DLQ](./screenshots/sqs-dlq.png)

---

## Step Functions Workflow

Demonstrates the deployed image-processing workflow.

![Step Functions Workflow](./screenshots/step-functions.png)

---

## Successful Processing Result

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
│   └── image-processing-workflow.json
│
└── infrastructure/
    └── README.md
```

---

# Cost Considerations

This project is designed primarily around AWS serverless and managed services.

For development and testing, the workload should be kept small and AWS Free Tier usage should be monitored.

Cost-control practices include:

* Using small test images
* Limiting test executions
* Monitoring Lambda usage
* Monitoring Step Functions state transitions
* Monitoring S3 storage
* Applying S3 lifecycle rules
* Avoiding unnecessary CloudFront traffic
* Deleting resources that are no longer required
* Monitoring AWS Billing and Free Tier usage

> AWS Free Tier limits and eligibility can change. Always verify current pricing and Free Tier terms before deployment.

---

# Key Learning Outcomes

This project provided hands-on experience with designing a complete event-driven serverless architecture.

### Technical

* Amazon S3 event notifications
* SQS queues and DLQs
* Lambda functions and Layers
* Step Functions workflows
* DynamoDB data modeling
* API Gateway integrations
* SNS notifications
* CloudFront distribution
* IAM permissions
* S3 lifecycle policies
* CloudWatch monitoring

### Architecture

The main architectural lesson is understanding **why each service is used and how the services work together**, rather than learning the services independently.

```text
Requirement
     │
     ├── Object Storage ───────► S3
     ├── Async Messaging ──────► SQS
     ├── Serverless Compute ───► Lambda
     ├── Workflow ─────────────► Step Functions
     ├── Metadata ─────────────► DynamoDB
     ├── Notifications ───────► SNS
     └── Global Delivery ──────► CloudFront
```

---

# Future Improvements

Possible future improvements include:

* Infrastructure as Code using Terraform
* CI/CD deployment using GitHub Actions
* Automated image format conversion
* Multiple thumbnail sizes
* Authentication using Amazon Cognito
* API request authorization
* More advanced Step Functions error handling
* CloudWatch dashboards and alarms
* Automated testing
* Object versioning
* Additional S3 lifecycle strategies

---

# Conclusion

This project demonstrates a complete **event-driven serverless image-processing architecture on AWS**.

The system separates image uploading, messaging, processing, storage, metadata management, notification, and content delivery into independent components.

The resulting architecture provides:

**Scalability + Resilience + Decoupling + Security + Serverless Operations + Global Content Delivery**

The main architectural pattern can be summarized as:

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
Deliver
```

---

## Author

**Hatem Nasser Fathey Alsayed**

Cloud & Infrastructure Engineering Portfolio

---

## Disclaimer

This project is intended for educational and portfolio purposes. AWS service availability, pricing, and Free Tier limits may change over time.

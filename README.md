# Project 2: Serverless Image Processing Pipeline with S3, SQS & Lambda
 
![Architecture](architecture_diagram.png)
 
> A fully serverless image processing pipeline built on AWS. Users upload images through a web or mobile application, which are then automatically validated, resized, watermarked, and delivered globally via CloudFront — with zero server management.
 
---
 
## Table of Contents
 
- [Overview](#overview)
- [Architecture Diagram](#architecture-diagram)
- [AWS Services Used](#aws-services-used)
- [How It Works — Step by Step](#how-it-works--step-by-step)
- [Step Functions Workflow](#step-functions-workflow)
- [Key Design Decisions](#key-design-decisions)
- [Prerequisites](#prerequisites)
- [Deployment](#deployment)
- [Learning Outcomes](#learning-outcomes)
---
 
## Overview
 
This project implements a **production-grade serverless image processing pipeline** on AWS. The architecture is fully event-driven: no servers to manage, no idle costs, and automatic scaling based on demand.
 
**What it does:**
- Accepts image uploads from a web or mobile application
- Automatically processes each image (validate → resize → watermark → store)
- Stores metadata (dimensions, status, timestamps) in DynamoDB
- Delivers processed images globally with low latency via CloudFront
- Notifies on job completion or failure via SNS
---
 
## Architecture Diagram
 
![Project 2 Architecture](architecture_diagram.png)
 
The architecture is organized into 4 main layers:
 
| Layer | Components |
|---|---|
| Ingestion | Web/Mobile App, API Gateway, Lambda, S3 Source Bucket |
| Queuing | SQS (Processing Queue + Dead Letter Queue) |
| Processing | AWS Step Functions, 5x Lambda Functions |
| Delivery | S3 Destination Bucket, CloudFront, End Users |
 
---
 
## AWS Services Used
 
### Core Services
 
| Service | Role in the Project |
|---|---|
| **Amazon API Gateway** | Exposes a REST endpoint that the app calls to request a Pre-signed URL |
| **AWS Lambda** | Generates Pre-signed URLs; executes each step of the image processing workflow |
| **Amazon S3 (Source)** | Stores original uploaded images; triggers SQS via event notifications |
| **Amazon S3 (Destination)** | Stores all processed image variants (thumbnail, medium, full-res) |
| **Amazon SQS** | Decouples S3 upload events from processing; buffers messages for Lambda |
| **Amazon SQS (DLQ)** | Dead Letter Queue — captures failed messages after 3 retries |
| **AWS Step Functions** | Orchestrates the multi-step processing workflow (state machine) |
| **Amazon DynamoDB** | Stores image metadata: key, name, dimensions, status, timestamps |
| **Amazon CloudFront** | CDN — serves processed images globally from edge locations |
| **Amazon SNS** | Sends notifications on job completion or failure |
| **Amazon CloudWatch** | Collects metrics, logs, and triggers alarms |
 
---
 
## How It Works — Step by Step
 
### Step 1 — User Opens the App
The user opens the **Web or Mobile Application** and selects an image to upload.
 
### Step 2 — Request a Pre-signed URL
The app sends a request to **API Gateway**, which invokes a **Lambda function**. Lambda generates a **Pre-signed S3 URL** — a temporary, secure permission to upload directly to S3 — and returns it to the app.
 
> **Why a Pre-signed URL?**
> It allows the user to upload directly to S3 without passing through Lambda, avoiding Lambda's 6MB payload limit and reducing cost.
 
### Step 3 — Upload Directly to S3
The app uses the Pre-signed URL to upload the image **directly to the S3 Source Bucket**, bypassing the backend entirely.
 
### Step 4 — S3 Event Notification → SQS
The moment the image lands in S3, an **Event Notification** is automatically sent to the **SQS Processing Queue**.
 
> **Why SQS?**
> SQS decouples the upload event from the processing logic. If 1,000 images arrive simultaneously, SQS absorbs the load and Lambda processes them at its own pace — no dropped messages, no crashes.
 
### Step 5 — Step Functions Polls SQS and Processes the Image
**AWS Step Functions** polls the SQS queue and starts a **state machine** for each image. If a message fails 3 times, it is automatically routed to the **Dead Letter Queue (DLQ)** for manual inspection.
 
### Step 6 — Processed Images Stored
Lambda (step 5.5) writes the processed image variants to the **S3 Destination Bucket** and saves all metadata to **DynamoDB**.
 
### Step 7 — Global Delivery via CloudFront
**CloudFront** sits in front of the S3 Destination Bucket and serves images from the nearest edge location to the end user — with HTTPS enforced and cache behaviors configured per path (`/thumb/*`, `/full/*`).
 
---
 
## Step Functions Workflow
 
The processing pipeline is orchestrated by **AWS Step Functions** using a Standard Workflow. Each step is an independent Lambda function:
 
```
SQS → Step Functions
         │
         ├── 5.1 Validate Image        (check MIME type, size, format)
         │
         ├── 5.2 Create Thumbnail      (resize to 150px, 800px, full-res)
         │
         ├── 5.3 Add Watermark         (stamp watermark overlay using Pillow/Sharp)
         │
         ├── 5.4 Extract Metadata      (get dimensions, format, file size)
         │
         └── 5.5 Store Processed Images
                  ├──→ S3 Destination Bucket  (save all image variants)
                  └──→ DynamoDB               (save metadata record)
 
Step Functions ──→ SNS       (notify: job completed or failed)
Step Functions ──→ CloudWatch (logs, metrics, alarms)
```
 
> **Lambda Layers** are used to package large image processing libraries (Pillow for Python, Sharp for Node.js) that exceed Lambda's default deployment size limit.
 
---
 
## Key Design Decisions
 
### Why SQS between S3 and Lambda?
Without SQS, S3 would invoke Lambda directly. Under high load, Lambda would receive thousands of simultaneous invocations, potentially throttling or failing. SQS acts as a buffer, enabling controlled, resilient processing with built-in retry logic.
 
### Why Step Functions instead of one Lambda?
A single Lambda doing all processing steps (validate + resize + watermark + store) would be difficult to debug and impossible to retry at a specific step. Step Functions breaks the workflow into independent states, each with its own retry and error-handling logic.
 
### Why Pre-signed URLs instead of direct upload through Lambda?
Lambda has a 6MB request body limit. High-resolution images can be 20–50MB. Pre-signed URLs allow direct upload to S3 without any size constraint on the backend.
 
### Why CloudFront in front of S3?
S3 alone serves files from a single region. CloudFront caches and serves images from 400+ edge locations worldwide, dramatically reducing latency for global users and offloading traffic from S3.
 
---

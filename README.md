# Moaz-Fayez-aws-serverless-image-processing-pipeline
Serverless Image Processing Pipeline built with AWS S3, SQS, Lambda, DynamoDB, and CloudFront
# Serverless Image Processing Pipeline on AWS

![Architecture Diagram](Project_Architecture.gif)
Table of Contents

    Solution Overview

    Architecture Diagram

    AWS CDK and Solutions Constructs

    Customizing the Solution

        Prerequisites for Customization

        1. Clone the repository

        2. Unit Test

        3. Build & Deploy

    Sample Lambda Code

    Collection of Operational Metrics

    External Contributors

    License

Solution Overview

The Serverless Image Processing Pipeline on AWS solution helps to embed images on websites and mobile applications to drive user engagement. It uses Pillow / Sharp to provide high-speed image processing without sacrificing image quality. To minimize costs of image optimization, manipulation, and processing, this solution automates version control and provides flexible storage and compute options for file reprocessing.

This solution automatically deploys and configures a serverless architecture optimized for dynamic image manipulation. Images can be rendered, resized, and watermarked spontaneously. It leverages:

    Amazon API Gateway: To generate secure Pre-signed URLs for client uploads.

    Amazon S3: For reliable, durable storage of raw and processed images.

    Amazon SQS & DLQ: For asynchronous queue management and resilient error handling.

    AWS Lambda & Amazon DynamoDB: For serverless processing execution and metadata tracking.

    Amazon CloudFront: For global content delivery with low latency.

Architecture Diagram

This architecture maintains a fully decoupled, event-driven flow divided into three key phases:

    Upload Phase: Users request Pre-signed URLs via API Gateway to upload raw images directly to the Source S3 Bucket, triggering an S3 Event Notification.

    Processing Phase: Messages are buffered in Amazon SQS (with a Dead-Letter Queue - DLQ for failed retries). AWS Lambda polls SQS, processes/resizes the image, saves metadata to Amazon DynamoDB, and uploads the output to S3.

    Storage & Delivery Phase: Output images in the Destination S3 Bucket are distributed globally via Amazon CloudFront.

AWS CDK and Solutions Constructs

AWS Cloud Development Kit (AWS CDK) and AWS Solutions Constructs make it easier to consistently create well-architected infrastructure applications. All AWS Solutions Constructs are reviewed by AWS and use best practices established by the AWS Well-Architected Framework. This solution utilizes:

    aws-cloudfront-s3

    aws-cloudfront-apigateway-lambda

In addition to AWS Solutions Constructs, the solution uses AWS CDK directly to synthesize and create underlying infrastructure resources.
Customizing the Solution
Prerequisites for Customization

    AWS Command Line Interface (AWS CLI) configured with proper IAM permissions.

    Node.js 20.x or later.

    Python 3.11+ (for Lambda execution environment).

1. Clone the repository
Bash

git clone https://github.com/MoazFayez/aws-serverless-image-processing-pipeline.git
cd aws-serverless-image-processing-pipeline
export MAIN_DIRECTORY=$PWD

2. Unit Test

After making changes, run unit tests to ensure all customizations pass test cases:
Bash

cd $MAIN_DIRECTORY/deployment
chmod +x run-unit-tests.sh && ./run-unit-tests.sh

3. Build & Deploy
Bash

cd $MAIN_DIRECTORY/source/constructs
npm run clean:install

# Bootstrap CDK environment
overrideWarningsEnabled=false npx cdk bootstrap --profile <PROFILE_NAME>

# Deploy Serverless Architecture Stack
overrideWarningsEnabled=false npx cdk deploy ImagePipelineStack \
  --parameters SourceBucketParameter=<MY_BUCKET> \
  --parameters AdminEmail=<MY_EMAIL> \
  --profile <PROFILE_NAME>

    Note:

        MY_BUCKET: Name of an existing S3 bucket or comma-separated bucket names.

        PROFILE_NAME: AWS CLI profile with deployment credentials for your preferred region.

        MY_EMAIL: Admin email for notification alerts and access policy configuration.

Sample Lambda Code

Below is the core implementation used by the AWS Lambda function (src/lambda_function.py) to process incoming images:
Python

import json
import os
import urllib.parse
import boto3
from PIL import Image
import io

s3_client = boto3.client('s3')
dynamodb = boto3.resource('dynamodb')

DEST_BUCKET = os.environ.get('DEST_BUCKET_NAME', 'processed-images-bucket')
TABLE_NAME = os.environ.get('METADATA_TABLE_NAME', 'image-metadata')

def lambda_handler(event, context):
    table = dynamodb.Table(TABLE_NAME)
    
    for record in event['Records']:
        body = json.loads(record['body'])
        if 'Records' not in body:
            continue
            
        for s3_record in body['Records']:
            source_bucket = s3_record['s3']['bucket']['name']
            object_key = urllib.parse.unquote_plus(s3_record['s3']['object']['key'])
            
            try:
                # 1. Fetch image from Source S3 Bucket
                response = s3_client.get_object(Bucket=source_bucket, Key=object_key)
                image_content = response['Body'].read()
                
                # 2. Resize image using Pillow
                image = Image.open(io.BytesIO(image_content))
                image.thumbnail((800, 800))
                
                buffer = io.BytesIO()
                image.save(buffer, format=image.format if image.format else 'JPEG')
                buffer.seek(0)
                
                # 3. Store processed image in Destination S3 Bucket
                processed_key = f"processed-{object_key}"
                s3_client.put_object(
                    Bucket=DEST_BUCKET,
                    Key=processed_key,
                    Body=buffer,
                    ContentType=response.get('ContentType', 'image/jpeg')
                )
                
                # 4. Log processing metadata into DynamoDB
                table.put_item(
                    Item={
                        'ImageID': object_key,
                        'OriginalBucket': source_bucket,
                        'ProcessedBucket': DEST_BUCKET,
                        'ProcessedKey': processed_key,
                        'Status': 'PROCESSED'
                    }
                )
                
            except Exception as e:
                print(f"Error processing key {object_key}: {str(e)}")
                raise e

    return {'statusCode': 200, 'body': json.dumps('Processing completed successfully

3. **Storage & Delivery Phase:**
   * **Destination S3 Bucket (`processed-images-bucket`):** Stores final processed images with S3 Lifecycle policies enabled for cost optimization.
   * **Amazon CloudFront Distribution:** Serves processed images globally at edge locations with low latency and edge-caching.

---

## 🛠️ Key AWS Services Used

| AWS Service | Role in Architecture |
| :--- | :--- |
| **Amazon S3** | Object storage for raw uploads and processed image outputs |
| **Amazon API Gateway** | Provides secure endpoints for generating Pre-signed S3 Upload URLs |
| **Amazon SQS & DLQ** | Decouples components and provides fault-tolerant message queueing |
| **AWS Lambda** | Serverless compute execution for image manipulation |
| **Amazon DynamoDB** | Fast NoSQL storage for tracking image processing metadata |
| **Amazon CloudFront** | Global Content Delivery Network (CDN) for low-latency image delivery |

---

## 💻 Sample Lambda Code (`src/lambda_function.py`)

Below is a reference Python implementation for the AWS Lambda processing function:

```python
import json
import os
import urllib.parse
import boto3
from PIL import Image
import io

s3_client = boto3.client('s3')
dynamodb = boto3.resource('dynamodb')

DEST_BUCKET = os.environ.get('DEST_BUCKET_NAME', 'processed-images-bucket')
TABLE_NAME = os.environ.get('METADATA_TABLE_NAME', 'image-metadata')

def lambda_handler(event, context):
    table = dynamodb.Table(TABLE_NAME)
    
    for record in event['Records']:
        # Parse SQS message body containing S3 event
        body = json.loads(record['body'])
        if 'Records' not in body:
            continue
            
        for s3_record in body['Records']:
            source_bucket = s3_record['s3']['bucket']['name']
            object_key = urllib.parse.unquote_plus(s3_record['s3']['object']['key'])
            
            try:
                # 1. Download image from Source S3 Bucket
                response = s3_client.get_object(Bucket=source_bucket, Key=object_key)
                image_content = response['Body'].read()
                
                # 2. Process Image (Resize)
                image = Image.open(io.BytesIO(image_content))
                image.thumbnail((800, 800))
                
                buffer = io.BytesIO()
                image.save(buffer, format=image.format)
                buffer.seek(0)
                
                # 3. Upload Processed Image to Destination S3 Bucket
                processed_key = f"processed-{object_key}"
                s3_client.put_object(
                    Bucket=DEST_BUCKET,
                    Key=processed_key,
                    Body=buffer,
                    ContentType=response['ContentType']
                )
                
                # 4. Write Metadata to DynamoDB
                table.put_item(
                    Item={
                        'ImageID': object_key,
                        'OriginalBucket': source_bucket,
                        'ProcessedBucket': DEST_BUCKET,
                        'ProcessedKey': processed_key,
                        'Status': 'PROCESSED'
                    }
                )
                
            except Exception as e:
                print(f"Error processing key {object_key}: {str(e)}")
                raise e

    return {'statusCode': 200, 'body': json.dumps('Processing complete')}

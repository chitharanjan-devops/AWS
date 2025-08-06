## Project Name

**Image Metadata Extractor with Email Notification**

---

## Tools/Technologies:

* AWS S3
* AWS Lambda
* AWS DynamoDB
* AWS SNS
* Python (in Lambda)
* AWS Console (no code editor needed)

---

# Step-by-Step Instructions

---

## STEP 1: Create S3 Bucket (For Image Uploads)

1. Go to [S3 Console](https://console.aws.amazon.com/s3/)
2. Click **Create bucket**
3. Enter:

   * Bucket name: `image-upload-metadata-demo` (or anything unique)
   * Region: Choose your nearest AWS region
4. Leave other options as default
5. Click **Create bucket**

---

## STEP 2: Create DynamoDB Table (To Store Metadata)

1. Go to [DynamoDB Console](https://console.aws.amazon.com/dynamodb/)
2. Click **Create table**
3. Enter:

   * Table name: `ImageMetadata`
   * Partition key: `ImageID` (String)
4. Click **Create table** (leave everything else default)

---

## STEP 3: Create SNS Topic (For Email Notification)

1. Go to [SNS Console](https://console.aws.amazon.com/sns/)
2. Click **Create topic**
3. Choose:

   * Type: **Standard**
   * Name: `ImageUploadNotification`
4. Click **Create topic**

---

## STEP 4: Subscribe to SNS (Your Email)

1. In your SNS topic, click **"Create subscription"**
2. Choose:

   * **Protocol**: Email
   * **Endpoint**: your email address
3. Click **Create subscription**
4. Check your email inbox and **confirm the subscription**

---

## STEP 5: Create IAM Role for Lambda

1. Go to [IAM Console](https://console.aws.amazon.com/iam/)
2. Click **Roles → Create Role**
3. Choose:

   * **Trusted Entity**: AWS service
   * **Use Case**: Lambda
4. Click **Next → Permissions**, then attach:

   * `AmazonS3ReadOnlyAccess`
   * `AmazonDynamoDBFullAccess`
   * `AmazonSNSFullAccess`
   * `AWSLambdaBasicExecutionRole`
5. Click **Next → Next → Create role**
6. Name it: `lambda-s3-dynamo-sns-role`

---

## STEP 6: Create Lambda Function

1. Go to [Lambda Console](https://console.aws.amazon.com/lambda/)
2. Click **Create function**
3. Enter:

   * Function name: `processImageMetadata`
   * Runtime: **Python 3.11**
   * Execution role: Use existing role → Select `lambda-s3-dynamo-sns-role`
4. Click **Create function**

---

## STEP 7: Add Lambda Code

1. Scroll to **Code Source**
2. Replace existing code with:

```python
import boto3
import os
import json
from PIL import Image
from datetime import datetime
import uuid

s3 = boto3.client('s3')
dynamodb = boto3.resource('dynamodb')
sns = boto3.client('sns')

table = dynamodb.Table('ImageMetadata')
SNS_TOPIC_ARN = 'arn:aws:sns:YOUR_REGION:YOUR_ACCOUNT_ID:ImageUploadNotification' 

def lambda_handler(event, context):
    for record in event['Records']:
        bucket = record['s3']['bucket']['name']
        key = record['s3']['object']['key']
        
        # Download file
        tmp_path = f'/tmp/{key}'
        s3.download_file(bucket, key, tmp_path)
        
        # Extract metadata
        with Image.open(tmp_path) as img:
            width, height = img.size
            format = img.format
            size = os.path.getsize(tmp_path)
        
        image_id = str(uuid.uuid4())
        timestamp = datetime.utcnow().isoformat()

        # Save to DynamoDB
        table.put_item(Item={
            'ImageID': image_id,
            'Filename': key,
            'FileSize': size,
            'Width': width,
            'Height': height,
            'Format': format,
            'UploadTimestamp': timestamp
        })

        # Send email via SNS
        message = f"""
        ✅ New Image Uploaded:
        🖼 Filename: {key}
        📐 Dimensions: {width} x {height}
        🗂 Format: {format}
        📦 Size: {size} bytes
        ⏰ Time: {timestamp}
        """
        sns.publish(
            TopicArn=SNS_TOPIC_ARN,
            Subject="New Image Uploaded",
            Message=message
        )

    return {
        'statusCode': 200,
        'body': json.dumps('Image processed successfully.')
    }
```

3. Replace `YOUR_REGION` and `YOUR_ACCOUNT_ID` with your actual AWS Region and Account ID (SNS gives you the ARN).
4. Click **Deploy**

---

## STEP 8: Add S3 Trigger to Lambda

1. In Lambda → Click **Add trigger**
2. Choose:

   * Source: **S3**
   * Bucket: `image-upload-metadata-demo`
   * Event type: **PUT**
   * Prefix: `images/` (optional)
3. Click **Add**

---

## STEP 9: Upload an Image to S3

You can upload using the AWS Console:

1. Go to S3 → Your bucket → Click **Upload**
2. Upload a `.jpg`, `.png`, etc. file

Or via CLI:

```bash
aws s3 cp myphoto.jpg s3://image-upload-metadata-demo/images/myphoto.jpg
```

---

## STEP 10: Verify Results

1.  **DynamoDB** → Open `ImageMetadata` → Confirm new row
2.  **Email Inbox** → Check for image upload notification
3.  **CloudWatch Logs** (in Lambda) → Debug if needed

---

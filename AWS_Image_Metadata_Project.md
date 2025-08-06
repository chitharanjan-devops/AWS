```
# 📸 AWS Project: Image Metadata Extractor with Email Notification

## 🚀 What You’ll Build
A fully serverless pipeline on AWS that:
- Uploads images to S3
- Triggers a Lambda function
- Extracts metadata (size, resolution, type)
- Stores metadata in DynamoDB
- Sends an email notification via SNS

---

## 🧰 Services Used
- **Amazon S3** — for storing uploaded images
- **AWS Lambda** — for processing images
- **Amazon DynamoDB** — for storing metadata
- **Amazon SNS** — for email notifications

---

## 🛠️ Step-by-Step Setup

### ✅ Step 1: Create S3 Bucket
1. Go to [S3 Console](https://console.aws.amazon.com/s3/)
2. Click **Create bucket**
3. Name: `image-upload-metadata-demo` (use a unique name)
4. Leave default settings and click **Create**

### ✅ Step 2: Create DynamoDB Table
1. Go to [DynamoDB Console](https://console.aws.amazon.com/dynamodb/)
2. Click **Create table**
3. Table name: `ImageMetadata`
4. Partition key: `ImageID` (String)
5. Click **Create table**

### ✅ Step 3: Create SNS Topic & Subscribe Email
1. Go to [SNS Console](https://console.aws.amazon.com/sns/)
2. Create topic → Type: **Standard**, Name: `ImageUploadNotification`
3. Click **Create topic**
4. Open the topic → Click **Create subscription**
   - Protocol: **Email**
   - Endpoint: your email
5. Confirm the email subscription from your inbox

### ✅ Step 4: Create IAM Role for Lambda
1. Go to [IAM Console](https://console.aws.amazon.com/iam/)
2. Click **Roles** → **Create Role**
3. Use Case: **Lambda**
4. Attach Policies:
   - `AmazonS3ReadOnlyAccess`
   - `AmazonDynamoDBFullAccess`
   - `AmazonSNSFullAccess`
   - `AWSLambdaBasicExecutionRole`
5. Name it: `lambda-s3-dynamo-sns-role`

### ✅ Step 5: Create Lambda Function
1. Go to [Lambda Console](https://console.aws.amazon.com/lambda/)
2. Create function → Name: `processImageMetadata`, Runtime: **Python 3.11**
3. Choose existing role: `lambda-s3-dynamo-sns-role`
4. Click **Create function**

### ✅ Step 6: Add Lambda Code

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
SNS_TOPIC_ARN = 'arn:aws:sns:YOUR_REGION:YOUR_ACCOUNT_ID:ImageUploadNotification'  # Replace this

def lambda_handler(event, context):
    for record in event['Records']:
        bucket = record['s3']['bucket']['name']
        key = record['s3']['object']['key']
        
        tmp_path = f'/tmp/{key}'
        s3.download_file(bucket, key, tmp_path)
        
        with Image.open(tmp_path) as img:
            width, height = img.size
            format = img.format
            size = os.path.getsize(tmp_path)
        
        image_id = str(uuid.uuid4())
        timestamp = datetime.utcnow().isoformat()

        table.put_item(Item={
            'ImageID': image_id,
            'Filename': key,
            'FileSize': size,
            'Width': width,
            'Height': height,
            'Format': format,
            'UploadTimestamp': timestamp
        })

        message = f\""" 
        ✅ New Image Uploaded:
        🖼 Filename: {key}
        📐 Dimensions: {width} x {height}
        🗂 Format: {format}
        📦 Size: {size} bytes
        ⏰ Time: {timestamp}
        \"""
        sns.publish(
            TopicArn=SNS_TOPIC_ARN,
            Subject="New Image Uploaded",
            Message=message
        )

    return {'statusCode': 200, 'body': json.dumps('Image processed.')}
```

> ⚠️ Replace `SNS_TOPIC_ARN` with your actual SNS ARN from AWS Console.

### ✅ Step 7: Add Trigger (S3 Upload Event)
1. In Lambda → **Configuration** → **Triggers**
2. Click **Add trigger**
3. Select **S3** → Bucket: `image-upload-metadata-demo`
4. Event type: **PUT**, Prefix: `images/` (optional)
5. Click **Add**

### ✅ Step 8: Upload an Image to S3
Use AWS Console or CLI:

```bash
aws s3 cp photo.jpg s3://image-upload-metadata-demo/images/photo.jpg
```

### ✅ Step 9: Check Output
- Go to **DynamoDB** → Check for new item
- Check your **email inbox** for notification
- Check **CloudWatch logs** if issues

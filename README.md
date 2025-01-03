# S3 to MongoDB Integration with AWS Lambda

This project demonstrates an AWS Lambda function that integrates Amazon S3 and MongoDB to handle file metadata. The Lambda function performs the following tasks:

- **Insert or Update:** On detecting an `ObjectCreated` event in S3, the Lambda function inserts or updates file metadata in MongoDB.
- **Delete:** On detecting an `ObjectRemoved` event in S3, the Lambda function removes the corresponding metadata from MongoDB.


## Architecture Overview

1. **Amazon S3**: Acts as the storage service for file uploads.
2. **AWS Lambda**: Processes S3 events and API Gateway requests.
3. **MongoDB**: Stores metadata for files (e.g., `bucket`, `file_path`, `etag`, `timestamp`).

---

## Prerequisites

### **Environment Setup**

1. **Docker**: Ensure Docker is installed for running MongoDB locally.
2. **Python 3.9**: Install Python for running the Lambda function locally.
3. **AWS CLI**: Install and configure the AWS CLI.

---

## Running MongoDB Locally

Run the following Docker command to start a MongoDB instance locally:

```bash
docker run -d \
  --name mongodb \
  -e MONGO_INITDB_ROOT_USERNAME=admin \
  -e MONGO_INITDB_ROOT_PASSWORD=admin123 \
  -p 27017:27017 \
  mongo:latest
```

**Connection String**:
```
mongodb://admin:admin123@localhost:27017
```

---

## Lambda Function Implementation

### **Python Script**
Below is the main Python script for the Lambda function:

```python
import json
import boto3
import base64
import pymongo

# MongoDB credentials and URI
MONGO_URI = "mongodb://admin:admin123@localhost:27017"
db_name = "file_metadata_db"
collection_name = "files"

# Initialize S3 client
s3_client = boto3.client('s3')

def lambda_handler(event, context):
    try:
        # MongoDB connection
        mongo_client = pymongo.MongoClient(MONGO_URI)
        db = mongo_client[db_name]
        collection = db[collection_name]

        # Process S3 events
        for record in event['Records']:
            s3_event = record['eventName']
            bucket_name = record['s3']['bucket']['name']
            file_path = record['s3']['object']['key']
            etag = record['s3']['object'].get('etag', None)

            if not etag:
                print(f"No ETag found for file {file_path} in bucket {bucket_name}")
                continue

            if 'ObjectCreated' in s3_event:
                print(f"Inserting/Updating metadata for file: {file_path}")
                collection.update_one(
                    {"etag": etag, "bucket": bucket_name, "file_path": file_path},
                    {"$set": {"etag": etag, "bucket": bucket_name, "file_path": file_path, "timestamp": record['eventTime']}},
                    upsert=True
                )

            elif 'ObjectRemoved' in s3_event:
                print(f"Deleting metadata for file: {file_path}")
                collection.delete_one({"etag": etag, "bucket": bucket_name, "file_path": file_path})

        return {
            "statusCode": 200,
            "body": json.dumps("S3 event processed successfully.")
        }

    except Exception as e:
        print(f"Error: {str(e)}")
        return {
            "statusCode": 500,
            "body": json.dumps(f"Error processing event: {str(e)}")
        }
```

---

## Deploying the Lambda Function

### **1. Create an IAM Role for Lambda**
Create an IAM role with the following policies:

- **AmazonS3FullAccess**
- **AWSLambdaBasicExecutionRole**

### **2. Package the Code**

1. Install the required dependencies:
   ```bash
   mkdir python
   pip install pymongo -t python
   zip -r function.zip python
   zip -g function.zip lambda_function.py
   ```

2. Upload the `function.zip` to AWS Lambda.

### **3. Configure S3 Event Notifications**

Set up an event notification on your S3 bucket:
- **Events**: `ObjectCreated` and `ObjectRemoved`
- **Destination**: The Lambda function ARN

---

## Testing the Function

### **Test S3 Events**
1. Upload a file to the S3 bucket:
   ```bash
   aws s3 cp testfile.txt s3://your-bucket-name/testfile.txt
   ```

2. Check MongoDB for metadata:
   ```bash
   docker exec -it mongodb mongo -u admin -p admin123
   use file_metadata_db
   db.files.find().pretty()
   ```

3. Delete the file:
   ```bash
   aws s3 rm s3://your-bucket-name/testfile.txt
   ```

4. Confirm the metadata is removed from MongoDB.

---

## Conclusion

This project provides a robust mechanism to handle file metadata for files stored in S3. By integrating AWS Lambda, MongoDB, and S3, you can automate metadata management efficiently. Contributions and suggestions are welcome!


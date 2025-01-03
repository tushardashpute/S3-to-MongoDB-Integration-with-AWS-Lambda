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
from pymongo import ReturnDocument

# MongoDB credentials and URI
MONGO_URI = "mongodb://admin:admin123@host_ip:27017"  # Update with actual MongoDB URI
DB_NAME = "file_metadata_db"
COLLECTION_NAME = "files"

# Initialize S3 client
s3_client = boto3.client('s3')

# MongoDB connection pool
mongo_client = pymongo.MongoClient(MONGO_URI)
db = mongo_client[DB_NAME]
collection = db[COLLECTION_NAME]

def lambda_handler(event, context):
    """
    AWS Lambda function to handle:
    - File download via API Gateway
    - File upload, modify, and delete triggered by S3 events
    - Idempotency logic with MongoDB to avoid re-processing the same file
    """
    try:
        # Check if the event is from API Gateway (for download)
        if 'queryStringParameters' in event:
            query_params = event['queryStringParameters']
            bucket_name = query_params.get('bucket', None)
            file_path = query_params.get('file', None)  # Includes full path if provided

            if not bucket_name or not file_path:
                return {
                    "statusCode": 400,
                    "body": json.dumps("Missing 'bucket' or 'file' query parameter.")
                }

            # Fetch the file from S3
            s3_response = s3_client.get_object(Bucket=bucket_name, Key=file_path)
            file_content = s3_response['Body'].read()

            # Extract the file name
            file_name = file_path.split('/')[-1]

            # Return the file content as Base64-encoded
            encoded_content = base64.b64encode(file_content).decode('utf-8')
            return {
                "statusCode": 200,
                "headers": {
                    "Content-Type": s3_response['ContentType'],
                    "Content-Disposition": f"attachment; filename={file_name}"
                },
                "body": encoded_content,
                "isBase64Encoded": True
            }

        # Handle S3 events (upload, modify, delete)
        elif 'Records' in event:
            for record in event['Records']:
                # Extract event details from S3 event
                s3_event = record['eventName']
                bucket_name = record['s3']['bucket']['name']
                file_path = record['s3']['object']['key']
                
                # Fetch file metadata from S3
                s3_response = s3_client.head_object(Bucket=bucket_name, Key=file_path)
                event_etag = s3_response['ETag'].strip('"')  # Remove quotes around ETag
                
                # Step 1: Find the file in MongoDB based on bucket name and file path
                file_record = collection.find_one({
                    "bucket": bucket_name,
                    "file_path": file_path
                })

                # Step 2: Handle new or modified files based on DB record
                if file_record is None:
                    # File doesn't exist in DB, so it's a new file
                    print(f"New file detected: {file_path}")

                    # Insert the new file record with its ETag into the database
                    collection.insert_one({
                        "bucket": bucket_name,
                        "file_path": file_path,
                        "etag": event_etag,
                        "timestamp": record['eventTime']
                    })

                else:
                    # File exists in the database, check if content has changed
                    db_etag = file_record.get('etag', None)
                    if db_etag == event_etag:
                        # ETag matches, file content hasn't changed
                        print(f"File {file_path} has not changed (ETag matches).")
                    else:
                        # ETag is different, meaning file content has changed
                        print(f"Modified file detected: {file_path}")

                        # Update the file record with the new ETag and timestamp
                        collection.find_one_and_update(
                            {"bucket": bucket_name, "file_path": file_path},
                            {
                                "$set": {
                                    "etag": event_etag,
                                    "timestamp": record['eventTime']
                                }
                            },
                            return_document=ReturnDocument.AFTER
                        )

                # Handle file deletion (remove metadata from DB)
                if 'ObjectRemoved' in s3_event:
                    print(f"File deleted: {file_path}")
                    collection.delete_one({"bucket": bucket_name, "file_path": file_path})

            return {
                "statusCode": 200,
                "body": json.dumps("S3 event processed successfully.")
            }

        else:
            return {
                "statusCode": 400,
                "body": json.dumps("Unsupported event source.")
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


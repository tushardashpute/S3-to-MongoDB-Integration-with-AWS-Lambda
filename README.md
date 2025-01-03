# S3 to MongoDB Integration with AWS Lambda

This project demonstrates an AWS Lambda function that integrates Amazon S3 and MongoDB to handle file metadata. The Lambda function performs the following tasks:

- **Insert or Update:** On detecting an `ObjectCreated` event in S3, the Lambda function inserts or updates file metadata in MongoDB.
- **Delete:** On detecting an `ObjectRemoved` event in S3, the Lambda function removes the corresponding metadata from MongoDB.
- **Download Support:** Provides an API Gateway integration to download S3 files via a Base64-encoded response.

---

## Architecture Overview

1. **Amazon S3**: Acts as the storage service for file uploads.
2. **AWS Lambda**: Processes S3 events and API Gateway requests.
3. **MongoDB**: Stores metadata for files (e.g., `bucket`, `file_path`, `etag`, `timestamp`).
4. **API Gateway**: Facilitates downloading files from S3 via HTTP requests.

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

## Creating a Lambda Layer

### **Steps to Create the Lambda Layer**

1. Create a directory for the layer:
   ```bash
   mkdir -p lambda-layer/python
   ```

2. Install dependencies (`pymongo` and `requests`) into the directory:
   ```bash
   pip3 install pymongo requests -t lambda-layer/python/
   ```

3. Package the layer into a ZIP file:
   ```bash
   cd lambda-layer
   zip -r pymongo-requests-layer.zip python/
   ```

4. Publish the layer to AWS Lambda:
   ```bash
   aws lambda publish-layer-version \
     --layer-name pymongo-requests-layer \
     --zip-file fileb://pymongo-requests-layer.zip \
     --compatible-runtimes python3.9
   ```

5. Note the ARN of the published layer and attach it to your Lambda function.

---

## Lambda Function Implementation

### **Updated Python Script**
The Lambda function code is available [here](https://raw.githubusercontent.com/tushardashpute/S3-to-MongoDB-Integration-with-AWS-Lambda/refs/heads/main/insert_to_mangodb.py). Make sure to use this updated version.

---

## Deploying the Lambda Function

### **1. Create an IAM Role for Lambda**
Create an IAM role with the following policies:

- **AmazonS3FullAccess**
- **AWSLambdaBasicExecutionRole**

### **2. Package the Code**

1. Download the updated `insert_to_mangodb.py` file:
   ```bash
   wget -O lambda_function.py https://raw.githubusercontent.com/tushardashpute/S3-to-MongoDB-Integration-with-AWS-Lambda/refs/heads/main/insert_to_mangodb.py
   ```

2. Package the Lambda function:
   ```bash
   zip function.zip lambda_function.py
   ```

3. Upload the `function.zip` to AWS Lambda.

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

## API Gateway Integration

### **Download File Endpoint**
1. Use the Lambda function to handle `GET` requests for downloading files.
2. The expected query parameters are:
   - `bucket`: Name of the S3 bucket.
   - `file`: Path to the file in the bucket.

3. Example request:
   ```bash
   curl "https://<api-gateway-id>.execute-api.<region>.amazonaws.com/dev/download?bucket=your-bucket-name&file=testfile.txt"
   ```

---

## Conclusion

This project provides a robust mechanism to handle file metadata for files stored in S3. By integrating AWS Lambda, MongoDB, and S3, you can automate metadata management efficiently. Contributions and suggestions are welcome!


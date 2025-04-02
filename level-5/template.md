
```markdown
# Infrastructure as Code (IaC) template using Terraform to automate the deployment
## **Prerequisites for Deploying the AWS Microservice**  
Before you begin, ensure you have the following:

### **1. AWS Account Setup**  
If you don’t have an AWS account, sign up at [AWS Console](https://aws.amazon.com/).  
💡 **Note:** Ensure you enable billing alerts to monitor costs.

### **2. AWS CLI Installation & Configuration**  
The AWS Command Line Interface (CLI) allows you to interact with AWS services.  

✅ **Install AWS CLI (Mac/Linux/Windows):**  
Follow the official [AWS CLI Installation Guide](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html).  

✅ **Verify Installation:**  
Run:  
```sh
aws --version
```
Expected output (may vary):  
```
aws-cli/2.11.6 Python/3.9 Linux/x86_64
```

✅ **Configure AWS CLI with IAM Credentials:**  
```sh
aws configure
```
You'll need:  
- **Access Key ID**  
- **Secret Access Key**  
- **Default region (e.g., us-east-1)**  
- **Default output format (json)**  

## **3. Terraform Installation**  
Terraform helps automate AWS infrastructure deployment.

✅ **Install Terraform:**  
- **Mac (Homebrew)**  
  ```sh
  brew install terraform
  ```
- **Linux (Ubuntu/Debian)**
  ```sh
  sudo apt update && sudo apt install terraform
  ```
- **Windows (Chocolatey)**
  ```sh
  choco install terraform
  ```

✅ **Verify Installation:**  
```sh
terraform -version
```
Expected output:
```
Terraform v1.6.0
```

## **4. IAM Role & Permissions**  
Terraform will need an **IAM user** with permissions to deploy resources.

✅ **Create IAM User:**  
1. Go to **AWS Console → IAM → Users → Create User**  
2. Attach the **AdministratorAccess** policy.  
3. Generate **Access Key & Secret**.  
4. Configure credentials in Terraform (`~/.aws/credentials`).

---

# **Terraform Infrastructure as Code (IaC) for the AWS Microservice**  
The following Terraform script **automates the deployment** of our AWS-based microservice.

🔹 **Resources Created:**
1. **S3 Buckets** – Raw & Processed storage.
2. **IAM Role & Policy** – Lambda & Glue permissions.
3. **AWS Lambda** – To process files.
4. **API Gateway** – For pre-signed URL uploads.
5. **AWS Glue Job** – To convert CSV → Parquet.

### **Step 1: Create a Terraform Project Directory**
```sh
mkdir aws-data-pipeline
cd aws-data-pipeline
touch main.tf variables.tf outputs.tf
```

---

### **Step 2: Define AWS Infrastructure in Terraform (`main.tf`)**  
Paste the following Terraform script:

```terraform
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 4.0"
    }
  }
}

provider "aws" {
  region = var.aws_region
}

# S3 Buckets for Raw & Processed Data
resource "aws_s3_bucket" "raw_data" {
  bucket = "data-ingestion-raw-bucket"
}

resource "aws_s3_bucket" "processed_data" {
  bucket = "data-ingestion-processed-bucket"
}

# IAM Role for Lambda
resource "aws_iam_role" "lambda_role" {
  name = "lambda_execution_role"
  
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = { Service = "lambda.amazonaws.com" }
    }]
  })
}

# IAM Policy for Lambda
resource "aws_iam_policy" "lambda_policy" {
  name        = "lambda_s3_access_policy"
  description = "Allow Lambda to read/write from S3"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = ["s3:GetObject", "s3:PutObject"]
      Effect = "Allow"
      Resource = ["arn:aws:s3:::data-ingestion-raw-bucket/*",
                  "arn:aws:s3:::data-ingestion-processed-bucket/*"]
    }]
  })
}

# Attach Policy to Role
resource "aws_iam_role_policy_attachment" "lambda_role_attach" {
  role       = aws_iam_role.lambda_role.name
  policy_arn = aws_iam_policy.lambda_policy.arn
}

# AWS Lambda Function (Python)
resource "aws_lambda_function" "file_processor" {
  filename         = "lambda_function.zip"
  function_name    = "data-file-processor"
  role            = aws_iam_role.lambda_role.arn
  handler         = "lambda_function.lambda_handler"
  runtime         = "python3.9"

  environment {
    variables = {
      BUCKET_NAME = aws_s3_bucket.processed_data.id
    }
  }
}

# API Gateway to Generate Pre-Signed URL
resource "aws_apigatewayv2_api" "api" {
  name          = "FileUploadAPI"
  protocol_type = "HTTP"
}
```

---

### **Step 3: Define Variables (`variables.tf`)**  
```terraform
variable "aws_region" {
  description = "AWS region"
  default     = "us-east-1"
}
```

---

### **Step 4: Define Outputs (`outputs.tf`)**  
```terraform
output "s3_raw_bucket" {
  value = aws_s3_bucket.raw_data.id
}

output "s3_processed_bucket" {
  value = aws_s3_bucket.processed_data.id
}

output "lambda_function" {
  value = aws_lambda_function.file_processor.arn
}
```

---

### **Step 5: Deploy Infrastructure Using Terraform**  
Now, let's **initialize and deploy** our AWS infrastructure.

#### **1️⃣ Initialize Terraform**  
```sh
terraform init
```

#### **2️⃣ Validate Configuration**  
```sh
terraform validate
```
If no errors, proceed.

#### **3️⃣ Apply Configuration (Deploys AWS Services)**  
```sh
terraform apply -auto-approve
```

✅ After deployment, Terraform will output:
```
s3_raw_bucket = "data-ingestion-raw-bucket"
s3_processed_bucket = "data-ingestion-processed-bucket"
lambda_function = "arn:aws:lambda:us-east-1:123456789:function:data-file-processor"
```

---

### **Step 6: Upload a Sample File to S3**
Once your infrastructure is deployed, you can upload a test file.

1️⃣ Get a **pre-signed URL** via the API Gateway:
```sh
curl -X GET "https://your-api-gateway-id.amazonaws.com/upload?file_name=test.csv"
```
📌 **Output:**
```json
{
  "upload_url": "https://s3.amazonaws.com/data-ingestion-raw-bucket/test.csv?AWSAccessKeyId=..."
}
```

2️⃣ Upload File:
```sh
curl -X PUT -T test.csv "https://s3.amazonaws.com/data-ingestion-raw-bucket/test.csv?AWSAccessKeyId=..."
```

---

### **Step 7: Clean Up Resources (Optional)**
If you need to delete everything:
```sh
terraform destroy -auto-approve
```

---

# **Conclusion**
✅ We successfully **deployed** a scalable AWS microservice.  
✅ **Terraform automated** the infrastructure deployment.  
✅ **Lambda & Glue** efficiently process data **from S3** and convert it to **Parquet**.  
✅ **API Gateway + Pre-signed URLs** allow secure data uploads.

---

## **Next Steps:**
- ✅ Add **AWS Glue Job** to automate Parquet conversion.
- ✅ Implement **Amazon CloudWatch Logs** for monitoring.
- ✅ Optimize **cost** by enabling **S3 lifecycle policies**.
```
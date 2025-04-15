# EKS-Deployment-with-Terraform-and-Jenkins-CICD


---

```markdown
# EKS Deployment with Terraform and Jenkins CI/CD

This project demonstrates how to provision an Amazon EKS (Elastic Kubernetes Service) cluster using Terraform, configure a secure remote backend using S3 and DynamoDB, deploy a sample application, and automate the entire infrastructure lifecycle using Jenkins CI/CD.

## Table of Contents

- [Project Overview](#project-overview)
- [Tools Used](#tools-used)
- [Infrastructure Setup](#infrastructure-setup)
- [Remote Backend Configuration](#remote-backend-configuration)
- [CI/CD Automation with Jenkins](#cicd-automation-with-jenkins)
- [How to Run This Project](#how-to-run-this-project)
- [Sample Scripts](#sample-scripts)
- [License](#license)

## Project Overview

This is a complete DevOps project that includes:

- Infrastructure provisioning with Terraform
- EKS cluster deployment on AWS
- Remote backend using S3 and DynamoDB
- Sample application deployment using `kubectl`
- CI/CD automation using Jenkins and Git

## Tools Used

- Terraform
- AWS CLI
- kubectl
- Amazon EKS
- S3 & DynamoDB
- Jenkins
- Git & GitHub

## Infrastructure Setup

1. Clone this repository and navigate into the project directory.
2. Define your AWS provider and resources in `main.tf`, `variables.tf`, and `outputs.tf`.
3. Initialize and apply your infrastructure:

   ```bash
   terraform fmt
   terraform init
   terraform validate
   terraform plan
   terraform apply
   ```

4. Authenticate with the EKS cluster:

   ```bash
   aws eks update-kubeconfig --region us-east-1 --name eks_cluster
   ```

5. Verify cluster setup:

   ```bash
   kubectl get nodes
   ```

## Remote Backend Configuration

Create and configure an S3 bucket and DynamoDB table to store and lock your Terraform state.

### Sample Backend Block

```hcl
terraform {
  backend "s3" {
    bucket         = "lili-terraform-state"
    key            = "global/s3/terraform.tfstate"
    region         = "us-east-1"
    dynamo_table   = "terraform-state"
    encrypt        = true
  }
}
```

Run:

```bash
terraform init
```

## CI/CD Automation with Jenkins

1. Initialize Git and push code to a remote repository:

   ```bash
   git init
   echo ".terraform/" >> .gitignore
   git add .
   git commit -m "Initial commit"
   git remote add origin <your-repo-url>
   git push -u origin main
   ```

2. Add AWS credentials in Jenkins via **Manage Jenkins > Credentials**.

3. Add a `Jenkinsfile` with pipeline stages such as:

   - Terraform Init & Apply
   - Kubernetes deployment

4. Create a Jenkins pipeline job, link it to the repo, and run the build.

5. After the job completes, get the LoadBalancer DNS from AWS:

   ```bash
   kubectl get svc
   ```

   Visit the external IP in your browser to access the app.

## Sample Scripts

### Create S3 and DynamoDB via Terraform

```hcl
provider "aws" {
  region = "us-east-1"
}

resource "aws_s3_bucket" "terraform_state" {
  bucket = "lili-terraform-state"

  lifecycle {
    prevent_destroy = true
  }
}

resource "aws_s3_bucket_versioning" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id

  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}

resource "aws_s3_bucket_public_access_block" "terraform_state" {
  bucket                  = aws_s3_bucket.terraform_state.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

resource "aws_dynamodb_table" "terraform_state" {
  name         = "terraform-state"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }
}
```

## License

This project is licensed under the MIT License. See the [LICENSE](LICENSE) file for details.
```

---

Let me know if you also want:

- A sample `Jenkinsfile`
- A sample Kubernetes `app-deployment.yaml`
- A folder structure guide

Happy to provide those next.

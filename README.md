

---

```markdown
# EKS Terraform Project with Remote Backend and Jenkins Automation

This project demonstrates how to provision an Amazon EKS cluster using Terraform, configure a remote backend with S3 and DynamoDB, and automate deployment through GitHub and Jenkins.

---

## Step 1: Build and Provision the EKS Infrastructure

```bash
export AWS_REGION=us-east-1
cd ~/eks-terraform-demo

terraform fmt
terraform init
terraform validate
terraform plan
terraform apply --auto-approve
```

---

## Step 2: Authenticate to Your EKS Cluster

```bash
aws eks update-kubeconfig --region us-east-1 --name eks_cluster
kubectl get nodes
```

---

## Step 3: Deploy and Verify Sample Application

```bash
kubectl apply -f deployment.yml
kubectl get pods
kubectl get svc
```

Access the application by visiting the LoadBalancer's EXTERNAL-IP in your browser.

---

## Step 4: Configure Remote Backend for Terraform

### Check Local State

```bash
ls
cat terraform.tfstate
terraform state list
```

### Define Backend Resources in `backend.tf`

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

output "s3_bucket_arn" {
  value = aws_s3_bucket.terraform_state.arn
}

output "dynamodb_table_name" {
  value = aws_dynamodb_table.terraform_state.name
}
```

### Add Backend Block to `provider.tf`

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

### Initialize Backend

```bash
terraform init
```

### Confirm Remote State

```bash
cat terraform.tfstate
```

Check AWS Console: `S3 > lili-terraform-state > global/s3/terraform.tfstate`

### Test Locking Behavior

```bash
terraform plan            # Run and press Ctrl+C mid-process
terraform plan            # Observe lock in effect
terraform plan -lock=false
terraform force-unlock <lock_id>
```

### Optional: Revert to Local State

```bash
# Comment backend block in provider.tf
terraform init -migrate-state
```

---

## Step 5: Automate with Git and Jenkins

### Git Setup

```bash
git init
git remote add origin https://github.com/<your-username>/<repo>.git

echo ".terraform/" >> .gitignore
echo "*.tfstate" >> .gitignore

git add .
git commit -m "Initial commit with Terraform for EKS"
git push -u origin main
```

### Jenkins Setup

1. Add AWS credentials in Jenkins:
   - Manage Jenkins > Credentials > Global > Add AWS Access and Secret Key.

2. Example `Jenkinsfile`:

```groovy
pipeline {
  agent any

  environment {
    AWS_ACCESS_KEY_ID     = credentials('aws-access-key-id')
    AWS_SECRET_ACCESS_KEY = credentials('aws-secret-access-key')
    AWS_DEFAULT_REGION    = 'us-east-1'
  }

  parameters {
    choice(name: 'ACTION', choices: ['apply', 'destroy'], description: 'Terraform action')
  }

  stages {
    stage('Checkout Code') {
      steps {
        git url: 'https://github.com/<your-username>/<repo>.git', branch: 'main'
      }
    }

    stage('Terraform Init') {
      steps {
        sh 'terraform init'
      }
    }

    stage('Terraform Plan') {
      steps {
        sh 'terraform plan'
      }
    }

    stage('Terraform Apply/Destroy') {
      steps {
        sh 'terraform ${ACTION} -auto-approve'
      }
    }

    stage('Deploy Application') {
      when {
        expression { params.ACTION == 'apply' }
      }
      steps {
        sh 'aws eks update-kubeconfig --region us-east-1 --name eks_cluster'
        sh 'kubectl apply -f deployment.yml'
        sh 'kubectl get pods'
      }
    }
  }
}
```

3. Create Jenkins pipeline job
   - Use "Pipeline script from SCM"
   - Use GitHub repo URL
   - Add build parameter: `ACTION` with choices `apply` and `destroy`
   - Trigger and run pipeline

---

## Step 6: Access the Application

```bash
kubectl get svc
```

Use the `EXTERNAL-IP` of the LoadBalancer service to access the app in your browser.

---

## License


```

---


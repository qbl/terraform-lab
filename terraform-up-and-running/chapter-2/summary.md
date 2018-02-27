# Getting Started with Terraform

## Setting Up AWS Account

1. Create New User
2. Grant Required Policies:  
   AmazonEC2FullAccess  
   AmazonS3FullAccess  
   CloudWatchFullAccess  
   IAMFullAccess  
   AmazonDynamoDBFullAccess  
   AmazonRDSFullAccess  

## Install Terraform

1. Download Terraform [source code](https://www.terraform.io/downloads.html)
2. In MacOS X, unzip the downloaded source code and move it to `/usr/local/bin/`
3. To verify, run `terraform` from your console
4. Export needed credentials from AWS in your console  
   `export AWS_ACCESS_KEY_ID=<your access key ID>`  
   `export AWS_SECRET_ACCESS_KEY=<your access key ID>`

## Deploy a Single Server

1. Create a file named "main.tf", fill it with the code below:

```
provider "aws" {
  region = "us-east-1"
}
resource "aws_instance" "example" {
  ami = "ami-40d28157"
  instance_type = "t2.micro"
}
```

2. Run `terraform init` to get all the required plugins
3. Run `terraform plan` to review operations that we will apply
4. Run `terraform apply` to actually apply our configuration
5. We can add instance name to our EC2 instance by editing "main.tf" as the code below:

```
provider "aws" {
  region = "us-east-1"
}
resource "aws_instance" "example" {
  ami = "ami-40d28157"
  instance_type = "t2.micro"

  tags {
    Name = "terraform-example"
  }
}

```

6. Then run `terraform plan` and `terraform apply` again

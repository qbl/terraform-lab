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

1. Create a file named "main.tf", fill it with the code below to indicate provider (AWS) and region:

```
provider "aws" {
  region = "us-east-1"
}
```

2. Add resource configuration to "main.tf" with the code below:

```
resource "aws_instance" "example" {
  ami = "ami-40d28157"
  instance_type = "t2.micro"
}
```

3. Run `terraform init` to get all the required plugins.
4. Run `terraform plan` to review operations that we will apply.
5. Run `terraform apply` to actually apply our configuration.
6. We can add instance name to our EC2 instance by editing "main.tf" as the code below:

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

## Deploy a Single Web Server

1. Edit our "main.tf" file to look like the code below:

```
resource "aws_instance" "example" {
  ami           = "ami-40d28157"
  instance_type = "t2.micro"

  user_data = <<-EOF
              #!/bin/bash
              echo "Hello, World" > index.html
              nohup busybox httpd -f -p 8080 &
              EOF

  tags {
    Name = "terraform-example"
  }
}
```

2. Add security group in AWS to enable EC2 instance to receive traffic on port 8080:

```
resource "aws_security_group" "instance" {
  name = "terraform-example-instance"

  ingress {
    from_port = 8080
    to_port = 8080
    protocol = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

3. Edit our configuration for EC2 to include vpc_security_group setting.

   Below we will see `${}` notation. It's Terraform's way to do string interpolation. The syntax is `${resource_type.name.attribute}`.

```
resource "aws_instance" "example" {
  ami = "ami-40d28157"
  instance_type = "t2.micro"
  vpc_security_group_ids = ["${aws_security_group.instance.id}"]

  user_data = <<-EOF
              #!/bin/bash
              echo "Hello, World" > index.html
              nohup busybox httpd -f -p 8080 &
              EOF

  tags {
    Name = "terraform-example"
  }
}
```

4. Run `terraform graph` to view the dependency graph in DOT format.
     
   Dependency graph determines the order in which operations are performed.

5. Run `terraform plan`.
6. Run `terraform apply`.
7. To verify if our installation is correct, execute:  
   `curl http://<EC2 Instance Public IP>:8080`

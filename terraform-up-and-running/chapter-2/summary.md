# Getting Started with Terraform

## 0. Setting Up AWS Account

To run code examples in this book, we need to setup an AWS account:

1. Create New User
2. Grant Required Policies:  
   AmazonEC2FullAccess  
   AmazonS3FullAccess  
   CloudWatchFullAccess  
   IAMFullAccess  
   AmazonDynamoDBFullAccess  
   AmazonRDSFullAccess  

## 1. Install Terraform

To install Terraform, follow this steps:

1. Download Terraform [source code](https://www.terraform.io/downloads.html)
2. In MacOS X, unzip the downloaded source code and move it to `/usr/local/bin/`
3. To verify, run `terraform` from your console
4. Export needed credentials from AWS in your console  
   `export AWS_ACCESS_KEY_ID=<your access key ID>`  
   `export AWS_SECRET_ACCESS_KEY=<your access key ID>`

## 2. Deploy a Single Server

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

## 3. Deploy a Single Web Server

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

## 4. Deploy a Configurable Web Server

In our previous example, you should notice that the value of our port (8080) appears multiple time. This violates the DRY principle: every piece of knowledge must have a single, unambiguous, authoritative representation within a system. Right now, if we were to change the port, we will have to change every single appearance of the value "8080" in our Terraform configuration file. Fortunately, Terraform allows us to define *input variables* with the following format:

```
variable "<name>" {
  [<config> ...]
}
```

The body of a variable can contain three parameters, all of them optional:
- description: description of the variable
- default: default value of the variable 
- type: either string, list, or map

Some examples of using Terraform variables:

```
variable "list_example" {
  description = "Variable with list type"
  type = "list"
  default = [1, 2, 3]
}
```

```
variable "list_example" {
  description = "Variable with map type"
  type = "map"
  default = {
    key1 = "value1"
    key2 = "value2"
    key3 = "value3"
  }
}
```

Now, if we add variable to add port in our "main.tf" without specifying its default value like this:

```
variable "server_port" {
  description = "The port we use for HTTP requests is:"
}
```

Terraform will prompt us to enter the value for port when execute `terraform plan`. Otherwise, we can also pass it as an argument like `terraform plan -var server_port="8080"`. We can use variables using string extrapolation syntax like `${var.variable_name}`.

Now, let's apply that to our "main.tf" configuration:

```
provider "aws" {
  region = "us-east-1"
}
resource "aws_instance" "example" {
  ami = "ami-40d28157"
  instance_type = "t2.micro"
  vpc_security_group_ids = ["${aws_security_group.instance.id}"]

  user_data = <<-EOF
              #!/bin/bash
              echo "Hello, World" > index.html
              nohup busybox httpd -f -p ${var.server_port} &
              EOF

  tags {
    Name = "terraform-example"
  }
}
resource "aws_security_group" "instance" {
  name = "terraform-example-instance"

  ingress {
    from_port = ${var.server_port}
    to_port = ${var.server_port}
    protocol = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
variable "server_port" {
  description = "The port we use for HTTP requests is: "
  default = 8080
}
```

In addition to accepting input variables, Terraform can also generating output variables. To do that, we need to specify a block in this format:

```
output "<name>" {
  value = <value>
}
```

In our case, we are going to need the public IP of our server instance. Rather than manually look it up in AWS web interface, we can tell Terraform to output the public IP. To do that, we need to add this block to our "main.tf" file:

```
output "public_ip" {
  value = "${aws_instance.example.public_ip}"
}
```

Now, when we run `terraform apply`, Terraform will not change anything (since we did not specify any changes) but it will output the public IP of our server instance.

## 5. Deploying a Cluster of Web Servers

In the real world, we rarely deploy our app to a single web server. We usually need to run a cluster of servers. To do this, in AWS we can use Auto Scaling Group (ASG). An ASG takes care a lot of tasks such as launching a cluster of EC2 instances, monitoring the health of of each instance, replacing failed instances, and adjusting the size of the cluster in response to load.

To create an ASG, first we need to change our configuration. Instead of using `aws_instance` resource, we will use `aws_launch_configuration`.

```
resource "aws_launch_configuration" "example" {
  ami = "ami-40d28157"
  instance_type = "t2.micro"
  security_groups = ["${aws_security_group.instance.id}"]

  user_data = <<-EOF
              #!/bin/bash
              echo "Hello, World" > index.html
              nohup busybox httpd -f -p ${var.server_port} &
              EOF

  lifecycle {
    create_before_destroy = true
  }
}
```

New thing from the code above is `lifecycle` parameter. What we did was telling Terraform to create a new instance of EC2 before deleting the old one. We will add this parameter to our security group configuration too.

```
resource "aws_security_group" "instance" {
  name = "terraform-example-instance"

  ingress {
    from_port = "${var.server_port}"
    to_port = "${var.server_port}"
    protocol = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  lifecycle {
    create_before_destroy = true
  }
}
```

Now we can create our ASG config.

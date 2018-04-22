# Why Terraform

## 0. Intro

In this chapter, before we learn anything about how to use Terraform, we try to answer the fundamental question first: "why Terraform?". To answer this, we need to understand a little bit about *software delivery*. Software is not completed when the code is working on programmer's code. Nor it is completed when it is committed and pushed to git repository. Nor it is completed when it is reviewed and marked as "ready to ship". It is only completed once it is delivered to end users.

Therefore, when we try to answer "why Terraform?", we need to understand the big picture of Software Delivery first. This chapter discusses about Software Delivery, covering:

1. The rise of DevOps
2. What is infrastructure as a code?
3. Benefits of intrastructure as a code
4. How Terraform works
5. How Terraform compares to other infrastructure as a code tools.

## 1. The Rise of DevOps

Before the age of cloud computing like Amazon Web Services, Microsoft Azure, and Google Cloud Platform, companies usually have a dedicated team that manage their infrastructure. This team is usually called Operations (or Ops) team. Back then, Ops team handle a lot of hardware related issues, from setting up to troubleshooting.

Nowadays, with the widespread adoption of cloud infrastructures, Ops team no longer have to deal with hardware. Instead, it has to work on tools such as Chef, Puppet, Terraform, and Docker to manage these cloud infrastructures. Therefore, Ops team spends more and more time writing software instead. This increasingly blurring restrictions of development (writing code) and operations gives birth to what now is called as DevOps.

There are a lot of different definitions of DevOps. Rather than defining what is DevOps, this book takes a simple approach by defining that "The goal of DevOps is to make software delivery vastly more efficient". While there are four core values of the DevOps movement (culture, automation, monitoring, and sharing), this book focuses on automation part of DevOps. By using Terraform, a DevOps team's goal is to automate software delivery process through code rather than manually executing shell commands.

## 2. What is Infrastructure as  Code?

The idea behind infrastructure as code (IAC) is to write and execute code to define, deploy, and update infrastructure. This is one of key insight of the DepOps movement: we can manage *almost* everything in code.

There are four categories of IAC tools:

1. Ad Hoc Scripts
2. Configuration Management Tools
3. Server Templating Tools
4. Server Provisioning Tools

Let's dig a little deeper on each item in the next sub chapters.

### 2.1. Ad Hoc Scripts

Writing ad hoc scripts is the most straightforward way to automate things. We just have to break down the tasks we want to accomplish into steps and then define each step in code using any scripting language. This is an example of bash script to get a PHP app up and running with an Apache server:

```
# Update the apt-get cache
sudo apt-get update

# Install PHP
sudo apt-get install -y php

# Install Apache
sudo apt-get install -y apache2

# Copy the code from the repository
sudo git clone https://github.com/brikis98/php-app.git /var/www/html/app

# Start Apache
sudo service apache2 start
```

While it is easy and quick to write, writing ad hoc scripts does not scale very well. It will be hard to use ad hoc script to manage dozens of servers, databases, load balancers, network configurations, and so on.

### 2.2. Configuration Management Tools

Configuration Management Tools are designed to install and manage software on existing servers. Chef, Puppet, Ansible, and Salt Stack fall under this category. Below is an example of an Ansible Role file named web-server.yml. It does the same thing as bash script in the previous example.

```
- name: Update the apt-get cache
  apt:
    update_cache: yes

- name: Install PHP
  apt:
    name: php

- name: Install Apache
  apt:
    name: apache2

- name: Copy the code from the repository
  git: repo=https://github.com/brikis98/php-app.git dest=/var/www/html/app

- name: Start Apache
  service: name=apache2 state=started enabled=yes
```

At a glance, it looks a lot like the bash script. However, a tool like Ansible offers several advantages over ad hoc scripts:

1. Coding Conventions  
   Ansible enforces a consistent and predictable structure so it is easier for any DevOps engineer to navigate the code.
2. Idempotence  
   Code that works correctly no matter how many times it is run is called an idempotent code. In our ad hoc script example above, it is hard to make our bash script idempotent. On a second run, the command "sudo apt-get install -y php" may raise error as PHP is already installed. However, with Ansible Role example above, Apache will be installed only if it is not installed already and will be started only if it is not running already.
3. Distribution  
   Ad hoc scripts are designed to run on a single machine. Configuration Management Tools are designed for managing large numbers of remote servers.

### 2.3. Server Templating Tools

Server Templating Tools have recently become an alternative as well as complement to Configuration Management Tools. The idea behind Server Templating Tools is to create an image of a server that captures a fully self-contained "snapshot" of the operating system, the software, the files, and all other relevant details. We can combine Server Templating Tools (such as Docker) with Configuration Management Tools (such as Ansible) to install an image on multiple servers.

There are two categories of tools to work with images:

1. Virtual Machines
   A virtual machine (VM) emulates an entire system, including the hardware. With VM, we run a hypervisor (such as VMWare or VirtualBox) to virtualize the underlying hardware in a machine. The benefits of VM are: VM Image can only see virtualized hardware so it is isolated from the host machine and other VM images, an image will run exactly the same in all environment. The drawback is that running VMs can incur a lot of resources overhead in the host machine. Tools such as Packer and Vagrant are used to create VM images as code.
2. Containers
   A container emulates the user space of an operating system. To run a container image, we need a container engine such as Docker or CoreOS rkt. The benefits of containers are: it is isolated from host machine and other containers, it runs exactly the same in all environments, and it incurs less resources overhead than VMs. The drawback is that containers' isolation with one another are not as secure since they share kernel and hardware. Tools such as Docker and CoreOS rkt are used to create container images as code.  

   Server templating is a key component for *immutable structure*. The idea behind immutable structure is that once deployed, a server should never be changed again. When we do need to change something, we should create a new image.  

   Below is an example of a Packer template called web-server.json. It does the same thing as bash script and Ansible script above. The differences are: this script creates an Amazon Machine Image (AMI) before installing PHP and Apache, and this script does not run the Apache server after it is installed.

```
{
  "builders": [{
    "ami_name": "packer-example",
    "instance_type": "t2.micro",
    "region": "us-east-1",
    "type": "amazon-ebs",
    "source_ami": "ami-40d28157",
    "ssh_username": "ubuntu"
  }],
  "provisioners": [{
    "type": "shell",
    "inline": [
      "sudo apt-get update",
      "sudo apt-get install -y php",
      "sudo apt-get install -y apache2",
      "sudo git clone https://github.com/brikis98/php-app.git /var/www/html/app"
    ]
  }]
}
```

### 2.4. Server Provisioning Tools

While Configuration Management Tools and Server Templating Tools define the codes that run on each server, Server Provisioning Tools are responsible for creating the server itself. In addition, we can use Server Provisioning Tools not only to create servers but also to create almost every aspect of infrastructure such as databases, caches, or load balancers. Terraform, CloudFormation, and OpenStack Heat fall in this category of tools.

## 3. Benefits of Infrastructure as Code

When infrastructure is defined as code, we can use software engineering practices to improve our software delivery process. These prcatices include:

1. Self-service
   If infrastructure is defined in code, the entire deployment process can be automated, and developers can kick off their own deployments whenever necessary.
2. Speed and Safety
   Since deployment process is automated, computer can execute it significantly faster than a person. It is also safer since the deployment process is more consistent and repeatable.
3. Documentation
   Infrastructure as code also acts as a documentation, allowing everyone in the team to understand how things work in their infrastructure.
4. Version Control
   Just as with any code, we can use version control to track changes made to our infrastructure.
5. Validation
   Just as with any code, we can use various methods to reduce the chance of deffects in our code such as code review or automated tests.
6. Reuse
   We can package our infrastructure as code into reusable modules.
7. Happiness
   Infrastructure as code allows computers to do what they do best (automation) and developers to do what they do best (coding).

## 4. How Terraform Works

Terraform is an open source tool written in Go. Terraform source code is compiled into a binary called 'terraform'. This binary can be used to deploy infrastructure from our local machine (or any machine) to where our servers need to be. Under the hood, terraform binary makes API calls to one or more providers (such as AWS, Azure, or Google Cloud). To tell terraform what API calls to make, we will need to specify what infrastructures we wish to create in a 'Terraform configurations' file. These configurations are the "code" in "infrastructure as code". Below is an example of Terraform configuration:

```
resource "aws_instance" "example" {
  ami           = "ami-40d28157"
  instance_type = "t2.micro"
}

resource "dnsimple_record" "example" {
  domain = "example.com"
  name   = "test"
  value  = "${aws_instance.example.public_ip}"
  type   = "A"
}
```

We can define our entire infrastructure in Terraform configuration files and commit those files to version control. We can run certain Terraform commands to actually deploy the infrastructure according to our configuration. 

## 5. How Terraform Compares to Other Infrastructure as a Code Tools

Knowing the plethora of options available to create our own infrastructure as code, how to choose which IAC to use? The book suggests to use these points when considering which IAC to use:

1. Configuration Management vs Provisioning
2. Mutable Infrastructure vs Immutable Infrastructure
3. Procedural Languange vs Declarative Language
4. Master vs Masterless
5. Agent vs Agentless
6. Large Community vs Small Community
7. Mature vs Cutting Edge
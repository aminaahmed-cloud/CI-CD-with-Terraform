# CI/CD with Terraform

## Project Overview

This project aims to integrate the "provision" stage into a CI/CD Pipeline using Terraform. Previously, the pipeline built Docker images and deployed them to existing servers. Now, with the integration of Terraform, the process of provisioning servers will be automated.

## Steps to Integrate Terraform in the CI/CD Pipeline:

### Create Jenkinsfile:

#### Steps to Do:

1. **Create SSH Key Pair on AWS:**
   - Generate an SSH key pair on AWS.
   - Add the SSH key pair into SSH Credentials for the pipeline.

2. **Install Terraform inside Jenkins Container:**
   -SSH into the Jenkins container.
   -Install Terraform and necessary dependencies.

```
# SSH into Jenkins container
ssh root@161.35.165.122

# Install Terraform
docker exec -it -u 0 c5f5770c87ce bash
cat /etc/os-release
apt-get update && sudo apt-get install -y gnupg software-properties-common
wget -O- https://apt.releases.hashicorp.com/gpg | gpg --dearmor | tee /usr/share/keyrings/hashicorp-archive-keyring.gpg > /dev/null
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update
sudo apt-get install terraform
terraform -v
```

3. **Terraform Configuration Files:**

   **.** Create main.tf and variables.tf files for Terraform configuration.

```
provider "aws" {
  region = var.region
}

resource "aws_vpc" "myapp-vpc" {
  cidr_block = var.vpc_cidr_block
  tags = {
    Name: "${var.env_prefix}-vpc"
  }
}

resource "aws_subnet" "myapp-subnet-1" {
  vpc_id = aws_vpc.myapp-vpc.id
  cidr_block = var.subnet_cidr_block
  availability_zone = var.avail_zone
    tags = {
    Name: "${var.env_prefix}-subnet-1"
  }
}

resource "aws_internet_gateway" "myapp-igw" {
  vpc_id = aws_vpc.myapp-vpc.id
  tags = {
    Name: "${var.env_prefix}-igw"
  }
}

resource "aws_default_route_table" "main-rtb" {
  default_route_table_id = aws_vpc.myapp-vpc.default_route_table_id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.myapp-igw.id
  }
  tags = {
    Name: "${var.env_prefix}-main-rtb"
  }
}

resource "aws_default_security_group" "default-sg" {
  vpc_id = aws_vpc.myapp-vpc.id

  ingress {
    from_port = 22
    to_port = 22
    protocol = "TCP"
    cidr_blocks = [var.my_ip, var.jenkins_ip]
  }

  ingress {
    from_port = 8080
    to_port = 8080
    protocol = "TCP"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port = 0
    to_port = 0
    protocol = "-1"
    cidr_blocks = ["0.0.0.0/0"]
    prefix_list_ids = []
  }

  tags = {
    Name: "${var.env_prefix}-default-sg"
  }
}

data "aws_ami" "latest-amazon-linux-image" {
  most_recent = true
  owners = ["amazon"]
  filter {
    name = "name" 
    values = ["amzn2-ami-kernel-*-x86_64-gp2"]
  }
  filter {
    name = "virtualization-type"
    values = ["hvm"]
  }
}

resource "aws_instance" "myapp-server" {
  ami = data.aws_ami.latest-amazon-linux-image.id
  instance_type = var.instance_type

  subnet_id = aws_subnet.myapp-subnet-1.id
  vpc_security_group_ids = [aws_default_security_group.default-sg.id]
  availability_zone = var.avail_zone

  associate_public_ip_address = true
  key_name = "myapp-key-pair"

  user_data = file("entry-script.sh")

  user_data_replace_on_change = true

  tags = {
    Name: "${var.env_prefix}-server"
  }
}

output "ec2-public_ip" {
  value = aws_instance.myapp-server.public_ip
}
```

---

```
variable vpc_cidr_block {
  default = "10.0.0.0/16"
}
variable subnet_cidr_block {
  default = "10.0.10.0/24"
}
variable avail_zone {
  default = "eu-west-2a"
}
variable env_prefix {
  default = "dev"
}
variable my_ip {
  default = "82.42.114.199/32"
}
variable jenkins_ip {
  default = "161.35.165.122/32"
}
variable instance_type {
  default = "t2.micro"
}
variable region {
  default = "eu-west-2"
}
```

4. **Provision Stage in Jenkinsfile:**

```

stage("provision server") {
  environment {
    AWS_ACCESS_KEY_ID = credentials('Jenkins_aws_access_key_id')
    AWS_SECRET_ACCESS_KEY = credentials('Jenkins-aws_secret_access_key')
    TF_VAR_env_prefix = 'test'
  }
  steps {
    script {
      dir('terraform') {
        sh "terraform init"
        sh "terraform apply --auto-approve"
        EC2_PUBLIC_IP = sh (
          script: "terraform output ec2_public_ip",
          returnStdout: true
        ).trim()
      }
    }
  } 
}

```

5. **Deploy Stage in Jenkinsfile:**
   -Modify the deploy stage in the Jenkinsfile to include the EC2 instance's public IP.
   -Create a script to deploy Docker images to the provisioned server.

```
stage("deploy") {
  environment {
    DOCKER_CREDS = credentials('docker-hub-repo')
  }
  steps {
    script {
      echo "waiting for EC2 server to initialize"
      sleep(time: 90, unit: "SECONDS")

      echo 'deploying docker image to EC2...'
      echo "${EC2_PUBLIC_IP}"      

      def shellCmd = "bash ./server-cmds.sh ${IMAGE_NAME} ${DOCKER_CREDS_USR} ${DOCKER_CREDS_PSW}"
      def ec2Instance = "ec2-user@${EC2_PUBLIC_IP}"

      sshagent(['server-ssh-key']) {
        sh "scp -o StrictHostKeyChecking=no server-cmds.sh ${ec2Instance}:/home/ec2-user"
        sh "scp -o StrictHostKeyChecking=no docker-compose.yaml ${ec2Instance}:/home/ec2-user"
        sh "ssh -o StrictHostKeyChecking=no ${ec2Instance} ${shellCmd}"
      }
    }
  }
}
```

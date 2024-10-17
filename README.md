![IndiaNIC-Symbol-Black.png](https://box.indianic.biz/index.php/s/aTcsj9zcdmTKEtS/download?path=%2FTemplate&files=logo-inic-black-img.jpg)

# **Pompah AWS Infrastructure**



### Production Deployment: 

This document aims at describing various components of the infrastructure for the **"pompah"** project. infrastructure is hosted the on AWS. The cloud infrastructure is for **production environment**.



### **Architecture Diagram**: 

------

![01_architecture.jpg](https://box.indianic.biz/index.php/s/aTcsj9zcdmTKEtS/download?path=%2FAWS%2Fpompah%2FArchitecture&files=01_architecture.jpg)



**Deployment URL:**

API : https://api.pompah.com

Admin : https://admin.pompah.com

Front : https://beta.pompah.com

Deployment Region :   AWS Mumbai Region 



### Components of architecture:

- S3 bucket to store terraform state file.
- VPC (subnets, Internet gateway, NAT gateway, Route tables (Public and Private), Subnets association, Security groups)
- Bastion host (EC2 instance)
- Redis Server (EC2 instance)
- Apache Kafka (EC2 instance)
- RDS (Postgresql)
- ECR
- EKS cluster with public and private endpoint 
- Jenkins Server (EC2 instance)
- Load balancer for EKS
- Route 53
- Project repositories and EKS deployment YAML files
- Jenkins pipelines 



Setup has been done using **Terraform**. We keep terraform state file into AWS S3 bucket. To start with terraform create provider.tf and s3.tf files.

Before we start using terraform script, we need to configure AWS credentials file for the perticular AWS Account.

Install aws cli. 

```bash
$ sudo apt install awscli
```

After this add "**credentials**" file. 

```bash
$ sudo nano .aws/credentials
```

```
[profile name] #Give any name (practice is to use project name)
aws_access_key_id = <aws access key>
aws_secret_access_key = <aws secret access key>
```



### Terraform Provider :

------

This is the terraform provider file. Provider is AWS because want to create infrastructure on AWS. Apart from provider, give details for the region and profile. (**provider.tf**)

```
#set region and projectname for the provider
provider "aws" {
  region  = var.region
  profile = var.projectName
}
```



### S3 Bucket :

------

Create **S3 bucket** using terraform script. **(It is recommanded to create S3 bucket manually)**

```
resource "aws_s3_bucket" "pompah-bucket" {
  bucket = "${var.projectName}-terraform-state"
  acl    = "private"

  lifecycle {
    prevent_destroy = true
  }

  versioning {
    enabled = true
  }

  server_side_encryption_configuration {
    rule {
      apply_server_side_encryption_by_default {
        sse_algorithm = "AES256"
      }
    }
  }

  tags = {
    Name        = "${var.projectName}-terraform-state"
  }
} 
```

run following commands. 

```
terraform init
terraform plan
terraform apply
```



### Terraform file to store variables :

All the **variables** related to different terraform resoreces are stored in **variables.tf** file. 

```
## Variables for env, region, and projectname  ##
variable "env" {
  type    = string
  default = "production"
}

variable "region" {
  type    = string
  default = "ap-south-1"
}

variable "projectName" {
  type    = string
  default = "pompah"
}

## VPC variables  ##

variable "cidr_blocks" {
  default     = ["10.2.0.0/16","10.2.1.0/24","10.2.2.0/24","10.2.3.0/24","10.2.11.0/24","10.2.12.0/24","10.2.13.0/24"]
  type        = list(string)
  description = "CIDR block for the VPC,public subnet 1,public subnet 2,public subnet 3,private subnet 1,private subnet 2,private subnet 3"
}

variable "availablity_zones" {
  default     = ["ap-south-1a","ap-south-1b","ap-south-1c"]
  type        = list(string)
  description = "Availablity zones for subnets"
}

variable "elastic-private-ip-range" {
  default     = "10.2.11.5"
  type        = string
  description = "Elastic private IP address range for Nat Gateway"
}

variable "destination-cidr-block" {
  default     = "0.0.0.0/0"
  type        = string
  description = "Destination CIDR Block for Nat and Internet Gateway"
}

## EC2 variables  ##
variable "ec2_ami" {
  default     = "ami-0567e0d2b4b2169ae"
  type        = string
  description = "ami image for EC2"
}

variable "ec2_bastion-host_instance_type" {
  default     = "t3.nano"
  type        = string
  description = "instance type for EC2"
}

variable "ec2_jenkins-server_instance_type" {
  default     = "t3a.medium"
  type        = string
  description = "instance type for EC2"
}

variable "ec2_redis_instance_type" {
  default     = "t3.small"
  type        = string
  description = "instance type for EC2"
}

variable "ec2_kafka_instance_type" {
  default     = "t3.medium"
  type        = string
  description = "instance type for EC2"
}

variable "ec2_key_name" {
  default     = "pompah"
  type        = string
  description = "key-pair to access EC2 instance"
}

variable "ec2_root_volume_type" {
  default     = "gp2"
  type        = string
  description = "EBS volume type of EC2 instance for root volume"
}

variable "ec2_root_volume_size" {
  default     = 20
  type        = number
  description = "EBS volume size of EC2 instance for root volume"
}

variable "ec2_jenkins_root_volume_size" {
  default     = 50
  type        = number
  description = "EBS volume size of EC2 instance for root volume"
}

variable "ec2_kafka_root_volume_size" {
  default     = 40
  type        = number
  description = "EBS volume size of EC2 instance for root volume"
}

## RDS variables for mysql  ##
variable "rds_engine" {
    default     = "postgres"
    type        = string
    description = "RDS engine"  
}

variable "rds_engine_version" {
    default     = "13.4"
    type        = string
    description = "RDS engine version"  
}

variable "rds_db_cluster_identifier" {
    default     = "postgresql-db-production"
    type        = string
    description = "RDS DB cluster identifier"  
}

variable "rds_master_username" {
    default     = "" 
    type        = string
    description = "RDS DB master username"  
    sensitive   = true
}

variable "rds_master_password" {
    default     = ""   
    type        = string
    description = "RDS DB root user password"    
    sensitive   = true
}

variable "rds_instance_class" {
    default     = "db.t3.small"
    type        = string
    description = "RDS DB instance class"  
}

variable "rds_allocated_storage" {
    default     = 20
    type        = number
    description = "RDS DB storage size"  
}

variable "rds_multi_az" {
    default     = false
    type        = bool
    description = "RDS DB multi AZ option selection"  
}

variable "rds_db_port" {
    default     = 5432
    type        = number
    description = "RDS DB port number"  
}

variable "rds_initial_db_name" {
    default     = "pompah_production_db"
    type        = string
    description = "RDS initial DB name"  
}

variable "rds_backup_retention_period" {
    default     = 7
    type        = number
    description = "RDS DB backup retention period"  
}

variable "rds_copy_tags_to_snapshot" {
    default     = true
    type        = bool
    description = "RDS DB copy tags to snapshot"  
}

variable "rds_auto_minor_version_upgrade" {
    default     = false
    type        = bool
    description = "RDS DB auto minor version upgrade"  
}

variable "rds_deletion_protection" {
    default     = true
    type        = bool
    description = "RDS DB delete protection"  
}

variable "rds_skip_final_snapshot" {
    default     = true
    type        = bool
    description = "RDS DB skip final snapshot"  
}
```



### S3 Backend :

------

Now, move terraform statefile to S3 bucket by creating **remote backend**. (**s3-beackend.tf**)

```
#S3 backend to store terraform statefile into terraform bucket
terraform {
  backend "s3" {
    bucket = "pompah-terraform-state"
    key    = "production/terraform.tfstate"
    region = "ap-south-1"
    profile = "pompah"
  }
}
```

Reinitialise the new backend by running the following command.

```
terraform init
```



### **VPC :**

------

Amazon VPC is a service that lets you launch AWS resources in a logically isolated virtual network that hold all our infrastructure. EKS cluster has specific requirement of VPC. 

This VPC has three public and three private[ ](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Subnets.html)subnets. One public and one private subnet are deployed to the same Availability Zone. The other public and private subnets are deployed to a second and third Availability Zones in the same Region. This option allows you to deploy your nodes to private subnets and allows Kubernetes to deploy load balancers to the public subnets that can load balance traffic to pods running on nodes in the private subnets. The nodes in private subnets can communicate with the cluster and other AWS services, and pods can communicate outbound to the internet through a NAT gateway.



Create **VPC**, 3 public subnets, 3 private subnets,  internet gateway, subnet associations with the route tables, elastic IPs (jenkins, NAT gateway, bastion host), NAT gateway, public and private route tables. (**vpc.tf**)

```
# Create VPC 
resource "aws_vpc" "pompah_vpc" {
  cidr_block           = var.cidr_blocks[0]
  enable_dns_hostnames = true
  enable_dns_support   = true
  instance_tenancy     = "default"

  tags = {
    Name        = "${var.projectName}-vpc"
  }
}

# Create Internet Gateways and attached to VPC.
resource "aws_internet_gateway" "pompah_ig" {
  vpc_id = aws_vpc.pompah_vpc.id

  tags = {
    Name        = "${var.projectName}-igw"
  }
}

#Create public subnet-1 
resource "aws_subnet" "pompah_pub_sub_1" {
  vpc_id            = aws_vpc.pompah_vpc.id 
  availability_zone = var.availablity_zones[0]
  cidr_block        = var.cidr_blocks[1]

  tags = {
    Name        = "${var.projectName}-pub-sub1"
    "kubernetes.io/cluster/pompah-eks-cluster" = "shared"
    "kubernetes.io/role/elb" = "1"
  }
}

#Create public subnet-2
resource "aws_subnet" "pompah_pub_sub_2" {
  vpc_id            = aws_vpc.pompah_vpc.id
  availability_zone = var.availablity_zones[1]
  cidr_block        = var.cidr_blocks[2]

  tags = {
    Name        = "${var.projectName}-pub-sub2"
    "kubernetes.io/cluster/pompah-eks-cluster" = "shared"
    "kubernetes.io/role/elb" = "1"
  }
}

#Create public subnet-3
resource "aws_subnet" "pompah_pub_sub_3" {
  vpc_id            = aws_vpc.pompah_vpc.id
  availability_zone = var.availablity_zones[2]
  cidr_block        = var.cidr_blocks[3]

  tags = {
    Name        = "${var.projectName}-pub-sub3"
    "kubernetes.io/cluster/pompah-eks-cluster" = "shared"
    "kubernetes.io/role/elb" = "1"
  }
}

#Create private subnet-1
resource "aws_subnet" "pompah_pri_sub_1" {
  vpc_id            = aws_vpc.pompah_vpc.id
  availability_zone = var.availablity_zones[0]
  cidr_block        = var.cidr_blocks[4]

  tags = {
    Name        = "${var.projectName}-pri-sub1"
    "kubernetes.io/cluster/pompah-eks-cluster" = "shared"
    "kubernetes.io/role/internal-elb" = "1"
  }
}

#Create private subnet-2
resource "aws_subnet" "pompah_pri_sub_2" {
  vpc_id            = aws_vpc.pompah_vpc.id
  availability_zone = var.availablity_zones[1]
  cidr_block        = var.cidr_blocks[5]

  tags = {
    Name        = "${var.projectName}-pri-sub2"
    "kubernetes.io/cluster/pompah-eks-cluster" = "shared"
    "kubernetes.io/role/internal-elb" = "1"
  }
}

#Create private subnet-3
resource "aws_subnet" "pompah_pri_sub_3" {
  vpc_id            = aws_vpc.pompah_vpc.id
  availability_zone = var.availablity_zones[2]
  cidr_block        = var.cidr_blocks[6]

  tags = {
    Name        = "${var.projectName}-pri-sub3"
    "kubernetes.io/cluster/pompah-eks-cluster" = "shared"
    "kubernetes.io/role/internal-elb" = "1"
  }
}

# Subnet association between a route table and a subnet1
resource "aws_route_table_association" "pompah_pri_sub_a" {
  subnet_id      = aws_subnet.pompah_pri_sub_1.id
  route_table_id = aws_route_table.pompah_private_rt.id
}

# Subnet association between a route table and a subnet2
resource "aws_route_table_association" "pompah_pri_sub_b" {
  subnet_id      = aws_subnet.pompah_pri_sub_2.id
  route_table_id = aws_route_table.pompah_private_rt.id
}

# Subnet association between a route table and a subnet3
resource "aws_route_table_association" "pompah_pri_sub_c" {
  subnet_id      = aws_subnet.pompah_pri_sub_3.id
  route_table_id = aws_route_table.pompah_private_rt.id
}


# Subnet association between a route table and a subnet1
resource "aws_route_table_association" "pompah_pub_sub_a" {
  subnet_id      = aws_subnet.pompah_pub_sub_1.id
  route_table_id = aws_default_route_table.pompah_default_rt.id
}

# Subnet association between a route table and a subnet2
resource "aws_route_table_association" "pompah_pub_sub_b" {
  subnet_id      = aws_subnet.pompah_pub_sub_2.id
  route_table_id = aws_default_route_table.pompah_default_rt.id
}

# Subnet association between a route table and a subnet2
resource "aws_route_table_association" "pompah_pub_sub_c" {
  subnet_id      = aws_subnet.pompah_pub_sub_3.id
  route_table_id = aws_default_route_table.pompah_default_rt.id
}

##Create Elastic IP for NAT Gateway
resource "aws_eip" "pompah_elastic-ip" {
  vpc                       = true
  associate_with_private_ip = var.elastic-private-ip-range

  tags = {
    Name        = "${var.projectName}-ngw-elastic-ip"
    Description = "Elastic IP for NAT Gateway"
  }
}

#Create Elastic IP for jenkins server
resource "aws_eip" "pompah_jenkins-server_elastic-ip" {
   instance = aws_instance.pompah_jenkins-server.id
   vpc      = true

   tags = {
     Name        = "${var.projectName}-jenkins-elastic-ip"
     Description = "Elastic IP for jenkins server"
   }
}

#Create Elastic IP for bastion host
resource "aws_eip" "pompah_bastion-host_elastic-ip" {
  instance = aws_instance.pompah_bastion-host.id
  vpc      = true

  tags = {
    Name        = "${var.projectName}-bastion-elastic-ip"
    Description = "Elastic IP for bastion host"
  }
}


#Create NAT Gateway
resource "aws_nat_gateway" "pompah_nat_gateway" {
  allocation_id = aws_eip.pompah_elastic-ip.id
  subnet_id     = aws_subnet.pompah_pub_sub_1.id

  tags = {
    Name        = "${var.projectName}-ngw"
  }
}

#Manage a default route table of a VPC (Public Route Table)
resource "aws_default_route_table" "pompah_default_rt" {
  default_route_table_id = aws_vpc.pompah_vpc.default_route_table_id

  route {
    cidr_block = var.destination-cidr-block
    gateway_id = aws_internet_gateway.pompah_ig.id
  }
  tags = {
    Name        = "${var.projectName}-public-rt"
  }
}

#Create Private Route table
resource "aws_route_table" "pompah_private_rt" {
  vpc_id = aws_vpc.pompah_vpc.id

  route {
    cidr_block     = var.destination-cidr-block
    nat_gateway_id = aws_nat_gateway.pompah_nat_gateway.id
  }
  tags = {
    Name        = "${var.projectName}-private-rt"
  }
}
```



### Security Groups :

Create **security groups** for EKS, Internal subnet routing, bastion host, and jenkins server. (**security-groups.tf**)

```
# pompah EKS control plan security group
resource "aws_security_group" "pompah_eks-control-plane_sg" {
  name        = "${var.projectName}-eks control plane"
  description = "Allow communication between worker nodes"
  vpc_id      = aws_vpc.pompah_vpc.id

  # ingress {
  #   description      = "https port"
  #   from_port        = 443
  #   to_port          = 443
  #   protocol         = "tcp"
  #   cidr_blocks      = ["0.0.0.0/0"]
  # }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name        = "${var.projectName}-eks control plane-sg"
  }
}

# pompah internal subnet routing security group
resource "aws_security_group" "pompah_isr_sg" {
  name        = "${var.projectName}-internal_subnet"
  description = "Allow internal subnet routing"
  vpc_id      = aws_vpc.pompah_vpc.id

  ingress {
    description      = "internal subnet routing"
    from_port        = 0
    to_port          = 0
    protocol         = "-1"
    cidr_blocks      = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name        = "${var.projectName}-isr-sg"
  }
}


# pompah bastion security group
resource "aws_security_group" "pompah_bastion_sg" {
  name        = "${var.projectName}-bastion"
  description = "Allow bastion connection"
  vpc_id      = aws_vpc.pompah_vpc.id

  ingress {
    description      = "bastion connection"
    from_port        = 22
    to_port          = 22
    protocol         = "tcp"
    cidr_blocks      = ["202.131.107.130/32"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name        = "${var.projectName}-bastion-sg"
  }
}

# pompah jenkins-server security group 
resource "aws_security_group" "pompah_jenkins-server_sg" {
  name        = "${var.projectName}-jenkins-sg"
  description = "Allow http, https, & ssh inbound traffic"
  vpc_id      = aws_vpc.pompah_vpc.id

  ingress {
    description      = "https port"
    from_port        = 443
    to_port          = 443
    protocol         = "tcp"
    cidr_blocks      = ["202.131.107.130/32"]
  }

  ingress {
    description      = "http port"
    from_port        = 80
    to_port          = 80
    protocol         = "tcp"
    cidr_blocks      = ["202.131.107.130/32"]
  }

  ingress {
    description      = "ssh"
    from_port        = 22
    to_port          = 22
    protocol         = "tcp"
    cidr_blocks      = ["202.131.107.130/32"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name        = "${var.projectName}-jenkins-server-sg"
  }
}
```



### **EC2 instances :**

------

EC2 instances to setup bastion host, jenkins server, redis server, and kafka server. **Bastion host** is deployed in **public subnets** whereas **redis and kafka** servers are deployed in **private subnets**.



To access EC2 instance using SSH, create .pem file as given below.

- Navigate to EC2 Console & in “**Network & Security**” section from the left panel, select “**Key Pairs**” 
- Click on  “**Create Key Pair**”
- Enter a key name, select type, and format. Then create key pair.

![02_ec2_key-pair.png](https://box.indianic.biz/index.php/s/aTcsj9zcdmTKEtS/download?path=%2FAWS%2Fpompah%2FEC2&files=02_ec2_key-pair.png)



Set **name** of created key pair (in our example "pompah") as a **value** for the variable **ec2_key_name** in variables.tf file and execute terraform script to create EC2 instances. (**ec2.tf**)

```
#create bastion host EC2 instance
resource "aws_instance" "pompah_bastion-host" {
  ami = var.ec2_ami
  instance_type = var.ec2_bastion-host_instance_type
  availability_zone = var.availablity_zones[0]
  subnet_id = aws_subnet.pompah_pub_sub_1.id
  vpc_security_group_ids = [aws_security_group.pompah_bastion_sg.id]
  key_name = var.ec2_key_name

  root_block_device {
      volume_type = var.ec2_root_volume_type
      volume_size = var.ec2_root_volume_size
      tags = {
        Name = "${var.projectName}-root-block"
      }
    }
  
  tags = {
    Name = "${var.projectName}_bastion-host-${var.env}"
  }
}

#create Redis server EC2 instance
resource "aws_instance" "pompah_redis" {
  ami = var.ec2_ami
  instance_type = var.ec2_redis_instance_type
  availability_zone = var.availablity_zones[1]
  subnet_id = aws_subnet.pompah_pri_sub_2.id
  vpc_security_group_ids = [aws_security_group.pompah_isr_sg.id]
  key_name = var.ec2_key_name

  root_block_device {
      volume_type = var.ec2_root_volume_type
      volume_size = var.ec2_root_volume_size
      tags = {
        Name = "${var.projectName}-root-block"
      }
    }
  
  tags = {
    Name = "${var.projectName}_bastion-host-${var.env}"
  }
}

#create Kafka server EC2 instance
resource "aws_instance" "pompah_kafka" {
  ami = var.ec2_ami
  instance_type = var.ec2_kafka_instance_type
  availability_zone = var.availablity_zones[2]
  subnet_id = aws_subnet.pompah_pri_sub_3.id
  vpc_security_group_ids = [aws_security_group.pompah_isr_sg.id]
  key_name = var.ec2_key_name

  root_block_device {
      volume_type = var.ec2_root_volume_type
      volume_size = var.ec2_kafka_root_volume_size
      tags = {
        Name = "${var.projectName}-root-block"
      }
    }
  
  tags = {
    Name = "${var.projectName}_bastion-host-${var.env}"
  }
}
```



### Redis Setup: (EC2 instance)

------

To **setup redis** on EC2 instance, **SSH** into the bastion host using **"pompah.pem"** and from bastion host **SSH** into **redis instance** then execute following commands.

Install redis using following command:

```
$ sudo apt-get install redis
```

To check that Redis is working, open up a Redis command line with the `redis-cli` command:

```
$ redis-cli
```

Test the connection with the `ping` command:

```
127.0.0.1:6379> ping
```

If Redis is working correctly, you will see the following:

```
Output
PONG
```

exit the Redis command line:

```bash
127.0.0.1:6379> quit
```



**Binding redis to EC2 instance's ip and setup password:** 

open configuration file.

```
$ sudo nano /etc/redis/redis.conf
```

Under the network section bind EC2 instance's ip as shown in figure.

![03_redis_ip-bind.png](https://box.indianic.biz/index.php/s/aTcsj9zcdmTKEtS/download?path=%2FAWS%2Fpompah%2FRedis&files=03_redis_ip-bind.png)

Under the security section set password (at highlighted rectangle) for the redis.

![04_redis_password-set.png](https://box.indianic.biz/index.php/s/aTcsj9zcdmTKEtS/download?path=%2FAWS%2Fpompah%2FRedis&files=04_redis_password-set.png)

Restart the redis service.

```
$ sudo systemctl restart redis.service
```

Now you can connect with redis with the `redis-cli` as follows.

```
$ redis-cli -h <redis-ip> -p <port> -a <redis-password>
```



### Kafka Setup : (EC2 instance)

------

To setup kafka on EC2, **SSH** into the bastion host using **"pompah.pem"** and from bastion host **SSH** into **kafka instance** then execute following commands.

Install Java:

```
$ sudo apt install openjdk-17-jre-headless
```

Downlaod kafka:

```
$ wget https://dlcdn.apache.org/kafka/2.6.3/kafka_2.12-2.6.3.tgz
```

Extract kafka:

```
$ tar zxvf kafka_2.12-2.6.3.tgz
```

Rename kafka_2.12-2.6.3.tgz to kafka:

```
$ mv kafka_2.12-2.6.3 kafka
```

Configure server.properties file:

```
$ cd /kafka/config
~/kafka/config$ vi server.properties
```

Add followings to the "Socket Server Settings" section as shown in the figure

**listeners=PLAINTEXT://<kafka server ip>:9092**

**advertised.listeners=PLAINTEXT://<kafka server ip>:9092**

![05_kafka_listeners-ip.png](https://box.indianic.biz/index.php/s/aTcsj9zcdmTKEtS/download?path=%2FAWS%2Fpompah%2FKafka&files=05_kafka_listeners-ip.png)



Start zookeeper service:

```
~/kafka/config$ cd ..
~/kafka$ bin/zookeeper-server-start.sh config/zookeeper.properties & disown
```

Start kafka service:

```
~/kafka$ bin/kafka-server-start.sh config/server.properties & disown
```



### **RDS Postgres :**

------

RDS postgres is setup in private subnet so that it can't accessible publicully.   

To store master username and password terraform environment variables are used. Because we don't want to pass credentials in terraform variable file.

```bash
export TF_VAR_rds_master_username = 'admin' <use your username>
export TF_VAR_rds_master_password = 'password' <use your password>
```



Terraform script to setup RDS postgres. (**rds.tf**)

```
resource "aws_db_instance" "pompah-postgresql-db" {
  #Engine options
  engine                 = var.rds_engine
  engine_version         = var.rds_engine_version
  
  #Settings
  identifier             = var.rds_db_cluster_identifier 
  username               = var.rds_master_username  
  password               = var.rds_master_password

  #DB instance class
  instance_class         = var.rds_instance_class  

  #Storage
  allocated_storage      = var.rds_allocated_storage  

  #Availability & durability
  multi_az               = var.rds_multi_az 

  #Connectivity
  db_subnet_group_name   = aws_db_subnet_group.pompah_db_subnet.name
  vpc_security_group_ids = [aws_security_group.pompah_isr_sg.id]
  port                   = var.rds_db_port   #Additional configuration
  
  #Additional configuration
  name                       = var.rds_initial_db_name #Database name
  backup_retention_period    = var.rds_backup_retention_period
  copy_tags_to_snapshot      = var.rds_copy_tags_to_snapshot
  auto_minor_version_upgrade = var.rds_auto_minor_version_upgrade
  deletion_protection        = var.rds_deletion_protection
  skip_final_snapshot        = var.rds_skip_final_snapshot  
  
}

resource "aws_db_subnet_group" "pompah_db_subnet" {  
    name       = "${var.projectName}-db-subnet-group"  
    subnet_ids = [aws_subnet.pompah_pri_sub_1.id,aws_subnet.pompah_pri_sub_2.id,aws_subnet.pompah_pri_sub_3.id] 

  tags = {    
      Name = "${var.projectName}-db-subnet"  
  }
}
```

#### **No need to restore dump as production setup required fresh database.** 



### Amazon Elastic Container Registry (Amazon ECR) :

------

Amazon ECR is a fully managed container registry offering  high-performance hosting, so you can reliably deploy application images  and artifacts anywhere.        

We need to create ECR private repos to store docker images of admin, api (10 micro services), and front. (**ecr.tf**)

```
#ECR for pompah admin 
resource "aws_ecr_repository" "pompah-admin" {
  name = "${var.projectName}-admin"
  
  tags = {
    Name        = "${var.projectName}-admin"
  }
}

#ECR for pompah api artist micro service
resource "aws_ecr_repository" "pompah-artist" {
  name = "${var.projectName}-artist"
  
  tags = {
    Name        = "${var.projectName}-artist"
  }
}

#ECR for pompah api auth micro service
resource "aws_ecr_repository" "pompah-auth" {
  name = "${var.projectName}-auth"
  
  tags = {
    Name        = "${var.projectName}-auth"
  }
}

#ECR for pompah api common micro service
resource "aws_ecr_repository" "pompah-common" {
  name = "${var.projectName}-common"
  
  tags = {
    Name        = "${var.projectName}-common"
  }
}

#ECR for pompah api contest micro service
resource "aws_ecr_repository" "pompah-contest" {
  name = "${var.projectName}-contest"
  
  tags = {
    Name        = "${var.projectName}-contest"
  }
}

#ECR for pompah api event micro service
resource "aws_ecr_repository" "pompah-event" {
  name = "${var.projectName}-event"
  
  tags = {
    Name        = "${var.projectName}-event"
  }
}

#ECR for pompah api master micro service
resource "aws_ecr_repository" "pompah-master" {
  name = "${var.projectName}-master"
  
  tags = {
    Name        = "${var.projectName}-master"
  }
}

#ECR for pompah api notification micro service
resource "aws_ecr_repository" "pompah-notification" {
  name = "${var.projectName}-notification"
  
  tags = {
    Name        = "${var.projectName}-notification"
  }
}

#ECR for pompah api retrive micro service
resource "aws_ecr_repository" "pompah-retrive" {
  name = "${var.projectName}-retrive"
  
  tags = {
    Name        = "${var.projectName}-retrive"
  }
}

#ECR for pompah api tutor micro service
resource "aws_ecr_repository" "pompah-tutor" {
  name = "${var.projectName}-tutor"
  
  tags = {
    Name        = "${var.projectName}-tutor"
  }
}

#ECR for pompah api user micro service
resource "aws_ecr_repository" "pompah-user" {
  name = "${var.projectName}-user"
  
  tags = {
    Name        = "${var.projectName}-user"
  }
}

#ECR for pompah front (React)
resource "aws_ecr_repository" "pompah-front" {
  name = "${var.projectName}-front"
  
  tags = {
    Name        = "${var.projectName}-front"
  }
}
```



### Amazon Elastic Container Service for Kubernetes (Amazon EKS) :

------

Amazon Elastic Container Service for Kubernetes (Amazon EKS) is a  managed service that makes it easy for users to run Kubernetes on AWS  without needing to stand up or maintain your own Kubernetes control  plane. Since Amazon EKS is a managed service it handles tasks such as  provisioning, upgrades, and patching.



#### How does Amazon EKS work?

![06_eks-intro.png](https://box.indianic.biz/index.php/s/aTcsj9zcdmTKEtS/download?path=%2FAWS%2Fpompah%2FEKS&files=06_eks-intro.png)



##### Steps to create Amazon EKS:

- Create an AWS IAM service role for EKS cluster 
- AWS VPC (**Already created with terraform**)
- Create Amazon EKS cluster
- Configure kubectl for Amazon EKS cluster
- Create an AWS IAM service role for worker nodes
- Launch and configure Amazon EKS worker nodes
- Deploy applications to the EKS



###### Step 1: Create an AWS IAM service role for EKS cluster

- Navigate to AWS IAM Console & in “**Roles**” section, click the “**Create role**” button
- Select “**AWS service**” as the type of entity, “**EKS**” as the service, and “**EKS cluster**” as use case
- Enter a name for the service role (ex. eks-cluster-role) and click “**Create role**” to create the role

![07_EKS-cluster-role.png](https://box.indianic.biz/index.php/s/aTcsj9zcdmTKEtS/download?path=%2FAWS%2Fpompah%2FEKS&files=07_EKS-cluster-role.png)



###### Step 3: Create Amazon EKS cluster

- Navigate to the Amazon EKS console and click on “**Create cluster**” button
- Enter details into the EKS cluster creation form such as cluster name, role ARN, VPC, subnets and security groups as shown in figures.
- Click “**Create**” to create the Amazon EKS cluster



**Configure cluster:**

Enter EKS cluster name and select IAM role "eks-cluster-role" that we have created in step-1 and click next.

![08_EKS-cluster-1.png](https://box.indianic.biz/index.php/s/aTcsj9zcdmTKEtS/download?path=%2FAWS%2Fpompah%2FEKS&files=08_EKS-cluster-1.png)



**Specifying networking:**

Select VPC "pompah-vpc" that you have created. Then select security group "pompah--eks control plane-sg" from dropdown.

![09-EKS-cluster-2.png](https://box.indianic.biz/index.php/s/aTcsj9zcdmTKEtS/download?path=%2FAWS%2Fpompah%2FEKS&files=09-EKS-cluster-2.png)



EKS cluster endpoint access selection.

There are three options.

1) **Public:** Select this option if you want to access EKS cluster outside of your Amazon VPC. Worker node traffic will leave your VPC to connect to the endpoint. This option is **not feasible** for the production as worker node traffic is directly exposed outside Amazon VPC.
2) **Public and private:** Select this option if you want to access EKS cluster out side of your Amazon VPC. Worker node traffic to the endpoint will stay within your VPC. **We have selected this option because we can access EKS cluster outside Amazon VPC and worker node traffic is not directly exposed outside Amazon VPC. (After selecting this option click next)**
3) **Private:** Select this option if you want to access EKS cluster within your Amazon VPC. Worker node traffic to the endpoint will stay within your VPC. **If you select this option then EKS cluster is accessible through worker nodes only.**

![10_EKS-cluster-3.png](https://box.indianic.biz/index.php/s/aTcsj9zcdmTKEtS/download?path=%2FAWS%2Fpompah%2FEKS&files=10_EKS-cluster-3.png)



**Configure logging:**

No need to enable logging. Click Next then **review and create**. 

![11_EKS-cluster-4.png](https://box.indianic.biz/index.php/s/aTcsj9zcdmTKEtS/download?path=%2FAWS%2Fpompah%2FEKS&files=11_EKS-cluster-4.png)



###### Step 4: Configure kubectl for Amazon EKS cluster

We can test if EKS cluster is accessible or not from our system (outside Amazon VPC). We need to install kubectl, aws cli, and eksctl.

Install kubectl. 

```
$ curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
$ sudo chmod +x kubectl
$ sudo mv kubectl /usr/local/bin/
```

Install aws cli. After this run "**aws configure**" to  to configure the settings that the AWS Command Line Interface (AWS CLI) uses to interact with AWS

```
$ sudo apt install awscli
$ aws configure
```

Install eksctl.

```
$ curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
$ sudo mv /tmp/eksctl /usr/local/bin/
```

Then run the following command to configure kubectl for your EKS cluster

```
$ aws eks --region <your region> update-kubeconfig --name <your EKS cluster name>
```



#### **Note: We will configure these things when we setup jenkins server beacuse we will be deploying micro services using jenkins pipeline.** 



###### Step 5: Create an AWS IAM service role for worker nodes

- Navigate to AWS IAM Console & in “**Roles**” section, click the “**Create role**” button
- Select “**AWS service**” as the type of entity, “**EC2**” as the service
- Attach policies as shown in the figure.
- Enter a name for the service role (ex. eks-node-group-role) and click “**Create role**” to create the role

![12_EKS_node-group-role.png](https://box.indianic.biz/index.php/s/aTcsj9zcdmTKEtS/download?path=%2FAWS%2Fpompah%2FEKS&files=12_EKS_node-group-role.png)



###### Step 6: Launch and configure Amazon EKS worker nodes

- Navigate to the Amazon EKS console, select “**pompah-eks-prod**” then go to the **"Compute"**, and click **"Add Node Group"**. 
- Enter details into the EKS cluster creation form such as cluster name, role ARN, VPC, subnets and security groups as shown in figures.
- Click “**Create**” to create the Node Group



**Configure Node Group:**

Enter Node group name and select IAM role "eks-node-group-role" that we have created in step-5 and click next.

![13_EKS_nodegroup-1.png](https://box.indianic.biz/index.php/s/aTcsj9zcdmTKEtS/download?path=%2FAWS%2Fpompah%2FEKS&files=13_EKS_nodegroup-1.png)



**Set compute and scaling configuration:**

Set scaling configuration as shown in the figure and click next.

![EKS_nodegroup-2.png](https://box.indianic.biz/index.php/s/aTcsj9zcdmTKEtS/download?path=%2FAWS%2Fpompah%2FEKS&files=14_EKS_nodegroup-2.png)



**Specifying networking:**

Select private subnets to put worker nodes and enable SSH. Click next. Then **review and create**.

![15_EKS_nodegroup-3.png](https://box.indianic.biz/index.php/s/aTcsj9zcdmTKEtS/download?path=%2FAWS%2Fpompah%2FEKS&files=15_EKS_nodegroup-3.png)





### **Jenkins server:**

------

**Jenkins server** (EC2 instnace) is deployed in **public subnet.**

```
#create jenkins server EC2 instance
resource "aws_instance" "pompah_jenkins-server" {
  ami = var.ec2_ami
  instance_type = var.ec2_jenkins-server_instance_type
  availability_zone = var.availablity_zones[0]
  subnet_id = aws_subnet.reserv-ae_pub_sub_1.id
  vpc_security_group_ids = [aws_security_group.pompah_jenkins-server_sg.id]
  key_name = var.ec2_key_name
  
  root_block_device {
      volume_type = var.ec2_root_volume_type
      volume_size = var.ec2_jenkins_root_volume_size
      tags = {
        Name = "${var.projectName}-root-block"
      }
    }

  user_data = file("jenkins.sh")

  tags = {
    Name = "${var.projectName}_jenkins-server-${var.env}"
  }
}
```

To setup jenknis on EC2, pass "jenkins.sh" as a user_data in above terraform script. Keep this file in same directory as terraform files.

```bash
#! /bin/bash
sudo apt-get update
sudo apt-get upgrade -y
sudo apt-get install nginx -y
sudo systemctl enable nginx
sudo apt install openjdk-11-jdk -y
wget -q -O - https://pkg.jenkins.io/debian/jenkins.io.key | sudo apt-key add -
sudo sh -c 'echo deb https://pkg.jenkins.io/debian binary/ > /etc/apt/sources.list.d/jenkins.list'
sudo apt-get update
sudo apt-get install jenkins -y
```

### Jenkins setup using nginx:

Once the jenkins is server is up, it can be run on web port (80) by setting up reverse proxy using nginx. 

SSH into the jenkins instance then replace the "**location**" code of default file as given below.

**path:** /etc/nginx/sites-enabled/default 

```bash
location / {
                # First attempt to serve request as file, then
                # as directory, then fall back to displaying a 404.
                #try_files $uri $uri/ =404;
                include /etc/nginx/proxy_params;
                proxy_pass          http://localhost:8080;
                proxy_read_timeout  90s;
        }
```

Restart the nginx. Atfer that it is accessible using public ip of EC2 instance.

```
$ sudo systemctl restart nginx
```

Start the jenkins using public ip: http://<jenkins EC2 public ip> and complete post installation setup.



**Install necessary tools on jenkins server:** 

Again SSH into jenkins server. Switch to root user.

```
$ sudo -i
```



Install kubectl to interact with the EKS cluster to make the deployments.

```
root# curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
root# chmod +x kubectl
root# mv kubectl /usr/local/bin/
```



Install aws cli to configure AWS account profile. After this run "**aws configure**" to  to configure the settings that the AWS Command Line Interface (AWS CLI) uses to interact with AWS

```
root# apt install awscli
root# aws configure
```



Install eksctl so that we can talk to the EKS cluster.

```
root# curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
root# mv /tmp/eksctl /usr/local/bin/
```



Configure config file of the EKS cluster to execute kubectl commands from the jenkins server. Run the following command to configure kubectl for your EKS cluster

```
root# aws eks --region <your region> update-kubeconfig --name <your EKS cluster name>
```



Install docker to execute docker commands such as "**docker build and docker push**" and jenkins user to docker group.

```
root# apt install docker.io
root# sudo usermod -aG docker jenkins
```

Give permission to jenkins user to run docker command.

```
root# chown jenkins:root /var/run/docker.sock
```



Install envsubst tools as a root. **This tool substitutes env variable/s from jenkins pipeline to kubernetes yaml file.**

```
root# apt-get install gettext-base
```





**Host entry for git (indianic) and EKS cluster endpoint:** 

Ping EKS cluster endpoint to findout its ip.

```
$ ping 33114CAD678EF4C9E6ACF10045ED9A14.sk1.ap-south-1.eks.amazonaws.com
```



![16_host-entry-eks-cluster.png](https://box.indianic.biz/index.php/s/aTcsj9zcdmTKEtS/download?path=%2FAWS%2Fpompah%2FJenkins&files=16_host-entry-eks-cluster.png)



Add git (indianic) and EKS cluster endpoint entries to /etc/hosts as shown in the figure.

![17_Jenkins-etc-hosts-file.png](https://box.indianic.biz/index.php/s/aTcsj9zcdmTKEtS/download?path=%2FAWS%2Fpompah%2FJenkins&files=17_Jenkins-etc-hosts-file.png)



### ALB for EKS :

------

Whenever we deploy any micro service/s with service type LoadBalancer, EKS will automatically creates a Classic Load balancer.  To use Applicatio Load Balancer with EKS need to do additional configuration. 



##### Steps to create ALB for AWS EKS:

- Tagging subnets of the VPC

- Create an IAM OIDC provider for your cluster

- Create an IAM policy for ALB

- Create an IAM role

- Create AWS Load Balancer Controller

- Deploy the AWS Load Balancer Controller

  

Perform steps on jenkins server. SSH into jenkins server.



**1. Tagging subnets of the VPC** 

- Navigate to VPC then subnets and add the following tags to private and public subnets.

Tag all public and private subnets that your cluster uses for load balancer resources with the following key-value pair:

```
Key: kubernetes.io/cluster/<Enter Cluster Name>
Value: shared
```

The cluster-name value is for your Amazon EKS cluster. The shared value allows more than one cluster to use the subnet.



To allow Kubernetes to use your private subnets for internal load balancers, tag all private subnets in your VPC with the following key-value pair:

```
Key: kubernetes.io/role/internal-elb
Value: 1
```



To allow Kubernetes to use only tagged subnets for external load balancers, tag all public subnets in your VPC with the following key-value pair:

```
Key: kubernetes.io/role/elb
Value: 1
```





**2. Create an IAM OIDC provider for your cluster**

EKS cluster has an OpenID Connect issuer URL associated with it. To use IAM roles for service accounts, an IAM OIDC provider must exist for your cluster.

```
$ eksctl utils associate-iam-oidc-provider --cluster <your EKS cluster name> --approve
```



**3. Create an IAM policy**

Create an IAM policy for the AWS Load Balancer Controller that allows it to make calls to AWS APIs on your behalf.

- Navigate to AWS IAM Console and in “**Policies**” section, click the “**Create Policy**” button
- Select “**JSON**”, paste contents from https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.3.0/docs/install/iam_policy.json as shown in the figure.
- Create a policy "AWSLoadBalancerControllerIAMPolicy"

 ![18_EKS_alb-iam-policy.png](https://box.indianic.biz/index.php/s/aTcsj9zcdmTKEtS/download?path=%2FAWS%2Fpompah%2FJenkins&files=18_EKS_alb-iam-policy.png)





**4. Create an IAM role**

Create an IAM role and annotate the Kubernetes service account that's named **aws-load-balancer-controller** in the **kube-system** namespace for the AWS Load Balancer Controller using eksctl or the AWS Management Console and kubectl. 

```
$ eksctl create iamserviceaccount \
  --cluster=<your EKS cluster name> \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --attach-policy-arn=arn:aws:iam::<AWS Account ID>:policy/AWSLoadBalancerControllerIAMPolicy \
  --override-existing-serviceaccounts \
  --approve
```

You can verify by executing following command.

```
$ eksctl get iamserviceaccount --cluster <your EKS cluster name>
```



**5. Create AWS Load Balancer Controller** 

Create ClusterRole, ClusterRoleBinding, and ServiceAccount for AWS Load Balancer Controller.

```
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-alb-ingress-controller/v1.1.8/docs/examples/rbac-role.yaml
```

 List service account created in above step.

```
$ kubectl get sa -n kube-system
```

Describe service account created in above step.

```
$ kubectl describe sa alb-ingress-controller -n kube-system
```



**6. Deploy the AWS Load Balancer Controller**

Download the alb-ingress-controller.yaml. uncomment --cluster-name as well as --aws-region lines and add the details.

```
$ wget https://raw.githubusercontent.com/kubernetes-sigs/aws-alb-ingress-controller/v1.1.8/docs/examples/alb-ingress-controller.yaml
```



![19_EKS-alb-depl.png](https://box.indianic.biz/index.php/s/aTcsj9zcdmTKEtS/download?path=%2FAWS%2Fpompah%2FJenkins&files=19_EKS-alb-depl.png)



Now, deploy the alb-controller.

```
$ kubectl apply -f alb-ingress-controller.yaml
```

Verify the deployment.

```
$ kubectl get deploy -n kube-system
```

Verify if alb-ingress-controller pod is running.

```
$ kubectl get pods -n kube-system
```

Check logs.

```
$ kubectl logs -f $(kubectl get po -n kube-system | egrep -o 'alb-ingress-controller-[A-Za-z0-9-]+') -n kube-system
```

Once the ALB controller is deployed. ALB for EKS can be created using "**kind: Ingress**" in deployment file. We see this in next section.



### Project repositories and EKS deployment YAML files : 

------

In this section, we understand project repositories structure, Kubernetes depolyment yamls, and Dockerfile.

There are three modules.

1. Admin - Python
2. API - Python (10 micro services)
3. Front - React



#### Admin :

Directory structure for admin.

![20_git-admin.png](https://box.indianic.biz/index.php/s/aTcsj9zcdmTKEtS/download?path=%2FGit-Repo%2Fpompah&files=20_git-admin.png)



Deployment file for the admin:

Create Kubernetes directory as shown in above figure and create **admin-depl.yaml** file inside it as given below.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: admin
spec:
  selector:
    matchLabels:
      app: admin
  replicas: 1
  template:
    metadata:
      labels:
        app: admin
    spec:
      containers:
      - name: admin
        image: 058279483918.dkr.ecr.ap-south-1.amazonaws.com/pompah-admin:$shortCommit
        ports:
        - containerPort: 8000
        imagePullPolicy: Always
---
apiVersion: v1
kind: Service
metadata:
  name: admin
  labels:
    app: admin
spec:
  ports:
  - name: https
    port: 8000
    protocol: TCP
    targetPort: 8000
  selector:
    app: admin
  type: NodePort
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: ingress-admin
  labels:
    app: admin
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/healthcheck-path: /
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:ap-south-1:058279483918:certificate/b917e83d-5832-4b5b-91e1-1d901de2248a
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}, {"HTTPS":443}]'
    alb.ingress.kubernetes.io/actions.ssl-redirect: '{"Type": "redirect", "RedirectConfig": { "Protocol": "HTTPS", "Port": "443", "StatusCode": "HTTP_301"}}'
spec:
  rules:
    - http:
        paths:
          - path: /*
            backend:
              serviceName: ssl-redirect
              servicePort: use-annotation
          - path: /*
            backend:
              serviceName: admin
              servicePort: 8000
```



This deployment file contains Deployment, Service, and Ingress (in our case it acts as ALB) for the admin.

In above yaml file, **<u>Ingress</u>** is very important because it creates Application Load Balancer for EKS and routing for the micro service/s. 



**<u>Importance of annotations :</u>**

Annotation is very important because it creates ALB. Let's understand it.

```yaml
annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/healthcheck-path: /
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:ap-south-1:058279483918:certificate/b917e83d-5832-4b5b-91e1-1d901de2248a
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}, {"HTTPS":443}]'
    alb.ingress.kubernetes.io/actions.ssl-redirect: '{"Type": "redirect", "RedirectConfig": { "Protocol": "HTTPS", "Port": "443", "StatusCode": "HTTP_301"}}'
```

First line "kubernetes.io/ingress.class: alb" indicates to **create ALB**.

Second line"alb.ingress.kubernetes.io/scheme: internet-facing" indicates to **have internet-facing ALB.**

Third line"alb.ingress.kubernetes.io/healthcheck-path: /" indicates **health check** path for ALB

Forth line "alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:ap-south-1:058279483918:certificate/b917e83d-5832-4b5b-91e1-1d901de2248a" indicates **domain certificate for ALB** .

Fifth line alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}, {"HTTPS":443}]' indicates to **create port 80 and port 443 of ALB.**

Sixth line alb.ingress.kubernetes.io/actions.ssl-redirect: '{"Type": "redirect", "RedirectConfig": { "Protocol": "HTTPS", "Port": "443", "StatusCode": "HTTP_301"}}' indicates **redirection rule from port 80 to port 443.**



**<u>Importance of rules :</u>**

Rules is very important because describes routing of ALB. Let's understand it.

```yaml
spec:
  rules:
    - http:
        paths:
          - path: /*
            backend:
              serviceName: ssl-redirect
              servicePort: use-annotation
          - path: /*
            backend:
              serviceName: admin
              servicePort: 8000
```



When you have redirection rule in the annotation like **Sixth line alb.ingress.kubernetes.io/actions.ssl-redirect: '{"Type": "redirect", "RedirectConfig": { "Protocol": "HTTPS", "Port": "443", "StatusCode": "HTTP_301"}}'** first routing rule must be like,

```yaml
- path: /*
            backend:
              serviceName: ssl-redirect
              servicePort: use-annotation
```

Subsequent routing rules can be added for the micro service/s as given below,

```yaml
- path: /*
            backend:
              serviceName: admin
              servicePort: 8000
```

 

Dockerfile:

This the dockerfile for the admin. **api (all micro services) followed same docker file as it is written in python.**

```dockerfile
FROM python:3.7-buster

ENV PYTHONUNBUFFERED 1

RUN apt-get update -y &&  apt-get install xvfb -y && apt-get install xfonts-100dpi xfonts-75dpi xfonts-scalable xfonts-cyrillic -y && apt-get install wkhtmltopdf -y

RUN mkdir /code

WORKDIR /code

COPY requirements.txt /code/

RUN pip install -r requirements.txt

COPY . /code/

#COPY ./config.ini.dev.example /code/config.ini

RUN python manage.py makemigrations logpipe

RUN python manage.py migrate

EXPOSE 8000

CMD python manage.py runserver 0.0.0.0:8000
```

Here "COPY ./config.ini.dev.example /code/config.ini" line is **commented out** as we keep configuration file inside the **jenkins** but not in the git repo.



#### API (micro services)

Directory structure for admin.

![21_git-api.png](https://box.indianic.biz/index.php/s/aTcsj9zcdmTKEtS/download?path=%2FGit-Repo%2Fpompah&files=21_git-api.png)





**Note:** <u>For API micro services (all), create directory as "Kubernetes" inside each micro services' directory. Inside "kubernetes" create deployment.yaml file. Below image shows the structure of **"artist"** micro service.</u>

![22_git-api-artist.png](https://box.indianic.biz/index.php/s/aTcsj9zcdmTKEtS/download?path=%2FGit-Repo%2Fpompah&files=22_git-api-artist.png)



Artist with Ingress Routing: (deployment.yaml)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: artist
spec:
  selector:
    matchLabels:
      app: artist
  replicas: 1
  template:
    metadata:
      labels:
        app: artist
    spec:
      containers:
      - name: artist
        image: 058279483918.dkr.ecr.ap-south-1.amazonaws.com/pompah-artist:$shortCommit
        ports:
        - containerPort: 8000
        imagePullPolicy: Always
---
apiVersion: v1
kind: Service
metadata:
  name: artist
  labels:
    app: artist
spec:
  ports:
  - name: https
    port: 8000
    protocol: TCP
    targetPort: 8000
  selector:
    app: artist
  type: NodePort
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: ingress-nginx
  labels:
    app: artist
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/healthcheck-path: /api/v1/healthcheck
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:ap-south-1:058279483918:certificate/b917e83d-5832-4b5b-91e1-1d901de2248a
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}, {"HTTPS":443}]'
    alb.ingress.kubernetes.io/actions.ssl-redirect: '{"Type": "redirect", "RedirectConfig": { "Protocol": "HTTPS", "Port": "443", "StatusCode": "HTTP_301"}}'
spec:
  rules:
    - http:
        paths:
          - path: /*
            backend:
              serviceName: ssl-redirect
              servicePort: use-annotation
          - path: /artist/*
            backend:
              serviceName: artist
              servicePort: 8000
          - path: /auth/*
            backend:
              serviceName: auth
              servicePort: 8000
          - path: /common/*
            backend:
              serviceName: common
              servicePort: 8000
          - path: /contest/*
            backend:
              serviceName: contest
              servicePort: 8000
          - path: /event/*
            backend:
              serviceName: event
              servicePort: 8000
          - path: /master/*
            backend:
              serviceName: master
              servicePort: 8000
          - path: /notification/*
            backend:
              serviceName: notification
              servicePort: 8000
          - path: /retrive/*
            backend:
              serviceName: retrive
              servicePort: 8000
          - path: /tutor/*
            backend:
              serviceName: tutor
              servicePort: 8000
          - path: /user/*
            backend:
              serviceName: user
              servicePort: 8000
```



**Note:** Follow same structure for remaining micro services like "artist" as explained above.

Auth: (deployment.yaml)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: auth
spec:
  selector:
    matchLabels:
      app: auth
  replicas: 1
  template:
    metadata:
      labels:
        app: auth
    spec:
      containers:
      - name: auth
        image: 058279483918.dkr.ecr.ap-south-1.amazonaws.com/pompah-auth:$shortCommit
        ports:
        - containerPort: 8000
        imagePullPolicy: Always
---
apiVersion: v1
kind: Service
metadata:
  name: auth
  labels:
    app: auth
spec:
  ports:
  - name: https
    port: 8000
    protocol: TCP
    targetPort: 8000
  selector:
    app: auth
  type: NodePort
```



Common: (deployment.yaml)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: common
spec:
  selector:
    matchLabels:
      app: common
  replicas: 1
  template:
    metadata:
      labels:
        app: common
    spec:
      containers:
      - name: common
        image: 058279483918.dkr.ecr.ap-south-1.amazonaws.com/pompah-common:$shortCommit
        ports:
        - containerPort: 8000
        imagePullPolicy: Always
---
apiVersion: v1
kind: Service
metadata:
  name: common
  labels:
    app: common
spec:
  ports:
  - name: https
    port: 8000
    protocol: TCP
    targetPort: 8000
  selector:
    app: common
  type: NodePort
```



Contest: (deployment.yaml)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: contest
spec:
  selector:
    matchLabels:
      app: contest
  replicas: 1
  template:
    metadata:
      labels:
        app: contest
    spec:
      containers:
      - name: contest
        image: 058279483918.dkr.ecr.ap-south-1.amazonaws.com/pompah-contest:$shortCommit
        ports:
        - containerPort: 8000
        imagePullPolicy: Always
---
apiVersion: v1
kind: Service
metadata:
  name: contest
  labels:
    app: contest
spec:
  ports:
  - name: https
    port: 8000
    protocol: TCP
    targetPort: 8000
  selector:
    app: contest
  type: NodePort
```



Event: (deployment.yaml)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: event
spec:
  selector:
    matchLabels:
      app: event
  replicas: 1
  template:
    metadata:
      labels:
        app: event
    spec:
      containers:
      - name: event
        image: 058279483918.dkr.ecr.ap-south-1.amazonaws.com/pompah-event:$shortCommit
        ports:
        - containerPort: 8000
        imagePullPolicy: Always
---
apiVersion: v1
kind: Service
metadata:
  name: event
  labels:
    app: event
spec:
  ports:
  - name: https
    port: 8000
    protocol: TCP
    targetPort: 8000
  selector:
    app: event
  type: NodePort
```



Master: (deployment.yaml)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: master
spec:
  selector:
    matchLabels:
      app: master
  replicas: 1
  template:
    metadata:
      labels:
        app: master
    spec:
      containers:
      - name: master
        image: 058279483918.dkr.ecr.ap-south-1.amazonaws.com/pompah-master:$shortCommit
        ports:
        - containerPort: 8000
        imagePullPolicy: Always
---
apiVersion: v1
kind: Service
metadata:
  name: master
  labels:
    app: master
spec:
  ports:
  - name: https
    port: 8000
    protocol: TCP
    targetPort: 8000
  selector:
    app: master
  type: NodePort
```



Notification: (deployment.yaml)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: notification
spec:
  selector:
    matchLabels:
      app: notification
  replicas: 1
  template:
    metadata:
      labels:
        app: notification
    spec:
      containers:
      - name: notification
        image: 058279483918.dkr.ecr.ap-south-1.amazonaws.com/pompah-notification:$shortCommit
        ports:
        - containerPort: 8000
        imagePullPolicy: Always
---
apiVersion: v1
kind: Service
metadata:
  name: notification
  labels:
    app: notification
spec:
  ports:
  - name: https
    port: 8000
    protocol: TCP
    targetPort: 8000
  selector:
    app: notification
  type: NodePort
```



Retrive: (deployment.yaml)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: retrive
spec:
  selector:
    matchLabels:
      app: retrive
  replicas: 1
  template:
    metadata:
      labels:
        app: retrive
    spec:
      containers:
      - name: retrive
        image: 058279483918.dkr.ecr.ap-south-1.amazonaws.com/pompah-retrive:$shortCommit
        ports:
        - containerPort: 8000
        imagePullPolicy: Always
---
apiVersion: v1
kind: Service
metadata:
  name: retrive
  labels:
    app: retrive
spec:
  ports:
  - name: https
    port: 8000
    protocol: TCP
    targetPort: 8000
  selector:
    app: retrive
  type: NodePort
```



Tutor: (deployment.yaml)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tutor
spec:
  selector:
    matchLabels:
      app: tutor
  replicas: 1
  template:
    metadata:
      labels:
        app: tutor
    spec:
      containers:
      - name: tutor
        image: 058279483918.dkr.ecr.ap-south-1.amazonaws.com/pompah-tutor:$shortCommit
        ports:
        - containerPort: 8000
        imagePullPolicy: Always
---
apiVersion: v1
kind: Service
metadata:
  name: tutor
  labels:
    app: tutor
spec:
  ports:
  - name: https
    port: 8000
    protocol: TCP
    targetPort: 8000
  selector:
    app: tutor
  type: NodePort
```



User: (deployment.yaml)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: user
spec:
  selector:
    matchLabels:
      app: user
  replicas: 1
  template:
    metadata:
      labels:
        app: user
    spec:
      containers:
      - name: user
        image: 058279483918.dkr.ecr.ap-south-1.amazonaws.com/pompah-user:$shortCommit
        ports:
        - containerPort: 8000
        imagePullPolicy: Always
---
apiVersion: v1
kind: Service
metadata:
  name: user
  labels:
    app: user
spec:
  ports:
  - name: https
    port: 8000
    protocol: TCP
    targetPort: 8000
  selector:
    app: user
  type: NodePort
```



#### Front:

Deployment file with Ingress (front-depl.yaml)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: front
spec:
  selector:
    matchLabels:
      app: front
  replicas: 1
  template:
    metadata:
      labels:
        app: front
    spec:
      containers:
      - name: front
        image: 058279483918.dkr.ecr.ap-south-1.amazonaws.com/pompah-front:$shortCommit
        ports:
        - containerPort: 5000
        imagePullPolicy: Always
---
apiVersion: v1
kind: Service
metadata:
  name: front
  labels:
    app: front
spec:
  ports:
  - name: https
    port: 5000
    protocol: TCP
    targetPort: 5000
  selector:
    app: front
  type: NodePort
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: ingress-front
  labels:
    app: front
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/healthcheck-path: /
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:ap-south-1:058279483918:certificate/b917e83d-5832-4b5b-91e1-1d901de2248a
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}, {"HTTPS":443}]'
    alb.ingress.kubernetes.io/actions.ssl-redirect: '{"Type": "redirect", "RedirectConfig": { "Protocol": "HTTPS", "Port": "443", "StatusCode": "HTTP_301"}}'
spec:
  rules:
    - http:
        paths:
          - path: /*
            backend:
              serviceName: ssl-redirect
              servicePort: use-annotation
          - path: /*
            backend:
              serviceName: front
              servicePort: 5000
```



Dockerfile:

```dockerfile
FROM docker.indianic.com/library/node:14.16.1

WORKDIR /usr/src

ENV PATH /usr/src/node_modules/.bin:$PATH

COPY package.json  ./package.json

RUN yarn install

COPY . ./

EXPOSE 5000

RUN yarn build:prod

CMD ["yarn","start:prod"]
```



### Jenkins pipelines for the micro services deployment :

------

last thing we need to do is to setup jenkins pipelines for the micro services. 

**Steps to setup pipeline:**

Install **Config File Provider** and **CloudBees AWS Credentials** plugins.



Install Config File Provider. This plugin provides a way to store and oragnize configuration file/s within jenkins.

![jenkins-conf-plugin](/home/indianic/Pompah_Keys/jenkins-conf-plugin.png)



Install CloudBees AWS Credentials. This plugin provides a way to store AWS credentials (Access Key and Secret Access Key).

![jenkins-aws-plugin](/home/indianic/Pompah_Keys/jenkins-aws-plugin.png)





**Admin Jenkins Pipeline:**

```
node() {
    cleanWs();

    stage("Checkout") {
        checkout([$class: 'GitSCM', branches: [[name: '*/master']], doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [], userRemoteConfigs: [[credentialsId: 'Git-credentials', url: 'http://git.indianic.com/FAF/F2021-6073/python-admin2.git']]])
    }
     
    configFileProvider([configFile(fileId: 'admin.config.ini', targetLocation: 'config.ini')]) {     
    stage("Build Docker Image") {
            env.shortCommit = sh(returnStdout: true, script: "git log -n 1 --pretty=format:'%h'").trim()
            // echo "${shortCommit}"
            sh "docker build -t 058279483918.dkr.ecr.ap-south-1.amazonaws.com/pompah-admin:${shortCommit} ."
    }
    }
     
    stage('Push Docker Image') {
        withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', accessKeyVariable: 'AWS_ACCESS_KEY_ID', credentialsId: 'pompah-user-aws', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']]) {
            sh "aws ecr get-login-password --region ap-south-1 | docker login --username AWS --password-stdin 058279483918.dkr.ecr.ap-south-1.amazonaws.com" 
            sh "docker push 058279483918.dkr.ecr.ap-south-1.amazonaws.com/pompah-admin:${shortCommit}"
        }
    }
    
    stage('Deploy to EKS') {
            //withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', accessKeyVariable: 'AWS_ACCESS_KEY_ID', credentialsId: 'pompah-user-aws', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']]) {
                    //sh "kubectl apply -f Kubernetes/admin-depl.yaml"
                    //sh "kubectl set image deployment/admin admin=058279483918.dkr.ecr.ap-south-1.amazonaws.com/pompah-admin:${shortCommit} --record"
            
                    sh "envsubst < Kubernetes/admin-depl.yaml | kubectl apply -f -"
            //}
    }    
    
}
```



**API Jenkins Pipeline:**

```
node() {
    cleanWs();

    stage("Checkout") {
        checkout([$class: 'GitSCM', branches: [[name: '*/master']], doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [], userRemoteConfigs: [[credentialsId: 'Git-credentials', url: 'http://git.indianic.com/FAF/F2021-6073/python-api.git']]])
    }
    
    stage('Input selectopn') {
        def userInput = input(
            id: 'userInput', message: 'Select microservice to apply change', 
            parameters: [
                            [$class: 'ChoiceParameterDefinition', defaultValue: 'None', description:'', name:'API microservices', choices: "artist\nauth\ncommon\ncontest\nevent\nmaster\nnotification\nretrive\ntutor\nuser"]
            ]
        )
        echo ("Image tags: "+userInput)
        env.Select = userInput
    } 

    configFileProvider([configFile(fileId: "${env.Select}.config.ini", targetLocation: "./${env.Select}/config.ini")]) {     
    stage("Build Docker Image") {
            env.shortCommit = sh(returnStdout: true, script: "git log -n 1 --pretty=format:'%h'").trim()
            // echo "${shortCommit}"
            sh "docker build -t 058279483918.dkr.ecr.ap-south-1.amazonaws.com/pompah-${env.Select}:${shortCommit} ./${env.Select}/"
    }
    }
    
    stage('Push Docker Image') {
        withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', accessKeyVariable: 'AWS_ACCESS_KEY_ID', credentialsId: 'pompah-user-aws', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']]) {
            sh "aws ecr get-login-password --region ap-south-1 | docker login --username AWS --password-stdin 058279483918.dkr.ecr.ap-south-1.amazonaws.com" 
            sh "docker push 058279483918.dkr.ecr.ap-south-1.amazonaws.com/pompah-${env.Select}:${shortCommit}"
        }
    }
    
    stage('Deploy to EKS') {
            //withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', accessKeyVariable: 'AWS_ACCESS_KEY_ID', credentialsId: 'pompah-user-aws', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']]) {
                    //sh "kubectl apply -f Kubernetes/admin-depl.yaml"
                    //sh "kubectl set image deployment/admin admin=058279483918.dkr.ecr.ap-south-1.amazonaws.com/pompah-admin:${shortCommit} --record"
            
                    sh "envsubst < ${env.Select}/Kubernetes/deployment.yaml | kubectl apply -f -"
            //}
    }
}
```



**Front Jenkins Pipeline:**

```
node() {
    cleanWs();

    stage("Checkout") {
        checkout([$class: 'GitSCM', branches: [[name: '*/master']], doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [], userRemoteConfigs: [[credentialsId: 'Git-credentials', url: 'http://git.indianic.com/FAF/F2021-6073/reactjs-front4.git']]])
    }
     
    configFileProvider([configFile(fileId: 'admin.config.ini', targetLocation: 'config.ini')]) {     
    stage("Build Docker Image") {
            env.shortCommit = sh(returnStdout: true, script: "git log -n 1 --pretty=format:'%h'").trim()
            // echo "${shortCommit}"
            sh "docker build -t 058279483918.dkr.ecr.ap-south-1.amazonaws.com/pompah-front:${shortCommit} ."
    }
    }
    
    stage('Push Docker Image') {
        withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', accessKeyVariable: 'AWS_ACCESS_KEY_ID', credentialsId: 'pompah-user-aws', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']]) {
            sh "aws ecr get-login-password --region ap-south-1 | docker login --username AWS --password-stdin 058279483918.dkr.ecr.ap-south-1.amazonaws.com" 
            sh "docker push 058279483918.dkr.ecr.ap-south-1.amazonaws.com/pompah-front:${shortCommit}"
        }
    }
    
    stage('Deploy to EKS') {
           // withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', accessKeyVariable: 'AWS_ACCESS_KEY_ID', credentialsId: 'pompah-user-aws', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']]) {
                    //sh "kubectl apply -f Kubernetes/front-depl.yaml"
                    //sh "kubectl set image deployment/front front=058279483918.dkr.ecr.ap-south-1.amazonaws.com/pompah-front:${shortCommit} --record"
                    sh "envsubst < Kubernetes/front-depl.yaml | kubectl apply -f -"
           // }
    } 
}
```



###                                                                        **<u>*Thanks for the reading*</u>**.


# AWS Terraform Modules 使用方法

Terraform的功能非常丰富，可以通过Providers来提供对多平台的支持，通过Provisioners来实现本地与远程的脚本调用等功能，支持ssh与winrm的连接方式，也能作为Chef Client的方式运行，通过Modules去重用组件提高开发效率，大数AWS资源都能通过官方托管的Module Sources来重用。

## 部署架构

<img src="https://i.loli.net/2021/08/24/VqzDuRyQtBOSJ8P.png" width=600 />

- 创建一个三层网络架构，服务器只能通过跳板机连接；
- web 服务器只能由跳板机连接，80 端口只能由 ELB 访问，服务器不分配公网IP，外网连接通过 NAT；
- 数据库服务器只能由 web 服务器连接 3306 端口；
- 服务器分布在多 AZ。

## 编写TF Code

### 创建项目目录

新建一个AWS-TF文件夹，将下面创建的4个文件放到同一目录。

参考链接：
<https://registry.terraform.io/providers/hashicorp/aws/latest>

### 编写variable.tf文件

```json
variable "aws_access_key" {
  
}
variable "aws_secret_key" {
  
}
variable "aws_region" {
  
}
variable "instance_ami" {
  
}
variable "instance_type" {
  
}
variable "tags" {
  
}
variable "aws_key_pair" {
  
}
```

### 编写terraform.tfvars文件

```json
aws_access_key = "access_key"
aws_secret_key = "secret_key"
aws_region = "cn-north-1"
instance_ami = "ami-fba67596"
instance_type = "t2.micro"
tags = {
    Owner = "demouser"
    Enviroment = "Dev"
    Name = "demo-terraform"
}
aws_key_pair = "demo-key-pair"
```

### 编写main.tf文件

```json
provider "aws" {
  access_key = var.aws_access_key
  secret_key = var.aws_secret_key
  region = var.aws_region
}

##################################################
### VPC Module
##################################################
module "vpc" {
  source = "terraform-aws-modules/vpc/aws"
  name = "my-demo-vpc" 
  cidr = "172.18.0.0/16"
  azs = [ "${var.aws_region}a" , "${var.aws_region}b" ]
  public_subnets = [ "172.18.0.0/24", "172.18.1.0/24" ]
  private_subnets = [ "172.18.2.0/24", "172.18.3.0/24" ]
  database_subnets = [ "172.18.4.0/24", "172.18.5.0/24" ]
  create_database_subnet_group = true
  enable_dns_hostnames = true
  enable_dns_support = true
  enable_nat_gateway = true
  single_nat_gateway = true
  one_nat_gateway_per_az = false
  tags = var.tags
}

# web security group

module "websg" {
  source = "terraform-aws-modules/security-group/aws"
  name = "web-service-sg"
  description = "Security group for HTTP and SSH within VPC"
  vpc_id = module.vpc.vpc_id
  ingress_rules = [ "http-80-tcp", "https-443-tcp", "ssh-tcp", "all-icmp" ]
  ingress_cidr_blocks = [ "0.0.0.0/0" ]
  egress_rules = [ "all-all" ]
  egress_cidr_blocks = [ "0.0.0.0/0" ]
}

# DB Security Group

module "dbsg" {
  source = "terraform-aws-modules/security-group/aws"
  name = "DB-Security-sg"
  description = "Security group for database within VPC"
  vpc_id = module.vpc.vpc_id
  ingress_with_source_security_group_id = [ {
    "rule" = "mysql-tcp"
    source_security_group_id = module.websg.security_group_id
  } ]
  egress_with_source_security_group_id = [ {
    "rule" = "all-all"
    source_security_group_id = module.websg.security_group_id
  } ]
}

# bastion host module

module "bastion" {
  source = "terraform-aws-modules/ec2-instance/aws"
  name = "my-demo-bastion-host"
  ami = var.instance_ami
  instance_type = var.instance_type
  vpc_security_group_ids = [ module.websg.security_group_id ]
  subnet_id = module.vpc.public_subnets[1]
  associate_public_ip_address = true
  key_name = var.aws_key_pair
  tags = var.tags
}

# web server module
module "web_server_1a" {
  source = "terraform-aws-modules/ec2-instance/aws"
  name = "my-demo-web-server-1a"
  instance_count = 1
  ami = var.instance_ami
  instance_type = var.instance_type
  vpc_security_group_ids = [ module.websg.security_group_id ]
  subnet_id = module.vpc.private_subnets[0]
  key_name = var.aws_key_pair
  tags = var.tags
  user_data = <<-EOF
    #!/bin/bash
    yum install httpd
    systemctl enable httpd
    systemctl start httpd
  EOF
}
module "web_server_1b" {
  source = "terraform-aws-modules/ec2-instance/aws"
  name = "my-demo-web-server-1b"
  ami = var.instance_ami
  instance_count = 1
  instance_type = var.instance_type
  vpc_security_group_ids = [ module.websg.security_group_id ]
  subnet_id = module.vpc.private_subnets[1]
  key_name = var.aws_key_pair
  tags = var.tags
  user_data = <<-EOF
    #!/bin/bash
    sudo yum install -y update
    sudo yum install -y httpd
    services restart httpd
  EOF
}

# module  alb

module "aws_alb" {
  source = "terraform-aws-modules/alb/aws"
  name = "my-demo-alb"
  load_balancer_type = "application"
  vpc_id = module.vpc.vpc_id
  subnets = module.vpc.private_subnets
  security_groups = [ module.websg.security_group_id ]
  http_tcp_listeners = [
    {
      port = 80
      protocol = "HTTP"
    }
  ]
  target_groups = [
    {
      name_prefix = "pref-"
      backend_protocol = "HTTP"
      backend_port = 80
      target_type = "instance"
      targets = {
        ec1 = {
          target_id = "${module.web_server_1a.id[0]}"
          port = 80
        },
        ec2 = {
          target_id = "${module.web_server_1b.id[0]}"
          port = 80
        }
      }
    }
    
  ]

  tags = var.tags
}

# module RDS

module "mysql01" {
  source = "terraform-aws-modules/rds/aws"
  identifier = "mysql01"
  name = "mysql01"
  engine = "mysql"
  engine_version = "8.0.23"
  instance_class = "db.t3.micro"
  allocated_storage = 20
  storage_type = "standard"
  username = "myadmin"
  password = "Password01!"
  port = "3306"
  multi_az = true
  vpc_security_group_ids = [ "${module.dbsg.security_group_id}" ]
  maintenance_window = "Mon:00:00-Mon:03:00"
  backup_window = "03:00-06:00"
  subnet_ids = [ "${module.vpc.database_subnets[0]}", "${module.vpc.database_subnets[1]}"]
  family = "mysql8.0"
  major_engine_version = "8.0"
  backup_retention_period = 0
  publicly_accessible = false
  parameters = [ {
    "name" = "character_set_client"
    "value" = "utf8"
  },
  {
    "name" = "character_set_server"
    "value" = "utf8"
  }
   ]
}
```

### 部署运行

#### 初始化

运行terraform init进行初始化，等待插件与Module自动下载

#### 查看计划

运行terraform plan执行计划，暂时忽略这个报错，有可能是安全组相互引用的问题，在后面分段运行即可。

#### 分块运行

由于资源相互引用，请按下面的顺序执行，也可以放一个shell脚本里面。运行过程有可能遇到时间过长，在控制台上看到资源都已经建好，请耐心等待。

terraform.exe apply -target module.vpc

### 资源回收

terraform destroy

# 使用 AWS CLI 创建 VPC

## 创建 VPC

1. 用下列 create-vpc 命令创建含 10.10.0.0/16 CIDR 块的 VPC

    ```shell
    aws ec2 create-vpc --cidr-block 10.10.0.0/16 --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=prd-test-vpc},{Key=createby,Value=bx}]' --query Vpc.VpcId --output text
    ```

2. 通过以下 create-subnet 命令，使用上一步中的 VPC ID 创建具有 10.10.1.0/24 CIDR 块的子网。

    ```shell
    aws ec2 create-subnet --vpc-id vpc-03b9fb89829ea1b1d --cidr-block 10.10.1.0/24 --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=prd-test-subnet-pri-1a},{Key=createby,Value=bx}]' --availability-zone ap-northeast-1a
    ```

根据tag搜索资源,并输出subnetid

```shell
aws ec2 describe-subnets --filters "Name=tag:Name,Values=prd-test-vpc" --query Subnets[*].SubnetId --output text
```

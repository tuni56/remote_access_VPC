# VPC with Private Access to S3 using AWS CLI

This repository provides step-by-step instructions to set up an AWS Virtual Private Cloud (VPC) with private access to Amazon S3 using a VPC Endpoint, configured entirely via the AWS Command Line Interface (CLI).

## Overview

The goal of this project is to create a secure architecture that enables resources in a private subnet to access Amazon S3 without exposing them to the internet. This is achieved using a **VPC Endpoint** for S3, avoiding the need for an Internet Gateway.

## Architecture Diagram

![Architecture Diagram]https://github.com/tuni56/remote_access_VPC/blob/main/DIAGRAM/aws_vpc_diagram.png


## Prerequisites

- **AWS CLI** installed and configured with proper IAM credentials (`aws configure`).
- IAM permissions to create and manage VPC resources, EC2 instances, and VPC endpoints.
- A terminal or shell to execute AWS CLI commands.

## Steps

### 1. Create a VPC

```bash
aws ec2 create-vpc --cidr-block 10.0.0.0/16 --tag-specifications \
'ResourceType=vpc,Tags=[{Key=Name,Value=MyPrivateVPC}]'
```

Take note of the `VpcId` from the output.

### 2. Create Subnets

- **Public Subnet**:

```bash
aws ec2 create-subnet --vpc-id <VpcId> --cidr-block 10.0.1.0/24 \
--availability-zone us-east-1a --tag-specifications \
'ResourceType=subnet,Tags=[{Key=Name,Value=PublicSubnet}]'
```

- **Private Subnet**:

```bash
aws ec2 create-subnet --vpc-id <VpcId> --cidr-block 10.0.2.0/24 \
--availability-zone us-east-1b --tag-specifications \
'ResourceType=subnet,Tags=[{Key=Name,Value=PrivateSubnet}]'
```

### 3. Create and Attach an Internet Gateway

```bash
aws ec2 create-internet-gateway --tag-specifications \
'ResourceType=internet-gateway,Tags=[{Key=Name,Value=MyIGW}]'
```

Attach the Internet Gateway:

```bash
aws ec2 attach-internet-gateway --vpc-id <VpcId> --internet-gateway-id <InternetGatewayId>
```

### 4. Create Route Tables

- **Public Route Table**:

```bash
aws ec2 create-route-table --vpc-id <VpcId> --tag-specifications \
'ResourceType=route-table,Tags=[{Key=Name,Value=PublicRouteTable}]'
```

Add a route for internet traffic:

```bash
aws ec2 create-route --route-table-id <RouteTableId> --destination-cidr-block 0.0.0.0/0 \
--gateway-id <InternetGatewayId>
```

Associate it with the public subnet:

```bash
aws ec2 associate-route-table --route-table-id <RouteTableId> --subnet-id <PublicSubnetId>
```

- **Private Route Table**:

```bash
aws ec2 create-route-table --vpc-id <VpcId> --tag-specifications \
'ResourceType=route-table,Tags=[{Key=Name,Value=PrivateRouteTable}]'
```

Associate it with the private subnet:

```bash
aws ec2 associate-route-table --route-table-id <PrivateRouteTableId> --subnet-id <PrivateSubnetId>
```

### 5. Create a VPC Endpoint for S3

```bash
aws ec2 create-vpc-endpoint --vpc-id <VpcId> \
--service-name com.amazonaws.us-east-1.s3 \
--vpc-endpoint-type Gateway \
--route-table-ids <PrivateRouteTableId> \
--tag-specifications 'ResourceType=vpc-endpoint,Tags=[{Key=Name,Value=S3Endpoint}]'
```

### 6. Launch an EC2 Instance in the Private Subnet

#### 1. Create a Key Pair

```bash
aws ec2 create-key-pair --key-name MyKeyPair --query 'KeyMaterial' --output text > MyKeyPair.pem
chmod 400 MyKeyPair.pem
```

#### 2. Create a Security Group

```bash
aws ec2 create-security-group --group-name MyPrivateSG --description "Private Subnet SG" \
--vpc-id <VpcId>
```

Allow SSH traffic (if using a bastion host):

```bash
aws ec2 authorize-security-group-ingress --group-id <SecurityGroupId> \
--protocol tcp --port 22 --cidr <BastionHostCIDR>
```

#### 3. Launch the EC2 Instance

```bash
aws ec2 run-instances --image-id ami-0c02fb55956c7d316 \
--instance-type t2.micro --key-name MyKeyPair \
--security-group-ids <SecurityGroupId> --subnet-id <PrivateSubnetId> \
--tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=PrivateEC2}]'
```

### 7. Test S3 Access

1. SSH into the EC2 instance (via a bastion host or AWS Session Manager).
2. Verify access to S3:

```bash
aws s3 ls
```

## Notes

- This setup ensures that private resources can access S3 securely without needing public IPs or an Internet Gateway.
- Adjust `Region` and `CIDR blocks` according to your environment.

## License

This project is licensed under the MIT License.



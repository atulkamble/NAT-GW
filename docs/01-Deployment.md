# Deployment Guide

[← Home](../README.md)

## 2. AWS CLI Commands

Run in the **same local terminal** to retain shell variables. Stop on any failed command instead of continuing with empty IDs.

### Step 1 – Configure environment

```
export AWS_DEFAULT_REGION=us-east-1
export AWS_PAGER=""
aws --version
export AZ=us-east-1a
export MY_IP="$(curl -fsS -4 https://checkip.amazonaws.com)/32"

aws sts get-caller-identity
```

### Step 2 – Create VPC

```
VPC_ID=$(aws ec2 create-vpc \
  --cidr-block 10.0.0.0/16 \
  --query 'Vpc.VpcId' --output text)

aws ec2 wait vpc-available --vpc-ids "$VPC_ID"
aws ec2 modify-vpc-attribute \
  --vpc-id "$VPC_ID" --enable-dns-support
aws ec2 modify-vpc-attribute \
  --vpc-id "$VPC_ID" --enable-dns-hostnames
```

### Step 3 – Create subnets

```
PUBLIC_SUBNET=$(aws ec2 create-subnet \
  --vpc-id "$VPC_ID" \
  --cidr-block 10.0.1.0/24 \
  --availability-zone "$AZ" \
  --query 'Subnet.SubnetId' --output text)

PRIVATE_SUBNET=$(aws ec2 create-subnet \
  --vpc-id "$VPC_ID" \
  --cidr-block 10.0.2.0/24 \
  --availability-zone "$AZ" \
  --query 'Subnet.SubnetId' --output text)

aws ec2 modify-subnet-attribute \
  --subnet-id "$PUBLIC_SUBNET" --map-public-ip-on-launch
aws ec2 modify-subnet-attribute \
  --subnet-id "$PRIVATE_SUBNET" --no-map-public-ip-on-launch
```

### Step 4 – Create Internet Gateway

```
IGW_ID=$(aws ec2 create-internet-gateway \
  --query 'InternetGateway.InternetGatewayId' \
  --output text)

aws ec2 attach-internet-gateway \
  --internet-gateway-id "$IGW_ID" \
  --vpc-id "$VPC_ID"
```

### Step 5 – Configure public route table

```
PUBLIC_RT=$(aws ec2 create-route-table \
  --vpc-id "$VPC_ID" \
  --query 'RouteTable.RouteTableId' --output text)

aws ec2 create-route \
  --route-table-id "$PUBLIC_RT" \
  --destination-cidr-block 0.0.0.0/0 \
  --gateway-id "$IGW_ID"

PUBLIC_ASSOC=$(aws ec2 associate-route-table \
  --subnet-id "$PUBLIC_SUBNET" \
  --route-table-id "$PUBLIC_RT" \
  --query AssociationId --output text)
```

### Step 6 – Create NAT Gateway

```
EIP_ID=$(aws ec2 allocate-address \
  --domain vpc \
  --query AllocationId --output text)

NAT_ID=$(aws ec2 create-nat-gateway \
  --subnet-id "$PUBLIC_SUBNET" \
  --allocation-id "$EIP_ID" \
  --query 'NatGateway.NatGatewayId' --output text)

aws ec2 wait nat-gateway-available \
  --nat-gateway-ids "$NAT_ID"

aws ec2 describe-nat-gateways --nat-gateway-ids "$NAT_ID" \
  --query 'NatGateways[0].[NatGatewayId,State,SubnetId]' --output table
```

### Step 7 – Configure private route table

```
PRIVATE_RT=$(aws ec2 create-route-table \
  --vpc-id "$VPC_ID" \
  --query 'RouteTable.RouteTableId' --output text)

aws ec2 create-route \
  --route-table-id "$PRIVATE_RT" \
  --destination-cidr-block 0.0.0.0/0 \
  --nat-gateway-id "$NAT_ID"

PRIVATE_ASSOC=$(aws ec2 associate-route-table \
  --subnet-id "$PRIVATE_SUBNET" \
  --route-table-id "$PRIVATE_RT" \
  --query AssociationId --output text)
```

### Step 8 – Create key pair and security groups

```
KEY_NAME="nat-key-$(date +%s)"

aws ec2 create-key-pair \
  --key-name "$KEY_NAME" \
  --query KeyMaterial --output text > "$KEY_NAME.pem"

chmod 400 "$KEY_NAME.pem"

PUBLIC_SG=$(aws ec2 create-security-group \
  --group-name nat-public-sg \
  --description "Public SSH" \
  --vpc-id "$VPC_ID" \
  --query GroupId --output text)

PRIVATE_SG=$(aws ec2 create-security-group \
  --group-name nat-private-sg \
  --description "Private SSH" \
  --vpc-id "$VPC_ID" \
  --query GroupId --output text)

aws ec2 authorize-security-group-ingress \
  --group-id "$PUBLIC_SG" \
  --protocol tcp --port 22 --cidr "$MY_IP"

aws ec2 authorize-security-group-ingress \
  --group-id "$PRIVATE_SG" \
  --protocol tcp --port 22 \
  --source-group "$PUBLIC_SG"
```

### Step 9 – Launch EC2 instances

```
AMI=$(aws ssm get-parameter \
  --name /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64 \
  --query Parameter.Value --output text)

PUBLIC_EC2=$(aws ec2 run-instances \
  --image-id "$AMI" \
  --instance-type t3.micro \
  --key-name "$KEY_NAME" \
  --subnet-id "$PUBLIC_SUBNET" \
  --security-group-ids "$PUBLIC_SG" \
  --associate-public-ip-address \
  --metadata-options HttpTokens=required,HttpEndpoint=enabled \
  --query 'Instances[0].InstanceId' --output text)

PRIVATE_EC2=$(aws ec2 run-instances \
  --image-id "$AMI" \
  --instance-type t3.micro \
  --key-name "$KEY_NAME" \
  --network-interfaces \
  "DeviceIndex=0,SubnetId=$PRIVATE_SUBNET,Groups=[$PRIVATE_SG],AssociatePublicIpAddress=false" \
  --metadata-options HttpTokens=required,HttpEndpoint=enabled \
  --query 'Instances[0].InstanceId' --output text)

aws ec2 wait instance-running \
  --instance-ids "$PUBLIC_EC2" "$PRIVATE_EC2"
```

### Step 10 – Get EC2 IP addresses

```
PUBLIC_IP=$(aws ec2 describe-instances \
  --instance-ids "$PUBLIC_EC2" \
  --query 'Reservations[0].Instances[0].PublicIpAddress' \
  --output text)

PRIVATE_IP=$(aws ec2 describe-instances \
  --instance-ids "$PRIVATE_EC2" \
  --query 'Reservations[0].Instances[0].PrivateIpAddress' \
  --output text)

echo "Public IP: $PUBLIC_IP"
echo "Private IP: $PRIVATE_IP"
```


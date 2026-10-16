# AWS NAT Gateway Project – CLI Practical

Objective: Allow a private EC2 instance to access the internet using NAT Gateway.

Region: `us-east-1` VPC: `10.0.0.0/16` Public Subnet: `10.0.1.0/24` Private Subnet: `10.0.2.0/24`

## 1. Architecture

```
Internet
   |
Internet Gateway
   |
Public Subnet (10.0.1.0/24)
   |-- Public EC2 (Bastion)
   |-- NAT Gateway (Elastic IP)
             |
        Private Route
             |
Private Subnet (10.0.2.0/24)
   |
Private EC2
   |
Outbound Internet via NAT
```

## 2. AWS CLI Commands

### Step 1 – Configure environment

```
export AWS_DEFAULT_REGION=us-east-1
export AZ=us-east-1a
export MY_IP="$(curl -fsS -4 https://checkip.amazonaws.com)/32"

aws sts get-caller-identity
```

### Step 2 – Create VPC

```
VPC_ID=$(aws ec2 create-vpc \
  --cidr-block 10.0.0.0/16 \
  --query 'Vpc.VpcId' --output text)

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
  --query 'Instances[0].InstanceId' --output text)

PRIVATE_EC2=$(aws ec2 run-instances \
  --image-id "$AMI" \
  --instance-type t3.micro \
  --key-name "$KEY_NAME" \
  --network-interfaces \
  "DeviceIndex=0,SubnetId=$PRIVATE_SUBNET,Groups=[$PRIVATE_SG],AssociatePublicIpAddress=false" \
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

## 3. Connect and test

Connect directly to private EC2 through the bastion from your laptop:

```
ssh -i "./$KEY_NAME.pem" \
  -J "ec2-user@$PUBLIC_IP" \
  "ec2-user@$PRIVATE_IP"
```

Run inside private EC2:

```
hostname -I
curl -4 -I https://www.google.com
curl -4 https://checkip.amazonaws.com
sudo dnf update -y
```

Verify NAT Elastic IP from your laptop:

```
aws ec2 describe-addresses \
  --allocation-ids "$EIP_ID" \
  --query 'Addresses[0].PublicIp' \
  --output text
```

The public IP returned from private EC2 should match the NAT Elastic IP.

## 4. Cleanup commands

Run from your original local terminal.

```
# Terminate instances
aws ec2 terminate-instances \
  --instance-ids "$PUBLIC_EC2" "$PRIVATE_EC2"

aws ec2 wait instance-terminated \
  --instance-ids "$PUBLIC_EC2" "$PRIVATE_EC2"

# Delete NAT
aws ec2 delete-nat-gateway --nat-gateway-id "$NAT_ID"
aws ec2 wait nat-gateway-deleted --nat-gateway-ids "$NAT_ID"

# Release Elastic IP
aws ec2 release-address --allocation-id "$EIP_ID"

# Remove route table associations
aws ec2 disassociate-route-table --association-id "$PUBLIC_ASSOC"
aws ec2 disassociate-route-table --association-id "$PRIVATE_ASSOC"

# Delete route tables
aws ec2 delete-route-table --route-table-id "$PUBLIC_RT"
aws ec2 delete-route-table --route-table-id "$PRIVATE_RT"

# Delete security groups
aws ec2 revoke-security-group-ingress \
  --group-id "$PRIVATE_SG" \
  --protocol tcp --port 22 \
  --source-group "$PUBLIC_SG"

aws ec2 delete-security-group --group-id "$PRIVATE_SG"
aws ec2 delete-security-group --group-id "$PUBLIC_SG"

# Delete subnets
aws ec2 delete-subnet --subnet-id "$PUBLIC_SUBNET"
aws ec2 delete-subnet --subnet-id "$PRIVATE_SUBNET"

# Delete Internet Gateway
aws ec2 detach-internet-gateway \
  --internet-gateway-id "$IGW_ID" --vpc-id "$VPC_ID"

aws ec2 delete-internet-gateway --internet-gateway-id "$IGW_ID"

# Delete VPC
aws ec2 delete-vpc --vpc-id "$VPC_ID"

# Delete key pair
aws ec2 delete-key-pair --key-name "$KEY_NAME"
rm -f "./$KEY_NAME.pem"
```

## 5. Points to remember

- Public subnet → Internet Gateway.
- Private subnet → NAT Gateway.
- NAT Gateway → Public subnet + Elastic IP.
- Private EC2 → No public IP.
- Bastion EC2 → SSH access to private EC2.
- NAT Gateway → Outbound IPv4 connectivity, not unsolicited inbound access.
- NAT Gateway and public IPv4 addresses incur charges until released or deleted.

Expected result: Private EC2 successfully accesses the internet through NAT Gateway without having a public IP address.

# AWS NAT Gateway Project – Complete Updated Guide with AWS CLI

Practical Hands-on Project

Project Name: AWS VPC Public and Private Subnet Networking with NAT Gateway

Difficulty: Beginner to Intermediate Cloud Provider: Amazon Web Services (AWS) Region: `us-east-1` Tools: AWS CLI v2, SSH, EC2, VPC, NAT Gateway Operating System: Amazon Linux 2023 Deployment Method: AWS CLI (Bash/Zsh on macOS or Linux)

## 1. Project overview

### Objective

Design and deploy a custom AWS VPC where a public EC2 instance acts as a bastion host and a private EC2 instance accesses the internet securely through a public NAT Gateway.

### Real-world use case

Organizations deploy backend applications, internal services, and private compute workloads without public IP addresses. These workloads may still need outbound internet access for software updates, package installation, and external APIs.

### AWS services used

| Service          | Purpose                                               |
| ---------------- | ----------------------------------------------------- |
| VPC              | Isolated virtual network                              |
| Public subnet    | Hosts bastion EC2 and NAT Gateway                     |
| Private subnet   | Hosts private EC2 workload                            |
| Internet Gateway | Connects VPC public resources to internet             |
| NAT Gateway      | Provides outbound IPv4 internet access to private EC2 |
| Elastic IP       | Static public IPv4 address for NAT                    |
| Route Tables     | Control subnet traffic routing                        |
| Security Groups  | Control instance inbound and outbound traffic         |
| EC2              | Public and private Linux servers                      |
| IAM              | Controls AWS API access                               |

## 2. Architecture

Outbound flow: Private EC2 → NAT Gateway → Internet Gateway → Internet. SSH administration: Laptop → Public EC2 → Private EC2.

## 3. Network configuration

| Resource            | Name             | Configuration     |
| ------------------- | ---------------- | ----------------- |
| VPC                 | `nat-vpc`        | `10.0.0.0/16`     |
| Public subnet       | `public-subnet`  | `10.0.1.0/24`     |
| Private subnet      | `private-subnet` | `10.0.2.0/24`     |
| Availability Zone   | —                | `us-east-1a`      |
| Internet Gateway    | `nat-igw`        | Attached to VPC   |
| NAT Gateway         | `nat-gateway`    | Public subnet     |
| Public route table  | `public-rt`      | `0.0.0.0/0 → IGW` |
| Private route table | `private-rt`     | `0.0.0.0/0 → NAT` |
| Public EC2          | `ec2-public`     | `t3.micro`        |
| Private EC2         | `ec2-private`    | `t3.micro`        |

## 4. Prerequisites

Verify AWS CLI installation, credentials, and identity:

```
aws --version
aws configure
aws sts get-caller-identity
```

Required IAM permissions include EC2/VPC networking operations, EC2 instance management, and SSM Parameter Store read access for the AMI lookup.

The following implementation creates a fresh environment rather than relying on old resource IDs.

## 5. Complete AWS CLI deployment

Run these commands in sequence from your local macOS/Linux terminal. Variables capture AWS-generated resource IDs automatically.

### Step 1 – Configure project variables

```
export AWS_DEFAULT_REGION=us-east-1
export AWS_PAGER=""
export AZ=us-east-1a

export VPC_CIDR=10.0.0.0/16
export PUBLIC_CIDR=10.0.1.0/24
export PRIVATE_CIDR=10.0.2.0/24

# Your current public IPv4 for SSH access
export MY_IP="$(curl -fsS -4 https://checkip.amazonaws.com)/32"

echo "AWS Region: $AWS_DEFAULT_REGION"
echo "Availability Zone: $AZ"
echo "SSH allowed from: $MY_IP"
```

### Step 2 – Create VPC

```
VPC_ID=$(aws ec2 create-vpc \
  --cidr-block "$VPC_CIDR" \
  --tag-specifications \
  'ResourceType=vpc,Tags=[{Key=Name,Value=nat-vpc}]' \
  --query 'Vpc.VpcId' \
  --output text)

aws ec2 wait vpc-available --vpc-ids "$VPC_ID"

aws ec2 modify-vpc-attribute \
  --vpc-id "$VPC_ID" \
  --enable-dns-support

aws ec2 modify-vpc-attribute \
  --vpc-id "$VPC_ID" \
  --enable-dns-hostnames

echo "VPC: $VPC_ID"
```

Verify:

```
aws ec2 describe-vpcs \
  --vpc-ids "$VPC_ID" \
  --output table
```

### Step 3 – Create public subnet

```
PUBLIC_SUBNET_ID=$(aws ec2 create-subnet \
  --vpc-id "$VPC_ID" \
  --cidr-block "$PUBLIC_CIDR" \
  --availability-zone "$AZ" \
  --tag-specifications \
  'ResourceType=subnet,Tags=[{Key=Name,Value=public-subnet}]' \
  --query 'Subnet.SubnetId' \
  --output text)

aws ec2 modify-subnet-attribute \
  --subnet-id "$PUBLIC_SUBNET_ID" \
  --map-public-ip-on-launch

echo "Public Subnet: $PUBLIC_SUBNET_ID"
```

### Step 4 – Create private subnet

```
PRIVATE_SUBNET_ID=$(aws ec2 create-subnet \
  --vpc-id "$VPC_ID" \
  --cidr-block "$PRIVATE_CIDR" \
  --availability-zone "$AZ" \
  --tag-specifications \
  'ResourceType=subnet,Tags=[{Key=Name,Value=private-subnet}]' \
  --query 'Subnet.SubnetId' \
  --output text)

aws ec2 modify-subnet-attribute \
  --subnet-id "$PRIVATE_SUBNET_ID" \
  --no-map-public-ip-on-launch

echo "Private Subnet: $PRIVATE_SUBNET_ID"
```

Verify subnets:

```
aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=$VPC_ID" \
  --query 'Subnets[*].[SubnetId,CidrBlock,AvailabilityZone,MapPublicIpOnLaunch]' \
  --output table
```

### Step 5 – Create Internet Gateway

```
IGW_ID=$(aws ec2 create-internet-gateway \
  --tag-specifications \
  'ResourceType=internet-gateway,Tags=[{Key=Name,Value=nat-igw}]' \
  --query 'InternetGateway.InternetGatewayId' \
  --output text)

aws ec2 attach-internet-gateway \
  --internet-gateway-id "$IGW_ID" \
  --vpc-id "$VPC_ID"

echo "Internet Gateway: $IGW_ID"
```

### Step 6 – Create public route table

```
PUBLIC_RT_ID=$(aws ec2 create-route-table \
  --vpc-id "$VPC_ID" \
  --tag-specifications \
  'ResourceType=route-table,Tags=[{Key=Name,Value=public-rt}]' \
  --query 'RouteTable.RouteTableId' \
  --output text)

aws ec2 create-route \
  --route-table-id "$PUBLIC_RT_ID" \
  --destination-cidr-block 0.0.0.0/0 \
  --gateway-id "$IGW_ID"

PUBLIC_RT_ASSOC_ID=$(aws ec2 associate-route-table \
  --route-table-id "$PUBLIC_RT_ID" \
  --subnet-id "$PUBLIC_SUBNET_ID" \
  --query 'AssociationId' \
  --output text)

echo "Public Route Table: $PUBLIC_RT_ID"
```

Public route table:

| Destination   | Target           |
| ------------- | ---------------- |
| `10.0.0.0/16` | local            |
| `0.0.0.0/0`   | Internet Gateway |

### Step 7 – Allocate Elastic IP

```
EIP_ALLOC_ID=$(aws ec2 allocate-address \
  --domain vpc \
  --tag-specifications \
  'ResourceType=elastic-ip,Tags=[{Key=Name,Value=nat-eip}]' \
  --query 'AllocationId' \
  --output text)

NAT_PUBLIC_IP=$(aws ec2 describe-addresses \
  --allocation-ids "$EIP_ALLOC_ID" \
  --query 'Addresses[0].PublicIp' \
  --output text)

echo "Elastic IP Allocation: $EIP_ALLOC_ID"
echo "NAT Public IP: $NAT_PUBLIC_IP"
```

### Step 8 – Create NAT Gateway

```
NAT_ID=$(aws ec2 create-nat-gateway \
  --subnet-id "$PUBLIC_SUBNET_ID" \
  --allocation-id "$EIP_ALLOC_ID" \
  --connectivity-type public \
  --tag-specifications \
  'ResourceType=natgateway,Tags=[{Key=Name,Value=nat-gateway}]' \
  --query 'NatGateway.NatGatewayId' \
  --output text)

echo "NAT Gateway: $NAT_ID"

aws ec2 wait nat-gateway-available \
  --nat-gateway-ids "$NAT_ID"
```

Verify:

```
aws ec2 describe-nat-gateways \
  --nat-gateway-ids "$NAT_ID" \
  --query 'NatGateways[*].[NatGatewayId,State,SubnetId]' \
  --output table
```

Expected state: `available`.

### Step 9 – Create private route table

```
PRIVATE_RT_ID=$(aws ec2 create-route-table \
  --vpc-id "$VPC_ID" \
  --tag-specifications \
  'ResourceType=route-table,Tags=[{Key=Name,Value=private-rt}]' \
  --query 'RouteTable.RouteTableId' \
  --output text)

aws ec2 create-route \
  --route-table-id "$PRIVATE_RT_ID" \
  --destination-cidr-block 0.0.0.0/0 \
  --nat-gateway-id "$NAT_ID"

PRIVATE_RT_ASSOC_ID=$(aws ec2 associate-route-table \
  --route-table-id "$PRIVATE_RT_ID" \
  --subnet-id "$PRIVATE_SUBNET_ID" \
  --query 'AssociationId' \
  --output text)

echo "Private Route Table: $PRIVATE_RT_ID"
```

Private route table:

| Destination   | Target      |
| ------------- | ----------- |
| `10.0.0.0/16` | local       |
| `0.0.0.0/0`   | NAT Gateway |

The subnet, IGW, NAT Gateway, and routing steps follow the AWS VPC CLI deployment model.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://docs.aws.amazon.com\&sz=32)

Amazon Virtual Private Cloud

+1



### Step 10 – Create SSH key pair

```
KEY_NAME="nat-lab-$(date +%s)"

aws ec2 create-key-pair \
  --key-name "$KEY_NAME" \
  --key-type rsa \
  --query 'KeyMaterial' \
  --output text > "$KEY_NAME.pem"

chmod 400 "$KEY_NAME.pem"

echo "Key Pair: $KEY_NAME"
```

Keep this PEM file on your local laptop.

### Step 11 – Create security groups

Public EC2 security group

```
PUBLIC_SG_ID=$(aws ec2 create-security-group \
  --group-name nat-public-sg \
  --description "Public bastion SSH security group" \
  --vpc-id "$VPC_ID" \
  --query 'GroupId' \
  --output text)

aws ec2 authorize-security-group-ingress \
  --group-id "$PUBLIC_SG_ID" \
  --protocol tcp \
  --port 22 \
  --cidr "$MY_IP"
```

Private EC2 security group

```
PRIVATE_SG_ID=$(aws ec2 create-security-group \
  --group-name nat-private-sg \
  --description "Private EC2 SSH from bastion" \
  --vpc-id "$VPC_ID" \
  --query 'GroupId' \
  --output text)

aws ec2 authorize-security-group-ingress \
  --group-id "$PRIVATE_SG_ID" \
  --protocol tcp \
  --port 22 \
  --source-group "$PUBLIC_SG_ID"
```

Security group summary:

| Instance    | Inbound      | Source                    |
| ----------- | ------------ | ------------------------- |
| Public EC2  | TCP 22       | Your public IPv4 `/32`    |
| Private EC2 | TCP 22       | Public EC2 security group |
| Both        | Outbound all | `0.0.0.0/0`               |

The outbound rule is the default security group egress configuration and is sufficient for this teaching lab.

### Step 12 – Retrieve Amazon Linux AMI

```
AMI_ID=$(aws ssm get-parameter \
  --name \
  /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64 \
  --query 'Parameter.Value' \
  --output text)

echo "AMI: $AMI_ID"
```

### Step 13 – Launch public EC2

```
PUBLIC_EC2_ID=$(aws ec2 run-instances \
  --image-id "$AMI_ID" \
  --instance-type t3.micro \
  --key-name "$KEY_NAME" \
  --subnet-id "$PUBLIC_SUBNET_ID" \
  --security-group-ids "$PUBLIC_SG_ID" \
  --associate-public-ip-address \
  --metadata-options \
  HttpTokens=required,HttpEndpoint=enabled \
  --tag-specifications \
  'ResourceType=instance,Tags=[{Key=Name,Value=ec2-public}]' \
  --query 'Instances[0].InstanceId' \
  --output text)

aws ec2 wait instance-running \
  --instance-ids "$PUBLIC_EC2_ID"

echo "Public EC2: $PUBLIC_EC2_ID"
```

### Step 14 – Launch private EC2

```
PRIVATE_EC2_ID=$(aws ec2 run-instances \
  --image-id "$AMI_ID" \
  --instance-type t3.micro \
  --key-name "$KEY_NAME" \
  --network-interfaces \
  "DeviceIndex=0,SubnetId=$PRIVATE_SUBNET_ID,Groups=[$PRIVATE_SG_ID],AssociatePublicIpAddress=false" \
  --metadata-options \
  HttpTokens=required,HttpEndpoint=enabled \
  --tag-specifications \
  'ResourceType=instance,Tags=[{Key=Name,Value=ec2-private}]' \
  --query 'Instances[0].InstanceId' \
  --output text)

aws ec2 wait instance-running \
  --instance-ids "$PRIVATE_EC2_ID"

echo "Private EC2: $PRIVATE_EC2_ID"
```

Using the network-interface configuration explicitly prevents a public IPv4 address from being assigned to the private EC2 instance.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://docs.aws.amazon.com\&sz=32)

AWS CLI 2.38.1 Command Reference



### Step 15 – Get EC2 IP addresses

```
PUBLIC_IP=$(aws ec2 describe-instances \
  --instance-ids "$PUBLIC_EC2_ID" \
  --query 'Reservations[0].Instances[0].PublicIpAddress' \
  --output text)

PRIVATE_IP=$(aws ec2 describe-instances \
  --instance-ids "$PRIVATE_EC2_ID" \
  --query 'Reservations[0].Instances[0].PrivateIpAddress' \
  --output text)

echo "Public EC2 IP: $PUBLIC_IP"
echo "Private EC2 IP: $PRIVATE_IP"
echo "NAT Gateway IP: $NAT_PUBLIC_IP"
```

### Step 16 – Save resource details

Create a file containing all generated resource IDs, so you can reconnect or clean up later.

```
cat > nat-project.env <<EOF
export AWS_DEFAULT_REGION="$AWS_DEFAULT_REGION"
export VPC_ID="$VPC_ID"
export PUBLIC_SUBNET_ID="$PUBLIC_SUBNET_ID"
export PRIVATE_SUBNET_ID="$PRIVATE_SUBNET_ID"
export IGW_ID="$IGW_ID"
export PUBLIC_RT_ID="$PUBLIC_RT_ID"
export PRIVATE_RT_ID="$PRIVATE_RT_ID"
export PUBLIC_RT_ASSOC_ID="$PUBLIC_RT_ASSOC_ID"
export PRIVATE_RT_ASSOC_ID="$PRIVATE_RT_ASSOC_ID"
export EIP_ALLOC_ID="$EIP_ALLOC_ID"
export NAT_ID="$NAT_ID"
export PUBLIC_SG_ID="$PUBLIC_SG_ID"
export PRIVATE_SG_ID="$PRIVATE_SG_ID"
export KEY_NAME="$KEY_NAME"
export PUBLIC_EC2_ID="$PUBLIC_EC2_ID"
export PRIVATE_EC2_ID="$PRIVATE_EC2_ID"
export PUBLIC_IP="$PUBLIC_IP"
export PRIVATE_IP="$PRIVATE_IP"
export NAT_PUBLIC_IP="$NAT_PUBLIC_IP"
EOF

chmod 600 nat-project.env
```

To restore variables in a new terminal:

```
source nat-project.env
```

Do not commit the PEM file or environment file containing infrastructure details to a public repository.

## 6. Connect to public and private EC2

### Step 17 – Connect to public EC2

On your local laptop:

```
eval "$(ssh-agent -s)"
ssh-add "./$KEY_NAME.pem"

ssh -A -i "./$KEY_NAME.pem" \
  ec2-user@"$PUBLIC_IP"
```

The public EC2 acts as a bastion.

### Step 18 – Connect to private EC2

Inside the public EC2 SSH session:

```
ssh ec2-user@10.0.2.x
```

Replace `10.0.2.x` with the actual private IP.

Alternatively, use a single command from your local laptop without agent forwarding:

```
ssh -i "./$KEY_NAME.pem" \
  -J "ec2-user@$PUBLIC_IP" \
  "ec2-user@$PRIVATE_IP"
```

This uses SSH ProxyJump. It avoids storing the private key on the bastion.

## 7. Test NAT Gateway

Run the following commands inside the private EC2 instance.

Test A – Network configuration

```
hostname -I
ip -4 addr
ip route
```

Test B – DNS resolution

```
getent hosts www.google.com
```

Test C – Internet connectivity

```
curl -4 -I --connect-timeout 10 https://www.google.com
```

Test D – Outbound public IP

```
curl -4 -s https://checkip.amazonaws.com
```

Expected: The output matches `NAT_PUBLIC_IP` from Step 15.

Test E – Package installation

```
sudo dnf check-update
sudo dnf install -y git
git --version
```

Test F – Verify external API access

```
curl -4 -s https://api.github.com
```

A successful HTTP response demonstrates outbound access.

### Expected results

| Test                                       | Expected               |
| ------------------------------------------ | ---------------------- |
| Private EC2 IP                             | `10.0.2.x`             |
| Public IP on private EC2                   | None                   |
| NAT Gateway status                         | `available`            |
| DNS lookup                                 | Resolves               |
| HTTPS request                              | Successful             |
| External observed IP                       | NAT Gateway Elastic IP |
| Package installation                       | Successful             |
| Direct inbound internet SSH to private EC2 | Not available          |

## 8. Verify AWS infrastructure

Run from your local laptop.

### Check VPC

```
aws ec2 describe-vpcs \
  --vpc-ids "$VPC_ID" \
  --output table
```

### Check subnets

```
aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=$VPC_ID" \
  --query 'Subnets[*].[SubnetId,CidrBlock,AvailabilityZone]' \
  --output table
```

### Check NAT Gateway

```
aws ec2 describe-nat-gateways \
  --nat-gateway-ids "$NAT_ID" \
  --output table
```

### Check route tables

```
aws ec2 describe-route-tables \
  --route-table-ids "$PUBLIC_RT_ID" "$PRIVATE_RT_ID" \
  --query 'RouteTables[*].{ID:RouteTableId,Routes:Routes}' \
  --output json
```

### Check EC2 instances

```
aws ec2 describe-instances \
  --instance-ids "$PUBLIC_EC2_ID" "$PRIVATE_EC2_ID" \
  --query 'Reservations[*].Instances[*].[InstanceId,State.Name,PrivateIpAddress,PublicIpAddress]' \
  --output table
```

## 9. Troubleshooting guide

| Issue                                  | Possible cause                        | Solution                              |
| -------------------------------------- | ------------------------------------- | ------------------------------------- |
| Cannot connect to public EC2           | SSH security group or wrong public IP | Verify port 22 and your `/32` source  |
| Cannot connect to private EC2          | Wrong private IP or SG                | Allow SSH from bastion SG             |
| Private EC2 has no internet            | Missing NAT route                     | Verify private route table            |
| NAT Gateway creation fails             | EIP, quota, or subnet issue           | Check NAT failure details             |
| DNS fails                              | VPC DNS configuration                 | Enable DNS support                    |
| `curl` times out                       | Routing, NACL, or SG issue            | Check route tables and outbound rules |
| `dnf` fails                            | DNS or internet access                | Test HTTPS and DNS separately         |
| NAT Gateway is deleted but EIP remains | EIP not automatically released        | Release EIP manually                  |

For NAT troubleshooting:

```
aws ec2 describe-nat-gateways \
  --nat-gateway-ids "$NAT_ID" \
  --query 'NatGateways[*].[State,FailureCode,FailureMessage]' \
  --output table
```

## 10. Delete project resources

NAT Gateway, EC2 instances, and public IPv4 addresses can generate charges. Delete resources when the lab is complete.

The cleanup sequence follows AWS dependency requirements.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://docs.aws.amazon.com\&sz=32)

AWS CLI 2.38.1 Command Reference

+2



### Step 1 – Restore project variables

```
source nat-project.env
```

### Step 2 – Terminate EC2 instances

```
aws ec2 terminate-instances \
  --instance-ids "$PUBLIC_EC2_ID" "$PRIVATE_EC2_ID"

aws ec2 wait instance-terminated \
  --instance-ids "$PUBLIC_EC2_ID" "$PRIVATE_EC2_ID"
```

### Step 3 – Delete NAT Gateway

```
aws ec2 delete-nat-gateway \
  --nat-gateway-id "$NAT_ID"

aws ec2 wait nat-gateway-deleted \
  --nat-gateway-ids "$NAT_ID"
```

### Step 4 – Release Elastic IP

```
aws ec2 release-address \
  --allocation-id "$EIP_ALLOC_ID"
```

### Step 5 – Delete route table associations

```
aws ec2 disassociate-route-table \
  --association-id "$PRIVATE_RT_ASSOC_ID"

aws ec2 disassociate-route-table \
  --association-id "$PUBLIC_RT_ASSOC_ID"
```

### Step 6 – Delete route tables

```
aws ec2 delete-route-table \
  --route-table-id "$PRIVATE_RT_ID"

aws ec2 delete-route-table \
  --route-table-id "$PUBLIC_RT_ID"
```

### Step 7 – Delete security groups

Remove the private security group's reference to the public security group first:

```
aws ec2 revoke-security-group-ingress \
  --group-id "$PRIVATE_SG_ID" \
  --protocol tcp \
  --port 22 \
  --source-group "$PUBLIC_SG_ID"

aws ec2 delete-security-group \
  --group-id "$PRIVATE_SG_ID"

aws ec2 delete-security-group \
  --group-id "$PUBLIC_SG_ID"
```

### Step 8 – Delete subnets

```
aws ec2 delete-subnet \
  --subnet-id "$PRIVATE_SUBNET_ID"

aws ec2 delete-subnet \
  --subnet-id "$PUBLIC_SUBNET_ID"
```

### Step 9 – Detach and delete Internet Gateway

```
aws ec2 detach-internet-gateway \
  --internet-gateway-id "$IGW_ID" \
  --vpc-id "$VPC_ID"

aws ec2 delete-internet-gateway \
  --internet-gateway-id "$IGW_ID"
```

### Step 10 – Delete VPC

```
aws ec2 delete-vpc \
  --vpc-id "$VPC_ID"
```

### Step 11 – Delete SSH key pair

```
aws ec2 delete-key-pair \
  --key-name "$KEY_NAME"

ssh-add -d "./$KEY_NAME.pem"
rm -f "./$KEY_NAME.pem"
```

### Step 12 – Verify cleanup

```
aws ec2 describe-vpcs \
  --vpc-ids "$VPC_ID"

aws ec2 describe-addresses \
  --allocation-ids "$EIP_ALLOC_ID"
```

Expected: The VPC and Elastic IP no longer exist. AWS may return resource-not-found errors, which are expected after deletion.

Finally, remove the saved environment file:

```
rm -f nat-project.env
```

## 11. Points to remember for training

| Concept           | Important point                                           |
| ----------------- | --------------------------------------------------------- |
| VPC               | Logically isolated AWS network                            |
| Public subnet     | Subnet with a default IPv4 route to IGW                   |
| Private subnet    | No direct default route to IGW                            |
| Internet Gateway  | Provides internet routing for public resources            |
| NAT Gateway       | Allows private workloads to initiate outbound connections |
| Elastic IP        | Public IPv4 address associated with public NAT Gateway    |
| Route table       | Determines the next hop for network traffic               |
| Security Group    | Stateful firewall for EC2                                 |
| NACL              | Stateless subnet-level traffic control                    |
| Bastion host      | Controlled SSH entry point                                |
| DNS               | Resolves domain names to IP addresses                     |
| High availability | Avoid single-AZ dependency for production workloads       |

### NAT Gateway vs Internet Gateway

| Feature                             | Internet Gateway                        | Public NAT Gateway                         |
| ----------------------------------- | --------------------------------------- | ------------------------------------------ |
| Public subnet connectivity          | Yes                                     | Provides outbound path for private subnets |
| Requires Elastic IP                 | No, not for the gateway itself          | Yes, for this zonal NAT configuration      |
| Initiates outbound internet traffic | Supports public resources               | Supports private workloads                 |
| Unsolicited inbound connections     | Possible to public resources if allowed | Not supported to private instances         |
| Route table target                  | `igw-...`                               | `nat-...`                                  |

Production note: This lab uses a zonal NAT Gateway in one Availability Zone. For multi-AZ production deployments, consider AZ-local NAT gateways or the regional NAT Gateway option supported by AWS, based on availability, routing, and cost requirements.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://docs.aws.amazon.com\&sz=32)

Amazon Virtual Private Cloud



## 12. Practical assignments

AWS NAT Gateway Lab Checklist

0/11 completed

Create VPC with CIDR 10.0.0.0/16

Create public and private subnets

Configure Internet Gateway and public route table

Create Elastic IP and NAT Gateway

Configure private route table

Launch public and private EC2 instances

Connect to private EC2 through bastion

Verify outbound internet access using curl

Install Git on private EC2 using dnf

Remove NAT route, observe failure, and restore it

Delete all AWS resources

&#x20;Copy assignment checklist

## 13. Project completion criteria

The project is successfully completed when all of the following conditions are satisfied:

- Public EC2 has a public IPv4 address and is reachable through SSH from your trusted IP.
- Private EC2 has only a private IPv4 address.
- Private EC2 is reachable through the bastion host.
- Private EC2 can resolve DNS and access HTTPS endpoints.
- Outbound IP observed from private EC2 matches the NAT Gateway Elastic IP.
- Package installation succeeds on the private EC2.
- All lab resources can be deleted without remaining dependencies.

Official AWS references: [Create a VPC with private subnets and NAT gateways](https://docs.aws.amazon.com/vpc/latest/userguide/create-a-vpc-with-private-subnets-and-nat-gateways-using-aws-cli.html?utm_source=chatgpt.com) · [Working with NAT gateways](https://docs.aws.amazon.com/vpc/latest/userguide/nat-gateway-working-with.html?utm_source=chatgpt.com) · [Delete a VPC](https://docs.aws.amazon.com/vpc/latest/userguide/delete-vpc.html?utm_source=chatgpt.com)

This is a complete AWS CLI training project, including creation, networking, instance deployment, connectivity verification, troubleshooting, and cleanup. The commands are structured for execution but have not been run against your AWS account.

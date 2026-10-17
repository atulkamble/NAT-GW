# Cleanup Guide

[← Home](../README.md)

## 4. Cleanup commands

Run from your original local terminal **with the original variables available**. These commands permanently delete this lab. If using a new terminal, you must restore your real IDs first.

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
ssh-add -d "./$KEY_NAME.pem" 2>/dev/null || true
rm -f "./$KEY_NAME.pem"
```


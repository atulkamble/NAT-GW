# Connectivity & Validation

[← Home](../README.md)

## 3. Connect and test

Allow instance boot/SSH initialization. This lab uses Amazon Linux 2023 (`ec2-user`). For the first SSH connection, verify the server host-key fingerprint before accepting it.

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
sudo dnf check-update || test "$?" -eq 100
sudo dnf install -y git
```

Verify NAT Elastic IP from your laptop:

```
aws ec2 describe-addresses \
  --allocation-ids "$EIP_ID" \
  --query 'Addresses[0].PublicIp' \
  --output text
```

The public IP returned from private EC2 **must match** the NAT Elastic IP. Check that the private instance has no public IPv4:

```bash
aws ec2 describe-instances --instance-ids "$PRIVATE_EC2" \
  --query 'Reservations[0].Instances[0].[PrivateIpAddress,PublicIpAddress]' --output table
aws ec2 describe-route-tables --route-table-ids "$PUBLIC_RT" "$PRIVATE_RT" \
  --query 'RouteTables[*].[RouteTableId,Routes]' --output json
```

**Expected:** public route `0.0.0.0/0 → igw-*`; private route `0.0.0.0/0 → nat-*`; private instance `PublicIpAddress` is `None`.

**Troubleshooting:** Confirm NAT is `available`, routes are associated with their intended subnets, default SG egress permits HTTPS, VPC DNS is enabled, and NACLs allow return traffic. SSH to the bastion is limited to `$MY_IP`; private SSH is limited to the bastion SG.

## Live AWS deployment verification

**Status: NOT RUN — awaiting execution in an AWS account.** Documentation review is not proof of deployment. Do not mark these checks passed until actual command output is collected.

From the same local terminal after Step 10, run:

```bash
aws ec2 describe-nat-gateways --nat-gateway-ids "$NAT_ID" \
  --query 'NatGateways[0].[NatGatewayId,State]' --output table
aws ec2 describe-instances --instance-ids "$PUBLIC_EC2" "$PRIVATE_EC2" \
  --query 'Reservations[*].Instances[*].[InstanceId,State.Name,PrivateIpAddress,PublicIpAddress]' --output table
aws ec2 describe-route-tables --route-table-ids "$PUBLIC_RT" "$PRIVATE_RT" \
  --query 'RouteTables[*].[RouteTableId,Routes]' --output json
NAT_EIP=$(aws ec2 describe-addresses --allocation-ids "$EIP_ID" \
  --query 'Addresses[0].PublicIp' --output text)
PRIVATE_EGRESS_IP=$(ssh -o BatchMode=yes -o ConnectTimeout=15 -i "./$KEY_NAME.pem" \
  -J "ec2-user@$PUBLIC_IP" "ec2-user@$PRIVATE_IP" \
  'curl -fsS -4 --connect-timeout 10 https://checkip.amazonaws.com')
echo "NAT EIP: $NAT_EIP | Private egress: $PRIVATE_EGRESS_IP"
test "$NAT_EIP" = "$PRIVATE_EGRESS_IP" && echo "PASS: NAT egress IP matches"
```

SSH requires that the bastion host key has been verified and accepted beforehand; `BatchMode=yes` will not prompt for credentials. If a check fails, troubleshoot before cleanup. **Never commit private keys, AWS credentials, or sensitive command output.**

| Check | Live result |
|---|---|
| NAT Gateway available | Pending |
| Both EC2 instances running | Pending |
| Bastion-to-private SSH | Pending |
| Private HTTPS connectivity | Pending |
| NAT EIP equals private egress IP | Pending |


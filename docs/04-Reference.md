# Key Concepts & Reference

[← Home](../README.md)

## 5. Points to remember

- Public subnet → Internet Gateway.
- Private subnet → NAT Gateway.
- NAT Gateway → Public subnet + Elastic IP.
- Private EC2 → No public IP.
- Bastion EC2 → SSH access to private EC2.
- NAT Gateway → Outbound IPv4 connectivity, not unsolicited inbound access.
- NAT Gateway and public IPv4 addresses incur charges until released or deleted.

**Expected result:** Private EC2 accesses internet via NAT Gateway without a public IP. Confirm IP equality with `checkip.amazonaws.com`, then perform cleanup. Verify VPC and Elastic IP deletion using `describe-vpcs` and `describe-addresses` (resource-not-found responses expected).

**AWS documentation:** [NAT gateway basics](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-nat-gateway.html) · [Create a NAT gateway](https://docs.aws.amazon.com/vpc/latest/userguide/nat-gateway-working-with.html) · [VPC routing](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Route_Tables.html)

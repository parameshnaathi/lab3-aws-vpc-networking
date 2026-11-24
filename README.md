# Lab 3 - AWS VPC & Networking From Scratch

This lab is designed for a **corporate laptop environment using AWS Console + AWS CloudShell**, so no local `sudo` access or AWS CLI installation is required.

The goal is to build and troubleshoot a production-style AWS network from scratch and understand the complete traffic path.

---

# Repository Structure

```text
lab3-aws-vpc-networking/
├── README.md
└── notes/
```

You can optionally store screenshots, command outputs, or architecture notes inside the `notes/` directory.

---

# Quick Command Reference

| Category | Command | Use | Simple Example |
|---|---|---|---|
| AWS Identity | `aws sts get-caller-identity` | Verify active AWS identity and account | `aws sts get-caller-identity` |
| Region | `export AWS_DEFAULT_REGION=...` | Set default AWS region | `export AWS_DEFAULT_REGION=ap-south-1` |
| VPC | `aws ec2 describe-vpcs` | List or inspect VPCs | `aws ec2 describe-vpcs` |
| VPC | `aws ec2 create-vpc` | Create a new VPC | `aws ec2 create-vpc --cidr-block 10.20.0.0/16` |
| Subnet | `aws ec2 describe-subnets` | List or inspect subnets | `aws ec2 describe-subnets` |
| Subnet | `aws ec2 create-subnet` | Create a subnet | `aws ec2 create-subnet --vpc-id <VPC_ID> --cidr-block 10.20.1.0/24` |
| IGW | `aws ec2 create-internet-gateway` | Create an Internet Gateway | `aws ec2 create-internet-gateway` |
| IGW | `aws ec2 attach-internet-gateway` | Attach IGW to VPC | `aws ec2 attach-internet-gateway --internet-gateway-id <IGW> --vpc-id <VPC>` |
| NAT | `aws ec2 create-nat-gateway` | Create NAT Gateway | `aws ec2 create-nat-gateway --subnet-id <PUBLIC_SUBNET> --allocation-id <EIP>` |
| Route Table | `aws ec2 describe-route-tables` | Inspect route tables and routes | `aws ec2 describe-route-tables` |
| Route Table | `aws ec2 create-route-table` | Create route table | `aws ec2 create-route-table --vpc-id <VPC_ID>` |
| Route | `aws ec2 create-route` | Add route to route table | `aws ec2 create-route --route-table-id <RT> --destination-cidr-block 0.0.0.0/0 --gateway-id <IGW>` |
| Route | `aws ec2 delete-route` | Remove a route | `aws ec2 delete-route --route-table-id <RT> --destination-cidr-block 0.0.0.0/0` |
| Security Group | `aws ec2 describe-security-groups` | Inspect SG rules | `aws ec2 describe-security-groups` |
| Security Group | `aws ec2 authorize-security-group-ingress` | Add SG inbound rule | `aws ec2 authorize-security-group-ingress --group-id <SG> ...` |
| Security Group | `aws ec2 revoke-security-group-ingress` | Remove SG inbound rule | `aws ec2 revoke-security-group-ingress --group-id <SG> ...` |
| NACL | `aws ec2 describe-network-acls` | Inspect Network ACLs | `aws ec2 describe-network-acls` |
| NACL | `aws ec2 create-network-acl-entry` | Add NACL allow/deny rule | `aws ec2 create-network-acl-entry --network-acl-id <NACL> ...` |
| EC2 | `aws ec2 run-instances` | Launch EC2 instance | `aws ec2 run-instances --image-id <AMI> --instance-type t3.micro ...` |
| EC2 | `aws ec2 describe-instances` | Inspect EC2 instances | `aws ec2 describe-instances` |
| EC2 | `aws ec2 terminate-instances` | Terminate EC2 instances | `aws ec2 terminate-instances --instance-ids <ID>` |
| VPC Endpoint | `aws ec2 create-vpc-endpoint` | Create VPC endpoint | `aws ec2 create-vpc-endpoint --vpc-id <VPC> --service-name com.amazonaws.ap-south-1.s3 ...` |
| SSM | `aws ssm describe-instance-information` | Check SSM managed instances | `aws ssm describe-instance-information` |
| Linux Route | `ip route` | Check instance OS routing | `ip route` |
| DNS | `getent hosts <name>` | Check DNS resolution | `getent hosts amazon.com` |
| HTTP | `curl` | Test connectivity | `curl https://checkip.amazonaws.com` |
| Port Test | `curl --connect-timeout` | Test application reachability | `curl --connect-timeout 5 http://10.20.11.10:8080` |

## Most Important Troubleshooting Sequence

```text
Application / Process
       ↓
Destination IP / DNS
       ↓
OS Route
       ↓
Subnet Route Table
       ↓
Security Group
       ↓
Network ACL
       ↓
NAT / IGW / VPC Endpoint
       ↓
Destination Security
       ↓
VPC Flow Logs
```

---

# 1. Lab Architecture

```text
                              Internet
                                 |
                         Internet Gateway
                                 |
                    +------------+------------+
                    |                         |
              Public Subnet A           Public Subnet B
              10.20.1.0/24              10.20.2.0/24
                    |
                NAT Gateway
                    |
             ------------------
             Private App Route
             ------------------
                 /         \
                /           \
     Private App A       Private App B
     10.20.11.0/24       10.20.12.0/24

                 \         /
                  \       /
             Private DB Subnets
          10.20.21.0/24  10.20.22.0/24
```

VPC CIDR:

```text
10.20.0.0/16
```

Subnets:

```text
AZ-A
Public-A       10.20.1.0/24
Application-A  10.20.11.0/24
Database-A     10.20.21.0/24

AZ-B
Public-B       10.20.2.0/24
Application-B  10.20.12.0/24
Database-B     10.20.22.0/24
```

---

# 2. Public, Private, and Isolated Subnets

## Public Subnet

```text
0.0.0.0/0 → Internet Gateway
```

## Private Application Subnet

```text
0.0.0.0/0 → NAT Gateway
```

## Isolated Database Subnet

```text
No 0.0.0.0/0 route
```

---

# 3. Open AWS CloudShell

Verify identity:

```bash
aws sts get-caller-identity
```

Set region:

```bash
export REGION=ap-south-1
export AWS_DEFAULT_REGION=$REGION
```

Verify:

```bash
echo $REGION
```

---

# 4. Select Two Availability Zones

```bash
AZ1=$(aws ec2 describe-availability-zones \
  --filters Name=state,Values=available \
  --query 'AvailabilityZones[0].ZoneName' \
  --output text)

AZ2=$(aws ec2 describe-availability-zones \
  --filters Name=state,Values=available \
  --query 'AvailabilityZones[1].ZoneName' \
  --output text)
```

Check:

```bash
echo $AZ1
echo $AZ2
```

---

# 5. Create VPC

```bash
VPC_ID=$(aws ec2 create-vpc \
  --cidr-block 10.20.0.0/16 \
  --tag-specifications \
  'ResourceType=vpc,Tags=[{Key=Name,Value=arrise-lab-vpc}]' \
  --query 'Vpc.VpcId' \
  --output text)
```

Enable DNS:

```bash
aws ec2 modify-vpc-attribute \
  --vpc-id "$VPC_ID" \
  --enable-dns-support '{"Value":true}'

aws ec2 modify-vpc-attribute \
  --vpc-id "$VPC_ID" \
  --enable-dns-hostnames '{"Value":true}'
```

Verify:

```bash
aws ec2 describe-vpcs --vpc-ids "$VPC_ID"
```

---

# 6. Create Six Subnets

```bash
PUBLIC_A=$(aws ec2 create-subnet \
  --vpc-id "$VPC_ID" \
  --availability-zone "$AZ1" \
  --cidr-block 10.20.1.0/24 \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=arrise-public-a}]' \
  --query 'Subnet.SubnetId' --output text)

PUBLIC_B=$(aws ec2 create-subnet \
  --vpc-id "$VPC_ID" \
  --availability-zone "$AZ2" \
  --cidr-block 10.20.2.0/24 \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=arrise-public-b}]' \
  --query 'Subnet.SubnetId' --output text)

APP_A=$(aws ec2 create-subnet \
  --vpc-id "$VPC_ID" \
  --availability-zone "$AZ1" \
  --cidr-block 10.20.11.0/24 \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=arrise-app-a}]' \
  --query 'Subnet.SubnetId' --output text)

APP_B=$(aws ec2 create-subnet \
  --vpc-id "$VPC_ID" \
  --availability-zone "$AZ2" \
  --cidr-block 10.20.12.0/24 \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=arrise-app-b}]' \
  --query 'Subnet.SubnetId' --output text)

DB_A=$(aws ec2 create-subnet \
  --vpc-id "$VPC_ID" \
  --availability-zone "$AZ1" \
  --cidr-block 10.20.21.0/24 \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=arrise-db-a}]' \
  --query 'Subnet.SubnetId' --output text)

DB_B=$(aws ec2 create-subnet \
  --vpc-id "$VPC_ID" \
  --availability-zone "$AZ2" \
  --cidr-block 10.20.22.0/24 \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=arrise-db-b}]' \
  --query 'Subnet.SubnetId' --output text)
```

Verify:

```bash
aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=$VPC_ID" \
  --query 'Subnets[*].[Tags[?Key==`Name`].Value|[0],SubnetId,CidrBlock,AvailabilityZone]' \
  --output table
```

---

# 7. Enable Public IP Mapping

```bash
aws ec2 modify-subnet-attribute --subnet-id "$PUBLIC_A" --map-public-ip-on-launch
aws ec2 modify-subnet-attribute --subnet-id "$PUBLIC_B" --map-public-ip-on-launch
```

Remember:

```text
Public IP alone does NOT make a subnet public.
```

It also needs:

```text
0.0.0.0/0 → Internet Gateway
```

---

# 8. Create Internet Gateway

```bash
IGW_ID=$(aws ec2 create-internet-gateway \
  --tag-specifications 'ResourceType=internet-gateway,Tags=[{Key=Name,Value=arrise-lab-igw}]' \
  --query 'InternetGateway.InternetGatewayId' \
  --output text)
```

Attach:

```bash
aws ec2 attach-internet-gateway \
  --internet-gateway-id "$IGW_ID" \
  --vpc-id "$VPC_ID"
```

---

# 9. Create Public Route Table

```bash
PUBLIC_RT=$(aws ec2 create-route-table \
  --vpc-id "$VPC_ID" \
  --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=arrise-public-rt}]' \
  --query 'RouteTable.RouteTableId' \
  --output text)
```

Add default route:

```bash
aws ec2 create-route \
  --route-table-id "$PUBLIC_RT" \
  --destination-cidr-block 0.0.0.0/0 \
  --gateway-id "$IGW_ID"
```

Associate public subnets:

```bash
aws ec2 associate-route-table --route-table-id "$PUBLIC_RT" --subnet-id "$PUBLIC_A"
aws ec2 associate-route-table --route-table-id "$PUBLIC_RT" --subnet-id "$PUBLIC_B"
```

Expected:

```text
10.20.0.0/16 → local
0.0.0.0/0    → Internet Gateway
```

---

# 10. Create NAT Gateway

Allocate Elastic IP:

```bash
EIP_ALLOC_ID=$(aws ec2 allocate-address \
  --domain vpc \
  --query 'AllocationId' \
  --output text)
```

Create NAT in Public-A:

```bash
NAT_ID=$(aws ec2 create-nat-gateway \
  --subnet-id "$PUBLIC_A" \
  --allocation-id "$EIP_ALLOC_ID" \
  --tag-specifications 'ResourceType=natgateway,Tags=[{Key=Name,Value=arrise-lab-nat}]' \
  --query 'NatGateway.NatGatewayId' \
  --output text)
```

Wait:

```bash
aws ec2 wait nat-gateway-available --nat-gateway-ids "$NAT_ID"
```

> NAT Gateway is billable. Delete it after completing the lab.

---

# 11. Create Application Route Table

```bash
APP_RT=$(aws ec2 create-route-table \
  --vpc-id "$VPC_ID" \
  --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=arrise-app-rt}]' \
  --query 'RouteTable.RouteTableId' \
  --output text)
```

Route through NAT:

```bash
aws ec2 create-route \
  --route-table-id "$APP_RT" \
  --destination-cidr-block 0.0.0.0/0 \
  --nat-gateway-id "$NAT_ID"
```

Associate app subnets:

```bash
aws ec2 associate-route-table --route-table-id "$APP_RT" --subnet-id "$APP_A"
aws ec2 associate-route-table --route-table-id "$APP_RT" --subnet-id "$APP_B"
```

---

# 12. Create Database Route Table

```bash
DB_RT=$(aws ec2 create-route-table \
  --vpc-id "$VPC_ID" \
  --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=arrise-db-rt}]' \
  --query 'RouteTable.RouteTableId' \
  --output text)
```

Associate DB subnets:

```bash
aws ec2 associate-route-table --route-table-id "$DB_RT" --subnet-id "$DB_A"
aws ec2 associate-route-table --route-table-id "$DB_RT" --subnet-id "$DB_B"
```

Do not add a default internet route.

Expected:

```text
10.20.0.0/16 → local
```

---

# 13. Create Security Groups

## Public Test SG

```bash
PUBLIC_SG=$(aws ec2 create-security-group \
  --group-name arrise-public-test-sg \
  --description "Public test instance" \
  --vpc-id "$VPC_ID" \
  --query 'GroupId' \
  --output text)
```

## Application SG

```bash
APP_SG=$(aws ec2 create-security-group \
  --group-name arrise-app-sg \
  --description "Application tier" \
  --vpc-id "$VPC_ID" \
  --query 'GroupId' \
  --output text)
```

Allow port 8080 only from Public-SG:

```bash
aws ec2 authorize-security-group-ingress \
  --group-id "$APP_SG" \
  --ip-permissions \
  "IpProtocol=tcp,FromPort=8080,ToPort=8080,UserIdGroupPairs=[{GroupId=$PUBLIC_SG}]"
```

## Database SG

```bash
DB_SG=$(aws ec2 create-security-group \
  --group-name arrise-db-sg \
  --description "Database tier" \
  --vpc-id "$VPC_ID" \
  --query 'GroupId' \
  --output text)
```

Allow PostgreSQL only from App-SG:

```bash
aws ec2 authorize-security-group-ingress \
  --group-id "$DB_SG" \
  --ip-permissions \
  "IpProtocol=tcp,FromPort=5432,ToPort=5432,UserIdGroupPairs=[{GroupId=$APP_SG}]"
```

---

# 14. Security Group vs NACL

| Security Group | Network ACL |
|---|---|
| Resource / ENI level | Subnet level |
| Stateful | Stateless |
| Allow rules | Allow + deny rules |
| Return traffic automatically allowed | Return traffic must be explicitly allowed |
| All rules evaluated | Lowest matching rule number wins |

### Interview Answer

> Security Groups protect resources and are stateful, so response traffic is automatically allowed for an established connection. Network ACLs protect subnets and are stateless, so both inbound and outbound traffic must be explicitly allowed. NACLs support allow and deny rules, and rules are evaluated in numerical order.

---

# 15. Create SSM IAM Role

Create trust policy:

```bash
cat > trust-policy.json <<'POLICY'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {"Service": "ec2.amazonaws.com"},
      "Action": "sts:AssumeRole"
    }
  ]
}
POLICY
```

Create role:

```bash
aws iam create-role \
  --role-name arrise-lab-ssm-role \
  --assume-role-policy-document file://trust-policy.json
```

Attach SSM policy:

```bash
aws iam attach-role-policy \
  --role-name arrise-lab-ssm-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore
```

Create profile:

```bash
aws iam create-instance-profile \
  --instance-profile-name arrise-lab-ssm-profile
```

Add role:

```bash
aws iam add-role-to-instance-profile \
  --instance-profile-name arrise-lab-ssm-profile \
  --role-name arrise-lab-ssm-role
```

---

# 16. Find Amazon Linux AMI

```bash
AMI_ID=$(aws ssm get-parameter \
  --name /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64 \
  --query 'Parameter.Value' \
  --output text)
```

---

# 17. Launch Public EC2

```bash
PUBLIC_INSTANCE=$(aws ec2 run-instances \
  --image-id "$AMI_ID" \
  --instance-type t3.micro \
  --subnet-id "$PUBLIC_A" \
  --security-group-ids "$PUBLIC_SG" \
  --iam-instance-profile Name=arrise-lab-ssm-profile \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=arrise-public-test}]' \
  --query 'Instances[0].InstanceId' \
  --output text)
```

---

# 18. Launch Private Application EC2

```bash
APP_INSTANCE=$(aws ec2 run-instances \
  --image-id "$AMI_ID" \
  --instance-type t3.micro \
  --subnet-id "$APP_A" \
  --security-group-ids "$APP_SG" \
  --iam-instance-profile Name=arrise-lab-ssm-profile \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=arrise-private-app}]' \
  --query 'Instances[0].InstanceId' \
  --output text)
```

Wait:

```bash
aws ec2 wait instance-status-ok \
  --instance-ids "$PUBLIC_INSTANCE" "$APP_INSTANCE"
```

---

# 19. Connect Using Session Manager

```text
EC2
 ↓
Instances
 ↓
Select Instance
 ↓
Connect
 ↓
Session Manager
 ↓
Connect
```

Benefits:

```text
No SSH key
No inbound port 22
No bastion required
```

---

# 20. Test Public EC2 Internet Connectivity

Inside public EC2:

```bash
curl https://checkip.amazonaws.com
ip route
getent hosts amazon.com
```

---

# 21. Test Private EC2 Internet Connectivity

Inside private EC2:

```bash
curl https://checkip.amazonaws.com
```

Expected path:

```text
Private EC2
   ↓
Private IP
   ↓
NAT Gateway
   ↓
Elastic IP
   ↓
Internet Gateway
   ↓
Internet
```

---

# 22. Run Test Application on Private EC2

Inside private EC2:

```bash
mkdir -p /tmp/lab-web

echo "ARRISE private application OK" > /tmp/lab-web/index.html

cd /tmp/lab-web
python3 -m http.server 8080 --bind 0.0.0.0 &
```

Get private IP from CloudShell:

```bash
APP_PRIVATE_IP=$(aws ec2 describe-instances \
  --instance-ids "$APP_INSTANCE" \
  --query 'Reservations[0].Instances[0].PrivateIpAddress' \
  --output text)
```

From public EC2:

```bash
curl http://$APP_PRIVATE_IP:8080
```

Expected:

```text
ARRISE private application OK
```

---

# 23. Failure Injection - Break Security Group

Remove 8080 rule:

```bash
aws ec2 revoke-security-group-ingress \
  --group-id "$APP_SG" \
  --ip-permissions \
  "IpProtocol=tcp,FromPort=8080,ToPort=8080,UserIdGroupPairs=[{GroupId=$PUBLIC_SG}]"
```

Test:

```bash
curl --connect-timeout 5 http://$APP_PRIVATE_IP:8080
```

Restore:

```bash
aws ec2 authorize-security-group-ingress \
  --group-id "$APP_SG" \
  --ip-permissions \
  "IpProtocol=tcp,FromPort=8080,ToPort=8080,UserIdGroupPairs=[{GroupId=$PUBLIC_SG}]"
```

---

# 24. Failure Injection - Break NAT Route

Delete private default route:

```bash
aws ec2 delete-route \
  --route-table-id "$APP_RT" \
  --destination-cidr-block 0.0.0.0/0
```

Restore:

```bash
aws ec2 create-route \
  --route-table-id "$APP_RT" \
  --destination-cidr-block 0.0.0.0/0 \
  --nat-gateway-id "$NAT_ID"
```

### Interview Lesson

```text
Private Route Table
      ↓
NAT Gateway
      ↓
Public Subnet
      ↓
Public Route Table
      ↓
Internet Gateway
```

---

# 25. Failure Injection - NACL Deny

Find default NACL:

```bash
DEFAULT_NACL=$(aws ec2 describe-network-acls \
  --filters "Name=vpc-id,Values=$VPC_ID" "Name=default,Values=true" \
  --query 'NetworkAcls[0].NetworkAclId' \
  --output text)
```

Create custom NACL:

```bash
LAB_NACL=$(aws ec2 create-network-acl \
  --vpc-id "$VPC_ID" \
  --tag-specifications 'ResourceType=network-acl,Tags=[{Key=Name,Value=arrise-app-test-nacl}]' \
  --query 'NetworkAcl.NetworkAclId' \
  --output text)
```

Deny 8080:

```bash
aws ec2 create-network-acl-entry \
  --network-acl-id "$LAB_NACL" \
  --ingress \
  --rule-number 100 \
  --protocol 6 \
  --port-range From=8080,To=8080 \
  --cidr-block 10.20.1.0/24 \
  --rule-action deny
```

Allow remaining inbound:

```bash
aws ec2 create-network-acl-entry \
  --network-acl-id "$LAB_NACL" \
  --ingress \
  --rule-number 200 \
  --protocol -1 \
  --cidr-block 0.0.0.0/0 \
  --rule-action allow
```

Allow outbound:

```bash
aws ec2 create-network-acl-entry \
  --network-acl-id "$LAB_NACL" \
  --egress \
  --rule-number 100 \
  --protocol -1 \
  --cidr-block 0.0.0.0/0 \
  --rule-action allow
```

Find association:

```bash
APP_NACL_ASSOC=$(aws ec2 describe-network-acls \
  --filters "Name=association.subnet-id,Values=$APP_A" \
  --query 'NetworkAcls[0].Associations[0].NetworkAclAssociationId' \
  --output text)
```

Associate custom NACL:

```bash
NEW_ASSOC=$(aws ec2 replace-network-acl-association \
  --association-id "$APP_NACL_ASSOC" \
  --network-acl-id "$LAB_NACL" \
  --query 'NewAssociationId' \
  --output text)
```

Test:

```bash
curl --connect-timeout 5 http://$APP_PRIVATE_IP:8080
```

Restore default NACL:

```bash
aws ec2 replace-network-acl-association \
  --association-id "$NEW_ASSOC" \
  --network-acl-id "$DEFAULT_NACL"
```

---

# 26. Create S3 Gateway VPC Endpoint

```bash
S3_ENDPOINT=$(aws ec2 create-vpc-endpoint \
  --vpc-id "$VPC_ID" \
  --vpc-endpoint-type Gateway \
  --service-name "com.amazonaws.$REGION.s3" \
  --route-table-ids "$APP_RT" "$DB_RT" \
  --query 'VpcEndpoint.VpcEndpointId' \
  --output text)
```

Inspect route table:

```bash
aws ec2 describe-route-tables \
  --route-table-ids "$APP_RT" \
  --query 'RouteTables[0].Routes' \
  --output table
```

### Interview Point

```text
Private EC2
     ↓
S3 Gateway Endpoint
     ↓
S3
```

This avoids sending S3 traffic through NAT Gateway.

---

# 27. NAT Gateway vs Internet Gateway

## Internet Gateway

```text
Public EC2
   ↓
Public IP
   ↓
Route → IGW
   ↓
Internet
```

## NAT Gateway

```text
Private EC2
   ↓
Private IP
   ↓
Route → NAT
   ↓
Elastic IP
   ↓
IGW
   ↓
Internet
```

### Interview Answer

> An Internet Gateway provides direct internet routing for resources that have public addressing and a route to the IGW. A NAT Gateway is mainly used for private IPv4 workloads that need outbound internet access without accepting unsolicited inbound internet traffic.

---

# 28. Production NAT Design

Lab:

```text
             NAT-A
               |
       App-A + App-B
```

Production pattern:

```text
AZ-A                       AZ-B

Public-A                   Public-B
   |                          |
NAT-A                      NAT-B
   |                          |
App-A                      App-B
```

Routes:

```text
App-A → NAT-A
App-B → NAT-B
```

---

# 29. Private EC2 Cannot Reach Internet - Troubleshooting

```text
1. Is DNS working?
       ↓
2. Correct private IP?
       ↓
3. Private route table:
   0.0.0.0/0 → NAT?
       ↓
4. NAT Gateway available?
       ↓
5. NAT in public subnet?
       ↓
6. Public route table:
   0.0.0.0/0 → IGW?
       ↓
7. IGW attached?
       ↓
8. SG egress allowed?
       ↓
9. NACL ingress/egress allowed?
       ↓
10. VPC Flow Logs
```

### Interview-Ready Answer

> I first verify DNS and the instance network configuration. Then I check whether the private subnet route table sends `0.0.0.0/0` to the NAT Gateway. I verify the NAT Gateway is available and located in a public subnet whose route table sends internet traffic to an Internet Gateway. Then I check security-group egress and both directions of the NACL because NACLs are stateless. If everything appears correct, I use VPC Flow Logs to determine whether traffic is accepted or rejected.

---

# 30. CIDR Planning Interview Point

Example:

```text
VPC = 10.20.0.0/16
Subnets = /24
```

Strong answer:

> I select CIDRs based on current workload requirements, future growth, environment separation, and future connectivity. The most important design consideration is avoiding overlap with other VPCs, on-premises networks, VPN ranges, Transit Gateway-connected networks, and future expansion.

---

# 31. Key Traffic Paths to Remember

## Public EC2 to Internet

```text
EC2
 ↓
Public IP
 ↓
Public Route Table
 ↓
Internet Gateway
 ↓
Internet
```

## Private EC2 to Internet

```text
Private EC2
 ↓
Private Route Table
 ↓
NAT Gateway
 ↓
Public Route Table
 ↓
Internet Gateway
 ↓
Internet
```

## Public EC2 to Private App

```text
Public EC2
 ↓
Public SG
 ↓
VPC Local Route
 ↓
App SG
 ↓
Private App EC2
```

## Application to Database

```text
Application EC2
 ↓
App SG
 ↓
VPC Local Route
 ↓
DB SG
 ↓
Database
```

---

# 32. Cleanup

## Terminate EC2

```bash
aws ec2 terminate-instances \
  --instance-ids "$PUBLIC_INSTANCE" "$APP_INSTANCE"

aws ec2 wait instance-terminated \
  --instance-ids "$PUBLIC_INSTANCE" "$APP_INSTANCE"
```

## Delete S3 Endpoint

```bash
aws ec2 delete-vpc-endpoints --vpc-endpoint-ids "$S3_ENDPOINT"
```

## Delete NAT Gateway and Release EIP

```bash
aws ec2 delete-nat-gateway --nat-gateway-id "$NAT_ID"
aws ec2 wait nat-gateway-deleted --nat-gateway-ids "$NAT_ID"
aws ec2 release-address --allocation-id "$EIP_ALLOC_ID"
```

## Delete Custom NACL

```bash
aws ec2 delete-network-acl --network-acl-id "$LAB_NACL"
```

## Delete Security Groups

```bash
aws ec2 delete-security-group --group-id "$DB_SG"
aws ec2 delete-security-group --group-id "$APP_SG"
aws ec2 delete-security-group --group-id "$PUBLIC_SG"
```

## Delete Route Tables

```bash
aws ec2 delete-route-table --route-table-id "$DB_RT"
aws ec2 delete-route-table --route-table-id "$APP_RT"
aws ec2 delete-route-table --route-table-id "$PUBLIC_RT"
```

## Delete Subnets

```bash
for SUBNET in \
  "$DB_A" "$DB_B" \
  "$APP_A" "$APP_B" \
  "$PUBLIC_A" "$PUBLIC_B"
do
  aws ec2 delete-subnet --subnet-id "$SUBNET"
done
```

## Delete Internet Gateway

```bash
aws ec2 detach-internet-gateway \
  --internet-gateway-id "$IGW_ID" \
  --vpc-id "$VPC_ID"

aws ec2 delete-internet-gateway \
  --internet-gateway-id "$IGW_ID"
```

## Delete VPC

```bash
aws ec2 delete-vpc --vpc-id "$VPC_ID"
```

## Delete IAM Resources

```bash
aws iam remove-role-from-instance-profile \
  --instance-profile-name arrise-lab-ssm-profile \
  --role-name arrise-lab-ssm-role

aws iam delete-instance-profile \
  --instance-profile-name arrise-lab-ssm-profile

aws iam detach-role-policy \
  --role-name arrise-lab-ssm-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore

aws iam delete-role \
  --role-name arrise-lab-ssm-role
```

---


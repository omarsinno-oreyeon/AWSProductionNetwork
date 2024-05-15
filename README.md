# 🌐 Oreyeon AWS Production Network Infrastructure

## Introduction
This repo will server as a guide reference to know how the previous production network infrastructure looked like, how the new network infrastructure is, and how we setup each.

It also presents the advantages and disadvantages of using either, and things to consider. In addition, it adds information regarding each step.

Finally, presents diagrams to represent the old and new structures.

## VPC Peering Setup
The previous setup required a deployment of set of resources and their connection per production account.

On the production account side:

1. Two isolated VPCs, one belonging to the internal account, and one belonging to the production account (Tyndall/Yuma/LEMD). **The VPC CIDR blocks must not overlap.**
2. The VPC in the production account has subnets deployed in 2 or more availability zones in the us-east-1 region.
3. The VPC contains both public and private subnets.
4. In the private subnet is deployed an RDS instance, that runs a relational production database containing metadata regarding the system deployed @ the client side.
5. The public subnet contains an EC2 instance that serves as the bastion host used to connect to the deployed RDS instance in the private subnet.
6. An internet gateway at the VPC level is used to connect to the internet.
7. A NAT Gateway is used to allow the database to access the EC2 instance.

To communicate with the internal account:
1. Establish a peering connection with the internal account.
2. Modify the routing tables on both production and internal accounts to allow traffic between accounts.
3. Modify production resource permissions: any resource deployed in production that needs to be accessed from internal requires to have the right service policies or bucket policies.
4. Modify development resource permissions: any resource deployed in development that needs to be accessed from production requires to have the right service policies or bucket policies.

### Diagram

![VPC Peering Diagram](./diagrams/VPC%20Peering%20Network%20Diagram.svg)


Green Flags:
- Works.
- Production RDS instance accessible through Bastion Host on production account's public subnet.
- Direct network traffic sent between VPCs.

Orange Flags:
- The infrastructure set up and deployment can be made through infrastructure as code services such as CloudFormation.

Red Flags:
- Expensive: IGW and NGW deployed on a per-client basis. I.e., for every client we have to deploy and manage IGWs and NGWs which accumulates a cost overhead and a resource management overhead.
- Distributed connections: peered connections must be created on a per-client basis, whenever a new production account is setup a new PCX must be made and modifications to the route tables must be made.


## Transit Gateway Setup

The new setup:
1. Create Transit Gateway (TGW).
2. Share across organization TGW using Resource Access Manager (RAM).
3. Create TGW attachments.
4. Add routes to attachments in TGW.
5. Add "0.0.0.0/0" destination route to TGW in public subnet route table.
6. Add "0.0.0.0/0" destination route to TGW in private subnet(s) route table(s).

# 🌐 Oreyeon AWS Production Network Infrastructure
This repo will server as a guide reference to know how the previous production network infrastructure looked like, how the new network infrastructure is, and how we setup each.

It also presents the advantages and disadvantages of using either, and things to consider. In addition, it adds information regarding each step.

Finally, presents diagrams to represent the old and new structures.

## Definitions

1. What is a VPC? Amazon Virtual Private Cloud allows you to launch AWS resources in a logically isolated virtual network that you define.

2. What are subnets? A subnet is a range of IP addresses in your VPC. A subnet must reside in a single Availability Zone. After you add subnets, you can deploy AWS resources in your VPC.

3. Private vs Public? A public subnet has a direct route to an internet gateway, can access the public internet. A private subnet does not have a direct route to an internet gateway, they require a NAT device to access the public internet.

4. What are Route Tables? Use route tables to determine where network traffic from your subnet or gateway is directed.

5. What's a peering connection? A VPC peering connection routes traffic between the resources in two VPCs. There is no transitivity between VPCs. Resources in peered VPCs can communicate with each other as if they are within the same network.

6. What is a transit gateway? A transit gateway acts as a central hub, to route traffic between your VPCs, VPN connections, and AWS Direct Connect connections.

7. What is a NAT Gateway? A NAT gateway is a Network Address Translation (NAT) service. You can use a NAT gateway so that instances in a private subnet can connect to services outside your VPC but external services cannot initiate a connection with those instances.

8. What is an Internet Gateway? An internet gateway is a horizontally scaled, redundant, and highly available VPC component that allows communication between your VPC and the internet.


## 🍐 VPC Peering Setup
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

### 📐 Diagram

![VPC Peering Diagram](./diagrams/VPC%20Peering%20Network%20Diagram.svg)


🟢 Works.\
🟢 Production RDS instance accessible through Bastion Host on production account's public subnet.\
🟢 Direct network traffic sent between VPCs.\
🟠 The infrastructure set up and deployment can be made through infrastructure as code services such as CloudFormation.\
🔴 Expensive: IGW and NGW deployed on a per-client basis. I.e., for every client we have to deploy and manage IGWs and NGWs which accumulates a cost overhead and a resource management overhead.\
🔴 Distributed connections: peered connections must be created on a per-client basis, whenever a new production account is setup a new PCX must be made and modifications to the route tables must be made.


## 🚌 Transit Gateway Setup


The new setup:
1. Create Transit Gateway (TGW).
2. Share across organization TGW using Resource Access Manager (RAM).
3. Create TGW attachments.
4. Add routes to attachments in TGW.
5. Add "0.0.0.0/0" destination route to TGW in public subnet route table.
6. Add "0.0.0.0/0" destination route to TGW in private subnet(s) route table(s).

### 📐 Diagram
![Transit Gateway Diagram](./diagrams/Transit%20Gateway%20Network%20Diagram.svg)

🟢 Works.\
🟢 Centralized architecture that simplifies design: any additional client would require a TGW attachment instead of a complete architectural setup.\
🟢 Less costly: avoids the deployment and management of multiple NGWs & IGWs since internet access depends only on central VPC.\
🟢 Avoids creating multiple peering connections.\
🟢 Ability to create multiple routing tables at the TGW level. Flexibility to design multiple networks and routes (we don't need to create multiple TGWs).\
🟢 More control over who has access to the production databases in the production accounts.\
🟠 The infrastructure set up and deployment can be made through infrastructure as code services such as CloudFormation.\
🟠 No direct communication between VPCs, only through the transit gateway which routes traffic according to RT.\
🟠 New clients require only TGW attachment and route table modification.\
🔴 Bastion Hosts will be down since traffic that used to be sent through NGWs then IGWs on the client side are now all redirected to TGW.*

\* Frankly, there is no need for public subnets, NGWs, nor IGWs in the production VPCs anymore since internet access is delivered through the centralized VPC.

If we would want to ditch the mentioned resources, we need to find an alternative to the bastion host:
1- Use Internal Production Database in the internal account as a centralized ?
2- Somehow give bastion host internet access through TGW?

## ClickUp Task Describing the process

This guide outlines the steps to connect multiple accounts' networks (i.e., VPCs) using a **Transit Gateway (TGW)**, ensuring connectivity between an internal account and production accounts. The configuration allows seamless access to production databases and data from the internal account.

### 1. **Transit Gateway (TGW) Setup**
- **Create Transit Gateway**: This is done once from the internal account.
- **Share Transit Gateway with the Organization**:
  - Ensure all production accounts are part of the AWS Organization.
  - Enable **auto-accept resources** so that new accounts added to the organization automatically accept the shared TGW resource.

### 2. **Transit Gateway Attachments**
- **Create TGW Attachments**:
  - From the production account, create a TGW attachment to the shared TGW.
  - The attachment should associate **private subnets** (not public subnets).
  - A pending request will appear in the internal account under **VPC > Transit Gateway Attachments**.
- **Accept the Pending Request**: Approve the TGW attachment request in the internal account.

### 3. **Internal VPC Modifications**
- **Public Route Table**:
  - Add the following route:
    - **Destination**: Production account CIDR block (e.g., `10.0.0.0/16`).
    - **Target**: Transit Gateway (e.g., `tgw-055e96e17b529b40d`).
  - Ensure the rest of the traffic forwards to an IGW (Internet Gateway).
- **Private Route Table**:
  - Ensure traffic forwards to an NGW (NAT Gateway).

### 4. **Create VPC in Production Account**
- Include at least 2 Availability Zones (AZs).
- Include **2 public subnets** and **2 private subnets**.
- Add a **VPC S3 Endpoint**.

### 5. **Create VPC Secrets Manager Endpoint in Production Account**
- Navigate to **VPC > Endpoints > Create Endpoint**.
- When creating the endpoint, ensure it is associated with the VPC's **private subnets**.

### 6. **Production VPC Modifications**
- **Private Route Tables**: Forward traffic to the Transit Gateway:
  - **Destination**: `0.0.0.0/0`
  - **Target**: Transit Gateway
- **Public Route Tables**: Forward traffic to the Transit Gateway:
  - **Destination**: `0.0.0.0/0`
  - **Target**: Transit Gateway.

### 7. **Transit Gateway Route Table**
- Add a route to the production account:
  - **CIDR**: CIDR block of the production account VPC.
  - **Attachment ID**: Select the TGW attachment from the production account VPC.

### 8. **Grant QueryCDI Permissions**
- Ensure the **QueryCDI** Lambda function in the internal account has permissions to access the following resources in the production account:
  - **IAM**, **S3**, **RDS**, **KMS**, and **Secrets Manager**.
- **Create Role in Production Account**:
  - Create the role `arn:aws:iam::ACCOUNT-NUMBER:role/CustomerDataIngestion-Development-Access`.
  - Use an existing production account as a template:
    - Navigate to **IAM > Roles > CustomerDataIngestion-Development-Access**.
    - Replicate the role in the new production account.

### 9. **Modify QueryCDI**
- Update the Lambda function to accept the airport name as a parameter.
- Based on the parameter:
  - Select the production account.
  - Assume the role defined above.
  - Retrieve the secret to access the RDS from the production account.

### 10. **Modify Security Groups**
- Allow traffic between:
  - **Lambda in the internal account**.
  - **RDS in the production account**.

## Deliverables
- Shared network and access between internal and production accounts.

## Optional Enhancements
- Fine-grained permissions for:
  - Security groups traffic.
  - Route tables.


## 🔍 References
References that will be helpful as general knowledge, or implementation guides for either method:

How to setup VPC Peering Infrastructure using IaC: https://cuteprogramming.blog/2024/02/25/setup-and-access-private-rds-database-via-a-bastion-host/.

Example of using transit gateway to centralize outbound routing to the internet, in other words having a central VPC that routes traffic to the internet whether the traffic starts from the VPC itself or attached VPCs: https://docs.aws.amazon.com/vpc/latest/tgw/transit-gateway-nat-igw.html

<p align="center"><img src="./diagrams/tgw-centralized-nat-igw.png" alt="Centralized Outbound Routing" width=50%/></p>

AWS Documentation on VPCs, contains all the definitions defined in the first definitions section, and many more details: https://docs.aws.amazon.com/vpc/latest/userguide/what-is-amazon-vpc.html

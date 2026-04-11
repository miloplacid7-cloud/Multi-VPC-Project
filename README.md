# Multi-VPC Networking

# & PrivateLink Architecture

```
AWS Cloud Infrastructure Project Documentation
```
## Overview

This project designs and implements a multi-VPC architecture where each application is isolated in
its own VPC and subnets. A Shared Services VPC hosts a private internal web application. Payments
and Analytics VPCs consume that application via AWS PrivateLink requiring no public IPs or direct
internet access for internal traffic. Security Groups and routing enforce least-privilege network
access throughout.

## Project Breakdown

- **Create three isolated VPCs** with public and private subnets (Payments, Analytics, Shared
    Services).
- **Configure Security Groups** for the shared application and each client VPC.
- **Deploy a private-only EC2 instance** running an internal web application in the Shared
    Services VPC.
- **Place the instance behind an internal Network Load Balancer (NLB)** for high
    availability.
- **Create a VPC Endpoint Service** for the NLB, acting as the PrivateLink provider.
- **Create Interface Endpoints** in the Payments and Analytics VPCs as PrivateLink consumers.

## VPC Architecture Summary

**VPC Name CIDR Block Public Subnet Internet GW**
Payments VPC 10.10.0.0/16 payments-public-subnet-
(10.10.1.0/24)
payments-igw
Analytics VPC 10.20.0.0/16 analytics-public-subnet-
(10.20.1.0/24)
analytics-igw
Shared Services VPC 10.30.0.0/16 shared-public-subnet-
(10.30.1.0/24)
shared-igw
Each VPC also contains two private subnets (/24) and a main route table for private traffic.


## Step 1: Create the Payments VPC

Begin by building the first isolated environment.

### Create the VPC

- Open the VPC Console and click Create VPC.
- Choose VPC only - to manually build all networking components.
- Name the VPC: payments-vpc
- Enter the CIDR block: 10.10.0.0/16
- Leave all other settings as default and create the VPC.

### Create Subnets

Go to Subnets → Create subnet, then create the following three subnets, selecting payments-vpc for
each:
**Subnet Name Type CIDR**
payments-public-subnet-1 Public 10.10.1.0/24
payments-private-subnet-1 Private 10.10.2.0/24
payments-private-subnet-2 Private 10.10.3.0/24

### Configure Route Tables

- Go to Route Tables → Create route table.


- Select payments-vpc and name it: payments-public-rt.
- Open payments-public-rt → Subnet associations → Edit subnet associations.
- Select payments-public-subnet-1 and save.
- Open the main route table (Main = Yes) → Subnet associations.
- Assign payments-private-subnet-1 and payments-private-subnet-2 to the main route table.


### Attach Internet Gateway

- Go to Internet Gateways → Create internet gateway.
- Set Name tag: payments-igw and create.
- Click Actions → Attach to VPC → select payments-vpc.
- Open payments-public-rt → Routes → Edit routes → Add route:
    ◦ Destination: 0.0.0.0/0


```
◦ Target: payments-igw
```
- **Save the route.** Payments VPC is now ready.


## Step 2: Create the Analytics VPC

Repeat a similar process for the Analytics environment, using a different CIDR range.

### Create the VPC

- Open the VPC Console → Create VPC → choose VPC only.
- Name the VPC: analytics-vpc
- Enter the CIDR block: 10.20.0.0/16
- Leave all other settings as default and click Create VPC.

### Create Subnets

```
Subnet Name Type CIDR
analytics-public-subnet-1 Public 10.20.1.0/24
analytics-private-subnet-1 Private 10.20.2.0/24
analytics-private-subnet-2 Private 10.20.3.0/24
```

### Configure Route Tables & Internet Gateway

- Create analytics-public-rt and associate analytics-public-subnet-1.



- Assign analytics-private-subnet-1 and analytics-private-subnet-2 to the main route table.
- Create internet gateway: analytics-igw and attach to analytics-vpc.



- Add default route in analytics-public-rt: Destination 0.0.0.0/0 → Target analytics-igw.
**Analytics VPC is now ready.**

## Step 3: Create the Shared Services VPC

This VPC hosts the internal web application and provides the PrivateLink Endpoint Service.

### Create VPC, Subnets, Route Tables & Gateway

- Open VPC Console → Create VPC → VPC only.
- Name: shared-services-vpc | CIDR: 10.30.0.0/16
- Create three subnets:
**Subnet Name Type CIDR**
shared-public-subnet-1 Public 10.30.1.0/24
shared-private-subnet-1 Private 10.30.2.0/24
shared-private-subnet-2 Private 10.30.3.0/24


- Important: Ensure both private subnets use different Availability Zones.


- Create shared-public-rt and associate shared-public-subnet-1.
- Assign both private subnets to the main route table.



- Create shared-igw, attach to shared-services-vpc, and add default route 0.0.0.0/0 →
    shared-igw in shared-public-rt.



**Shared Services VPC is now ready.**

## Step 4: Configure Security Groups

AWS automatically creates a default security group for each new VPC, but those have overly
permissive rules. We create clean, purpose-built security groups for each VPC to enforce
least-privilege access.
**SG Name VPC Inbound Rule**
shared-app-sg shared-services-vpc HTTP (port 80) from 10.0.0.0/16
payments-client-sg payments-vpc SSH (port 22) from payments-client-sg
analytics-client-sg analytics-vpc Outbound only (Allow All)
All outbound rules remain as default (Allow All). The Shared Services VPC can receive internal
HTTP traffic, and both client VPCs can send traffic out without any public internet exposure.

## Step 5: Deploy the Internal Service

Now deploy the private EC2 instance that will act as the shared internal application, the PrivateLink
target.

### Launch the EC2 Instance

- EC2 Console → Launch instance. Name: shared-services-app.


- AMI: Amazon Linux 2023 | Instance type: t2.micro or t3.micro.
- Network settings → VPC: shared-services-vpc | Subnet: shared-private-subnet-1.
- Auto-assign public IP: Disabled (no public exposure).
- Security group: select shared-app-sg.
- Scroll to Advanced details → User data and paste your Apache installation script.


- Launch the instance.

### Instance Initialization

Wait 1-2 minutes for 2/2 status checks to pass. The User Data script automatically handles:

- Installing Apache (httpd)
- Starting the web server
- Serving the internal HTML page from /var/www/html/index.html
No manual SSH or EC2 Instance Connect is required.

### Why No Direct Login

The instance is intentionally inaccessible from the internet because:

- It resides in a private subnet
- No public IP is assigned
- No Internet Gateway route exists for private subnets
- No bastion host or SSM endpoints are required for this project

## Step 6: Set Up the PrivateLink Provider

PrivateLink operates in two halves: the provider side (Shared Services VPC) and the consumer side
(Payments & Analytics VPCs). This step completes the provider setup.

### Create a Target Group

- EC2 Console → Target Groups → Create target group.


- Target type: Instances | Name: shared-services-tg | Protocol: TCP | Port: 80.
- VPC: shared-services-vpc. Leave health check settings on default.
- Click Next, then select shared-services-app → Include as pending below.
- Click Create target group.


### Create the Internal Network Load Balancer

- Load Balancers → Create Load Balancer → Network Load Balancer.
- Name: shared-services-nlb, Scheme: Internal, IP Address Type: IPv4.


- Availability Zones: select shared-private-subnet-1 and shared-private-subnet-2.
- Security group: shared-app-sg.
- Listener: Protocol TCP | Port 80 | Forward to shared-services-tg.
- Click Create Load Balancer.


### Create the PrivateLink Endpoint Service

- VPC Console → Endpoint Services → Create Endpoint Service.
- Name: shared-internal-service | Load balancer type: Network Load Balancer.
- Select NLB: shared-services-nlb.


- Acceptance settings: leave Require acceptance enabled (gives full control over which VPCs
    connect).
- Click Create Endpoint Service.
- Copy the generated Service Name (format: com.amazonaws.vpce.<id>.<region>.vpce-svc).
    Payments and Analytics VPCs will need this.

## Step 7: Connect Consumer VPCs via PrivateLink

Create Interface Endpoints in the Payments and Analytics VPCs to privately connect to the Shared
Services application.

### Create Interface Endpoint: Payments VPC

- VPC Console → Endpoints → Create endpoint. Name: payments-endpoint.
- Type: Endpoint services that use NLBs and GWLBs.
- Paste your PrivateLink Service Name and click Verify.
- VPC: payments-vpc | Security group: payments-client-sg.


- Click Create endpoint. Status will show: Pending Acceptance.

### Create Interface Endpoint: Analytics VPC

- VPC Console → Endpoints → Create endpoint. Name: analytics-endpoint.
- Type: Endpoint services that use NLBs and GWLBs.


- Paste the PrivateLink Service Name → Verify.
- VPC: analytics-vpc | Click Create endpoint.

### Approve Both Endpoint Connections

Because the Endpoint Service requires manual acceptance, approve each request from the provider
side:

- VPC Console → Endpoint Services → shared-internal-service.
- Scroll to Endpoint connections you will see entries for both Payments and Analytics VPCs
    (status: Pending acceptance).


- Select each request → Actions → Accept endpoint connection.
- Once accepted, status changes to: Available.

## Architecture & Design Overview

### Traffic Flow

All internal traffic flows through the following path, never touching the public internet:
**Payments VPC → Interface Endpoint → PrivateLink → Internal NLB → Target Group →
Shared Services EC2**

### What We Built

- **Three isolated VPCs** : Payments, Analytics, and Shared Services. Each environment
    operates independently with controlled access to shared internal services, reflecting real
    enterprise separation for security and compliance.
- **Private internal service** : An EC2 instance in the Shared Services VPC running Apache,
    representing any internal API (billing, identity, logging, central metrics).
- **PrivateLink Provider** : A Target Group, internal NLB, and VPC Endpoint Service that
    expose the internal API securely without any public access.
- **PrivateLink Consumers** : Interface Endpoints in both Payments and Analytics VPCs. All
    traffic stays inside AWS's private network no VPC peering, no NAT Gateways, no public
    exposure.

### Enterprise Access Patterns

In real-world architectures, PrivateLink services are consumed only from:

- EC2 instances inside private VPCs
- Corporate networks connected via VPN or AWS Direct Connect
- Bastion hosts (jump hosts) inside AWS
- Internal developer machines on secure networks
They are never meant to be reached from a laptop on the public internet, this is the strongest possible
security boundary.


## Verification & Validation

### Proving No Internet Exposure

From your local machine, attempt to reach the NLB DNS:
$ curl <nlb-dns-name>
# Expected result: Timeout or Connection refused
A timeout confirms the architecture is correct:

- The NLB is internal-only and not internet-facing
- The EC2 application is not exposed to the internet
- PrivateLink is the only access path
- Route tables are blocking all internet-bound paths
- No part of the architecture is publicly reachable
If you were able to open the Shared Services application from your local browser, the
architecture would be incorrect. **The inability to access it is the strongest indicator that
PrivateLink is working securely and as designed.**


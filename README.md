# Secure VPC Build and Web Server Deployment on AWS

## Project Overview

This project demonstrates how to design and build a production-style AWS network from scratch and deploy a working web server on top of it. A custom VPC was built with public and private subnets across two Availability Zones, routing was configured through an Internet Gateway and a NAT Gateway, access was locked down with a Security Group, and an EC2 instance was launched to serve a web application over HTTP.

The main goal of this project was to demonstrate hands-on experience with core AWS networking concepts (VPC, subnets, route tables, IGW, NAT) and how they work together to support a secure, highly-available application architecture.

## Architecture

```
Internet
   ↓
Internet Gateway (IGW)
   ↓
Public Route Table ──── Public Subnet 1 (10.0.0.0/24)
   │                     Public Subnet 2 (10.0.2.0/24) → EC2 "Web Server 1" (Apache)
   │
Private Route Table ──── Private Subnet 1 (10.0.1.0/24)
                          Private Subnet 2 (10.0.3.0/24)
                               ↓
                          NAT Gateway → outbound internet access only
```

**VPC:** `Lab VPC` — `10.0.0.0/16`

| Subnet | CIDR | Type |
|---|---|---|
| Public Subnet 1 | 10.0.0.0/24 | Public |
| Private Subnet 1 | 10.0.1.0/24 | Private |
| Public Subnet 2 | 10.0.2.0/24 | Public |
| Private Subnet 2 | 10.0.3.0/24 | Private |

- **Internet Gateway (IGW):** Enables inbound/outbound internet access for the public subnets.
- **NAT Gateway (single AZ):** Lets private subnet instances reach the internet outbound (updates, package installs) without being publicly reachable themselves.
- **Route Tables:** A Public Route Table associated with both public subnets, and a Private Route Table associated with both private subnets.

## AWS Services Used

- Amazon VPC
- Subnets (public and private)
- Route Tables
- Internet Gateway (IGW)
- NAT Gateway
- Security Groups
- Amazon EC2

## How It Works

1. A custom VPC (`10.0.0.0/16`) is created as the network foundation.
2. The VPC wizard generates the baseline networking components — subnets, route tables, IGW, and NAT.
3. Two additional subnets (a second public and a second private) are added to extend the design across a second Availability Zone.
4. Each subnet is associated with the correct route table so public subnets route to the IGW and private subnets route outbound through the NAT Gateway.
5. A Security Group is created to control inbound traffic at the instance level.
6. An EC2 instance is launched into a public subnet, bootstrapped via user data, and tested from a browser to confirm the web app is reachable.

## Implementation

### Step 1 – VPC and Subnet Setup
**Lab VPC** (`10.0.0.0/16`) was created using the VPC wizard, which generated the initial public/private subnet pair, route tables, IGW, and NAT Gateway. The design was then extended to a second Availability Zone by adding:
- **Public Subnet 2** — `10.0.2.0/24`
- **Private Subnet 2** — `10.0.3.0/24`

This multi-AZ layout reflects a high-availability design pattern, even though this lab only deploys a single EC2 instance.

### Step 2 – Routing Configuration
- **Public Subnet 2** was associated with the **Public Route Table** (route to the Internet Gateway).
- **Private Subnet 2** was associated with the **Private Route Table** (route out through the NAT Gateway).
- Result: public subnets have a direct path to the internet, while private subnets can only reach the internet outbound through NAT — they remain unreachable from the outside.

### Step 3 – Security Group Configuration
A **Web Security Group** was created with:
- Inbound rule: **HTTP (port 80)** from **Anywhere-IPv4 (0.0.0.0/0)**
- Everything else blocked by default (Security Groups are implicit deny)

### Step 4 – EC2 Deployment
An **Amazon Linux 2**, **t3.micro** instance named **Web Server 1** was launched into **Public Subnet 2** with a public IP enabled and the **Web Security Group** attached. The instance was bootstrapped with the following user data script:

```bash
#!/bin/bash
yum install -y httpd mysql php
wget https://aws-tc-largeobjects.s3.us-west-2.amazonaws.com/CUR-TF-100-RESTRT-1/267-lab-NF-build-vpc-web-server/s3/lab-app.zip
unzip lab-app.zip -d /var/www/html/
chkconfig httpd on
service httpd start
```

This installs Apache, PHP, and MySQL client tools, pulls down a sample application from S3, deploys it to the web root, and starts the Apache service on boot.

### Step 5 – Testing
- Waited for the instance to reach **2/2 status checks passed** in the EC2 console.
- Opened the instance's **Public IPv4 DNS** in a browser.
- Confirmed the web application loaded successfully over HTTP.

## Screenshots

*(Screenshots to be added, each with a short caption.)*

- VPC and subnet layout in the AWS console
- Route table associations
- Security Group inbound rules
- EC2 instance details showing 2/2 status checks passed
- Web application loading successfully in the browser

## Challenges and Troubleshooting

- **http/https:** When attempting to open the website in a new tab, the browser kept returning an error. After about an hour of debugging it turned out that a browser extension was turning the http address into an https address which caused it to return an error. After running it on the mobile version of Firefox which didn't have the extension, it displayed the website.

## Security Considerations

- **Network segmentation:** Public and private subnets separate internet-facing resources from internal-only resources.
- **Security Groups:** Instance-level firewall allowing only HTTP (80) inbound; all other inbound traffic is denied by default.
- **NAT Gateway:** Lets private subnet resources initiate outbound connections without exposing them to inbound internet traffic.
- **Least privilege routing:** Private subnets have no direct route to the Internet Gateway, so they can't be reached directly from the internet.

## Key Takeaways

This project provided practical experience with:

- Designing a VPC and planning IPv4 subnet CIDR ranges
- Separating public and private subnets and understanding when to use each
- The difference between an Internet Gateway and a NAT Gateway, and when each is appropriate
- Configuring Security Groups as instance-level firewalls
- Launching and bootstrapping EC2 instances with user data scripts
- Validating a deployment end-to-end, from instance status checks to a working web page in the browser

## Outcome

The project was successfully completed and tested. The final environment consists of a custom two-AZ VPC with properly segmented public/private subnets and routing, a locked-down Security Group, and an EC2 instance running Apache that serves a web application reachable over HTTP from the internet.

## Future Improvements

- Add Infrastructure as Code (Terraform or AWS CloudFormation) to make the network reproducible
- Place the EC2 instance behind an Application Load Balancer across both public subnets for higher availability
- Add a second NAT Gateway (one per AZ) to remove the single point of failure
- Add CloudWatch monitoring and alarms for the EC2 instance
- Move to HTTPS with a TLS certificate instead of plain HTTP

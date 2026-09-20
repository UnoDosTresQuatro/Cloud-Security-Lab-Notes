# AWS Secure VPC Lab

**Network Segmentation | Access Control | Monitoring | Incident Response**

## Overview

This project documents my hands-on work building and securing a segmented network in AWS.

I manually configured the environment to better understand how VPC networking, routing, Security Groups, Network ACLs, monitoring, and incident response work together in a cloud environment.

The lab uses a public-facing web server with private application and database servers. I tested communication between each tier, configured CloudWatch logging, troubleshot configuration and permission issues, and practiced incident response by isolating an EC2 instance with a dedicated isolation Security Group.

## Project Goals

The goal of this lab was to move beyond learning individual AWS services and understand how multiple networking and security controls work together inside a cloud environment.

During the project, I focused on:

- Building a custom VPC and subnet structure
- Separating public-facing and private resources
- Controlling traffic with Security Groups
- Testing Network ACL behavior and rule precedence
- Configuring public and private routing
- Providing outbound internet access to private resources through a NAT Gateway
- Testing communication between the web, application, and database tiers
- Sending Apache logs to Amazon CloudWatch
- Troubleshooting AWS permissions, Linux permissions, and connectivity issues
- Practicing incident response by isolating an EC2 instance

## Architecture Diagram

The lab was built inside a custom `10.0.0.0/16` VPC with separate public and private network segments.

The public subnet (`10.0.1.0/24`) contains the web server and provides the internet-facing portion of the environment.

The private subnet (`10.0.2.0/24`) contains the application and database servers. These systems are not intended to be directly reachable from the public internet.

Traffic between the tiers is restricted using Security Groups so that each server only communicates with the resources required for its role.

> Architecture diagram will be added here.

## Architecture Components

| Component | Purpose |
|---|---|
| VPC | Provides the isolated AWS network for the lab |
| Public Subnet | Hosts the internet-facing web server |
| Private Subnet | Hosts the application and database servers |
| Internet Gateway | Provides internet connectivity for public resources |
| NAT Gateway | Provides outbound internet access for private resources |
| Public Route Table | Routes internet-bound public traffic to the Internet Gateway |
| Private Route Table | Routes outbound private traffic through the NAT Gateway |
| Web Security Group | Controls access to the web tier |
| App Security Group | Allows application traffic from the web tier |
| DB Security Group | Allows database traffic from the application tier |
| Network ACLs | Provides subnet-level stateless traffic filtering |
| CloudWatch | Collects Apache logs for monitoring and investigation |
| CL-Isolation-SG | Restricts an EC2 instance during incident-response containment |

## Security Design

The environment follows a tiered access model rather than allowing every server to communicate freely.

**Internet → Web Server**

Public web traffic is allowed to reach the web tier.

**Web Server → Application Server**

Application traffic is allowed on TCP port `8080`.

**Application Server → Database Server**

Database traffic is allowed on TCP port `3306`.

Direct communication paths that are not required by the architecture are restricted.

This design helped me understand how network segmentation can reduce unnecessary access between systems and limit the potential impact of a compromised resource.

## Next Sections

The remainder of this project documents:

- Traffic Flow
- Subnet Design
- Route Tables
- Security Groups
- Security Groups vs. Network ACLs
- NACL Implementation and Testing
- EC2 Deployment
- Connectivity Testing
- CloudWatch Logging and Monitoring
- Troubleshooting
- Incident Response and Host Isolation
- What I Learned
- AI-Assisted Learning

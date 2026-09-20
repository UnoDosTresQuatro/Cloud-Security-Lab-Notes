# AWS Secure VPC Lab

**Network Segmentation | Access Control | Monitoring | Incident Response**

## Overview

This project documents my hands-on work building and securing a segmented network in AWS.

I built the environment manually through the AWS console to develop a better understanding of how VPC networking, routing, Security Groups, Network ACLs, monitoring, and incident response work together.

The environment uses a public web tier and private application and database tiers. I configured the network controls between each layer, tested allowed and blocked traffic, implemented CloudWatch logging, troubleshot configuration issues, and practiced isolating an EC2 instance during an incident-response scenario.

---

## Architecture

```text
┌─────────────────────────────────────────────────────────────────────┐
│                       AWS REGION (us-east-1)                        │
│                                                                     │
│   ┌────────────────────── VPC 10.0.0.0/16 ──────────────────────┐   │
│   │                                                             │   │
│   │   PUBLIC SUBNET                    PRIVATE SUBNET            │   │
│   │   10.0.1.0/24                      10.0.2.0/24              │   │
│   │                                                             │   │
│   │   ┌─────────────────┐              ┌─────────────────┐      │   │
│   │   │  CL-Web-Server  │              │  CL-App-Server  │      │   │
│   │   │    10.0.1.11    │── TCP 8080 ─►│    10.0.2.85    │      │   │
│   │   └─────────────────┘              └────────┬────────┘      │   │
│   │                                             │               │   │
│   │                                         TCP 3306            │   │
│   │                                             │               │   │
│   │                                             ▼               │   │
│   │                                    ┌─────────────────┐      │   │
│   │   ┌─────────────────┐              │  CL-DB-Server   │      │   │
│   │   │   NAT Gateway   │              │   10.0.2.133    │      │   │
│   │   └─────────────────┘              └─────────────────┘      │   │
│   │                                                             │   │
│   └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Architecture Components

| Component | Purpose |
|---|---|
| VPC | Isolated network boundary for the lab |
| Public Subnet | Hosts the public-facing web server |
| Private Subnet | Hosts the application and database servers |
| Internet Gateway | Provides internet connectivity to the VPC |
| NAT Gateway | Provides outbound internet access for private resources |
| Web Server | Public-facing EC2 instance |
| App Server | Private EC2 instance running the application tier |
| DB Server | Private EC2 instance representing the database tier |
| Security Groups | Instance-level stateful access control |
| Network ACLs | Subnet-level stateless traffic filtering |
| CloudWatch | Centralized collection of Apache logs |

---

## Traffic Flow

```text
INBOUND / APPLICATION FLOW

Internet
   │
   ▼
Internet Gateway
   │
   ▼
CL-Web-Server
10.0.1.11
   │
   │ TCP 8080
   ▼
CL-App-Server
10.0.2.85
   │
   │ TCP 3306
   ▼
CL-DB-Server
10.0.2.133


PRIVATE OUTBOUND FLOW

Private Resources
   │
   ▼
Private Route Table
   │
   ▼
NAT Gateway
   │
   ▼
Internet Gateway
   │
   ▼
Internet
```

The application path only permits the communication required between tiers. The database is not directly exposed to the web tier or public internet.

---

## Subnet Design

| Subnet | CIDR Range | Internet Access | Purpose |
|---|---|---|---|
| Public | `10.0.1.0/24` | Direct through IGW | Web tier and NAT Gateway |
| Private | `10.0.2.0/24` | Outbound through NAT | Application and database tiers |

The public and private subnet design separates internet-facing resources from systems that do not require direct public exposure.

---

## Route Tables

```text
PUBLIC ROUTE TABLE

Destination        Target
--------------------------------
10.0.0.0/16        local
0.0.0.0/0          Internet Gateway


PRIVATE ROUTE TABLE

Destination        Target
--------------------------------
10.0.0.0/16        local
0.0.0.0/0          NAT Gateway
```

The public route allows internet-bound traffic to reach the Internet Gateway.

The private route provides outbound connectivity through the NAT Gateway without directly exposing the private instances to inbound internet traffic.

---

## Security Group Design

```text
Internet
   │
   ▼
┌─────────────────┐
│   Web Security  │
│      Group      │
└────────┬────────┘
         │
         │ TCP 8080
         ▼
┌─────────────────┐
│   App Security  │
│      Group      │
└────────┬────────┘
         │
         │ TCP 3306
         ▼
┌─────────────────┐
│    DB Security  │
│      Group      │
└─────────────────┘
```

The Security Groups were configured around the required communication path rather than allowing unrestricted communication between the instances.

This allowed me to test both permitted and intentionally blocked connections between the three tiers.

---

## Security Groups vs. Network ACLs

| Security Groups | Network ACLs |
|---|---|
| Applied at the instance/ENI level | Applied at the subnet level |
| Stateful | Stateless |
| Allow rules only | Allow and deny rules |
| Return traffic is automatically allowed | Return traffic must be explicitly permitted |
| Rules are evaluated as a whole | Rules are processed by rule number |
| Can reference other Security Groups | Primarily uses IP/CIDR-based rules |

During the lab, I tested NACL rule precedence by introducing a lower-numbered deny rule and observing how it affected traffic.

This helped demonstrate the difference between stateful instance-level filtering and stateless subnet-level filtering.

---

## Connectivity Testing

I tested communication between the different tiers to verify that segmentation was working as intended.

```text
TEST                                      RESULT

Home → App private IP                     BLOCKED
Web → App :8080                           ALLOWED
App → DB :3306                            ALLOWED
Web → DB :3306                            BLOCKED
```

These tests confirmed that simply being inside the same VPC does not mean every system should be able to communicate with every other system.

Access still depends on routing and the security controls applied to the traffic path.

---

## CloudWatch Logging

I configured the CloudWatch Agent on the web server to collect Apache logs.

The monitored files included:

```text
/var/log/httpd/access_log
/var/log/httpd/error_log
```

During configuration, I encountered issues involving IAM permissions, an incorrect log path, and Linux file permissions.

Troubleshooting these problems helped me understand that cloud monitoring depends on both AWS permissions and permissions inside the operating system.

---

## Troubleshooting

Several parts of the lab did not work correctly on the first attempt.

Issues I worked through included:

- IAM role and permission configuration
- CloudWatch Agent configuration
- Apache log-path configuration
- Linux directory and file permissions
- Security Group connectivity
- Network ACL rule ordering
- Private-instance connectivity
- SSH access between instances

Rather than rebuilding the environment when something failed, I worked through each layer to determine where communication or permissions were breaking.

---

## Incident Response and Host Isolation

I also practiced a basic containment scenario using an isolation Security Group.

`CL-Isolation-SG` was created with tightly restricted access and then attached to the web server in place of its normal Security Group.

```text
NORMAL STATE

CL-Web-Server
     │
     ▼
CL-Web-SG


CONTAINMENT

CL-Web-Server
     │
     ▼
CL-Isolation-SG
```

The purpose of this exercise was to simulate isolating a potentially compromised EC2 instance while retaining controlled administrative access for investigation.

---

## What I Learned

This project helped connect several concepts that I had previously studied separately.

The biggest takeaway was that VPC security is built in layers. Subnets determine where resources live, route tables determine where traffic can travel, Security Groups control instance-level access, Network ACLs provide subnet-level filtering, and monitoring provides visibility into what is happening inside the environment.

Testing failed connections was just as useful as testing successful ones because it forced me to identify which control was responsible for allowing or blocking the traffic.

---

## AI-Assisted Learning

I used ChatGPT as a learning and troubleshooting resource while completing this project.

AI assistance was used to help explain unfamiliar concepts, troubleshoot configuration problems, organize documentation, and challenge my understanding of the environment.

The AWS resources, Linux commands, connectivity tests, troubleshooting steps, and incident-response exercises documented in this project were performed hands-on by me.

My goal in using AI was not simply to receive configuration steps, but to understand why each component was required and be able to explain the architecture and security decisions myself.

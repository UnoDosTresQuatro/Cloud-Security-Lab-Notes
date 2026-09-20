# AWS Cowrie SSH Honeypot Lab

**SSH Honeypot | Threat Monitoring | Log Analysis | Linux Networking**

## Overview

This project documents my hands-on deployment and investigation of a Cowrie SSH honeypot running on an AWS EC2 instance.

The goal was to build an isolated internet-facing system that could receive unsolicited SSH traffic while keeping the real administrative SSH service separate from the honeypot.

I configured Cowrie to listen on TCP port `2222`, moved the EC2 instance's real OpenSSH service to TCP port `22222`, and used Linux `nftables` to redirect incoming traffic from the standard SSH port `22` to Cowrie.

Once the honeypot was reachable from the internet, I monitored Cowrie's JSON logs and began analyzing real unsolicited connections. I used source IP addresses, session IDs, SSH client versions, authentication events, connection duration, and other event data to reconstruct individual sessions.

I also generated a controlled SSH session so I could compare known activity against unknown internet traffic.

---

## Lab Architecture

```text
                         INTERNET
                             │
                             │ TCP 22
                             ▼
                    ┌─────────────────┐
                    │ AWS EC2 Instance│
                    │ Sec-Lab-Cowrie  │
                    └────────┬────────┘
                             │
                             ▼
                    ┌─────────────────┐
                    │    nftables     │
                    │   22 → 2222     │
                    └────────┬────────┘
                             │
                             ▼
                    ┌─────────────────┐
                    │     Cowrie      │
                    │  SSH Honeypot   │
                    │    TCP 2222     │
                    └────────┬────────┘
                             │
                             ▼
                       cowrie.json
                             │
                             ▼
                        Log Analysis


ADMINISTRATION PATH

Administrator
     │
     │ TCP 22222
     ▼
┌─────────────────┐
│  Real OpenSSH   │
│     sshd        │
└─────────────────┘
```

The key part of the architecture is that the honeypot and the real administrative SSH service do not use the same listening port.

Internet traffic targeting normal SSH on TCP `22` is redirected to Cowrie, while legitimate administrative access uses TCP `22222`.

---

## Port Design

| Port | Service | Purpose |
|---|---|---|
| `22` | nftables redirect | Public-facing SSH entry point |
| `2222` | Cowrie | SSH honeypot service |
| `22222` | OpenSSH / sshd | Real administrative SSH access |

This design allowed Cowrie to appear on the standard SSH port without requiring the Cowrie process itself to directly listen on privileged TCP port `22`.

It also kept the actual administrative SSH service separate from the honeypot.

---

## Cowrie Deployment

Cowrie was installed on a Linux EC2 instance using a Python virtual environment.

The honeypot was configured to listen on:

```text
0.0.0.0:2222
```

I verified the listening service during troubleshooting:

```text
LISTEN 0 50 0.0.0.0:2222 0.0.0.0:*
```

The real OpenSSH service was configured separately on:

```text
0.0.0.0:22222
[::]:22222
```

This meant the real SSH daemon was no longer serving administrative SSH on the standard port being presented to internet scanners.

---

## SSH Traffic Redirection

Because Cowrie listened internally on TCP `2222`, I used `nftables` to redirect incoming TCP `22` traffic to the honeypot.

```text
Internet
   │
   │ TCP 22
   ▼
nftables
   │
   │ redirect
   ▼
TCP 2222
   │
   ▼
Cowrie
```

I inspected the active rules with:

```bash
sudo nft list ruleset
```

The nftables configuration was saved to:

```text
/etc/nftables.conf
```

so the redirect could persist rather than existing only as a temporary runtime rule.

---

## Administrative SSH Separation

The real OpenSSH service was moved to TCP port `22222`.

This produced two separate traffic paths:

```text
HONEYPOT TRAFFIC

Internet
   │
   ▼
TCP 22
   │
   ▼
nftables
   │
   ▼
TCP 2222
   │
   ▼
Cowrie


ADMINISTRATIVE TRAFFIC

Administrator
   │
   ▼
TCP 22222
   │
   ▼
Real sshd
```

Separating the two services allowed me to administer the EC2 instance without using the honeypot service itself.

---

## Local Testing

Before relying on external traffic, I tested Cowrie directly from the EC2 instance.

```bash
ssh root@127.0.0.1 -p 2222
```

This allowed me to verify Cowrie independently of the AWS Security Group, public networking, and TCP `22` redirect.

Testing locally helped separate application problems from networking problems.

If Cowrie did not respond locally, I could investigate Cowrie itself. If it worked locally but could not be reached externally, I could move outward and investigate the firewall, Security Group, routing, or redirect configuration.

---

## Service Management

Cowrie was configured to run as a Linux `systemd` service so the honeypot would not depend on an interactive terminal remaining open.

I checked the service using:

```bash
systemctl status cowrie
```

During setup, the service had trouble locating Cowrie's `twistd` executable.

Because Cowrie was installed inside a Python virtual environment, the systemd service environment needed to correctly locate the executable.

Troubleshooting this issue helped me understand the difference between running an application manually from an activated Python environment and having Linux launch the application as a managed service.

---

## Cowrie Logging

Cowrie records activity as structured events inside its JSON log.

The primary log I analyzed was:

```text
cowrie.json
```

Instead of treating the log as one large block of text, I focused on individual fields that could help reconstruct a session.

Important fields included:

```text
TIME
EVENT
SESSION
SOURCE IP
SOURCE PORT
CLIENT VERSION
USERNAME
PASSWORD
COMMAND
DURATION
DETAIL
```

Cowrie generates multiple events during a single SSH interaction, so the session ID became especially important for connecting related events.

---

## Commands Used During the Lab

### Verify Listening Ports

I used:

```bash
ss -tln
```

and:

```bash
ss -tulnp
```

to inspect listening TCP ports and verify that Cowrie and the real SSH service were running on the intended ports.

The final design was:

```text
TCP 2222   → Cowrie
TCP 22222  → Real OpenSSH / sshd
```

### Test Cowrie Locally

```bash
ssh root@127.0.0.1 -p 2222
```

This allowed me to verify Cowrie directly without relying on the external AWS network path.

### Inspect nftables

```bash
sudo nft list ruleset
```

This allowed me to verify the TCP `22 → 2222` redirect.

### Check Cowrie Service

```bash
systemctl status cowrie
```

This was used to verify that Cowrie was running correctly as a managed Linux service.

### View Recent Cowrie Events

```bash
tail -n 20 var/log/cowrie/cowrie.json
```

This allowed me to inspect recently generated events.

### Monitor Events in Real Time

```bash
tail -f var/log/cowrie/cowrie.json
```

This allowed me to watch new Cowrie events appear as connections reached the honeypot.

---

## Understanding Cowrie Events

One SSH connection can generate several different Cowrie events.

A session can progress through events such as:

```text
cowrie.session.connect
        │
        ▼
cowrie.client.version
        │
        ▼
cowrie.client.kex
        │
        ▼
Authentication Events
        │
        ▼
Terminal / TTY Events
        │
        ▼
Command Events
        │
        ▼
cowrie.session.closed
```

Not every connection progresses through every stage.

Some systems connect and disconnect almost immediately, while others continue far enough to perform SSH negotiation, submit credentials, open a terminal, or execute commands.

This made the session ID important because it allowed multiple log entries to be correlated as part of the same interaction.

---

## Controlled SSH Session

I generated my own known SSH interaction with the honeypot so I could see what a successful Cowrie session looked like in the logs.

The controlled test included:

```text
SSH Client: OpenSSH_for_Windows_9.5
Username:   root
Password:   password
Result:     Cowrie authentication accepted
Duration:   approximately 3 minutes
```

Cowrie generated multiple events associated with the same session:

```text
Connection
    │
    ▼
SSH Client Identification
    │
    ▼
Key Exchange
    │
    ▼
Authentication
    │
    ▼
Cowrie Login Accepted
    │
    ▼
Terminal Session
    │
    ▼
Session Closed
```

This gave me a baseline where I already knew what activity had occurred.

I could then compare this known session against unsolicited internet connections and identify differences in duration, SSH client information, authentication activity, and session behavior.

---

## Unsolicited Internet Activity

Once TCP port `22` was exposed to the internet and redirected to Cowrie, the honeypot began recording connections from external systems.

One example captured in the logs was:

```text
Source IP: 176.212.91.116
Destination: 10.10.1.174:2222
```

Cowrie recorded multiple connection attempts from this source using different source ports.

Example:

```text
TIME:    2026-09-19T19:17:01.209116Z
EVENT:   cowrie.session.connect
SESSION: ab2ad8d46a6d

DETAIL:
New connection:
176.212.91.116:56908
        │
        ▼
10.10.1.174:2222
```

The session then closed almost immediately:

```text
EVENT: cowrie.session.closed

DETAIL:
Connection lost after 0 milliseconds
```

A second connection from the same source appeared shortly afterward using another source port.

This was useful because it demonstrated that repeated connections from one IP do not necessarily represent separate people. Automated systems can repeatedly probe the same service using new TCP connections.

---

## Recent Findings

As I continued monitoring the honeypot, I observed multiple external systems interacting with the SSH service.

The goal of the investigation was not simply to collect IP addresses. I wanted to determine how far each connection progressed and what information Cowrie recorded about it.

### Finding 1 — Go SSH Client

One unsolicited session presented the SSH client version:

```text
SSH-2.0-Go
```

The session was very short and did not progress into meaningful interactive activity.

The Cowrie events provided information such as:

```text
SSH Client: SSH-2.0-Go
Session Duration: Very short
Credentials: None observed
Commands: None observed
```

Because Go-based SSH software can be used for many purposes, the client string alone does not prove malicious intent.

What I could establish from the logs was that an automated-looking SSH client reached the honeypot and disconnected without progressing into an interactive session.

---

## Finding 2 — ZGrab SSH Survey

Another connection produced a more descriptive SSH client banner:

```text
SSH-2.0-ZGrab ZGrab SSH Survey
```

This was especially useful because the software identified itself during SSH negotiation.

The interaction did not progress into a normal interactive login session.

No commands were observed from the session.

The finding demonstrated that SSH client metadata can provide additional context beyond simply knowing that a remote IP connected to TCP port `22`.

```text
Remote Connection
       │
       ▼
Cowrie Records Client Version
       │
       ▼
SSH-2.0-ZGrab ZGrab SSH Survey
       │
       ▼
Additional Investigation Context
```

Rather than labeling every connection as a successful attack, I used the recorded behavior to describe what could actually be supported by the logs.

---

## Short-Lived Connection Analysis

Several connections closed very quickly.

An example was:

```text
Connection established
        │
        ▼
Cowrie session created
        │
        ▼
Little or no further interaction
        │
        ▼
Connection closed
```

Possible explanations for this type of behavior include automated scanning, service discovery, SSH fingerprinting, or a client deciding not to continue after receiving information from the SSH service.

The important lesson was that:

```text
Connection ≠ Authentication
Authentication ≠ Command Execution
Command Execution ≠ Full Compromise
```

Each stage requires evidence from the logs.

---

## Session Correlation

Instead of analyzing individual log lines independently, I used Cowrie session IDs to group related activity.

```text
Source IP
    │
    ▼
Connection
    │
    ▼
Session ID
    │
    ├── Client Version
    │
    ├── Key Exchange
    │
    ├── Authentication
    │
    ├── Terminal Activity
    │
    ├── Commands
    │
    └── Session Closed
```

This allowed me to reconstruct the sequence of events associated with one SSH connection.

A source IP tells me where the connection came from.

The session ID helps tell me what happened after it arrived.

---

## Investigation Workflow

As I analyzed Cowrie events, I began using a repeatable investigation process:

```text
Who connected?
      │
      ▼
What source IP was recorded?
      │
      ▼
What session ID belongs to the connection?
      │
      ▼
What SSH client identified itself?
      │
      ▼
Was authentication attempted?
      │
      ▼
Were credentials submitted?
      │
      ▼
Was a shell opened?
      │
      ▼
Were commands executed?
      │
      ▼
How did the session end?
```

This changed the project from simply running a honeypot into practicing basic security-event investigation.

---

## What the Captured Traffic Demonstrated

The sessions showed that SSH activity can stop at different stages.

```text
TCP Connection
      │
      ▼
SSH Client Identification
      │
      ▼
Key Exchange
      │
      ▼
Authentication Attempt
      │
      ▼
Honeypot Login
      │
      ▼
Interactive Session
      │
      ▼
Command Execution
```

Not every connection reached the bottom of this sequence.

Some connections disappeared almost immediately.

Others revealed SSH client information.

My controlled test progressed further and generated authentication and terminal-related events.

Seeing these differences in the logs helped me understand why individual events need to be correlated before determining what actually occurred.

---

## Security Observations

This lab helped me understand the difference between seeing network activity and determining what that activity actually means.

### Public Services Receive Unsolicited Traffic

Once the SSH honeypot became reachable from the internet, it began receiving connections without being advertised.

This demonstrated why internet-facing services should be treated as potentially discoverable.

### A Connection Is Not a Compromise

One of the most important lessons from analyzing the logs was learning not to overstate what an event means.

```text
Connection
    ≠
Successful Authentication
    ≠
Command Execution
    ≠
System Compromise
```

A `cowrie.session.connect` event proves that a connection reached Cowrie.

Additional events are needed to determine whether authentication was attempted, credentials were submitted, a shell was opened, or commands were executed.

### Client Metadata Adds Context

SSH client banners such as:

```text
SSH-2.0-Go
```

and:

```text
SSH-2.0-ZGrab ZGrab SSH Survey
```

provided additional information about the software interacting with the honeypot.

Client identification alone does not establish intent, but it can help provide context during an investigation.

### Session IDs Are Important

A source IP by itself does not tell the complete story.

Cowrie's session identifiers allowed multiple events to be connected together and analyzed as one interaction.

```text
Source IP
   │
   ▼
Session ID
   │
   ├── Connection
   ├── Client Information
   ├── Authentication
   ├── Terminal Activity
   ├── Commands
   └── Disconnect
```

This introduced me to the basic idea of event correlation used in security monitoring.

---

## Troubleshooting

A large part of this project involved troubleshooting rather than simply installing Cowrie.

### Cowrie Listening Port

I first needed to confirm that Cowrie was actually listening on the expected port.

```bash
ss -tln
```

This confirmed:

```text
0.0.0.0:2222
```

If nothing was listening on the port, changing AWS networking rules would not solve the problem.

This reinforced the importance of troubleshooting from the application outward.

---

### SSH Command Syntax

While testing Cowrie locally, I initially entered the SSH command incorrectly.

The correct format was:

```bash
ssh root@127.0.0.1 -p 2222
```

This helped reinforce the structure of an SSH command:

```text
ssh USER@HOST -p PORT
```

---

### Separating Cowrie From Real SSH

Running a honeypot on an SSH server required making sure the real administrative SSH service did not conflict with the honeypot.

The final design became:

```text
TCP 22
   │
   ▼
nftables
   │
   ▼
TCP 2222
   │
   ▼
Cowrie


TCP 22222
   │
   ▼
Real OpenSSH
```

This allowed the honeypot to receive standard SSH traffic while preserving a separate administrative path.

---

### nftables Configuration

I had to verify that traffic reaching TCP port `22` was actually being redirected to Cowrie.

I inspected the active configuration using:

```bash
sudo nft list ruleset
```

This helped me understand that Cowrie could be working correctly on `2222` while external connections still failed if the redirect between `22` and `2222` was not configured correctly.

---

### systemd and the Python Virtual Environment

Cowrie was installed inside a Python virtual environment.

When I attempted to run Cowrie as a managed `systemd` service, the service initially had trouble locating the required `twistd` executable.

The application could work when launched manually from the virtual environment while still failing when started by `systemd`.

This introduced an important Linux administration concept:

```text
Interactive Shell Environment
             ≠
      systemd Environment
```

The service configuration needed access to the correct Cowrie/Python environment.

After troubleshooting the service configuration, Cowrie could run independently of my interactive SSH session.

---

### Understanding Cowrie Authentication

Testing the honeypot also required understanding that authentication inside Cowrie is simulated.

A successful login to Cowrie does not mean the user authenticated to the underlying EC2 operating system.

```text
Remote User
     │
     ▼
Cowrie Authentication
     │
     ▼
Simulated Environment

NOT

Remote User
     │
     ▼
Real EC2 Operating System
```

This distinction is one of the reasons a honeypot can safely collect information about attempted interactions without intentionally providing access to the real host.

---

### Reading JSON Logs

At first, the amount of information inside `cowrie.json` made individual events difficult to interpret.

Breaking the events into fields made the logs easier to investigate:

```text
Timestamp
Source IP
Session ID
Event Type
SSH Client
Username
Password
Command
Duration
```

From there, I could follow a single session rather than reading unrelated events chronologically.

---

## Lessons From Troubleshooting

The troubleshooting process gave me a better method for diagnosing service connectivity.

Instead of immediately changing AWS settings when something failed, I learned to check the environment layer by layer.

```text
Is the application running?
          │
          ▼
Is it listening on the expected port?
          │
          ▼
Does it work locally?
          │
          ▼
Is the Linux firewall/redirect correct?
          │
          ▼
Does AWS allow the traffic?
          │
          ▼
Can the service be reached externally?
```

This approach prevents randomly changing configurations without first identifying which layer is actually failing.

---

## What I Learned

This project started as an exercise in deploying a honeypot, but it became an introduction to several areas of cybersecurity at the same time.

I gained hands-on experience with:

- AWS EC2 networking
- Linux listening ports
- SSH services
- `nftables`
- Port redirection
- Python virtual environments
- `systemd` services
- Cowrie SSH emulation
- JSON security logs
- Session correlation
- SSH client identification
- Basic security-event investigation

One of my biggest takeaways was that collecting logs is only the beginning.

The real value comes from asking questions about those logs and connecting related events together.

For example:

```text
An IP connected.
```

is less useful than:

```text
An IP connected
      │
      ▼
using this SSH client
      │
      ▼
created this session
      │
      ▼
attempted these actions
      │
      ▼
and disconnected this way.
```

That is the beginning of turning raw event data into an investigation.

---

## Future Improvements

The next step for this project would be sending Cowrie events to a centralized security monitoring platform.

I explored deploying a SIEM but decided not to keep an additional AWS instance running because of the resource requirements and cost associated with that deployment.

A future version of the lab could integrate Cowrie with a SIEM to practice:

```text
Cowrie
   │
   ▼
Centralized Log Collection
   │
   ▼
SIEM
   │
   ├── Search
   ├── Dashboards
   ├── Correlation
   └── Alerts
```

Other future improvements could include analyzing authentication attempts, commonly attempted usernames, commands, source-address frequency, and activity patterns across longer monitoring periods.

---

## Portfolio Takeaway

This project demonstrates more than the installation of a honeypot.

I built and troubleshot a system that combined:

```text
AWS
 │
 ▼
Linux Networking
 │
 ▼
SSH Port Separation
 │
 ▼
nftables Redirection
 │
 ▼
Cowrie
 │
 ▼
Structured Logging
 │
 ▼
Event Correlation
 │
 ▼
Security Analysis
```

The project gave me practical experience following network traffic from the cloud layer through the Linux operating system and into an application, then using the application's logs to investigate the resulting activity.

---

## AI-Assisted Learning

I used ChatGPT as a learning and troubleshooting resource throughout this project.

AI assistance was used to help explain unfamiliar Linux, AWS, networking, Cowrie, and log-analysis concepts; troubleshoot configuration problems; test my understanding; and organize the final project documentation.

The AWS configuration, Linux commands, SSH testing, Cowrie deployment, `nftables` configuration, service troubleshooting, traffic monitoring, and log investigation documented in this project were performed hands-on by me.

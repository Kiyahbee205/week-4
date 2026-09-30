# Week 4: EC2 Evidence Lab

## HarborTech Ticket Summary

**Ticket:** TKT-2026-0004
**Client:** Riverside Goods
**Issue:** The client web workload could not be reached through HTTP.

The purpose of this lab was to provision a disposable Amazon Linux EC2 web server, investigate the network configuration, identify the cause of the web-access problem, document the evidence, apply an authorized corrective change, and verify the result.

During the lab, I experienced errors with several CloudShell commands. Some were caused by command syntax, missing resource information, or not having an instance ID available. I documented the errors instead of making up results.

## Client Impact

The missing HTTP access rule could prevent users from reaching the web server through port 80. The security group allowed SSH traffic on TCP port 22, but the evidence collected showed that inbound TCP port 80 was not initially allowed.

Because the server was intended to provide a web page, the missing HTTP rule could make the service unavailable to outside users even if the web server itself was configured correctly.

## Environment and Resource Names

**AWS Region:** Learner Lab Region
**VPC ID:** `vpc-06f0a1ee8424f42dd`
**Security Group:** `launch-wizard-1`
**Security Group ID:** `sg-01fd53d990c7fd090`

The security group was associated with the VPC listed above.

Sensitive AWS account information and personal identifiers were intentionally excluded from this public GitHub document.

## AWS Documentation Evidence

### Security Groups

**Source:** Amazon EC2 User Guide

> “A security group acts as a virtual firewall for your EC2 instances to control incoming and outgoing traffic.”

This supports the troubleshooting process because security-group rules determine what network traffic can reach the EC2 instance. Since the original security group did not contain an inbound TCP 80 rule, HTTP traffic was not allowed through that security group.

AWS documentation:
https://docs.aws.amazon.com/pdfs/AWSEC2/latest/UserGuide/ec2-ug.pdf#ec2-security-groups

### EC2 User Data

**Source:** Amazon EC2 User Guide

User data can be passed to an EC2 instance when it is launched and can contain commands used to configure the instance.

User data can help document how the web server was intended to be configured, but user data by itself does not prove that Apache is currently running or that the web page is reachable.

AWS documentation:
https://docs.aws.amazon.com/pdfs/AWSEC2/latest/UserGuide/ec2-ug.pdf

### Instance Metadata

**Source:** Amazon EC2 User Guide

> “Instance metadata properties are divided into categories, for example, host name, events, and security groups.”

Instance metadata can provide information about the running EC2 instance. This can be useful for confirming instance information from inside the guest operating system.

AWS documentation:
https://docs.aws.amazon.com/pdfs/AWSEC2/latest/UserGuide/ec2-ug.pdf#instancedata-data-retrieval

## CloudShell Command Record

I used AWS CloudShell to inspect the AWS environment and security-group configuration.

### Identity Verification

Command:

```bash
aws sts get-caller-identity
```

The command successfully verified that CloudShell was operating under the expected AWS Learner Lab identity.

**Public GitHub evidence:** Account numbers, user IDs, ARNs, and personal names were removed for security and privacy.

### Region Check

Command:

```bash
aws configure get region
```

No usable Region value was returned in the evidence collected.

### VPC Discovery

Command attempted:

```bash
aws ec2 describe-vpcs \
--query 'Vpcs[].[VpcId,CidrBlock,IsDefault]' \
--output table
```

The command returned an AWS CLI argument error:

```text
aws: [ERROR]: Unknown options:
Vpcs[].[VpcId,CidrBlock,IsDefault],
--output,
table,
--query
```

This was treated as a command syntax/argument problem rather than evidence of an AWS networking failure.

## Baseline Evidence

The security-group inspection command was:

```bash
aws ec2 describe-security-groups
```

The relevant security group was:

```text
GroupName: launch-wizard-1
GroupId: sg-01fd53d990c7fd090
VPC: vpc-06f0a1ee8424f42dd
```

The inbound rules showed:

```text
Protocol: TCP
From Port: 22
To Port: 22
CIDR: 0.0.0.0/0
```

There was no inbound TCP port 80 rule in the security-group evidence collected.

The security group did allow outbound IPv4 traffic.

## Root-Cause Analysis

The strongest evidence collected points to the security-group configuration as the network-level cause of the HTTP access problem.

The security group allowed inbound SSH traffic on TCP port 22, but no inbound TCP port 80 rule was present. Since HTTP normally uses TCP port 80, outside HTTP traffic would not be allowed through this security group.

I did not conclude that the operating system or Apache installation was damaged because I did not collect enough successful guest-level evidence to support that conclusion.

Several commands also produced errors, so complete end-to-end testing could not be confirmed from every required test path.

## Corrective Action

The planned corrective action was to authorize inbound TCP port 80 on the security group.

Command:

```bash
aws ec2 authorize-security-group-ingress \
  --group-id sg-01fd53d990c7fd090 \
  --protocol tcp \
  --port 80 \
  --cidr 0.0.0.0/0
```

**Evidence status:** The command was part of the corrective-action procedure, but a successful command output was not collected in my available evidence.

A production environment should not receive this type of network change without authorization because allowing `0.0.0.0/0` would expose the service to the public internet.

## Verification Evidence

The intended verification command was:

```bash
aws ec2 describe-security-groups \
  --group-ids sg-01fd53d990c7fd090 \
  --query 'SecurityGroups[0].IpPermissions' \
  --output json
```

The expected evidence would show an inbound TCP rule for port 80.

The HTTP test would then use the EC2 instance's public IPv4 address.

Because I was unable to successfully collect all of the required instance information and verification outputs, I am not claiming that the final web-page test succeeded.

## IMDSv2 and Guest Evidence

The intended guest-level checks were:

```bash
sudo systemctl status httpd --no-pager
```

```bash
curl -I http://localhost
```

For IMDSv2:

```bash
TOKEN=$(curl -X PUT "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
```

Then:

```bash
curl -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/instance-id
```

These commands were intended to verify Apache locally and retrieve the EC2 instance ID using IMDSv2.

**Evidence status:** Complete successful outputs were not collected.

## Stop/Start Lifecycle Test

The intended lifecycle test was to stop the EBS-backed EC2 instance, wait for it to reach the stopped state, start it again, and then retrieve the new public IPv4 address.

Example commands:

```bash
aws ec2 stop-instances --instance-ids INSTANCE_ID
```

```bash
aws ec2 wait instance-stopped --instance-ids INSTANCE_ID
```

```bash
aws ec2 start-instances --instance-ids INSTANCE_ID
```

```bash
aws ec2 wait instance-running --instance-ids INSTANCE_ID
```

The purpose of this test was to observe which EC2 resources persisted and whether the automatically assigned public IPv4 address changed after stopping and starting the instance.

**Evidence status:** Complete stop/start output was not collected.

## Cleanup Evidence

The intended cleanup process was:

```bash
aws ec2 terminate-instances --instance-ids INSTANCE_ID
```

Then:

```bash
aws ec2 wait instance-terminated --instance-ids INSTANCE_ID
```

After termination, the security group could be removed if it was no longer attached:

```bash
aws ec2 delete-security-group \
  --group-id sg-01fd53d990c7fd090
```

**Evidence status:** Complete termination and security-group deletion output was not collected.

No claim is being made that these cleanup commands successfully completed.

## Escalation and Change-Control Notes

One change I would not make in a production environment without authorization is opening inbound TCP port 80 to `0.0.0.0/0`.

The risk is that the change could expose a production web service to the public internet. Before making the change, I would obtain approval from the appropriate system owner or network/security administrator.

I would also document the current security-group rules, the reason for the change, the expected result, and a rollback plan. If the change caused an unexpected exposure, the rollback would be to remove the newly added rule and verify that the original security-group configuration was restored.

## Lessons Learned

This lab showed me that troubleshooting an EC2 web server requires checking multiple layers instead of immediately assuming the server itself is broken. A web page can fail because of the application, operating system, instance configuration, security group, or network settings.

I also learned that command syntax and resource IDs are important when working in CloudShell. I received errors with several commands because information was missing or the command was not entered correctly. Instead of making up results, I documented the errors and used the evidence I could verify.

The biggest lesson for me was to observe, verify, document, analyze, recommend, and escalate instead of making assumptions.

## Professional Vocabulary

* **EC2:** Amazon Elastic Compute Cloud service used to provide virtual servers.
* **Security Group:** A virtual firewall that controls inbound and outbound traffic for EC2 resources.
* **Inbound Rule:** A rule that determines what traffic is allowed to reach a resource.
* **TCP Port 80:** The standard port commonly used for HTTP web traffic.
* **HTTP:** Hypertext Transfer Protocol used for web communication.
* **User Data:** Commands or scripts provided to an EC2 instance during launch.
* **Instance Metadata:** Information about an EC2 instance that can be retrieved from inside the instance.
* **IMDSv2:** Version 2 of the EC2 Instance Metadata Service that uses a session token.
* **CloudShell:** A browser-based shell environment provided by AWS.
* **EBS:** Elastic Block Store, which provides persistent block storage for EC2.
* **Public IPv4 Address:** An internet-routable IPv4 address assigned to an EC2 instance.
* **Change Control:** The process of reviewing and approving changes before they are made to an environment.
* **Root Cause:** The underlying reason a problem occurred.
* **Verification:** Testing or reviewing evidence to confirm whether a change produced the expected result.

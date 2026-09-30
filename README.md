# week-4
# Week 4: EC2 Evidence Lab

## HarborTech Ticket Summary

**Ticket:** TKT-2026-0004

The purpose of this lab was to investigate a web-server reachability problem on a disposable Amazon EC2 workload. The investigation followed a troubleshooting process of verifying the AWS environment, reviewing the security-group configuration, testing HTTP reachability, identifying the supported corrective action, verifying the result, and documenting the instance lifecycle and cleanup.

## Client Impact

The reported symptom was that the EC2 web page could not be reached through HTTP. The initial investigation showed that the security group allowed inbound SSH traffic on TCP port 22 but did not contain an inbound HTTP rule for TCP port 80. Because the security group controls traffic entering the EC2 instance, the missing HTTP rule was a significant finding in the investigation.

The evidence did not initially prove that Apache itself was malfunctioning. The security-group configuration and HTTP reachability had to be evaluated separately from the application/service running inside the instance.

## Environment and Resource Names

* **AWS service:** Amazon EC2
* **AWS Region:** `us-east-1`
* **VPC:** `vpc-06f0a1ee8424f42dd`
* **Security group:** `launch-wizard-1`
* **Security group ID:** `sg-01fd53d990c7fd090`
* **Other observed security group:** `default`
* **Lab workload:** Disposable Amazon EC2 web server
* **Web service:** Apache/httpd
* **Inbound HTTP port:** TCP 80
* **Existing inbound rule before correction:** TCP 22 from `0.0.0.0/0`

Sensitive AWS account identifiers and personal identity information have intentionally been omitted from this public GitHub record.

## AWS Documentation Evidence

### AWS Source 1 — Security Groups and Inbound HTTP

**Document Title:** *Amazon EC2 User Guide*
**PDF Page:** 3296
**Quotation:** “A security group acts as a virtual firewall for your EC2 instances to control incoming and outgoing traffic.”

**AWS PDF:** https://docs.aws.amazon.com/pdfs/AWSEC2/latest/UserGuide/ec2-ug.pdf#ec2-security-groups

This source supports my interpretation of inbound HTTP access because AWS explains that security groups act as a virtual firewall for EC2 instances. Inbound rules control traffic coming into the instance, so allowing HTTP traffic requires a security-group rule that permits incoming web traffic to reach the instance. My baseline evidence showed that the security group allowed TCP 22 but did not contain an inbound TCP 80 rule.

### AWS Source 2 — User Data

**Document Title:** *Amazon EC2 User Guide*
**PDF Page:** 3706
**Quotation:** “When you launch an EC2 instance, you can pass a shell script to the instance using user data. Note that user data is base64 encoded, so you need to decode the user data to read the script.”

**AWS PDF:** https://docs.aws.amazon.com/pdfs/AWSEC2/latest/UserGuide/ec2-ug.pdf#ec2-security-groups

User data can show that an EC2 instance was given commands or scripts to perform configuration tasks, such as installing and starting a web server. However, user data alone cannot prove that the web service is currently running or accessible because providing the commands does not guarantee that they successfully completed. Additional evidence, such as checking the Apache service status or accessing the website, is needed to prove that the web service is actually running and reachable.

### AWS Source 3 — Instance Metadata

**Document Title:** *Instance Metadata*
**PDF Page:** 1641
**Quotation:** “Instance metadata properties are divided into categories, for example, host name, events, and security groups.”

**AWS PDF:** https://docs.aws.amazon.com/pdfs/AWSEC2/latest/UserGuide/ec2-ug.pdf#instancedata-data-retrieval

Instance metadata provides information about an EC2 instance and can be retrieved while the instance is running. This information can help identify the instance and retrieve instance-specific information, but metadata by itself does not prove that a web service is running or that the website is accessible from outside the instance. That distinction is important because the investigation required both instance identity evidence and separate web-service/network evidence.

## CloudShell Command Record

The first AWS identity check was performed with `aws sts get-caller-identity`. The command successfully confirmed that the CloudShell session was operating under the expected temporary lab role. Personal identity information and account identifiers are omitted from this public record.

### AWS Identity Check

```bash
aws sts get-caller-identity
```

Relevant result:

```text
UserId: [REDACTED]
Account: [REDACTED]
Arn: arn:aws:sts::[REDACTED]:assumed-role/voclabs/[REDACTED]
```

This confirmed the AWS identity being used by CloudShell. It did not by itself prove that the correct VPC, subnet, security group, or EC2 instance had been selected.

### Region Check

```bash
aws configure get region
```

The workload environment used `us-east-1`, as also shown by the ARN associated with the security group.

### Unsuccessful VPC Discovery Attempt

The first VPC discovery command was entered incorrectly:

```bash
aws ec2 describe-vpcs \
  --query 'Vpcs[].[VpcId,CidrBlock,IsDefault]' \
  --output table
```

The AWS CLI returned an error:

```text
aws: [ERROR]: Unknown options: Vpcs[].[VpcId,CidrBlock,IsDefault], --output, table, --query
```

This was a command-entry/argument parsing problem rather than evidence of an AWS networking failure. The error was documented instead of being treated as evidence about the EC2 environment.

### Security Group Inspection

The security-group inspection command was:

```bash
aws ec2 describe-security-groups
```

The relevant security group was:

```text
GroupId: sg-01fd53d990c7fd090
GroupName: launch-wizard-1
VpcId: vpc-06f0a1ee8424f42dd
```

The inbound rule was:

```text
IpProtocol: tcp
FromPort: 22
ToPort: 22
CidrIp: 0.0.0.0/0
```

There was no inbound TCP port 80 rule shown in the baseline configuration.

## Baseline Evidence

### Evidence A — Instance ID and AMI ID

**Instance ID:** `[ADD ACTUAL INSTANCE ID]`

**AMI ID:** `[ADD ACTUAL AMI ID]`

The instance ID identifies the EC2 workload being investigated, while the AMI ID identifies the image from which the instance was launched. These identifiers establish which resources were involved, but they do not prove that Apache was running or that the web page was reachable.

### Evidence B — Initial Public IPv4 Address

**Initial public IPv4:** `[ADD ACTUAL PUBLIC IP]`

The public IPv4 address identifies the network endpoint used for the initial HTTP test. Having a public IPv4 address does not by itself prove that HTTP traffic can reach the web server.

### Evidence C — Status Checks

**Status-check result:** `[ADD ACTUAL OUTPUT]`

The status checks provide evidence about the EC2 instance's infrastructure and system status. Passing status checks would not by themselves prove that Apache was running or that the security group allowed HTTP traffic.

### Evidence D — Security Group Before Fix

The baseline security group was `sg-01fd53d990c7fd090`. Its inbound configuration contained TCP port 22 from `0.0.0.0/0`, but no TCP port 80 rule.

This is strong evidence of the network-control configuration before correction. It does not by itself prove that Apache was stopped or misconfigured.

### Evidence E — Failed HTTP Test

**Initial HTTP test:** `[ADD ACTUAL CURL/HTTP OUTPUT]`

The failed HTTP test demonstrates that the web page could not be reached through the tested path at that time. The failed test alone does not identify the exact cause, which is why the security-group configuration and guest-side Apache evidence must be considered separately.

## Baseline Evidence Interpretation

The baseline evidence established the identity and configuration of the workload, the initial network endpoint, the EC2 status, and the security-group rules. The strongest configuration finding was that the security group allowed inbound SSH on TCP 22 but did not allow inbound HTTP on TCP 80. The evidence did not justify concluding that the operating system or Apache installation was corrupted.

## Root-Cause Analysis

The evidence supports a network-access configuration issue rather than an instance rebuild scenario. The security group functioned as the relevant inbound control point, and its baseline configuration lacked an inbound TCP 80 rule required for HTTP access.

The failed HTTP test was consistent with the missing rule, but the failed test alone was not sufficient to establish the root cause. The security-group inspection provided the configuration evidence needed to connect the symptom to the missing inbound HTTP rule.

## Corrective Action

The corrective action was to add the minimum required inbound HTTP rule to the existing security group rather than rebuilding the EC2 instance.

Security group:

```text
sg-01fd53d990c7fd090
```

Corrective rule:

```text
Protocol: TCP
Port: 80
Source: 0.0.0.0/0
```

**Command used:**

```bash
aws ec2 authorize-security-group-ingress \
  --group-id sg-01fd53d990c7fd090 \
  --protocol tcp \
  --port 80 \
  --cidr 0.0.0.0/0
```

The change addressed the identified network-control problem without replacing the workload.

## Verification Evidence

The security group was inspected again after the corrective action to verify that the TCP 80 rule was present.

```bash
aws ec2 describe-security-groups \
  --group-ids sg-01fd53d990c7fd090 \
  --query 'SecurityGroups[0].IpPermissions' \
  --output json
```

**Post-change output:** `[ADD ACTUAL OUTPUT]`

The final workload-path test should also be documented here:

```bash
curl -I http://[NEW_OR_CURRENT_PUBLIC_IP]
```

**HTTP result:** `[ADD ACTUAL OUTPUT]`

The AWS control-plane verification establishes that the security-group rule was changed. The HTTP test provides separate evidence that the workload path was tested after the change.

## IMDSv2 and Guest Evidence

Inside the EC2 instance, Apache should be verified independently from the AWS control plane.

```bash
sudo systemctl status httpd --no-pager
```

```bash
curl -I http://localhost
```

The local test addresses whether the web server responds from inside the instance. This is different from the CloudShell AWS CLI security-group output because the AWS CLI describes AWS-side configuration, while the local `curl` and Apache service status provide evidence about the guest operating system and application.

For IMDSv2, the instance metadata token is requested first:

```bash
TOKEN=$(curl -X PUT "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
```

The instance ID can then be retrieved using the token:

```bash
curl -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/instance-id
```

**Actual IMDSv2 output:** `[ADD ACTUAL INSTANCE ID]`

The IMDSv2 result identifies the running EC2 instance through the instance metadata service. It does not prove that Apache is running or that the public website is reachable.

## Stop/Start Lifecycle Test

**Status:** `[ADD YOUR ACTUAL STOP/START EVIDENCE]`

Before stopping the instance, record:

```text
Instance ID: [ADD]
Public IPv4: [ADD]
Web-page content: [ADD]
```

Stop command:

```bash
aws ec2 stop-instances --instance-ids [INSTANCE_ID]
```

Wait command:

```bash
aws ec2 wait instance-stopped --instance-ids [INSTANCE_ID]
```

Verify:

```bash
aws ec2 describe-instances \
  --instance-ids [INSTANCE_ID] \
  --query 'Reservations[0].Instances[0].State.Name' \
  --output text
```

Start command:

```bash
aws ec2 start-instances --instance-ids [INSTANCE_ID]
```

Wait command:

```bash
aws ec2 wait instance-running --instance-ids [INSTANCE_ID]
```

Retrieve the new public IPv4 address:

```bash
aws ec2 describe-instances \
  --instance-ids [INSTANCE_ID] \
  --query 'Reservations[0].Instances[0].PublicIpAddress' \
  --output text
```

Final HTTP test:

```bash
curl -I http://[NEW_PUBLIC_IP]
```

### Lifecycle Analysis

The EBS-backed instance should retain its instance identity and EBS-backed data through a normal stop/start cycle. An automatically assigned public IPv4 address can change after the instance is stopped and started, so the new address must be retrieved before retesting the web page.

This lifecycle behavior is consistent with AWS documentation describing EC2 instance state changes and the behavior of EBS-backed instances and automatically assigned public IPv4 addresses.

## Cleanup Evidence

**Status:** `[ADD ACTUAL CLEANUP EVIDENCE]`

Terminate the disposable instance:

```bash
aws ec2 terminate-instances --instance-ids [INSTANCE_ID]
```

Wait for termination:

```bash
aws ec2 wait instance-terminated --instance-ids [INSTANCE_ID]
```

Verify termination:

```bash
aws ec2 describe-instances \
  --instance-ids [INSTANCE_ID] \
  --query 'Reservations[0].Instances[0].State.Name' \
  --output text
```

After the instance is no longer attached to the security group, attempt deletion:

```bash
aws ec2 delete-security-group \
  --group-id sg-01fd53d990c7fd090
```

**Deletion result:** `[ADD SUCCESSFUL OUTPUT OR EXACT AWS ERROR]`

If AWS returns a dependency error, the exact error should be documented and the remaining resource dependency identified rather than bypassed.

## Escalation and Change-Control Notes

The corrective action was limited to the security-group rule required for HTTP access. In a production environment, this type of network change would require authorization and change-control review before implementation.

Rebuilding the instance was not justified because the available evidence pointed to an inbound network-control issue rather than operating-system or Apache corruption. Rebuilding would have introduced additional changes and could have destroyed useful evidence without addressing the identified security-group configuration.

One alternative ruled out was rebuilding the EC2 instance. The existing instance and its configuration remained useful for testing, and the missing TCP 80 rule provided a smaller supported corrective action.

A limitation of the investigation was that some commands did not complete successfully during the lab, so only the evidence actually obtained should be treated as verified.

## Lessons Learned

This lab reinforced the importance of troubleshooting one layer at a time instead of immediately rebuilding a workload. A failed web-page test does not automatically mean that Apache or the operating system is broken.

I also learned that AWS control-plane evidence and guest-level evidence answer different questions. Security-group inspection shows how AWS is configured to control network traffic, while local Apache checks show what is happening inside the instance.

The lifecycle test also demonstrates why instance identity, EBS persistence, and public IPv4 addressing need to be considered separately when documenting EC2 behavior.

## Professional Vocabulary

* **EC2:** Amazon Elastic Compute Cloud service used to provide virtual servers.
* **Security group:** A virtual firewall that controls inbound and outbound traffic for an EC2 resource.
* **Inbound rule:** A security-group rule controlling traffic entering the resource.
* **HTTP:** Web protocol commonly using TCP port 80.
* **EBS:** Elastic Block Store, which provides persistent block storage for EC2.
* **IMDSv2:** Version 2 of the EC2 Instance Metadata Service, using a session token to retrieve instance metadata.
* **Instance ID:** Unique identifier assigned to an EC2 instance.
* **Public IPv4 address:** Publicly reachable IPv4 address assigned to the instance when applicable.
* **User data:** Startup configuration or scripting passed to an EC2 instance during launch.
* **Control plane:** AWS-side configuration and management layer used to inspect or modify resources.
* **Guest operating system:** The operating system running inside the EC2 instance.
* **Root cause:** The underlying condition supported by evidence as responsible for the observed symptom.
* **Corrective action:** A controlled change intended to address the identified cause.
* **Verification:** Testing performed after a change to determine whether the expected result occurred.
* **Lifecycle:** The sequence of EC2 instance states and related behavior from launch through stop/start or termination.
* **Change control:** The authorization and documentation process used before making changes to managed workloads.
* **Rollback:** Returning a system or configuration to its previous known state after an unsuccessful change.
* **Escalation:** Passing an issue to an authorized person or team when additional access, approval, or expertise is required.

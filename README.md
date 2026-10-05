# Automating Infrastructure Compliance with AWS Config

A hands-on AWS lab where I built a full detect-and-fix compliance pipeline: AWS Config rules flagged noncompliant EC2 instances and S3 buckets, and AWS Systems Manager Automation documents fixed them automatically — no manual remediation steps once it was wired up.

## Scenario
A company needed to enforce two security requirements across its cloud environment: EC2 instances used for testing a pre-release application couldn't have public IP addresses, and S3 buckets needed tiered protections (versioning for medium-security data, versioning plus access logging for high-security data like source code). The task was to detect violations automatically and remediate the common ones without manual intervention.

## What I did

### 1. Set up AWS Config with 1-click setup
Enabled AWS Config to record all resource types, using the AWS-managed service-linked role and an auto-created S3 bucket for configuration history — before any rules existed, the dashboard showed zero rules and zero resources evaluated.

### 2. Reviewed the resource inventory
Filtered the resource inventory to confirm Config was tracking two EC2 instances and several S3 buckets, including the two with specific compliance requirements for this exercise.

### 3. Created an EC2 public-IP rule
Added the AWS managed rule `ec2-instance-no-public-ip` (named `isolated-compute-rule`), which flags any EC2 instance with a public IP as noncompliant — enforcing that test instances stay fully isolated from the internet.

### 4. Created three S3 security rules
- `s3-block-public-access` (`s3-bucket-public-read-prohibited`) — applied to all buckets, checking Block Public Access settings, bucket policy, and ACLs
- `s3-bucket-versioning-on` (`s3-bucket-versioning-enabled`) — applied to all buckets
- `s3-bucket-logging-on` (`s3-bucket-logging-enabled`) — scoped to one specific bucket via its Resource identifier, since only the high-security bucket required logging

### 5. Identified noncompliant resources
After the rules evaluated, the dashboard showed three noncompliant rules and at least five noncompliant resources — drilled into one noncompliant bucket specifically and confirmed both its versioning and logging rules were failing.

### 6. Configured automatic remediation for each rule
- EC2 rule → `AWS-TerminateEC2Instance` (terminates any instance found with a public IP)
- S3 versioning rule → `AWS-ConfigureS3BucketVersioning` (enables versioning on the flagged bucket)
- S3 logging rule → `AWS-ConfigureS3BucketLogging` (enables logging, targeting a dedicated log bucket with the correct grantee permissions for the S3 Log Delivery group)

Each remediation action used a scoped `AutomationServiceRole` with only the permissions needed to perform its specific fix.

### 7. Triggered re-evaluation and verified remediation
Manually re-evaluated each rule to trigger auto-remediation on the existing noncompliant resources, then confirmed in the actual services:
- The EC2 instance with a public IP had been **terminated**; the private-IP instance was untouched and running
- Both target S3 buckets now showed versioning enabled, and the high-security bucket showed logging enabled
- Cross-checked the same results in Systems Manager's Automation execution history, showing successful runs

## Key takeaways
- Detection and remediation are two separate configurations in AWS Config — a rule flags noncompliance, but nothing gets fixed until a remediation action is explicitly attached to it
- Auto-remediation only fires on evaluations that happen *after* the remediation action is saved — resources that were already noncompliant need an explicit re-evaluation to trigger the fix
- Scoping a rule to one specific resource (via Resource identifier) rather than an entire resource type lets you apply tiered requirements — all buckets needed versioning, but only one needed logging too

## Tools
AWS Config, AWS Systems Manager Automation, Amazon EC2, Amazon S3, IAM

---
*Completed as an AWS hands-on lab, including a passed knowledge check assessment.*

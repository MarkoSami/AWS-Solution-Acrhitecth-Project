# Deployment guide, validation plan and cleanup

This guide describes how to build the architecture in [README.md](../README.md) in the AWS Management Console. All resources are in `us-east-1` unless stated. Names use the prefix `saa-`.

> Run this in a sandbox account. Several resources (NAT gateways, RDS Multi-AZ, ALB, WAF) bill hourly, so follow the cleanup section when finished.

## Prerequisites

- An AWS account with administrator access for the build
- A registered domain in Route 53 (or a hosted zone you control)
- AWS CLI v2 configured (`aws sts get-caller-identity` works)

## Step 1: Network (VPC)

1. Create a VPC `saa-vpc`, CIDR `10.0.0.0/16`, DNS hostnames enabled.
2. Create six subnets using the CIDR plan in the README (public, private-app and private-data in `us-east-1a` and `us-east-1b`).
3. Create and attach an Internet Gateway.
4. Create two NAT gateways (one in each public subnet) with Elastic IPs.
5. Create route tables:
   - `saa-public-rt`: `0.0.0.0/0` → Internet Gateway; associate both public subnets.
   - `saa-app-rt-a` and `saa-app-rt-b`: `0.0.0.0/0` → NAT gateway in the same AZ; associate the matching app subnet.
   - `saa-data-rt`: local only; associate both data subnets.
6. Create a **gateway** VPC endpoint for S3 and associate it with both app route tables.
7. (Optional, for fully private access) Create interface endpoints for `ssm`, `ssmmessages` and `ec2messages` so Session Manager works without internet.

## Step 2: Security groups

| Name | Inbound rule |
|---|---|
| `sg-alb` | TCP 443 and 80 from `0.0.0.0/0` (later: the CloudFront managed prefix list) |
| `sg-app` | TCP 80 from `sg-alb` |
| `sg-db` | TCP 3306 (or 5432) from `sg-app` |

Leave default outbound rules. Do **not** open port 22 anywhere.

## Step 3: Database

1. Create a DB subnet group from the two private-data subnets.
2. Create an RDS instance (MySQL 8 or PostgreSQL): **Multi-AZ**, gp3 storage, `sg-db`, **not publicly accessible**, encryption on, automated backups 7 days, deletion protection on.
3. Choose **Manage master credentials in AWS Secrets Manager**. Enable rotation (for example every 30 days).

## Step 4: Compute

1. Create an IAM role `saa-ec2-role` with `AmazonSSMManagedInstanceCore`, `CloudWatchAgentServerPolicy`, and an inline policy allowing `secretsmanager:GetSecretValue` on the database secret only.
2. Create a Launch Template `saa-lt`:
   - Amazon Linux 2023, `t3.small` (or similar)
   - Instance profile `saa-ec2-role`, security group `sg-app`, no key pair
   - Metadata options: **IMDSv2 required**
   - User data: install a web server and the application, add a `/health` endpoint that returns HTTP 200, and install the CloudWatch agent.
3. Create the Target Group `saa-tg` (HTTP 80, health check path `/health`) and the ALB `saa-alb` (internet-facing, both public subnets, `sg-alb`).
4. Request an ACM certificate for the ALB, then add an HTTPS :443 listener forwarding to `saa-tg`, and an HTTP :80 listener that redirects to HTTPS.
5. Create the Auto Scaling group `saa-asg`: both private-app subnets, attach `saa-tg`, health check type **ELB**, grace period 300 s, min 2 / desired 2 / max 6.
6. Add two target tracking policies: average CPU 50%, and ALB request count per target (for example 1000).

## Step 5: Protection (WAF)

1. Create a regional web ACL and associate it with `saa-alb`.
2. Add AWS managed rule groups: Core rule set, Known bad inputs, SQL database. Add a rate-based rule (for example 2000 requests per 5 minutes per IP).
3. Start the managed rules in **Count** mode, review the logs, then switch to **Block**.

## Step 6: Edge (S3, CloudFront, Route 53)

1. Create a private S3 bucket for static assets (Block Public Access on, versioning on) and upload sample files under `assets/`.
2. Request an ACM certificate in `us-east-1` for the site name.
3. Create a CloudFront distribution with two origins:
   - S3 bucket with **Origin Access Control**; add the generated bucket policy.
   - The ALB (custom origin, HTTPS only) using an origin hostname such as `origin.example.com`.
4. Behaviors: `/assets/*` → S3 with the caching-optimized policy; default → ALB with caching disabled and all viewer headers/cookies/query strings forwarded. Redirect HTTP to HTTPS.
5. Route 53: alias `www` → CloudFront; alias `origin` → ALB with an HTTPS health check.
6. Hardening: add a secret custom header in the CloudFront ALB origin settings and require it in an ALB listener rule; restrict `sg-alb` to the CloudFront managed prefix list.

## Step 7: Monitoring

1. Create an SNS topic `saa-alerts` and confirm an email subscription.
2. Create the alarms listed in the README (section 7) with `saa-alerts` as the action.
3. Create a CloudWatch dashboard with the key ALB, ASG and RDS widgets.

## Validation plan

| # | Test | How | Expected result |
|---|---|---|---|
| 1 | Site reachable | Open `https://www.<domain>` | Page loads over HTTPS; HTTP redirects to HTTPS |
| 2 | Static caching | `curl -I https://www.<domain>/assets/<file>` twice | Second response shows a CloudFront cache hit (`X-Cache: Hit from cloudfront`) |
| 3 | No public compute | Check EC2 console | Instances have no public IPv4 address |
| 4 | Bastion-free access | Connect with Session Manager | Shell opens; port 22 closed in every security group |
| 5 | Instance failure | Terminate one instance | ALB keeps serving; ASG launches a replacement |
| 6 | Auto Scaling | Generate load (for example `hey -z 5m -c 200 https://www.<domain>/`) | Instance count rises above 2, then falls after the test |
| 7 | Database failover | Reboot the RDS instance **with failover** | App reconnects after about 1 to 2 minutes; same endpoint |
| 8 | WAF | Request `/?q=1' OR '1'='1` | HTTP 403 from WAF (after switching to Block mode) |
| 9 | Alarm path | Stop the app on all instances or lower an alarm threshold | SNS email received |
| 10 | Origin lock | Call the ALB DNS name directly | Rejected (403 or timeout) once the origin lock is in place |

Record screenshots of tests 5 to 8 and add them under `docs/evidence/` if you deploy.

## Cleanup (reverse order to avoid dependency errors)

1. Delete CloudFront distribution (disable first, then delete) and Route 53 records.
2. Delete the WAF web ACL association and web ACL.
3. Delete the ASG, then the ALB, target group and Launch Template.
4. Delete the RDS instance (turn off deletion protection; decide whether to keep a final snapshot) and the DB subnet group.
5. Delete the Secrets Manager secret, SNS topic, alarms and dashboard.
6. Delete the NAT gateways and **release the Elastic IPs**.
7. Delete VPC endpoints, route tables, subnets, Internet Gateway, security groups, then the VPC.
8. Empty and delete the S3 buckets. Delete ACM certificates and IAM roles.

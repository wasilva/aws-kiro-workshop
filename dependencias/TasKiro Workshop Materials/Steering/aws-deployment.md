---
inclusion: manual
---

# AWS Deployment Standards — TasKiro

> Example steering: defines how TasKiro (and similar Bun-native apps) is
> deployed on AWS. It also serves as teaching material to demonstrate the use
> of Steerings in Kiro.

## How this steering is included

The `inclusion` front-matter controls when these rules enter the context:

- `inclusion: always` — always included (the default when there is no front-matter).
- `inclusion: fileMatch` + `fileMatchPattern: 'Dockerfile'` — only when a
  matching file is opened/read.
- `inclusion: manual` — included on demand when you reference it with `#` in
  chat. **Used here** so that deployment is an intentional action in the demo.

Switch the mode depending on what you want to demonstrate.

## When to apply

Apply these rules to any provisioning, deployment, or infrastructure change
task for this project's AWS infrastructure.

## Reference architecture (do not deviate without approval)

```
Internet → CloudFront (HTTPS, *.cloudfront.net domain, static asset caching)
         → internet-facing ALB (public subnets, 2 AZs)
         → Bun EC2 (private subnet, no public IP) — SQLite on EBS
         → NAT Gateway (egress for the private subnet)
```

- The application runs as **a single Bun process** (`bun src/index.tsx`),
  serving the frontend and the ElysiaJS API on the same port.
- `/api/*` is forwarded **without caching**; the remaining paths may be cached.
- ALB target health check: `GET /api/health`.

## Non-negotiable constraints

1. **SQLite = single instance.** `bun:sqlite` is single-writer against a file. Do
   NOT enable Auto Scaling or more than one application instance. Real
   persistence lives on the EBS volume.
2. **No Docker.** The app runs Bun directly on EC2 via user-data. Do not introduce
   containers/images in this scenario.
3. **The app instance is private.** No public IP, no application port open to the
   internet — only the ALB talks to the EC2 instance.
4. **Administrative access via SSM Session Manager.** Opening SSH (22) or any
   `0.0.0.0/0` ingress on the EC2 instance is prohibited.
5. **No secrets in user-data or in the code.** Use SSM Parameter Store/Secrets
   Manager when there is a secret (there are none today).

## Compute (EC2)

- **OS:** Amazon Linux 2023 (AL2023). SSM agent pre-installed and a glibc base
  compatible with the standard Bun binary for Linux.
- **Architecture/family:** Graviton (ARM64), burstable **T** family.
- **Size:** `t4g.small` (2 vCPU / 2 GiB). 2 GiB accommodates client bundling
  at runtime; do not use 1 GiB without switching to a pre-build at boot.
- **Storage:** **encrypted gp3 EBS**; the `taskiro.sqlite` file lives on
  that volume (persistence).
- **AMI:** resolve the latest AL2023 ARM64 AMI via SSM Parameter
  (`/aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-arm64`) instead
  of pinning an AMI ID.
- **Metadata:** require **IMDSv2** (`HttpTokens=required`) at instance creation.
- **Bootstrap:** user-data installs Bun, downloads the code from the S3 bucket, and
  registers a **systemd** service that runs `bun src/index.tsx` with
  `NODE_ENV=production`.

## Networking

- VPC `10.20.0.0/16`. Public subnets `10.20.0.0/24` and `10.20.1.0/24`
  (ALB + NAT); private subnets `10.20.10.0/24` and `10.20.11.0/24` (EC2), across 2 AZs.
- A single NAT Gateway (largest fixed cost) for egress from the private subnets.
- Fixed CIDRs ensure reproducibility across runs.

## Naming and tagging conventions

- Prefix for all resources: `taskiro-` (e.g., `taskiro-vpc`,
  `taskiro-alb`, `taskiro-ec2-role`).
- **Tag-on-create**: apply tags at creation time (`TagSpecifications`),
  not afterward.
- Required tags on every resource that supports them:
  - `Name` — human-readable name with prefix (e.g., `taskiro-ec2`)
  - `Project = taskiro`
  - `Environment = demo`
  - `ManagedBy = kiro`
  - `Owner = <owner>`
  - `CostCenter = demo`
- Default region: `us-east-1`.

> **Critical — the `Project = taskiro` tag on the EC2 instance is load-bearing for permission.**
> The operator role only receives `ssm:SendCommand` and `ec2:GetConsoleOutput` on
> instances with `aws:ResourceTag/Project = taskiro`. Apply the tag on
> `RunInstances` via `TagSpecifications` (never with `CreateTags` alone afterward):
> if the instance does not have the tag at creation time, inspecting the
> bootstrap via Run Command and reading the console fail with `AccessDenied`.

## Security (AWS Well-Architected — Security pillar)

- **Least privilege.** The deployment role uses scoped permissions
  (see `docs/aws-least-privilege-permissions.md`). `*FullAccess` and
  `IAMFullAccess` are prohibited.
- `iam:PassRole` always restricted to the `taskiro-*` role and the
  `ec2.amazonaws.com` service.
- Encryption enabled: EBS volume and the S3 code bucket (SSE).
- S3 code bucket with **Block Public Access** enabled.
- Minimal Security Groups: the ALB allows 80/443 from the internet; the EC2
  instance allows the app port **only** from the ALB's Security Group.
- **IMDSv2 required** on the EC2 instance (`HttpTokens=required`) — mitigates
  credential theft via SSRF.
- **Restrict the ALB to CloudFront.** Since the ALB is internet-facing, anyone
  could reach its DNS directly, bypassing CloudFront. Mitigate with one of these
  options: (a) ALB SG ingress limited to the managed prefix list
  `com.amazonaws.global.cloudfront.origin-facing`; or (b) a secret header injected
  by CloudFront and validated at the ALB/app. CloudFront must redirect
  viewer HTTP → HTTPS.

## Provisioning tool

- Use the **`cloud-architect`** power, **`call_aws`** tool (the `awsapi` MCP
  server), to run the AWS CLI. Do not use the CDK flow in this scenario.
- Estimate cost with the `awspricing` server before creating paid resources
  (especially the NAT Gateway and EC2 hours).

## Workflow (rules for the agent)

1. **Confirm before creating paid or destructive resources.** The NAT Gateway, EC2,
   ALB, and CloudFront have a cost — present an estimate and wait for approval.
2. **Provision in order and report each step:** networking → security → IAM →
   EC2 → ALB/target group → CloudFront.
3. **Record the created IDs** (VPC, subnets, SG, instance, ALB, distribution)
   as you go, to enable rollback.
4. **Verify health** after deployment: ALB target `healthy` and `200` at
   `/api/health` via the ALB DNS, then via the CloudFront domain.
5. **On a permission error**, report the denied action and the exact missing
   permission — do not try to work around it with broader permissions.
6. **Ensure the `Project=taskiro` tag on the EC2 instance at creation.** Apply it on
   `RunInstances` (via `TagSpecifications`, never with `CreateTags` alone afterward) and,
   right after creating the instance, confirm with `describe-instances` before
   relying on Run Command or `GetConsoleOutput` — without the tag, both fail with
   `AccessDenied` (details in the "Naming and tagging conventions" section).

## Observability

- Application logs via systemd's journald; accessible through SSM (no SSH).
- Forward system/app logs to CloudWatch Logs when useful (group
  `/taskiro/app`), with short retention (e.g., 7 days) to contain cost.
- ALB access logs to an S3 bucket are optional in the demo.

## Backup and data

- `taskiro.sqlite` is the single data point. For any environment beyond a
  disposable demo, schedule a periodic copy of the file to S3 (systemd timer) or
  a snapshot of the EBS volume.
- `seed()` recreates the demo state when the database is empty — useful in an
  ephemeral environment, but not a substitute for a backup.

## Application update (without recreating infra)

- Publishing a new version = re-upload the code to the S3 bucket and restart the service
  via **SSM Run Command** (`systemctl restart taskiro`). Do not recreate the VPC/ALB/EC2
  for a simple code deployment.
- Invalidate in CloudFront only the changed static paths.

## Idempotency

- Before creating, look for existing resources by the `Project=taskiro` tag to
  avoid duplicating the VPC/ALB/distribution on re-runs.

## Verification and rollback

- Done criterion: the CloudFront URL serves the app and the `/api/*` calls
  work (creating/listing tasks persists to SQLite).
- Keep a teardown procedure that removes all `taskiro-*` resources
  (includes releasing the NAT's Elastic IP to avoid charges).

## Cost (transparency)

- Largest fixed cost of the scenario: **NAT Gateway** (~$32/month + traffic).
- A single small EC2 + ALB + CloudFront round out the cost.
- Because this is a **demo** environment, tear it down when not in use.

## Scope notice

This app **has no authentication**. Exposing it publicly means an open API —
acceptable only for demonstration. Do not treat it as a production pattern.

# Three-Tier WordPress on AWS (CloudFormation)

A single CloudFormation template (`vpc.yaml`) that provisions a highly available,
secure, three-tier WordPress deployment on AWS, with a GitHub Actions workflow
(`.github/workflows/deploy.yml`) for continuous deployment.

## Architecture

```
                       Internet
                          |
                   CloudFront (HTTPS)
                          |
                   AWS WAF Web ACL
             (SQL injection + XSS managed rules)
                          |
          Application Load Balancer (public subnets)
                          |
        +-----------------+-----------------+
        |                                   |
   WordPress EC2                       WordPress EC2        Auto Scaling Group
   (public subnet AZ-a)                (public subnet AZ-b)  (min 2, max 3)
        |         |                         |
        |         +--- EFS (wp-content, shared) ---+
        |                                   |
        +------ ElastiCache Redis (object cache) --+   (private subnets)
        |                                   |
        +------ RDS MySQL primary (writes) -+
        |              |
        |         RDS MySQL read replica (reads, different AZ)
```

### Tiers

| Tier | Services |
|------|----------|
| **Web / presentation** | CloudFront, WAF, Application Load Balancer, Auto Scaling Group of WordPress EC2 instances |
| **Application / cache** | EFS (shared `wp-content`), ElastiCache Redis (object cache) |
| **Data** | RDS MySQL primary + read replica, Secrets Manager (DB credentials) |

### Key design choices

- **Two subnets per tier across two Availability Zones.** Required by the RDS DB
  subnet group, the Application Load Balancer, and the EFS mount targets.
- **No NAT Gateway.** Private subnets have no outbound internet route. The web
  instances run in public subnets with public IPs, which is how they reach the
  internet (WordPress download, drop-ins) during boot.
- **DB credentials in Secrets Manager.** The password is auto-generated and never
  stored in the template or passed on the CLI. The RDS instance resolves it via
  `{{resolve:secretsmanager:...}}` and the web instances read it at boot through
  a scoped IAM role.
- **Read/write splitting via HyperDB.** Writes go to the RDS primary; reads are
  routed to the read replica.
- **Object caching via Redis.** The WordPress Redis Object Cache drop-in reduces
  repeated database queries and offloads the RDS instance.

## Resources created

- **Networking:** VPC, 2 public subnets, 2 private subnets, Internet Gateway,
  route tables and associations.
- **Compute:** Launch Template + Auto Scaling Group (WordPress), IAM instance role.
- **Load balancing:** Application Load Balancer, target group, HTTP listener.
- **Database:** RDS MySQL primary, RDS read replica, DB subnet group, Secrets
  Manager secret + target attachment.
- **Shared storage:** EFS file system + a mount target per AZ.
- **Cache:** ElastiCache Redis replication group + cache subnet group.
- **Edge security:** CloudFront distribution, WAFv2 Web ACL (SQLi + Common/XSS
  managed rule groups).
- **Monitoring:** CloudWatch dashboard (EC2 CPU, RDS CPU, ALB RequestCount).
- **Backup:** AWS Backup vault, daily plan (30-day retention), backup selection
  for RDS, EFS, and EC2, plus an AWS Backup IAM role.
- **Security groups:** ALB, web server, database, EFS, and cache — each scoped so
  downstream tiers only accept traffic from the tier above them.

## Prerequisites

- An AWS account and the AWS CLI installed and configured.
- **Region: `us-east-1`.** The WAF Web ACL is `CLOUDFRONT`-scoped, which AWS only
  allows in `us-east-1`. Deploy the whole stack there, or split the WAF into a
  separate `us-east-1` stack if you move regions.
- An existing **EC2 key pair** in the target region. Create one with:
  ```powershell
  aws ec2 create-key-pair --key-name wordpress-key --query "KeyMaterial" --output text > wordpress-key.pem
  ```
- Correct system clock. AWS SigV4 rejects requests when the clock drifts more
  than ~5 minutes (`SignatureDoesNotMatch`). Keep time in sync.

## Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `VpcCidr` | `10.0.0.0/16` | VPC CIDR block |
| `PublicSubnetCidr` | `10.0.1.0/24` | First public subnet |
| `PublicSubnet2Cidr` | `10.0.4.0/24` | Second public subnet (ALB) |
| `PrivateSubnetCidr` | `10.0.2.0/24` | First private subnet |
| `PrivateSubnet2Cidr` | `10.0.3.0/24` | Second private subnet |
| `AvailabilityZone` | _(required)_ | AZ for public subnet 1 + private subnet 1 |
| `AvailabilityZone2` | _(required)_ | AZ for public subnet 2 + private subnet 2 |
| `ReadReplicaAvailabilityZone` | _(required)_ | AZ for the RDS read replica |
| `KeyName` | _(required)_ | Existing EC2 key pair name |
| `SSHLocation` | `0.0.0.0/0` | CIDR allowed to SSH to the web servers |
| `DBName` | `wordpress` | Initial database name |
| `DBUsername` | `wpadmin` | DB master username |
| `DBInstanceClass` | `db.t3.micro` | RDS instance class |
| `DBAllocatedStorage` | `20` | RDS storage (GiB) |
| `DBEngineVersion` | `8.0` | MySQL engine version |
| `CacheNodeType` | `cache.t3.micro` | ElastiCache Redis node type |
| `InstanceType` | `t3.micro` | Web server instance type |
| `LatestAmazonLinux2Ami` | _(SSM path)_ | Resolves to the latest Amazon Linux 2 AMI |
| `ASGMinSize` | `2` | Minimum web instances |
| `ASGMaxSize` | `3` | Maximum web instances |
| `ASGDesiredCapacity` | `2` | Desired web instances |

> The DB password is **not** a parameter. It is generated and stored in Secrets
> Manager automatically.

## Deploy

> The template creates named IAM roles, so `--capabilities CAPABILITY_NAMED_IAM`
> is required.

```powershell
aws cloudformation deploy `
  --template-file vpc.yaml `
  --stack-name three-tier-vpc `
  --parameter-overrides `
      AvailabilityZone=us-east-1a `
      AvailabilityZone2=us-east-1b `
      ReadReplicaAvailabilityZone=us-east-1c `
      KeyName=wordpress-key `
  --capabilities CAPABILITY_NAMED_IAM
```

The AZ parameters and the key pair must all belong to the region you deploy into.

### Validate before deploying

```powershell
aws cloudformation validate-template --template-body file://vpc.yaml
```

## Continuous deployment (GitHub Actions)

`.github/workflows/deploy.yml` deploys the stack on every push to `main` (and via
manual dispatch). It authenticates to AWS using **GitHub OIDC** — no long-lived
access keys are stored. GitHub requests a short-lived token and assumes an IAM
role you create once.

### One-time AWS setup

1. **Create a GitHub OIDC identity provider** in IAM (only needed once per account):
   - Provider URL: `https://token.actions.githubusercontent.com`
   - Audience: `sts.amazonaws.com`

2. **Create an IAM role** that trusts that provider, scoped to your repository.
   Example trust policy (replace `OWNER/REPO`):

   ```json
   {
     "Version": "2012-10-17",
     "Statement": [
       {
         "Effect": "Allow",
         "Principal": {
           "Federated": "arn:aws:iam::<ACCOUNT_ID>:oidc-provider/token.actions.githubusercontent.com"
         },
         "Action": "sts:AssumeRoleWithWebIdentity",
         "Condition": {
           "StringEquals": {
             "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
           },
           "StringLike": {
             "token.actions.githubusercontent.com:sub": "repo:OWNER/REPO:ref:refs/heads/main"
           }
         }
       }
     ]
   }
   ```

   Attach a permissions policy allowing CloudFormation plus the resource types in
   the template (EC2/VPC, RDS, ELBv2, AutoScaling, EFS, ElastiCache, IAM,
   Secrets Manager, CloudWatch, CloudFront, WAFv2, Backup).

3. **Add the role ARN as a repository variable** (not a secret):
   Settings → Secrets and variables → Actions → **Variables** tab →
   `AWS_DEPLOY_ROLE_ARN = arn:aws:iam::<ACCOUNT_ID>:role/<role-name>`

### One-time AWS setup via CLI

Run these in PowerShell. Replace `OWNER/REPO` with your GitHub repo
(e.g. `mazpu/Three-Tier-Web-Application`). The commands are idempotent-ish; skip
step 1 if the OIDC provider already exists in your account.

```powershell
# --- Variables ---
$Repo    = "OWNER/REPO"
$Region  = "us-east-1"
$RoleName = "github-actions-cfn-deploy"
$Account = (aws sts get-caller-identity --query Account --output text)

# 1. Create the GitHub OIDC identity provider (once per account).
#    If it already exists, this returns EntityAlreadyExists -- safe to ignore.
aws iam create-open-id-connect-provider `
  --url "https://token.actions.githubusercontent.com" `
  --client-id-list "sts.amazonaws.com" `
  --thumbprint-list "ffffffffffffffffffffffffffffffffffffffff"

# 2. Write the trust policy scoped to this repo's main branch.
@"
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::$Account:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": { "token.actions.githubusercontent.com:aud": "sts.amazonaws.com" },
        "StringLike":   { "token.actions.githubusercontent.com:sub": "repo:$Repo`:ref:refs/heads/main" }
      }
    }
  ]
}
"@ | Out-File -Encoding ascii trust-policy.json

# 3. Create the role with that trust policy.
aws iam create-role `
  --role-name $RoleName `
  --assume-role-policy-document file://trust-policy.json

# 4. Attach a least-privilege inline policy scoped to this template's services.
#    deploy-policy.json is included in the repo.
aws iam put-role-policy `
  --role-name $RoleName `
  --policy-name three-tier-deploy `
  --policy-document file://deploy-policy.json

# 5. Print the role ARN -- paste this into the AWS_DEPLOY_ROLE_ARN repo variable.
aws iam get-role --role-name $RoleName --query "Role.Arn" --output text
```

> **Thumbprint note:** GitHub's OIDC token is validated against AWS's trust store,
> and for `token.actions.githubusercontent.com` the thumbprint value is no longer
> actually verified by IAM — the placeholder above is accepted. If your account
> enforces thumbprint validation, fetch the current one from GitHub's certificate
> chain instead.

> **Permissions note:** `deploy-policy.json` grants the role access only to the
> services this template manages (CloudFormation, EC2/VPC, RDS, ELBv2,
> AutoScaling, EFS, ElastiCache, CloudWatch, CloudFront, WAFv2, Backup, Secrets
> Manager, plus the IAM actions needed to create the stack's own roles). The
> service actions use `*` resources because CloudFormation creates these
> resources dynamically; tighten with resource ARNs or tag conditions if your
> security posture requires it.

### Set the GitHub repository variable via CLI

Using the GitHub CLI (`gh`):

```powershell
gh variable set AWS_DEPLOY_ROLE_ARN `
  --body "arn:aws:iam::<ACCOUNT_ID>:role/github-actions-cfn-deploy" `
  --repo OWNER/REPO
```

### Workflow configuration

The AZs, region, key pair name, and stack name are plain (non-sensitive) values
set directly in the workflow's `env:` block — edit them there if they change. The
only thing stored in GitHub is the role ARN variable above.

> The workflow needs `permissions: id-token: write` to request the OIDC token;
> this is already set in `deploy.yml`.

## Outputs

| Output | Description |
|--------|-------------|
| `WordPressURL` | Public HTTPS URL via CloudFront (WAF-protected) |
| `CloudFrontDomainName` | CloudFront distribution domain |
| `LoadBalancerDNSName` | ALB DNS name |
| `VpcId` | VPC ID |
| `PublicSubnetId`, `PublicSubnet2Id` | Public subnet IDs |
| `PrivateSubnetId`, `PrivateSubnet2Id` | Private subnet IDs |
| `DBEndpointAddress`, `DBEndpointPort` | RDS primary endpoint |
| `ReadReplicaEndpointAddress` | RDS read replica endpoint |
| `RedisEndpointAddress` | ElastiCache Redis primary endpoint |
| `DBSecretArn` | Secrets Manager ARN for DB credentials |
| `EFSFileSystemId` | EFS file system ID |
| `WebACLArn` | WAF Web ACL ARN |
| `BackupVaultName`, `BackupPlanId` | AWS Backup vault and plan |
| `DashboardURL` | CloudWatch dashboard link |

## Verifying after deploy

- Open the `WordPressURL` output in a browser to complete WordPress setup.
- SSH into an instance (if `SSHLocation` allows your IP) and inspect
  `/var/log/user-data.log` to confirm EFS mounted, Redis/HyperDB drop-ins
  installed, and WordPress downloaded.
- CloudFront takes 5-15 minutes to reach the `Deployed` state.
- Backups run on the daily schedule (05:00 UTC); trigger an on-demand backup from
  the AWS Backup console to verify immediately.

## Cleanup

```powershell
aws cloudformation delete-stack --stack-name three-tier-vpc
aws cloudformation wait stack-delete-complete --stack-name three-tier-vpc
```

Notes on deletion behavior:
- The **RDS primary** and **EFS** have `DeletionPolicy: Snapshot`, so a final
  snapshot/recovery point is retained when the stack is deleted.
- The **read replica** is deleted with the stack (no snapshot).
- Backup recovery points in the vault are retained per the plan's lifecycle; the
  vault must be empty before it can be removed.

## Known limitations and caveats

- **Region lock:** CLOUDFRONT-scoped WAF forces `us-east-1` for a single-stack
  deployment.
- **Direct ALB access:** The ALB is still publicly resolvable, so traffic can
  bypass CloudFront/WAF by hitting the ALB DNS name directly. Restricting the ALB
  security group to CloudFront (prefix list or a secret origin header) is a
  recommended hardening step not yet applied.
- **Read-splitting and object cache disabled by default:** The RDS read replica
  and ElastiCache Redis cluster are provisioned, but WordPress is not wired to
  them automatically. The HyperDB `db.php` and Redis `object-cache.php` drop-ins
  were found to cause site-wide HTTP 500s when installed blindly at boot (they
  load before everything and fail hard on any incompatibility, and because
  `wp-content` lives on shared EFS a single bad drop-in poisons every instance).
  The PHP redis extension and `WP_REDIS_*` constants are in place, so enable the
  Redis Object Cache plugin from the WordPress admin once the site is healthy.
  For read-splitting, add a tested HyperDB config pointing at the replica
  endpoint.
- **Boot-time downloads:** WordPress core is fetched from the internet during
  instance boot via the public-subnet outbound path.
- **Runtime-verified:** The stack was deployed to `us-east-1` and the WordPress
  site confirmed serving HTTP 200 through CloudFront (health checks passing,
  2/2 targets healthy). An `X-Forwarded-Proto` fix in `wp-config.php` prevents an
  HTTP/HTTPS redirect loop behind CloudFront. If you redeploy and instances fail
  health checks, inspect `/var/log/user-data.log` on an instance.
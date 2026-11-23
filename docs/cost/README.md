# Cost

**Source of the figures:** my AWS Pricing Calculator estimate (link to be added). No figure on this page is estimated outside the calculator.

| | USD |
| --- | ---: |
| Total per month | *from my calculator* |
| Total per year (12 months) | *from my calculator* |

## What to enter in the AWS Pricing Calculator

**Region:** Europe (Ireland) `eu-west-1` unless stated otherwise. On-Demand prices.

| Service | What it does in the design | Configuration to enter |
| --- | --- | --- |
| AWS Fargate | Runs the photo app | Linux, x86; 2 tasks; 0.5 vCPU and 1 GB per task; 24 hours a day; 20 GB ephemeral storage |
| Elastic Load Balancing | The ALB behind CloudFront | 1 Application Load Balancer; expected new connections per second |
| Amazon RDS for PostgreSQL | Photo details | `db.t4g.micro`, **Multi-AZ**; 20 GB gp3; backup storage up to the database size (no extra charge) |
| AWS Secrets Manager | Holds the database password | 1 secret |
| Amazon S3, Standard / Intelligent-Tiering (Ireland) | Primary photo bucket | Expected GB of photos; PUT requests = uploads per month; GET requests = cache misses only |
| Amazon S3, Glacier Instant Retrieval (**London** `eu-west-2`) | Replica photo bucket | Same GB as the primary |
| S3 Cross-Region Replication | Copies photos to London | Data transfer inter-Region: GB uploaded per month |
| Amazon CloudFront | Serves photos from the edge | Data transfer out to the internet: GB viewed per month; HTTPS requests per month; regions Europe and Africa |
| Amazon VPC | Interface endpoints | 4 interface endpoints (`ecr.api`, `ecr.dkr`, `logs`, `secretsmanager`) × 2 Availability Zones; data processed |
| Amazon CloudWatch | Container logs | Logs ingested (GB) with 14-day retention |
| Amazon ECR | Container images | About 1 GB stored (15 images kept) |

**Not billed for this use:**
- the S3 gateway endpoint;
- ECS itself (you pay for Fargate);
- Application Auto Scaling;
- IAM and the GitHub OIDC provider;
- CloudFormation and Git sync;
- AWS CodeConnections;
- AWS Budgets (first two budgets).

GitHub Actions runs outside AWS.

## What drives the cost, and how it is kept down

| Cost driver | Control in the design |
| --- | --- |
| Fargate (2 tasks all month) | 0.5 vCPU / 1 GB per task, `MaxTasks: 4` caps busy days, and scaling adds tasks only when CPU passes 50%. |
| RDS Multi-AZ | The smallest Graviton class (`db.t4g.micro`) with storage autoscaling up to 50 GB, so I only pay for what is used. |
| Photo storage growth | Photos move to **Intelligent-Tiering** after 30 days, which lowers the price of photos nobody views. Deleted versions expire after 90 days in the primary bucket. |
| The copy in London | **Glacier Instant Retrieval**, the cheapest class that still returns a photo in milliseconds in a disaster. |
| Viewing growth | **CloudFront cache**: repeat views do not reach the ALB, the containers or S3. Data from the ALB to CloudFront is free. |
| Interface endpoints | Billed per hour per zone. They still cost less than a NAT gateway, and they keep the containers off the internet. |
| Logs | 14-day retention; Container Insights off. |
| Unexpected spend | AWS Budgets emails at 80% of actual and 100% of forecast spend, against `MonthlyBudgetUsd` in `deployments/dev/6-service.yaml`. |

## Alternatives considered (cost and resilience)

| Option | Why rejected |
| --- | --- |
| Photos in the database | Database storage costs far more per GB than S3, and backups grow with every photo. |
| NAT gateway instead of endpoints | Gives the app a path to the internet, and costs more per hour plus per GB. |
| Aurora instead of RDS | Higher minimum cost for a small table. |
| Replica in S3 Standard | It costs more per GB to store a copy that is read only in a disaster. |
| No CDN | Every view costs data transfer from the ALB and container time. |

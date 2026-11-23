# Discovery Report: Storing and Serving Delivery Photos

| | |
| --- | --- |
| Lab | Week 5 Lab 1: Storing and Serving Delivery Photos |
| Name | Jado Fils SEZIKEYE |
| Date | 2026-11-23 |
| Client | Onward Logistics (see the [Client Profile](../client-profile/README.md)) |

## 1. What I found

### The client's need

Onward needs to keep every rider's delivery photo safe for as long as it is needed as dispute evidence. Customers and merchants anywhere must be able to view their photos quickly, and nobody outside Onward's system may see or share them.

### Solutions that exist

| Solution | How it works | Trade-off for this client |
| --- | --- | --- |
| Store photos inside the database | Each photo is saved as binary data in the same table as its details. | **Strength:** one place to back up. **Weakness:** the database grows fast, gets slow and expensive, and is hard to copy to another Region; it also fails the client's check that photos are not in the database. |
| Object storage plus a database for the details | Photos go to an object store (Amazon S3); a relational database keeps the details and the photo's key. The app serves the photo. | **Strength:** cheap, durable storage that grows without limits, can be copied to a second Region automatically and kept after a deletion. **Weakness:** two stores must be kept in step, which the app has to handle. |
| Public object storage or a CDN link per photo | Photos are stored in a bucket and viewed through a public or pre-signed link. | **Strength:** the simplest and fastest to build. **Weakness:** anyone holding the link can see and share the photo. That breaks the Head of Security's requirement. |

### What happens with no solution

- **If photos are lost:** Onward loses its evidence in delivery disputes. It must refund or re-deliver without proof, so money is lost on margins that are already thin.
- **If photos leak:** merchants and customers can file copyright and privacy complaints. Data protection rules apply in every market Onward works in, so a leak is also a legal risk. Customer trust, which Onward calls its product, suffers.
- **Code Spaces, 2014:** the company kept its data and its backups in one AWS account. An attacker deleted both, and the company shut down within days.
- **S3 outage, 2017:** an outage in a single AWS Region (us-east-1) made many services unavailable for hours, which shows why one Region is not enough for data that must never be lost.

### Sources

- [Amazon S3 replication](https://docs.aws.amazon.com/AmazonS3/latest/userguide/replication.html)
- [Retaining multiple versions of objects with S3 Versioning](https://docs.aws.amazon.com/AmazonS3/latest/userguide/Versioning.html)
- [Blocking public access to your Amazon S3 storage](https://docs.aws.amazon.com/AmazonS3/latest/userguide/access-control-block-public-access.html)
- [Controlling access from VPC endpoints with bucket policies](https://docs.aws.amazon.com/AmazonS3/latest/userguide/example-bucket-policies-vpc-endpoint.html)
- [Amazon RDS Multi-AZ DB instance deployments](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Concepts.MultiAZSingleStandby.html)
- [Password management with Amazon RDS and AWS Secrets Manager](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/rds-secrets-manager.html)
- [Restoring a DB instance to a specified time](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_PIT.html)
- [Restrict access to Application Load Balancers (CloudFront)](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/restrict-access-to-load-balancer.html)
- [Choosing the price class for a CloudFront distribution](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/PriceClass.html)
- [Summary of the Amazon S3 service disruption in us-east-1 (2017)](https://aws.amazon.com/message/41926/)
- Code Spaces shutdown after its AWS account was compromised (2014): InfoWorld, "Murder in the Amazon cloud"

## 2. The business value of my solution

### My chosen approach

I chose **object storage plus a database**:
- photos are stored in a private, versioned S3 bucket in Ireland, and copied automatically to a second bucket in London;
- their details are kept in a managed PostgreSQL database;
- one containerized application is the only way in. It uploads photos, lists them and streams them to viewers through CloudFront.

This suits Onward better than the other options because:
- storage cost grows slowly as photos pile up;
- every photo survives a mistake, an attack or the loss of a whole Region;
- photos cannot be opened or shared outside the application.

### Services I will use

| Service | The job it does | Why this service, used this way |
| --- | --- | --- |
| Amazon S3 (Ireland) | Holds the photo files | Durable and cheap per GB. **Versioning** keeps deleted photos. **Block Public Access** plus a bucket policy that allows downloads only through the VPC endpoint means no direct access. Old photos move to **Intelligent-Tiering** to lower cost. |
| S3 Cross-Region Replication → S3 (London) | Second copy of every photo | Survives the loss of the Region or the bucket. Deletions are **not** replicated, and nobody may permanently delete a version. The copy uses **Glacier Instant Retrieval**, which is cheap but still instant in a disaster. |
| Amazon RDS for PostgreSQL | Photo details, linked to each photo's S3 key | Managed, not self-managed. **Multi-AZ** survives the loss of a zone. **7-day point-in-time restore** covers mistakes. It sits in isolated subnets and is not publicly accessible. |
| AWS Secrets Manager | Holds the database password | RDS creates and rotates the password, and the app reads it at runtime with its IAM role. No password exists in code or Git. |
| Amazon ECS on AWS Fargate | Runs the application | No servers to manage. Two or more tasks in private subnets across two zones, with automatic replacement and scaling. |
| Application Load Balancer | Spreads requests across the tasks | Spans two zones and health-checks each task. It accepts traffic only from CloudFront. |
| Amazon CloudFront | Serves photos close to customers | Photos are cached at edge locations, including in Africa, with HTTPS, HTTP/2 and HTTP/3. Repeat views cost less and skip the app. |
| VPC endpoints (S3, ECR, Logs, Secrets Manager) | Private paths to AWS services | No NAT gateway and no internet route from the containers or the database. |
| Amazon ECR + GitHub OIDC | Stores the app image; keyless pushes from GitHub | No long-lived keys. Only the main branch of the app repository can push. |
| CloudFormation + Git sync | Builds everything from Git | Repeatable and reviewed. The whole setup can be rebuilt from the repositories. |
| AWS Budgets, CloudWatch Logs | Cost alerts; container logs | Spend is visible and alerted. Logs are kept for 14 days. |

### Who benefits, and how I will know

| Stakeholder | Benefit | How I would know |
| --- | --- | --- |
| Chief Technology Officer | Photos load quickly in every city, however many people view them. | The second request for a photo shows `Hit from cloudfront`. The app keeps answering when a task or a zone is stopped. |
| Head of Security | Photos are seen only through Onward's system; there are no credentials in code and no public database. | A direct S3 request returns 403, even for an administrator. The database has `PubliclyAccessible = false`. gitleaks passes on both repositories. |
| Head of Operations | No photo is ever lost, even if a Region or the bucket is lost, and deletions can be undone. | A photo uploaded today has `ReplicationStatus = COMPLETED` and exists in London. A deleted photo is restored from the app. |
| Finance Lead | Clear, controlled cost that grows slowly with photos and views. | The AWS Pricing Calculator estimate is linked in the cost page. Budget alerts are active. |
| Merchants and customers | Their photos are safe, private and quick to open on mobile connections. | Fewer disputes left unresolved for lack of evidence, and no leaked photos. |

### Limits and risks of my design

- **One Region for details:** if Ireland is lost, the photo *details* in RDS are not in London. The photos are, and they can be matched again by delivery reference from the key and object metadata. Copying RDS snapshots to London would close this gap.
- **Replication is not instant:** a photo uploaded seconds before a Regional failure may not be in London yet. S3 Replication Time Control would guarantee 15 minutes, at extra cost.
- **No sign-in:** anyone with the application URL can view the gallery. The lab does not require sign-in, but production needs it, and then signed cookies on CloudFront.
- **Edge cache after deletion:** a deleted photo can still be served from a CloudFront edge for up to a day, unless the cache is invalidated.
- **Origin traffic:** CloudFront reaches the ALB over HTTP inside AWS, because there is no custom domain or certificate in this lab. Viewers always use HTTPS.

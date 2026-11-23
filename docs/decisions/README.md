# Decisions

Each decision with its reason in business terms, and the options I rejected (company standard: *write down each decision with its reason*).

## D1. Photos in Amazon S3; their details in Amazon RDS

**Decision:** photo files go in an S3 bucket. Their details (title, description, delivery reference, city, uploader) go in RDS for PostgreSQL. Each row stores the photo's S3 key and version, which is the link between the two.

| Rejected | Why |
| --- | --- |
| Photos inside the database (BYTEA/BLOB) | It makes the database large, slow and expensive to back up. Database storage costs several times more per GB than S3. The lab's check "the photo file is not stored in the database" forbids it. |
| Amazon EFS | Built for shared file systems for servers. It costs more per GB, has no built-in replication to a second Region that keeps deleted files, and no edge caching. |
| Amazon EBS | Attached to one server in one Availability Zone. Fargate tasks cannot share it, and it does not survive the loss of that zone. |

## D2. A Multi-AZ RDS for PostgreSQL instance, not self-managed

**Decision:** RDS for PostgreSQL 17 on `db.t4g.micro`, Multi-AZ, encrypted, in isolated subnets.
- **Multi-AZ:** a standby in the second zone takes over automatically.
- **Backups:** 7-day automated backups give point-in-time restore. Deletion protection is on, and a final snapshot is taken if the stack is deleted.

| Rejected | Why |
| --- | --- |
| PostgreSQL on EC2 or in a container | The lab forbids a self-managed database. It would also leave patching, backups and failover to a small team. |
| Aurora PostgreSQL | Its minimum cost is higher. A Multi-AZ instance already meets the need for a small table of photo details. |
| Single-AZ RDS | The app must keep working if an Availability Zone is lost, and a Single-AZ database would make the database the weak point. |

## D3. No credentials anywhere in code, images or Git

**Decision:**
- RDS creates the database password itself and keeps it in **Secrets Manager** (`ManageMasterUserPassword`). The app reads it at runtime with the ECS **task role**.
- S3 is reached with the same task role.
- GitHub Actions signs in with **OIDC**.
- CI fails on any committed credential (gitleaks across the full history).

The app re-reads the secret if the database rejects the password, so RDS's automatic password rotation does not break new connections.

| Rejected | Why |
| --- | --- |
| Password in an environment variable or properties file | It is visible to anyone who can read the task definition or the repository. |
| ECS `secrets` injection at start-up | It works, but after a rotation the running tasks keep the old password until they restart. |
| IAM database authentication | Adds a database user set-up step and 15-minute tokens, more than this lab needs. I note it as a next step. |

## D4. Photos viewable only through the Onward application

**Decision:** two layers, so that one mistake does not expose the photos.
- **Block Public Access** is on.
- A **bucket policy** refuses `GetObject` / `GetObjectVersion` unless the request comes through this VPC's S3 gateway endpoint. Only the application's containers are in that VPC.

The app streams photos from `/api/photos/<id>/image`. The browser never learns the bucket or key.

| Rejected | Why |
| --- | --- |
| Public bucket or public object URLs | Anyone could see and share the photos. This breaks the Head of Security's requirement. |
| S3 pre-signed URLs | Anyone holding the link can open the photo until it expires, so it can be passed around. |
| CloudFront with Origin Access Control directly on the bucket | Exposes a CDN URL for each photo file outside the application. Serving through the app keeps one controlled path. I keep this option for a later stage, with signed cookies once sign-in exists. |

## D5. Every photo survives the loss of a Region or of the bucket

**Decision:** **S3 Cross-Region Replication** from Ireland to Frankfurt. Both buckets are versioned.
- **Delete markers are not replicated**, so a deletion in Ireland never removes the Frankfurt copy.
- Both bucket policies deny `s3:DeleteObjectVersion` to everyone, so no person or attacker can permanently delete a photo version. Only lifecycle rules expire old versions, and only in the primary bucket.
- The replica uses **S3 Glacier Instant Retrieval**: the copy is read only in a disaster, and this class still returns a photo in milliseconds.

| Rejected | Why |
| --- | --- |
| Same-Region backup only (AWS Backup, versioning) | It does not survive the loss of the Region. |
| Replicating delete markers | A deletion, accidental or malicious, would then reach the copy too. |
| Replication Time Control (15-minute guarantee) | Costs more. Normal replication usually finishes in minutes, which is enough for dev. It is the first upgrade for prod. |

## D6. Recovery from accidental deletion

**Decision:**
- **Photos:** S3 versioning. A deleted photo is kept as a previous version for 90 days in Ireland, and indefinitely in Frankfurt. The app's *Restore* button copies that version back. The app has no permission to delete versions.
- **Details:** rows are soft-deleted (`deleted_at`), so *Restore* brings them back. For a mistake made in the database itself, RDS point-in-time restore goes back to any second in the last 7 days.

## D7. High availability

**Decision:**
- At least two Fargate tasks, one in each Availability Zone. ECS replaces a failed task. Zone rebalancing spreads tasks again after a zone recovers.
- The ALB spans both zones, and RDS is Multi-AZ.
- The `/health` check deliberately does not test the database, so a database failover does not mark every container unhealthy at once.

## D8. Fast, low-cost viewing in different cities (stretch)

**Decision:** **Amazon CloudFront** in front of the ALB.
- **Photos** (`/api/photos/*/image`) use the managed *CachingOptimized* policy: cached at the edge for a day, compressed, HTTP/2 and HTTP/3.
- **Everything else** uses *CachingDisabled*: uploads, lists, pages.
- **Price class 200** includes edge locations in Africa, where the customers are, and Europe, without the most expensive locations.
- **Lock-down:** the ALB accepts only CloudFront's origin-facing IP ranges (AWS-managed prefix list), so viewers cannot bypass the cache.

Cost of viewing: data sent from CloudFront to viewers costs less per GB than from the ALB to the internet, and data from the ALB to CloudFront is free. Cache hits also cost nothing on the ALB, the containers or S3 requests. Viewing cost therefore grows more slowly than viewing itself.

| Rejected | Why |
| --- | --- |
| No CDN | Every view travels from Ireland to West Africa and through the containers. It is slower on mobile networks and costs more per view as viewing grows. |
| S3 Transfer Acceleration | It speeds up *uploads* to S3, not viewing, and it needs direct bucket access, which the Head of Security forbids. |
| PriceClass_All | Adds the most expensive edge locations, with little gain for West African customers. |
| PriceClass_100 | Leaves out the edge locations in Africa. |

## D9. Infrastructure by Git sync; separate repositories

**Decision:**
- All resources come from CloudFormation templates deployed by Git sync. The only exceptions are the bootstrap stacks (the GitHub connection and the Git sync role), which Git sync cannot create for itself.
- Application code and infrastructure are in separate repositories.
- Releases are a reviewed change to `ImageTag` in this repository.
- Names use `onward-dev-photos-*`, so this lab never collides with other labs in the same account.
- There are no inline IAM policies; CI enforces this.

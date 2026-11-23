# Demo: proving the client's checks

`URL` is the output `PhotoServiceUrl` of `onward-dev-photos-cdn`. The commands assume `aws configure` / `aws login` for the account, with Region `eu-west-1` unless stated otherwise.

```bash
ACCOUNT=$(aws sts get-caller-identity --query Account --output text)
BUCKET=onward-dev-photos-$ACCOUNT
REPLICA=onward-dev-photos-replica-$ACCOUNT
```

## 1. A photo uploaded through the app appears in the gallery with its details

Open `URL`, upload a photo with a title, description, delivery reference and city. It appears in the gallery with those details. Open **About** to see my name and the lab name.

## 2. The photo is not in the database, and the database is not reachable from the internet

```bash
# The details are in RDS; the file is in S3 under the key stored in that row.
aws s3api list-object-versions --bucket $BUCKET --prefix photos/ --query 'Versions[].[Key,VersionId,Size]' --output table

# Not publicly accessible, private subnets only, security group accepts the containers only.
aws rds describe-db-instances --db-instance-identifier onward-dev-photos-db \
  --query 'DBInstances[0].[PubliclyAccessible,MultiAZ,Endpoint.Address]'
aws ec2 describe-security-groups --filters Name=group-name,Values=onward-dev-photos-db-sg \
  --query 'SecurityGroups[0].IpPermissions'
```

The endpoint name resolves to a private `10.20.20.x` / `10.20.21.x` address. The database route table has no internet route. The table holds `object_key` (text), never the image bytes: see [V1__create_photo.sql](https://github.com/jadofils/aws-db-s3-storage-app/blob/main/app/src/main/resources/db/migration/V1__create_photo.sql).

## 3. A direct request to the storage location is refused

```bash
KEY=$(aws s3api list-objects-v2 --bucket $BUCKET --prefix photos/ --query 'Contents[0].Key' --output text)
curl -I "https://$BUCKET.s3.eu-west-1.amazonaws.com/$KEY"          # 403: no public access
aws s3 cp "s3://$BUCKET/$KEY" /tmp/photo                            # AccessDenied even for me: not from the VPC endpoint
curl -I "http://<ALB DNS name>/"                                    # times out: the ALB accepts CloudFront only
```

The photo loads only from `URL/api/photos/<id>/image`.

## 4. No credentials in the repositories or the application code

- Both repositories run **gitleaks** over their full history on every push. The CI badge or run is green.
- The app gets its database password from Secrets Manager and its AWS access from the ECS task role. See `DatabaseConfig.java`.
- GitHub Actions signs in with OIDC.

## 5. A deleted photo or record can be recovered

In the app, click **Delete** on a photo. It moves to *Recently deleted*. Click **Restore** and it is back.

Behind the scenes, S3 shows the delete marker and the earlier version:

```bash
aws s3api list-object-versions --bucket $BUCKET --prefix "$KEY" \
  --query '{versions:Versions[].[VersionId,IsLatest],markers:DeleteMarkers[].[VersionId,IsLatest]}'
```

For a record broken in the database itself, use RDS point-in-time restore to a new instance (any second in the last 7 days):

```bash
aws rds describe-db-instances --db-instance-identifier onward-dev-photos-db --query 'DBInstances[0].LatestRestorableTime'
```

## 6. A photo uploaded today also exists in the second Region

```bash
aws s3api head-object --bucket $BUCKET --key "$KEY" --query ReplicationStatus          # COMPLETED
aws s3api head-object --bucket $REPLICA --key "$KEY" --region eu-central-1 --query '[StorageClass,ContentLength]'
```

**Protected if Ireland is lost:** every photo (Frankfurt copy, deletions not replicated).

**Not protected:**
- the photo *details* in RDS (latest snapshot only, same Region);
- the running application;
- uploads made in the last few minutes before the failure, if they had not yet replicated.

To recover, the stacks can be rebuilt from Git in another Region, pointing at the Frankfurt bucket.

## 7. The application keeps working if a container or an Availability Zone is lost

```bash
# Stop one task: the ALB keeps serving from the other zone, ECS starts a replacement.
TASK=$(aws ecs list-tasks --cluster onward-dev-photos --query 'taskArns[0]' --output text)
aws ecs stop-task --cluster onward-dev-photos --task $TASK
```

Refresh **About** several times. *Answered by* shows the host of each task: `10.20.10.x` is in zone a, `10.20.11.x` in zone b.

For a zone, **RDS** can show a failover with `aws rds reboot-db-instance --db-instance-identifier onward-dev-photos-db --force-failover`.

## 8. Stretch: photos load quickly in different locations

```bash
curl -sI "$URL/api/photos/<id>/image" | grep -iE 'x-cache|x-amz-cf-pop|age|cache-control'
```

The first request shows `Miss from cloudfront`; the next one shows `Hit from cloudfront` with an `Age`. `x-amz-cf-pop` is the edge location that answered.

## 9. Cost and alternatives

See [cost](../cost/README.md) and [decisions](../decisions/README.md).

## 10. The whole setup can be rebuilt from the repositories

Follow *Setting it up from nothing* in the [README](../../README.md): two bootstrap stacks, then seven Git sync stacks.

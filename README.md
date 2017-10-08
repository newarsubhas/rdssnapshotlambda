# iam_policy.json
```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "arn:aws:logs:*:*:*"
    },
    {
      "Action": [
        "rds:CreateDBSnapshot",
        "rds:DeleteDBSnapshot",
        "rds:DescribeDBSnapshots"
      ],
      "Effect": "Allow",
      "Resource": "*"
    }
  ]
}


```
# lambda_handler.py
```
import boto3
import time
from botocore.client import ClientError
from datetime import datetime, timedelta, tzinfo

region='us-east-1'

rds = boto3.client('rds', region_name=region)

class JST(tzinfo):
    def utcoffset(self, dt):
        return timedelta(hours=9)
    def dst(self, dt):
        return timedelta(0)
    def tzname(self, dt):
        return 'JST'

def create_snapshot(prefix, instanceid):
    snapshotid = "-".join([prefix, datetime.now(tz=JST()).strftime("%Y-%m-%dT%H%M")])
    
    for i in range(0, 5):
        try:
            snapshot = rds.create_db_snapshot(
                DBSnapshotIdentifier=snapshotid,
                DBInstanceIdentifier=instanceid
            )
            return
        except ClientError as e:
                print(str(e))
        time.sleep(1)

def delete_snapshots(prefix, days):
    snapshots = rds.describe_db_snapshots()
    now = datetime.utcnow().replace(tzinfo=None)
    for snapshot in snapshots['DBSnapshots']:
        # creating snapshot
        if not snapshot.has_key('SnapshotCreateTime'):
            continue
        
        delta = now - snapshot['SnapshotCreateTime'].replace(tzinfo=None)
        if snapshot['DBSnapshotIdentifier'].startswith(prefix) and delta.days >= days:
            rds.delete_db_snapshot(DBSnapshotIdentifier=snapshot['DBSnapshotIdentifier'])

def lambda_handler(event, context):
    delete_days = 1
    snapshot_prefix = "manually-snapshot-myrdsdatabase"
    instances = ["myrdsdatabase"]
    
    for instance in instances:
        create_snapshot(snapshot_prefix, instance)
        
    delete_snapshots(snapshot_prefix, delete_days)
    
    snapshots = rds.describe_db_snapshots()
    for snapshot in snapshots['DBSnapshots']:
        response = snapshot['DBSnapshotArn']
        print(response)

```


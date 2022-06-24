# Bootstrap juju controller on AWS and deploy Ceph juju bundle

```
# Bootstrap juju controller on AWS
juju bootstrap aws/eu-central-1 aws

# Deploy Ceph + RadosGW bundle
juju deploy ./ceph-bundle.yaml

# Watch the model deploying until all units are in active/idle state
watch -c juju status --color
```

# Verify integration of Ceph RADOS Gateway with Ceph
```
juju ssh ceph-radosgw/0 'sudo rados --user rgw.$(hostname) df'
```

# Configure a user for Ceph Radosgw
```
# Create user in Ceph Radosgw
# https://docs.ceph.com/en/latest/radosgw/admin/
juju ssh ceph-radosgw/0 \
  'sudo radosgw-admin --user rgw.$(hostname) user create \
    --uid=ubuntu --display-name="Ubuntu"'
{
    "user_id": "ubuntu",
    "display_name": "Ubuntu",
    "email": "",
    "suspended": 0,
    "max_buckets": 1000,
    "subusers": [],
    "keys": [
        {
            "user": "ubuntu",
            "access_key": "YST4CQX5W3NWTKV4XCJR",
            "secret_key": "t9LlAnAEJAPrxMZvUcD4RrmsS2fdozm6IH8mMWE9"
        }
    ],
    "swift_keys": [],
    "caps": [],
    "op_mask": "read, write, delete",
    "default_placement": "",
    "default_storage_class": "",
    "placement_tags": [],
    "bucket_quota": {
        "enabled": false,
        "check_on_raw": false,
        "max_size": -1,
        "max_size_kb": 0,
        "max_objects": -1
    },
    "user_quota": {
        "enabled": false,
        "check_on_raw": false,
        "max_size": -1,
        "max_size_kb": 0,
        "max_objects": -1
    },
    "temp_url_keys": [],
    "type": "rgw",
    "mfa_ids": []
}
 
# Install `aws` CLI:
# https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html
 
# Configure AWS CLI
# Values used below come from the output of the `radosgw-admin user
# create` command (see `access_key` and `secret_key` fields)
export AWS_ACCESS_KEY_ID=YST4CQX5W3NWTKV4XCJR
export AWS_SECRET_ACCESS_KEY=t9LlAnAEJAPrxMZvUcD4RrmsS2fdozm6IH8mMWE9

# Read ceph-radosgw IP
CEPH_RADOSGW_IP=$(juju status ceph-radosgw --format json | jq -r '.machines[]."dns-name"')

# Create a bucket
aws --endpoint=http://$CEPH_RADOSGW_IP:80 \
  s3api create-bucket --bucket test-bucket
 
# List buckets
aws --endpoint=http://$CEPH_RADOSGW_IP:80 s3api list-buckets
{
    "Buckets": [
        {
            "Name": "test-bucket",
            "CreationDate": "2022-06-24T06:28:41.149000+00:00"
        }
    ],
    "Owner": {
        "DisplayName": "Ubuntu",
        "ID": "ubuntu"
    }
}
```

# Clean up

When done with testing, tear down the whole environment:
```
juju kill-controller aws --timeout 0 --yes
```

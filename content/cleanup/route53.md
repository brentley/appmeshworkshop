---
title: "Cleanup the Route53 Hosted Zone"
date: 2018-08-07T12:37:34-07:00
weight: 60
---

* Delete the Route53 Hosted Zone.

```bash
 # Define variables #
HOSTED_ZONE_ID=$(aws route53 list-hosted-zones-by-name \
    --dns-name appmeshworkshop.hosted.local \
    --max-items 1 | \
  jq -r ' .HostedZones | first | .Id');
  CRYSTAL_RECORD_SET=$(aws route53 list-resource-record-sets --hosted-zone-id=$HOSTED_ZONE_ID | \
 jq -r '.ResourceRecordSets[] | select (.Name == "crystal.appmeshworkshop.hosted.local.")');
  NODEJS_RECORD_SET=$(aws route53 list-resource-record-sets --hosted-zone-id=$HOSTED_ZONE_ID | \
 jq -r '.ResourceRecordSets[] | select (.Name == "nodejs.appmeshworkshop.hosted.local.")');

# Create temaplate file
cat <<-EOF > /tmp/delete_r53.json
{
  "Comment": "DELETE crystal.appmeshworkshop.hosted.local and nodejs.appmeshworkshop.hosted.local",
  "Changes": [
    {
      "Action": "DELETE",
      "ResourceRecordSet": $CRYSTAL_RECORD_SET
    },
    {
      "Action": "DELETE",
      "ResourceRecordSet": $NODEJS_RECORD_SET    
    }
  ]
}
EOF

# Delete hosted zone
aws route53 change-resource-record-sets \
  --hosted-zone-id $HOSTED_ZONE_ID \
  --change-batch file:///tmp/delete_r53.json

aws route53 delete-hosted-zone \
--id $HOSTED_ZONE_ID
```
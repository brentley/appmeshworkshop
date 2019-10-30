---
title: "Cleanup the SSM resources"
date: 2018-08-07T13:37:53-07:00
weight: 30
---

* Delete the SSM association.

```bash
# Define variables #
ASSOCIATION=$(aws ssm list-associations \
    --association-filter key=AssociationName,value=appmesh-workshop-state | \
  jq -r ' .Associations | first | .AssociationId')
# Delete ssm association #
aws ssm delete-association \
  --association-id $ASSOCIATION
```

* Delete the SSM document.

```bash
# Delete ssm document #
aws ssm delete-document \
  --name appmesh-workshop-installenvoy
```
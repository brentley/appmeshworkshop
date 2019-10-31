---
title: "Delete the baseline template"
date: 2018-08-07T12:37:34-07:00
weight: 70
---

* Delete the infrastructure deployed by the baseline template.

```bash
 # Define variables #
STACK_NAME=$(jq < cfn-output.json -r '.StackName');
# Delete cloudformation stack #
aws cloudformation delete-stack \
  --stack-name $STACK_NAME
aws cloudformation wait stack-delete-complete \
  --stack-name $STACK_NAME
```
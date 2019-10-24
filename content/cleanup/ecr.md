---
title: "Cleanup the ECR repositories"
date: 2018-08-07T12:37:34-07:00
weight: 50
---

* Delete the container images.

```bash
 # Define variables #
CRYSTAL_ECR_REPO=$(jq < cfn-output.json -r '.CrystalEcrRepo' | cut -d'/' -f2)
NODEJS_ECR_REPO=$(jq < cfn-output.json -r '.NodeJSEcrRepo' | cut -d'/' -f2)
# Delete ecr images #
aws ecr list-images \
  --repository-name $CRYSTAL_ECR_REPO | \
jq -r ' .imageIds[] | [ .imageDigest ] | @tsv ' | \
  while IFS=$'\t' read -r imageDigest; do 
    aws ecr batch-delete-image \
      --repository-name $CRYSTAL_ECR_REPO \
      --image-ids imageDigest=$imageDigest
  done
aws ecr list-images \
  --repository-name $NODEJS_ECR_REPO | \
jq -r ' .imageIds[] | [ .imageDigest ] | @tsv ' | \
  while IFS=$'\t' read -r imageDigest; do 
    aws ecr batch-delete-image \
      --repository-name $NODEJS_ECR_REPO \
      --image-ids imageDigest=$imageDigest
  done
```
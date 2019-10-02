---
title: "Build the EKS cluster"
chapter: false
weight: 42
---

The following script will do the following:

 - Confirm your IAM role is set correctly
 - Build the EKS Control Plane
 - Build the EKS Workers on the private subnets in our existing VPC

#### Run the build script!
```bash
cd ~/environment
scripts/build-eks
```


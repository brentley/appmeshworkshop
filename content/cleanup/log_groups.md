---
title: "Delete the CW log groups"
date: 2018-08-07T12:37:34-07:00
weight: 80
---

* Delete the Log Groups from CloudWatch logs.

```bash
# Delete CloudWatch log groups
aws logs delete-log-group --log-group-name appmesh-workshop-crystal-envoy
aws logs delete-log-group --log-group-name appmesh-workshop-frontend-envoy

```
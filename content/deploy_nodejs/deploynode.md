---
title: "Deploy the NodeJS Service"
date: 2018-09-18T16:01:14-05:00
weight: 5
---

```bash
cd ~/environment

# deploy with 3 replicas
yq w ecsdemo-nodejs/kubernetes/deployment.yaml spec.replicas 3 | kubectl apply -f -

# deploy service type LoadBalancer rather than default ClusterIP
yq w ecsdemo-nodejs/kubernetes/service.yaml spec.type LoadBalancer | kubectl apply -f -
```

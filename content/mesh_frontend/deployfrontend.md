---
title: "Install the envoy proxy"
date: 2018-09-18T17:40:09-05:00
weight: 25
---

Let’s bring up the Ruby Frontend!

Copy/Paste the following commands into your Cloud9 workspace:

```
cd ~/environment/ecsdemo-frontend
kubectl apply -f kubernetes/deployment.yaml
kubectl apply -f kubernetes/service.yaml
```

We can watch the progress by looking at the deployment status:
```
kubectl get deployment ecsdemo-frontend
```

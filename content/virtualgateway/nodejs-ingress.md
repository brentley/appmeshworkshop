---
title: "Virtual Gateway Ingress to NodeJS service"
date: 2018-09-18T17:39:30-05:00
weight: 10
---

So first thing we need to do is to edit our K8S Namespace to include a label that is used by the Virtual Gateway we are about to create.

```bash
kubectl label namespace appmesh-workshop-ns gateway=ingress-gw
```


---
title: "Create the service mesh"
date: 2018-09-18T16:01:14-05:00
weight: 5
---

**AWS App Mesh** is a service mesh based on the **Envoy proxy**. It standardizes how microservices communicate, giving you end-to-end visibility and helping to ensure high-availability for your applications.

Throughout this workshop we will use App Mesh, to gain consistent visibility and network traffic controls on our microservices running in **Amazon EC2, AWS Fargate and Amazon EKS**. 

Let's start by creating the service mesh. A service mesh is a logical boundary for network traffic between the services that reside within it.

```bash
# Create mesh #
aws appmesh create-mesh \
  --mesh-name appmesh-workshop \
  --spec egressFilter={type=DROP_ALL}
```

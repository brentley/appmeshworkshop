---
title: "Update the crystal code"
date: 2018-09-18T16:01:14-05:00
weight: 5
---

Our strategy will be to deploy a new version of the crystal service and begin shifting traffic to it. If all goes well, then we will continue to shift more traffic to the new version until it is serving 100% of all requests. 

**Canary release** is a software development strategy in which a new version of a service (as well as other software) is deployed as a canary release for testing purposes, and the base version remains deployed as a production release for normal operations.

In a canary release deployment, total traffic is separated at random into a production release and a canary release with a pre-configured ratio. Typically, the canary release receives a small percentage of the traffic and the production release takes up the rest. The updated service features are only visible to the traffic through the canary. You can adjust the canary traffic percentage to optimize test coverage or performance.

Remember, in App Mesh, every version of a service is ultimately backed by actual running code somewhere (Fargate tasks in the case of crystal), so each service will have it's own virtual node representation in the mesh that provides this conduit.

Additionaly, there is the physical deployment of the application itself to a compute environment. Both crystal deployments will run on ECS using the Fargate launch type. Our goal is to test with a portion of traffic going to the new version, ultimately increasing to 100% of traffic.

* On your Cloud9 environment open the **ecsdemo-crystal** project

```bash
# Create mesh #
aws appmesh create-mesh \
  --mesh-name appmesh-workshop \
  --spec egressFilter={type=DROP_ALL}
```
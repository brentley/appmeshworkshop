---
title: "Create the virtual services"
date: 2018-09-18T17:39:30-05:00
weight: 20
---

A virtual service is an abstraction of a real service that is provided by a virtual node in the mesh. Virtual nodes act as a logical pointer to a particular task group, such as an Amazon ECS service or a Kubernetes deployment.

Let’s create the virtual services for our application microservices:

```
aws appmesh create-virtual-service --mesh-name AppMesh-Workshop \
                                   --virtual-service-name frontend.appmeshworkshop.service
```

```
aws appmesh create-virtual-service --mesh-name AppMesh-Workshop \
                                   --virtual-service-name crystal.appmeshworkshop.service
```

```
aws appmesh create-virtual-service --mesh-name AppMesh-Workshop \
                                   --virtual-service-name nodejs.appmeshworkshop.service
```
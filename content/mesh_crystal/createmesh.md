---
title: "Create the service mesh"
date: 2018-09-18T16:01:14-05:00
weight: 5
---

**AWS App Mesh** is a service mesh based on the **Envoy proxy**. It standardizes how microservices communicate, giving you end-to-end visibility and helping to ensure high-availability for your applications.

Throughout this workshop we will use AWS App Mesh, to gain consistent visibility and network traffic controls on our microservices running in **Amazon EC2, AWS Fargate and Amazon EKS**. 

A service mesh is composed of two parts, a control plane and a data plane. AWS App Mesh is a managed control plane so you as a user don't need to install or manage any servers to run it. The data plane for App Mesh is the open source Envoy proxy. It is your responsability to add the Envoy proxy as a side car to each microservice you want to expose and manage via App Mesh.

In the next few sections, you will perform 2 broad set of tasks: 

* You will use the AWS CLI to create and configure the App Mesh resources needed to represent the microservices available in your environment like virtual services and virtual nodes. 

* You will add a docker container running the Envoy proxy to each compute environment where you have microservices running. AWS App Mesh supports EC2, ECS and EKS and will show you how to add Envoy in each of this environments, without modifying your microservice source code. The process for enabling the Envoy proxy is different for each compute environment. In the following sections you will perform the changes needed to enable each compute environment so it can be 

A key feature of Envoy worth mentioning is that it exposes a set of APIs. These APIs are leverage by AWS App Mesh to dynamically configure Envoy's routing logic, freeing developers from the tedious task of manually updating config files.

Now every time you launch an Envoy enabled microservice, the Envoy proxy will contact the AWS App Mesh Management API to subscribe to resource information for Listeners, Clusters, Routes, and Endpoints. The connection from the Envoy proxy to the AWS App Mesh management API endpoint is held, which allows AWS App Mesh to stream updates to the Envoy proxy as users introduce changes to either of the resources listed above.   

Let's start by creating the service mesh. A service mesh is a logical boundary for network traffic between the services that reside within it.

```bash
# Create mesh #
aws appmesh create-mesh \
  --mesh-name appmesh-workshop \
  --spec egressFilter={type=DROP_ALL}
```

---
title: "Distributed Tracing with X-Ray"
chapter: true
weight: 30
---

# Add End-to-End Tracing Capabilities

![x-ray](/images/app_mesh_architecture/AppMeshWorkshopXray.png)

[AWS X-Ray](https://aws.amazon.com/xray/) integrates with AWS App Mesh to manage Envoy proxies for microservices. App Mesh provides a version of Envoy that you can configure to send trace data to the X-Ray daemon running in a container of the same task or pod. X-Ray supports tracing with the following App Mesh compatible services:

* Amazon Elastic Container Service (Amazon ECS)
* Amazon Elastic Kubernetes Service (Amazon EKS)
* Amazon Elastic Compute Cloud (Amazon EC2)

Let's now enable the X-Ray integration for the Envoy proxies:

{{% children showhidden="false" %}}

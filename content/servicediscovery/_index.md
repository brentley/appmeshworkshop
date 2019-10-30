---
title: "Cloud Map Service Discovery"
chapter: true
weight: 35
---

# Use Cloud Map based service discovery 

![monitoring](/images/app_mesh_architecture/AppMeshWorkshopCloudMap.png)

Until now, the way our frontend application running in the EC2 instances was talking to the backend Crystal service running in the ECS Cluster by using a dedicated Load Balancer.

Part of the transition to microservices and modern architectures involves having dynamic, autoscaling, and robust services that can respond quickly to failures and changing loads. A modern architectural best practice is to loosely couple these services by allowing them to specify their own dependencies. Compared to dedicated load balancing, service discovery (client side load balancing) can help improve resiliency, and convenience in dynamic and large microservice environments.

[AWS Cloud Map](https://aws.amazon.com/cloud-map/) is a cloud resource discovery service. Cloud Map enables you to name your application resources with custom names, and it automatically updates the locations of these dynamically changing resources.

The objective of this chapter is to change the way as the Frontend application talks to the Crystal Backend application by configuring the integration between AWS App Mesh and AWS Cloud Map.


{{% children showhidden="false" %}}

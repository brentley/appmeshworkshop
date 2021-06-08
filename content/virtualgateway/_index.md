---
title: "Virtual Gateway"
chapter: true
weight: 38
---

# Adding Ingress with Virtual Gateway

In this section of workshop you will add an Ingress so that clients external to the mesh can interact with the (virtual) services that were already brought into the mesh, as is the case of the nodeJS EKS-based service.

So what is an Ingress you may ask? Well, it’s a set of components that provide external client access to the services that reside inside the mesh. 

As you may recall, we have a frontend running on EC2 with 2 dependencies, one of them on ECS and another one on EKS. 

So under this arrangement there is not need for an Ingress for, say,  the EKS backed service because its clients also reside inside the mesh (the app running on EC2). But what if you had an external client (sitting outside the mesh, in the same VPC) for instance a curler client that needs to access the Virtual Service represented by our EKS service? Ingress are the way to enable such communications.

AppMesh offers a construct called Virtual Gateway, that provides this ingress functionality. You can read more about it [here](https://docs.aws.amazon.com/app-mesh/latest/userguide/virtual_gateways.html). The newer versions of the AppMesh controllers provide CDR for Virtual Gateways and Virtual Routes. So you can create a VG as you would with any other AppMesh construct, by leveraging the kubectl tool.

So what happens when you create such component? The AppMesh controller will automatically deploy an AWS NLB inside your VPC and you get to define whether the NLB will be internal only or externally available (internet facing). Additionally, the controller will go ahead and create a new K8S deployment for the Envoy containers that will be the target of all the traffic that the NLB receives from its clients. Based on routing rules (path or header based at the time of this writing) that you define, the fleet of envoys that receive the traffic from the NLB will further route those requests to the corresponding Virtual Service inside the mesh.

Here is a diagram of the new architecture with the Virtual gateway in place. 

Let’s get started!

![virtualgateway](/images/virtual-gateway.png)
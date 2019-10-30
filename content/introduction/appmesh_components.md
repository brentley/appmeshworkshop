---
title: "App Mesh components"
date: 2018-10-03T10:15:55-07:00
draft: false
weight: 40
---

App Mesh is made up of the following components:

**Service mesh** – A service mesh is a logical boundary for network traffic between the services that reside within it.

**Virtual services** – A virtual service is an abstraction of a real service that is provided by a virtual node directly or indirectly by means of a virtual router.

**Virtual nodes** – A virtual node acts as a logical pointer to a particular task group, such as an ECS service or a Kubernetes deployment. When you create a virtual node, you must specify the service discovery name for your task group.

**Envoy proxy** – The Envoy proxy configures your microservice task group to use the App Mesh service mesh traffic rules that you set up for your virtual routers and virtual nodes. You add the Envoy container to your task group after you have created your virtual nodes, virtual routers, routes, and virtual services.

**Virtual routers** – The virtual router handles traffic for one or more virtual services within your mesh.

**Routes** – A route is associated with a virtual router, and it directs traffic that matches a service name prefix to one or more virtual nodes.

---
title: "Mesh the Crystal Service"
chapter: true
weight: 15
---

# Mesh the Backend Crystal Service

![mesh backend](/images/app_mesh_architecture/mesh_crystal.png)

At this point, you should have the application up and running in your lab environment with the EC2 instances serving the frontend and the ECS service managing the backend tasks.

In this chapter, our goal is to create the Mesh and edit your backend deployment in order to have the Envoy containers running and intercepting the network traffic from your ECS tasks.

To accomplish this, let's run the following steps:

{{% children showhidden="false" %}}

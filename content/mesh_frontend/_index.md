---
title: "Mesh the Frontend Service"
chapter: true
weight: 20
---

# Mesh the Frontend Ruby Service

![mesh frontend](/images/app_mesh_architecture/AppMeshWorkshopFrontend.png)

Now that we already have the AppMesh taking care of our backend network traffic, it's time to put our frontend application inside the Mesh.

In this chapter we will install and configure the Envoy proxy into our EC2 instances that are running the Frontend application by using the [AWS Systems Manager (SSM)](https://aws.amazon.com/systems-manager/).

To do so, follow these steps:

{{% children showhidden="false" %}}

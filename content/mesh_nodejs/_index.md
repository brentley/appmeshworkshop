---
title: "Mesh the NodeJS Service"
chapter: true
weight: 20
---

# Mesh the NodeJS Service running on EKS

![mesh backend nodejs](/images/app_mesh_architecture/AppMeshWorkshopMeshBackendNodeJS.png)

Now that App Mesh is taking care of our Crystal backend network traffic, the next step is to put our NodeJS backend application inside the Mesh.

In this chapter we will install AWS App Mesh Controller For Kubernetes and inject the Envoy proxy into our EKS pods.

To do so, follow these steps:

{{% children showhidden="false" %}}

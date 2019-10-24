---
title: "Deployment Strategies"
chapter: true
weight: 40
---

# Roll out updates to the example microservices

![deployment](/images/app_mesh_architecture/AppMeshWorkshopCloudMap.png)

**Canary release** is a software development strategy in which a new version of a service (as well as other software) is deployed as a canary release for testing purposes, and the base version remains deployed as a production release for normal operations.

In this chapter, let's create a new version of the **Crystal backend** application and use the AppMesh to control the Canary deployment process where we will start sending only a small part of the traffic to the new version. If all goes well, then we will continue to shift more traffic to the new version until it is serving 100% of all requests.



{{% children showhidden="false" %}}

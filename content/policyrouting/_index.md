---
title: "Header Based Routing"
chapter: true
weight: 45
---

# Routing to specific microservices

![microservices](/images/crystal.svg)

AWS AppMesh allows you to implement advanced routing capabilities between your microservices. One of the ways to achieve this is to working with **Header based routing**.

Using header-based routing, you can create patterns such as session persistence (sticky sessions) or an enhanced experience using "state". HTTP header-based routing enables using HTTP header information as a basis to determine how to route a request. This might be a standard header, like Accept or Cookie, or it might be a custom header, like my-own-header-key-value.

Header-based routing can also be used to enable use cases such as A/B testing (e.g.: custom headers using any string), canary or blue/green deployments, delivering different pages or user experiences based on categories of devices. (e.g.: using regex in header), handling traffic from different browsers differently (e.g. using user-agent) or configuring access restrictions based on IP address or CDN. (e.g. using X-Forwarded-for).  

In this step, let's configure a route that checks if the header `canary_fleet` is equals true. If so, the requests will be send to the `crystal-sd-green` version of the Crystal backend service.



{{% children showhidden="false" %}}

---
title: "Create a discovery service"
date: 2018-09-18T17:39:30-05:00
weight: 10
---

**AWS Cloud Map** is a cloud resource discovery service. Cloud Map enables you to name your application resources with custom names, and it automatically updates the locations of these dynamically changing resources.

Part of the transition to microservices and modern architectures involves having dynamic, autoscaling, and robust services that can respond quickly to failures and changing loads. A modern architectural best practice is to loosely couple these services by allowing them to specify their own dependencies. Compared to dedicated load balancing, service discovery (client side load balancing) can help improve resiliency, and convenience in dynamic and large micrsoervice environments.

The Crystal backend service operates behind an internal (dedicated) load balancer. We will now configure it to use Amazon ECS Service Discovery. Service discovery uses AWS Cloud Map API actions to manage HTTP and DNS namespaces for Amazon ECS services.

{{% notice info %}}

In ECS, Service Discovery can **only** be enabled at service creation time. In other words, you can't update an existing servie to use service discovery. Given our Crystal service was not enabled for service discovery at creation time, we will have to create a new version which does indeed leverages ECS's support for service discovery.
{{% /notice  %}}

So given the dependencies between the services, we will proceed in the following order to enable Service Discovery:

We will start of by configuring a namespace and a service in Cloud Map. 
We will then create the App Mesh resources needed to represent the new version of our ECS-based Crystal service. 
Finally we will create a new service in ECS for the Crystal backend.

* Let's create a namespace in Cloud Map to hold our service. We will name it **appmeshworkshop.pvt.local**  

```bash
# Define variables #
VPC_ID=$(jq < cfn-output.json -r '.VpcId');
# Create cloud map service #
aws servicediscovery create-private-dns-namespace \
      --name appmeshworkshop.pvt.local \
      --description 'App Mesh Workshop private DNS namespace' \
      --vpc $VPC_ID
```

**TODO: Add script to wait for creation**

* Let's create a new service in Cloud Map in the namespace we just created. We will name it **crystal** so it's FQDN will become **crystal.appmeshworkshop.pvt.local**

```bash
# Define variables #
NAMESPACE=$(aws servicediscovery list-namespaces | \
      jq -r ' .Namespaces[] | 
        select ( .Properties.HttpProperties.HttpName == "appmeshworkshop.pvt.local" ) | .Id ');
# Create cloud map service #
aws servicediscovery create-service \
      --name crystal-blue \
      --description 'Discovery service for the Crystal service (blue)' \
      --namespace-id $NAMESPACE \
      --dns-config 'RoutingPolicy=MULTIVALUE,DnsRecords=[{Type=A,TTL=300}]' \
      --health-check-custom-config FailureThreshold=1
```

Go back to the AWS Admin console and locate the Cloud Map service. Expand the left hand side section and click on the namespace with Name  **appmeshworkshop.pvt.local**

We are ready to start using the service we just defined in Cloud Map. Let's leverage ECS integration with Cloud Map to configure
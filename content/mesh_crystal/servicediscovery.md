---
title: "Add service discovery"
date: 2018-09-18T17:39:30-05:00
weight: 10
---

App Mesh supports microservice applications that use service discovery naming for their components. 

We will be using **AWS Cloud Map** to discover new versions of our Crystal backend service, instead of using an Application Load Balancer. Cloud Map enables you to name your application resources with custom names, and it automatically updates the locations of these dynamically changing resources. 

* Let's create a service in Cloud Map for our crystal microservice

```
NAMESPACE_ID=$(jq < cfn-output.json -r '.NamespaceId'); \
aws servicediscovery create-service \
      --name crystal \
      --namespace-id $NAMESPACE_ID \
      --description 'Discovery service for the crystal service' \
      --dns-config 'RoutingPolicy=MULTIVALUE,DnsRecords=[{Type=SRV,TTL=60}]' \
      --health-check-custom-config FailureThreshold=1
```

{{% notice note %}}
Check the json output, and write down the value of the **Id** that AWS Cloud Map assigned to the service you just created.
{{% /notice %}}
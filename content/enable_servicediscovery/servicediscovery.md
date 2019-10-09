---
title: "Create a discovery service"
date: 2018-09-18T17:39:30-05:00
weight: 10
---

**AWS Cloud Map** is a cloud resource discovery service. Cloud Map enables you to name your application resources with custom names, and it automatically updates the locations of these dynamically changing resources.

Part of the transition to microservices and modern architectures involves having dynamic, autoscaling, and robust services that can respond quickly to failures and changing loads. A modern architectural best practice is to loosely couple these services by allowing them to specify their own dependencies. Compared to dedicated load balancing, service discovery (client side load balancing) can help improve resiliency, and convenience in dynamic and large micrsoervice environments.

The Crystal backend service operates behind an internal (dedicated) load balancer. We will  now configure it to use Amazon ECS Service Discovery. Service discovery uses AWS Cloud Map API actions to manage HTTP and DNS namespaces for Amazon ECS services.

* Let's create a new service in Cloud Map.

```bash
NAMESPACE=$(jq < cfn-output.json -r '.NamespaceId');
aws servicediscovery create-service \
      --name crystal \
      --description 'Discovery service for the Crystal service' \
      --namespace-id $NAMESPACE \
      --dns-config 'RoutingPolicy=MULTIVALUE,DnsRecords=[{Type=SRV,TTL=60}]'
```
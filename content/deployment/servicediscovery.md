---
title: "Create a discovery service"
date: 2018-09-18T17:39:30-05:00
weight: 10
---

* Let's create a new service in Cloud Map.

```bash
# Define variables #
NAMESPACE=$(aws servicediscovery list-namespaces | \
  jq -r ' .Namespaces[] | 
    select ( .Properties.HttpProperties.HttpName == "appmeshworkshop.pvt.local" ) | .Id ');
# Create cloud map service #
aws servicediscovery create-service \
  --name crystal-green \
  --description 'Discovery service for the Crystal service (green)' \
  --namespace-id $NAMESPACE \
  --dns-config 'RoutingPolicy=MULTIVALUE,DnsRecords=[{Type=A,TTL=300}]' \
  --health-check-custom-config FailureThreshold=1
```
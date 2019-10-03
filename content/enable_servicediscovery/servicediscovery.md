---
title: "Create a new Crystal service"
date: 2018-09-18T17:39:30-05:00
weight: 10
---

**AWS Cloud Map** is a cloud resource discovery service. Cloud Map enables you to name your application resources with custom names, and it automatically updates the locations of these dynamically changing resources.

Part of the transition to microservices and modern architectures involves having dynamic, autoscaling, and robust services that can respond quickly to failures and changing loads. A modern architectural best practice is to loosely couple these services by allowing them to specify their own dependencies. Compared to dedicated load balancing, service discovery (client side load balancing) can help improve resiliency, and convenience in dynamic and large micrsoervice environments.

The Crystal backend service operates behind an internal (dedicated) load balancer. We will  now configure it to use Amazon ECS Service Discovery. Service discovery uses AWS Cloud Map API actions to manage HTTP and DNS namespaces for Amazon ECS services.

* Let's create a new service in Cloud Map

```bash
NAMESPACE=$(jq < cfn-output.json -r '.NamespaceId');
aws servicediscovery create-service \
      --name crystal \
      --namespace-id $NAMESPACE \
      --dns-config 'RoutingPolicy=MULTIVALUE,DnsRecords=[{Type=SRV,TTL=60}]'
```

* Time to create a new service in ECS

```bash
CLUSTER_NAME=$(jq < cfn-output.json -r '.EcsClusterName')
TASK_DEF_ARN=$(jq < cfn-output.json -r '.CrystalTaskDefinition')
PRIVSUB1=$(jq < cfn-output.json -r '.PrivateSubnetOne')
PRIVSUB2=$(jq < cfn-output.json -r '.PrivateSubnetTwo')
PRIVSUB3=$(jq < cfn-output.json -r '.PrivateSubnetThree')
SECGROUP=$(jq < cfn-output.json -r '.ContainerSecurityGroup')
aws ecs create-service \
  --cluster $CLUSTER_NAME \
  --service-name CrystalService-SD \
  --task-definition "$(echo $TASK_DEF_ARN | awk -F: '{$7=$7+1}1' OFS=:)" \
  --desired-count 3 \
  --platform-version LATEST \
  --service-registries \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[$PRIVSUB1,$PRIVSUB2,$PRIVSUB3],securityGroups=[$SECGROUP],assignPublicIp=DISABLED}"
```
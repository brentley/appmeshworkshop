---
title: "Create a discovery service"
date: 2018-09-18T17:39:30-05:00
weight: 10
---
The Crystal backend service operates behind an internal (dedicated) load balancer. We will now configure it to use Amazon ECS Service Discovery. Service discovery uses AWS Cloud Map API actions to manage HTTP and DNS namespaces for Amazon ECS services.

Given the dependencies between Cloud Map, ECS and App Mesh, you will proceed in the following order to enable Service Discovery:

1. We will start of by configuring a namespace and a service in Cloud Map.
2. We will then create the App Mesh resources needed to represent the new version of our ECS-based Crystal service.
3. Finally we will create a new service in ECS for the Crystal backend.

* Let's create a namespace in Cloud Map to hold the service. Name it **appmeshworkshop.pvt.local**  

```bash
# Define variables #
VPC_ID=$(jq < cfn-output.json -r '.VpcId');
# Create cloud map namespace #
OPERATION_ID=$(aws servicediscovery create-private-dns-namespace \
    --name appmeshworkshop.pvt.local \
    --description 'App Mesh Workshop private DNS namespace' \
    --vpc $VPC_ID | \
  jq -r ' .OperationId ')
_operation_status() {
  aws servicediscovery get-operation \
    --operation-id $OPERATION_ID | \
  jq -r '.Operation.Status '
}
until [ $(_operation_status) != "PENDING" ]; do
  echo "Namespace is creating ..."
  sleep 10s
  if [ $(_operation_status) == "SUCCESS" ]; then
    echo "Namespace created"
    break
  fi
done
```

* Create a service inside the namespace created in the step above. Name it **crystal**. The service's FQDN becomes **crystal.appmeshworkshop.pvt.local**

```bash
# Define variables #
NAMESPACE=$(aws servicediscovery list-namespaces | \
  jq -r ' .Namespaces[] |
    select ( .Properties.HttpProperties.HttpName == "appmeshworkshop.pvt.local" ) | .Id ');
# Create cloud map service #
aws servicediscovery create-service \
  --name crystal \
  --description 'Discovery service for the Crystal service' \
  --namespace-id $NAMESPACE \
  --dns-config 'RoutingPolicy=MULTIVALUE,DnsRecords=[{Type=A,TTL=300}]' \
  --health-check-custom-config FailureThreshold=1
```

Go back to the AWS Admin console and locate the Cloud Map service. Expand the left hand side section and click on the namespace  **appmeshworkshop.pvt.local**. Take a look at the service definition.

We are ready to start using the service we just defined in Cloud Map. Let's leverage ECS integration with Cloud Map to configure it.
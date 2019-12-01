---
title: "Cleanup Kubernetes components"
date: 2018-08-07T12:37:34-07:00
weight: 15
---

* Delete the components from your EKS cluster:

```bash
# Delete the virtual service
kubectl delete virtualservice nodejs.appmeshworkshop.hosted.local -n appmesh-workshop-ns
# Delete the virtual node
kubectl delete virtualnode nodejs-app -n appmesh-workshop-ns
# Delete the deployment
kubectl delete deployment nodejs-app -n appmesh-workshop-ns
```

Now, let's deregister the instances from the Cloud Map service discovery:

```bash
NAMESPACE=$(aws servicediscovery list-namespaces | \
  jq -r ' .Namespaces[] | 
    select ( .Properties.HttpProperties.HttpName == "appmeshworkshop.pvt.local" ) | .Id ');
SERVICE_ID=$(aws servicediscovery list-services --filters Name="NAMESPACE_ID",Values=$NAMESPACE,Condition="EQ" | jq -r ' .Services[] | [ .Id ] | @tsv ' )
aws servicediscovery list-instances --service-id $SERVICE_ID | jq -r ' .Instances[] | [ .Id ] | @tsv ' |\
  while IFS=$'\t' read -r instanceId; do 
    aws servicediscovery deregister-instance --service-id $SERVICE_ID --instance-id $instanceId
  done
```

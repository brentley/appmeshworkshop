---
title: "Cleanup the namespace"
date: 2018-08-07T13:37:53-07:00
weight: 20
---

* Delete the services in the namespace.

```bash
# Define variables #
NAMESPACE=$(aws servicediscovery list-namespaces | \
  jq -r ' .Namespaces[] | 
    select ( .Properties.HttpProperties.HttpName == "appmeshworkshop.pvt.local" ) | .Id ');
# Delete cloud map services #
aws servicediscovery list-services \
  --filters Name="NAMESPACE_ID",Values=$NAMESPACE,Condition="EQ" | \
jq -r ' .Services[] | [ .Id ] | @tsv ' | \
  while IFS=$'\t' read -r serviceId; do 
    aws servicediscovery delete-service \
      --id $serviceId
  done
```

* Delete the namespace.

```bash
# Define variables #
NAMESPACE=$(aws servicediscovery list-namespaces | \
  jq -r ' .Namespaces[] | 
    select ( .Properties.HttpProperties.HttpName == "appmeshworkshop.pvt.local" ) | .Id ');
# Delete cloud map namespace #
OPERATION_ID=$(aws servicediscovery delete-namespace \
    --id $NAMESPACE | \
  jq -r ' .OperationId ')
_operation_status() {
  aws servicediscovery get-operation \
    --operation-id $OPERATION_ID | \
  jq -r '.Operation.Status '
}
until [ $(_operation_status) != "PENDING" ]; do
  echo "Namespace is deleting ..."
  sleep 10s
  if [ $(_operation_status) == "SUCCESS" ]; then
    echo "Namespace deleted"
    break
  fi
done
```
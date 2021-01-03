---
title: "Cleanup the mesh"
date: 2018-08-07T12:37:34-07:00
weight: 40
---

* Delete the virtual services.

```bash
# Delete app mesh virtual services #
aws appmesh list-virtual-services \
  --mesh-name appmesh-workshop | \
jq -r ' .virtualServices[] | [.virtualServiceName] | @tsv ' | \
  while IFS=$'\t' read -r virtualServiceName; do 
    aws appmesh delete-virtual-service \
      --mesh-name appmesh-workshop \
      --virtual-service-name $virtualServiceName 
  done
```

* Delete the virtual routers.

```bash
# Delete app mesh virtual routers #
aws appmesh list-virtual-routers \
  --mesh-name appmesh-workshop | \
jq -r ' .virtualRouters[] | [.virtualRouterName] | @tsv ' | \
  while IFS=$'\t' read -r virtualRouterName; do 
    aws appmesh list-routes \
      --mesh-name appmesh-workshop \
      --virtual-router-name $virtualRouterName | \
    jq -r ' .routes[] | [ .routeName] | @tsv ' | \
      while IFS=$'\t' read -r routeName; do 
        aws appmesh delete-route \
          --mesh appmesh-workshop \
          --virtual-router-name $virtualRouterName \
          --route-name $routeName
      done
    aws appmesh delete-virtual-router \
      --mesh-name appmesh-workshop \
      --virtual-router-name $virtualRouterName 
  done
```

* Delete the virtual nodes.

```bash
# Delete app mesh virtual nodes #
aws appmesh list-virtual-nodes \
  --mesh-name appmesh-workshop | \
jq -r ' .virtualNodes[] | [.virtualNodeName] | @tsv ' | \
  while IFS=$'\t' read -r virtualNodeName; do 
    aws appmesh delete-virtual-node \
      --mesh-name appmesh-workshop \
      --virtual-node-name $virtualNodeName 
  done
```

* Delete the mesh.

```bash
# Delete app mesh mesh #
aws appmesh delete-mesh \
  --mesh-name appmesh-workshop
```

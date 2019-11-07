---
title: "Create the virtual service"
date: 2018-08-07T08:30:11-07:00
weight: 15
---

Now that the Mesh is already created and everything is properly installed in the EKS cluster, it's time to create the resources in App Mesh for the NodeJS application.

Start by creating the virtual node with the following commands:

```bash
# Define environment variable
NODEJS_LB_URL=$(kubectl get service eks-nodejs-app -n appmesh-workshop-ns -o json | jq -r '.status.loadBalancer.ingress[].hostname')

# Create Virtual Node yaml file
cat <<"EOF" > ~/environment/eks-scripts/virtual-node.yml
apiVersion: appmesh.k8s.aws/v1beta1
kind: VirtualNode
metadata:
  name: nodejs-application
  namespace: appmesh-workshop-ns
spec:
  meshName: appmesh-workshop
  listeners:
    - portMapping:
        port: 3000
        protocol: http
  serviceDiscovery:
    dns:
      hostName: $NODEJS_LB_URL
EOF

# Apply the configuration
kubectl apply -f  ~/environment/eks-scripts/virtual-node.yml
```

Now, create your virtual service:

```bash
# Create Virtual Service yaml file
cat <<"EOF" > ~/environment/eks-scripts/virtual-service.yml
apiVersion: appmesh.k8s.aws/v1beta1
kind: VirtualService
metadata:
  name: nodejs.appmeshworkshop.hosted.local
  namespace: appmesh-workshop-ns
spec:
  meshName: appmesh-workshop
  virtualRouter:
    listeners:
      - portMapping:
          port: 3000
          protocol: http
  routes:
    - name: route-to-nodejs-app
      http:
        match:
          prefix: /
        action:
          weightedTargets:
            - virtualNodeName: nodejs-application
              weight: 1
EOF

# Apply the configuration
kubectl apply -f  ~/environment/eks-scripts/virtual-service.yml
```

Now let's name the `appmesh-workshop-ns` namespace so we will have the envoy sidecar being injected in the application pods:

```bash
kubectl label namespace appmesh-workshop-ns appmesh.k8s.aws/sidecarInjectorWebhook=enabled
```
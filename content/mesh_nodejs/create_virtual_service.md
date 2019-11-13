---
title: "Create the virtual service"
date: 2018-08-07T08:30:11-07:00
weight: 15
---

Now that the Mesh is already created and everything is properly installed in the EKS cluster, it's time to create the resources in App Mesh for the NodeJS application.

Even though the Mesh is already created in the App Mesh service, we still need to create a Mesh inside the EKS cluster, so the controller will be able to interact with the App Mesh service and manage components in it. Let's now create a Mesh inside the cluster with the following commands:

```bash
# Create the Mesh yaml file
cat <<"EOF" > ~/environment/eks-scripts/mesh.yml
apiVersion: appmesh.k8s.aws/v1beta1
kind: Mesh
metadata:
  name: appmesh-workshop
EOF

# Apply the configuration
kubectl apply -f  ~/environment/eks-scripts/mesh.yml
```


Start by creating the virtual node with the following commands:

```bash
# Define environment variable
NODEJS_LB_URL=$(kubectl get service nodejs-app-service -n appmesh-workshop-ns -o json | jq -r '.status.loadBalancer.ingress[].hostname')

# Create Virtual Node yaml file
cat <<EOF > ~/environment/eks-scripts/virtual-node.yml
apiVersion: appmesh.k8s.aws/v1beta1
kind: VirtualNode
metadata:
  name: nodejs-app
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
cat <<EOF > ~/environment/eks-scripts/virtual-service.yml
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
            - virtualNodeName: nodejs-app
              weight: 1
EOF

# Apply the configuration
kubectl apply -f  ~/environment/eks-scripts/virtual-service.yml
```


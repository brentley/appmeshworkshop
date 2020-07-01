---
title: "Create the virtual service"
date: 2018-08-07T08:30:11-07:00
weight: 15
---

With the Mesh already created and the App Mesh controller properly installed in the EKS cluster, it's nearly time to create the resources in App Mesh for the NodeJS application. Before we get there, we need to perform a couple of steps.

First, we need to affiliate the application running in EKS with the Mesh we created previously. This is done by adding a `mesh` label to the application namespace.

```bash
kubectl label namespace appmesh-workshop-ns mesh=appmesh-workshop
```

At this point, we will also configure the App Mesh Controller to automatically inject the Envoy sidecar proxy containers into our application pods in this namespace. This is done by adding a second label to the namespace.

```bash
kubectl label namespace appmesh-workshop-ns appmesh.k8s.aws/sidecarInjectorWebhook=enabled
```

Finally, we'll create a Mesh object within the EKS cluster. Even though the mesh is already created in the App Mesh service, we will need to create a mesh object so the controller will be able to interact with the App Mesh service and manage components within it.

Let's now create a Mesh inside the cluster with the following commands. Note, the Mesh specification indicates the namespace `mesh` label value in the match for namespaceSelector. This is what binds this namespace to our mesh.

```bash
# Create the Mesh yaml file
cat <<"EOF" > ~/environment/eks-scripts/mesh.yml
apiVersion: appmesh.k8s.aws/v1beta2
kind: Mesh
metadata:
  name: appmesh-workshop
spec:
  namespaceSelector:
    matchLabels:
      mesh: appmesh-workshop
EOF

# Apply the configuration
kubectl apply -f  ~/environment/eks-scripts/mesh.yml
```

To now configure our service to run within App Mesh, we'll create a VirtualNode for the nodejs-app Deployment and Service. 

```bash
# Define environment variable
NODEJS_LB_URL=$(kubectl get service nodejs-app-service -n appmesh-workshop-ns -o json | jq -r '.status.loadBalancer.ingress[].hostname')

# Create Virtual Node yaml file
cat <<EOF > ~/environment/eks-scripts/virtual-node.yml
apiVersion: appmesh.k8s.aws/v1beta2
kind: VirtualNode
metadata:
  name: nodejs-app
  namespace: appmesh-workshop-ns
spec:
  podSelector:
    matchLabels:
      app: nodejs-app
  listeners:
    - portMapping:
        port: 3000
        protocol: http
  serviceDiscovery:
    dns:
      hostname: $NODEJS_LB_URL
EOF

# Apply the configuration
kubectl apply -f  ~/environment/eks-scripts/virtual-node.yml
```

Now, create a VirtualService and a VirtualRouter.

```bash
# Create Virtual Service yaml file
cat <<EOF > ~/environment/eks-scripts/virtual-service-and-router.yml
---
apiVersion: appmesh.k8s.aws/v1beta2
kind: VirtualService
metadata:
  name: nodejs
  namespace: appmesh-workshop-ns
spec:
  awsName: nodejs.appmeshworkshop.hosted.local
  provider:
    virtualRouter:
      virtualRouterRef:
        name: nodejs-router
---
apiVersion: appmesh.k8s.aws/v1beta2
kind: VirtualRouter
metadata:
  name: nodejs-router
  namespace: appmesh-workshop-ns
spec:
  listeners:
    - portMapping:
        port: 3000
        protocol: http
  routes:
    - name: route-to-nodejs-app
      httpRoute:
        match:
          prefix: /
        action:
          weightedTargets:
            - virtualNodeRef:
                name: nodejs-app
              weight: 1
---
EOF

# Apply the configuration
kubectl apply -f  ~/environment/eks-scripts/virtual-service-and-router.yml
```


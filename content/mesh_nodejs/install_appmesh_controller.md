---
title: "Install AWS App Mesh Controller For K8s"
date: 2018-08-07T08:30:11-07:00
weight: 10
---

AWS App Mesh Controller For K8s is a controller to help manage App Mesh resources for a Kubernetes cluster. The controller watches custom resources for changes and reflects those changes into the App Mesh API. It is accompanied by the deployment of three custom resource definitions (CRDs): meshes, virtualnodes, and virtualservices. These map to App Mesh API objects which the controller manages for you.

To start, add the eks-charts helm repository:

```bash
helm repo add eks https://aws.github.io/eks-charts
```

Create the `appmesh-system` namespace:

```bash
kubectl create ns appmesh-system
```

Apply the CRDs:

```bash
kubectl apply -f https://raw.githubusercontent.com/aws/eks-charts/master/stable/appmesh-controller/crds/crds.yaml
```

Install the appmesh-controller:

```bash
helm upgrade -i appmesh-controller eks/appmesh-controller --namespace appmesh-system
```

Install the mutating admission webhook (aws-app-mesh-inject):

```bash
helm upgrade -i appmesh-inject eks/appmesh-inject \
--namespace appmesh-system \
--set mesh.create=false \
--set mesh.name=appmesh-workshop
```
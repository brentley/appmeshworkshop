---
title: "Install AWS App Mesh Controller For K8s"
date: 2018-08-07T08:30:11-07:00
weight: 10
---

[AWS App Mesh Controller For K8s](https://github.com/aws/aws-app-mesh-controller-for-k8s) manages App Mesh resources within your Kubernetes clusters. The controller is accompanied by Custom Resource Definitions (CRDs) that allow you to define App Mesh components such as Meshes and VirtualNodes using the Kubernetes API just as you define native Kubernetes objects such as Deployments and Services. These custom resources map to App Mesh API objects which the controller manages for you. The controller watches these custom resources for changes and reflects those changes into the App Mesh API.

The controller is installed via Helm Chart. Follow [this link](https://helm.sh/docs/intro/install/) to install the latest version of Helm. Once done, you should see a version of 3.0 or higher.

```bash
helm version --short
```

Now, add the EKS Charts repo to Helm.

```bash
helm repo add eks https://aws.github.io/eks-charts

helm repo list | grep eks-charts
```

Create the `appmesh-system` namespace and attach IAM Policies for AWS App Mesh and AWS Cloud Map access.

{{% notice tip %}}
If you are new to the `IAM Roles for Service Accounts (IRSA)` concept, [Click here](/beginner/110_irsa/) for me information.
{{% /notice %}}

```bash
kubectl create ns appmesh-system

# Create your OIDC identity provider for the cluster
eksctl utils associate-iam-oidc-provider \
  --cluster appmesh-workshop \
  --approve

# Create an IAM role for the appmesh-controller service account
eksctl create iamserviceaccount \
  --cluster appmesh-workshop \
  --namespace appmesh-system \
  --name appmesh-controller \
  --attach-policy-arn  arn:aws:iam::aws:policy/AWSCloudMapFullAccess,arn:aws:iam::aws:policy/AWSAppMeshFullAccess \
  --override-existing-serviceaccounts \
  --approve
```

Now install App Mesh Controller into the `appmesh-system` namespace using the project's Helm chart, specifying the service account you just created.

```bash
helm upgrade -i appmesh-controller eks/appmesh-controller \
  --namespace appmesh-system \
  --set region=${AWS_REGION} \
  --set serviceAccount.create=false \
  --set serviceAccount.name=appmesh-controller
```

To verify the installation was successful, list the objects in the `appmesh-system` namespace and ensure the `appmesh-controller` pod instance is in a `Running` state before moving on.

```bash
kubectl -n appmesh-system get all
```

You can also see that the App Mesh Custom Resource Definitions were installed.

```bash
kubectl get crds | grep appmesh
```


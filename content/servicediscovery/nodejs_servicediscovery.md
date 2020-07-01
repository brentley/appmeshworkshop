---
title: "NodeJS app with Cloud Map"
date: 2018-09-18T17:39:30-05:00
weight: 80
---

In order to leverage the integration between the NodeJS app running in the EKS cluster and the AWS Cloud Map service, we will be using the ExternalDNS Kubernetes connector.

Since we are using the `AWS App Mesh Controller For K8s` and it is already integrated with Cloud Map, let's change the `Virtual Node` configuration, so it can register the Pods IP addresses to the Cloud Map:

```bash
# Update the Virtual Node yaml file
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
    awsCloudMap:
      namespaceName: appmeshworkshop.pvt.local
      serviceName: nodejs
EOF

# Apply the configuration
kubectl apply -f  ~/environment/eks-scripts/virtual-node.yml
```

After applying these changes, your pods IP Addresses will be added to the `appmeshworkshop.pvt.local` domain name, previously created in the Cloud Map. You can check that everything is working properly by using the AWS cli command bellow:

```bash
aws servicediscovery discover-instances \
--namespace appmeshworkshop.pvt.local \
--service-name nodejs
```

You can compare the IP addresses presented in the `Instance ID` fields with your Pods IP Addresses presented in the `IP` field of the following command: 

```bash
kubectl get pods -n appmesh-workshop-ns -o wide
```

Or you can access the [Cloud Map console](https://console.aws.amazon.com/cloudmap/home), click in the `appmeshworkshop.pvt.local` domain name and then in the `nodejs` service name to see that your pods are registered under `Service instances`:


![cloud map](/images/cloud_map/check_pods.png)

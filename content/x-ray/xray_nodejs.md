---
title: "Enable X-Ray for NodeJS App"
date: 2018-09-18T16:01:14-05:00
weight: 11
---

Let's now enable the X-Ray integration to the NodeJS app currently running on the EKS cluster. 

The first step is to change the configuration of the App Mesh controller in order to automatically add the X-Ray container to new pods and configure the Envoy proxy containers to send data to them:

```bash
helm upgrade -i appmesh-controller eks/appmesh-controller \
  --namespace appmesh-system \
  --set region=${AWS_REGION} \
  --set serviceAccount.create=false \
  --set serviceAccount.name=appmesh-controller \
  --set tracing.enabled=true \
  --set tracing.provider=x-ray \
  --set sidecar.image.repository=840364872350.dkr.ecr.us-west-2.amazonaws.com/aws-appmesh-envoy \
  --set sidecar.image.tag=v1.12.5.0-prod
```

Now, let's restart our pods again in order to force the injection of the X-Ray container:

```bash
kubectl -n appmesh-workshop-ns rollout restart deployment nodejs-app
```

Monitor the restarts and move on once the new pods are all in a `Running` state.

```bash
kubectl -n appmesh-workshop-ns get pods -w
```

Let's now check if the X-Ray container was injected in your pod:

```bash
POD=$(kubectl -n appmesh-workshop-ns get pods -o jsonpath='{.items[0].metadata.name}')
kubectl -n appmesh-workshop-ns get pods ${POD} -o jsonpath='{.spec.containers[*].name}'; echo
```

You should see a `xray-daemon` container in the pod along with the app and Envoy containers.


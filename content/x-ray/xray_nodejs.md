---
title: "Enable X-Ray for NodeJS App"
date: 2018-09-18T16:01:14-05:00
weight: 11
---

Let's now enable the X-Ray integration to the NodeJS app currently running on the EKS cluster. 

The first step is to change the configurations in the App Mesh injector in order to add the X-Ray container to the pods and configure the Envoy proxy to send data to it:

```bash
helm upgrade -i appmesh-inject eks/appmesh-inject \
--namespace appmesh-system \
--set mesh.create=false \
--set mesh.name=appmesh-workshop \
--set tracing.enabled=true \
--set tracing.provider=x-ray
```

Now, let's restart our pods again in order to force the injection of the X-Ray container:

```bash
kubectl delete pods -n appmesh-workshop-ns --all 
```

Let's now check if the X-Ray container was injected in your pod:

```bash
# Get a single pod name
NODEJS_POD=$(kubectl get pod --no-headers=true -o \
name -n appmesh-workshop-ns \
| awk -F "/" '{print $2}' | \
head -n 1)

# Describe pod configuration
kubectl describe pod $NODEJS_POD -n appmesh-workshop-ns
```

You should see information about the X-Ray daemon container:

```text
  xray-daemon:
    Container ID:   docker://47197806db6f658047b12f3905be802ce1e0f58b02b9f94d2ef733e98b6fe4a0
    Image:          amazon/aws-xray-daemon
    Image ID:       docker-pullable://amazon/aws-xray-daemon@sha256:5d30d974bd7bd5a864a2d6d4d4696902153d364cc0943efca8ea44e6bf16c1c2
    Port:           2000/UDP
    Host Port:      0/UDP
    State:          Running
      Started:      Fri, 08 Nov 2019 23:32:09 +0000
    Ready:          True
    Restart Count:  0
    Requests:
      cpu:        10m
      memory:     32Mi
    Environment:  <none>
    Mounts:       <none>
Conditions:
  Type              Status
  Initialized       True 
  Ready             True 
  ContainersReady   True 
  PodScheduled      True 
Volumes:
  default-token-82w67:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-82w67
    Optional:    false
QoS Class:       Burstable
Node-Selectors:  <none>
Tolerations:     node.kubernetes.io/not-ready:NoExecute for 300s
                 node.kubernetes.io/unreachable:NoExecute for 300s
```


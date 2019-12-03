---
title: "Inject the Envoy proxy"
date: 2018-08-07T08:30:11-07:00
weight: 15
---
The first step to work with the `appmesh-injector` component is to tag the `appmesh-workshop-ns` namespace so we will have the envoy sidecar being injected in the application pods:

```bash
kubectl label namespace appmesh-workshop-ns appmesh.k8s.aws/sidecarInjectorWebhook=enabled
```

Now, let's restart the pods in order to add the Envoy proxy to them using the App Mesh injector component, which is already installed.

To do so, run the following command:

```bash
# Restart pods
kubectl delete pods -n appmesh-workshop-ns --all
```

{{% notice info %}}
This command will restart all the pods present in the `appmesh-workshop-ns` namespace. Note that this command should not be used in a production environment and we are only using it now because we just have one deployment running in this namespace.
{{% /notice %}}

After restarting all the pods, you can check if the envoy proxy was added to it with the following commands:

```bash
# Get a single pod name
NODEJS_POD=$(kubectl get pod --no-headers=true -o \
  name -n appmesh-workshop-ns | \
  awk -F "/" '{print $2}' | \
  head -n 1)

# Describe pod configuration
kubectl describe pod $NODEJS_POD -n appmesh-workshop-ns
```

Take a look in the output of this command. You should see the Envoy container image as part of your pod:

```text
  envoy:
    Container ID:   docker://5a1e9b03d8c89e6db4797010b1970ee8f1ee930c565990960f7753a06a912970
    Image:          840364872350.dkr.ecr.us-west-2.amazonaws.com/aws-appmesh-envoy:v1.11.2.0-prod
    Image ID:       docker-pullable://840364872350.dkr.ecr.us-west-2.amazonaws.com/aws-appmesh-envoy@sha256:69632b54c1390724fe491192c2e937d8cf8bb4060c038117c555cd6b8add2e1a
    Port:           9901/TCP
    Host Port:      0/TCP
    State:          Running
      Started:      Wed, 06 Nov 2019 02:20:56 +0000
    Ready:          True
    Restart Count:  0
    Requests:
      cpu:     10m
      memory:  32Mi
    Environment:
      APPMESH_VIRTUAL_NODE_NAME:  mesh/appmesh-workshop/virtualNode/nodejs-application-appmesh-workshop-ns
      APPMESH_PREVIEW:            0
      ENVOY_LOG_LEVEL:            info
      AWS_REGION:                 us-west-2
    Mounts:                       <none>
```

It means that the injector is working fine and the Envoy is already running inside your NodeJS pod.

Now, it's time to test and see if the responses are being sent by the Envoy proxy. Let's access one of the EC2 instances serving the frontend again:

```bash
AUTO_SCALING_GROUP=$(jq < cfn-output.json -r '.RubyAutoScalingGroupName');
TARGET_EC2=$(aws ec2 describe-instances \
    --filters Name=tag:aws:autoscaling:groupName,Values=$AUTO_SCALING_GROUP | \
  jq -r ' .Reservations | first | .Instances | first | .InstanceId')
aws ssm start-session --target $TARGET_EC2
```

And curl the NodeJS microservice:

```bash
curl -v http://nodejs.appmeshworkshop.hosted.local:3000/
```

You should see a response comming from the NodeJS microservice running in the EKS cluster. In this response, look for `envoy` in the `server` parameter:

```text
*   Trying 10.0.100.206...
* TCP_NODELAY set
* Connected to nodejs.appmeshworkshop.hosted.local (10.0.100.206) port 3000 (#0)
> GET / HTTP/1.1
> Host: nodejs.appmeshworkshop.hosted.local:3000
> User-Agent: curl/7.61.1
> Accept: */*
> 
< HTTP/1.1 200 OK
< x-powered-by: Express
< content-type: text/plain; charset=utf-8
< content-length: 65
< etag: W/"41-Cr+iGiCIiMfq96LyNt+F6PjOvZs"
< date: Fri, 08 Nov 2019 19:20:23 GMT
< x-envoy-upstream-service-time: 3
< server: envoy
< 
Node.js backend: Hello! from 10.0.102.185 in AZ-c commit 4252202
* Connection #0 to host nodejs.appmeshworkshop.hosted.local left intact
```

This means that the responses from the NodeJS application are passing in the Envoy proxy.

Now you can terminate the session:

```bash
exit 
```

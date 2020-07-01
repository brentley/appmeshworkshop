---
title: "Inject the Envoy proxy"
date: 2018-08-07T08:30:11-07:00
weight: 15
---
Now that our objects are in place, we have one remaining step to add our `nodejs` application service to the mesh. We've labeled the namespace so that all new pods will have their Envoy sidecar containers automatically injected, but our pods are already up and running. To inject the sidecar proxies, we'll need to restart those pods.

To do so, run the following command:

```bash
kubectl -n appmesh-workshop-ns rollout restart deployment nodejs-app
```

This command will restart all the pods present in the `appmesh-workshop-ns` namespace. You can monitor the restart by watching the pods roll out.

```bash
kubectl -n appmesh-workshop-ns get pods -w
```

Once you see all pods in a `Running` state, verify that the envoy proxy containers were added by checking the number of container instances in each pod. You should now see `2/2` for each pod.

To dig a little deeper, get the running containers from one of the new pods. Note that your pod names will differ, and you can choose any of your running pods.

```bash
POD=$(kubectl -n appmesh-workshop-ns get pods -o jsonpath='{.items[0].metadata.name}')
kubectl -n appmesh-workshop-ns get pods ${POD} -o jsonpath='{.spec.containers[*].name}'; echo
```

You will see `envoy` listed as one of the two containers, which means that sidecar injection in the controller is working and that Envoy is running inside your NodeJS pods.

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

You should see a response coming from the NodeJS microservice running in the EKS cluster. In this response, look for `envoy` in the `server` parameter:

```text
*   Trying 10.0.102.194...
* TCP_NODELAY set
* Connected to nodejs.appmeshworkshop.hosted.local (10.0.102.194) port 3000 (#0)
> GET / HTTP/1.1
> Host: nodejs.appmeshworkshop.hosted.local:3000
> User-Agent: curl/7.61.1
> Accept: */*
> 
< HTTP/1.1 200 OK
< x-powered-by: Express
< content-type: text/plain; charset=utf-8
< content-length: 64
< etag: W/"40-fCLGgbgasYs+i/eJyifWO2rSi40"
< date: Wed, 01 Jul 2020 00:31:28 GMT
< x-envoy-upstream-service-time: 4
< server: envoy
< 
Node.js backend: Hello! from 10.0.102.41 in AZ-c commit 219f52d
* Connection #0 to host nodejs.appmeshworkshop.hosted.local left intact
```

This means that the responses from the NodeJS application are passing in the Envoy proxy.

* Terminate the session.

```bash
exit 
```

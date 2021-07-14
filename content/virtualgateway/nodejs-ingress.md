---
title: "Virtual Gateway Ingress to NodeJS service"
date: 2018-09-18T17:39:30-05:00
weight: 10
---

So first thing we need to do is to edit our K8S Namespace to include a label that is used by the Virtual Gateway we are about to create.

```bash
kubectl label namespace appmesh-workshop-ns gateway=ingress-gw
```
Let’s now create our Virtual Gateway manifest file

```bash
# Create Virtual Service yaml file
cat <<EOF > ~/environment/eks-scripts/app-mesh-virtual-gateway.yml
---
apiVersion: v1
kind: Service
metadata:
  name: ingress-gw
  namespace: appmesh-workshop-ns
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
    service.beta.kubernetes.io/aws-load-balancer-internal: (http://service.beta.kubernetes.io/aws-load-balancer-internal:) "true"
spec:
  type: LoadBalancer
  ports:
    - port: 80
      targetPort: 8088
      name: http
  selector:
    app: ingress-gw
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ingress-gw
  namespace: appmesh-workshop-ns
spec:
  replicas: 2
  selector:
    matchLabels:
      app: ingress-gw
  template:
    metadata:
      labels:
        app: ingress-gw
    spec:
      containers:
        - name: envoy
          image: 840364872350.dkr.ecr.$AWS_REGION.amazonaws.com/aws-appmesh-envoy:v1.16.1.1-prod
          ports:
            - containerPort: 8088
---
apiVersion: appmesh.k8s.aws/v1beta2
kind: VirtualGateway
metadata:
  name: ingress-gw
  namespace: appmesh-workshop-ns
spec:
  namespaceSelector:
    matchLabels:
      gateway: ingress-gw
  podSelector:
    matchLabels:
      app: ingress-gw
  listeners:
    - portMapping:
        port: 8088
        protocol: http
---
apiVersion: appmesh.k8s.aws/v1beta2
kind: GatewayRoute
metadata:
  name: gateway-route-eks
  namespace: appmesh-workshop-ns
spec:
  httpRoute:
    match:
      prefix: "/eks/"
    action:
      target:
        virtualService:
          virtualServiceRef:
            name: nodejs
---
EOF
```
If you take a close look at the previous manifest file, you will notice we added one GatewayRoute with a single route for path /eks. In practical terms this means any requests that arrive at the NLB with path /eks will get rerouted to the Virtual Service nodeJS.

Let’s see if it works! We have an EC2 instance sitting inside our VPC but is not part of our EKS cluster.

Connect to the EC2 instance in order to test reachability from external requests to the mesh using the virtual gateway ingress

```bash
EXTERNAL_EC2=$(aws ec2 describe-instances --filters Name=tag:Usage,Values=ExternalEC2Instance | jq -r '.Reservations[].Instances[].InstanceId')

aws ssm start-session --target $EXTERNAL_EC2
Starting session with SessionId: xxxxx-03f74a1b6fabf65d4
```

We add the Kube context for EKS connectivity 
```bash
aws eks --region us-west-2 update-kubeconfig --name appmesh-workshop
Added new context arn:aws:eks:xxxxx:xxxxxxx:cluster/appmesh-workshop to /home/ssm-user/.kube/config
```

And finally, let’s get the FQDN of the NLB that was created as part of the K8s Service of type LoadBalancer.

```bash
NLB_ENDPOINT=$(kubectl get service -n appmesh-workshop-ns -o json | jq -r ".items[].status.loadBalancer.ingress[].hostname")
```

Let’s try to connect 

```bash
curl -v $NLB_ENDPOINT/eks/
```

You should receive a response similar to the one below:

```bash
*   Trying 10.0.101.45...
* TCP_NODELAY set
* Connected to a9919a32f014444d2bc66103e18ee816-8e1fb5e7a8e603d3.elb.us-west-2.amazonaws.com (10.0.101.45) port 80 (#0)
> GET /eks/ HTTP/1.1
> Host: a9919a32f014444d2bc66103e18ee816-8e1fb5e7a8e603d3.elb.us-west-2.amazonaws.com
> User-Agent: curl/7.61.1
> Accept: */*
> 
< HTTP/1.1 200 OK
< x-powered-by: Express
< content-type: text/plain; charset=utf-8
< content-length: 64
< etag: W/"40-usK7ZSptGiNHjPKT0n79jNRx8l8"
< date: Mon, 10 May 2021 16:57:06 GMT
< x-envoy-upstream-service-time: 3
< server: envoy
< 
Node.js backend: Hello! from 10.0.101.93 in AZ-b commit c61db33
* Connection #0 to host a9919a32f014444d2bc66103e18ee816-8e1fb5e7a8e603d3.elb.us-west-2.amazonaws.com left intact
```

---
title: "Deploy the NodeJS Service"
date: 2018-09-18T16:01:14-05:00
weight: 5
draft: true
---
Since we already have the frontend application running, it's time to deploy the NodeJS backend service to your EKS cluster.


Let's start creating a deployment for the NodeJs application in the EKS cluster:

```bash
# Create deployment yaml file
cat <<-EOF > ~/environment/eks-scripts/nodejs-deployment.yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: eks-nodejs-app
  labels:
    app: eks-nodejs-app
  namespace: default
spec:
  replicas: 3
  selector:
    matchLabels:
      app: eks-nodejs-app
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: eks-nodejs-app
    spec:
      containers:
      - image: brentley/ecsdemo-nodejs:latest
        imagePullPolicy: Always
        name: eks-nodejs-app
        ports:
        - containerPort: 3000
          protocol: TCP

EOF
# Deploy NodeJS App to EKS
kubectl apply -f ~/environment/eks-scripts/nodejs-deployment.yml
```

Now, let's create a service so we can expose the application in the cluster:

```bash
# Create service yaml file
cat <<-EOF > ~/environment/eks-scripts/nodejs-service.yml
apiVersion: v1
kind: Service
metadata:
  name: eks-nodejs-app
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-internal: "true"

spec:
  selector:
    app: eks-nodejs-app
  ports:
   -  protocol: TCP
      port: 3000
      targetPort: 3000
  type: LoadBalancer
EOF

# Create service in EKS Cluster
kubectl apply -f ~/environment/eks-scripts/nodejs-service.yml
```

Next step is to make sure that the frontend is able to talk to the NodeJS backend App. To do so, let's create a Route53 record set using the following commands:

```bash
# Define variables
HOSTED_ZONE_ID=$(aws route53 list-hosted-zones-by-name \
--dns-name appmeshworkshop.hosted.local \
--max-items 1 | \
jq -r ' .HostedZones | first | .Id' \
| cut -d '/' -f3);
RECORD_SET=$(aws route53 list-resource-record-sets --hosted-zone-id=$HOSTED_ZONE_ID | \
jq -r '.ResourceRecordSets[] | select (.Name == "crystal.appmeshworkshop.hosted.local.") | '.AliasTarget.HostedZoneId'');
NODEJS_LB_URL=$(kubectl get service eks-nodejs-app -o json | jq -r '.status.loadBalancer.ingress[].hostname')

# Create Route53 batch file
cat <<-EOF > /tmp/add_nodejs_recordset.json
{
    "Comment": "CREATE nodejs.appmeshworkshop.hosted.local",
    "Changes": [
        {
            "Action": "CREATE",
            "ResourceRecordSet": {
                "AliasTarget": {
                    "HostedZoneId": "$RECORD_SET",
                    "EvaluateTargetHealth": false,
                    "DNSName": "$NODEJS_LB_URL."
                },
            "Type": "A",
            "Name": "nodejs.appmeshworkshop.hosted.local."
            }
        }
    ]
}
EOF

# Change route53 record set
aws route53 change-resource-record-sets \
--hosted-zone-id $HOSTED_ZONE_ID \
--change-batch file:///tmp/add_nodejs_recordset.json
```

After deploying your application, you will be able to see the NodeJS info in your frontend application.
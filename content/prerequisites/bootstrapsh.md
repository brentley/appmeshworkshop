---
title: "Run the bootstrap scripts"
chapter: false
weight: 23
---

The following bootstrap scripts will (1) build the Docker images, (2) push them to the ECR repository, (3) create the ECS services, and (4) build the EKS cluster.

* Create the bootstrap scripts

```bash

# bootstrap script
cat > ~/environment/scripts/bootstrap <<-"EOF"

#!/bin/bash -ex

echo 'Fetching CloudFormation outputs'
~/environment/scripts/fetch-outputs
echo 'Building Docker Containers'
~/environment/scripts/build-containers
echo 'Creating the ECS Services'
~/environment/scripts/create-ecs-service
echo 'Creating the EKS Cluster'
~/environment/scripts/build-eks

EOF

# fetch-outputs script
cat > ~/environment/scripts/fetch-outputs <<-"EOF"

#!/bin/bash -ex

STACK_NAME=appmesh-workshop
aws cloudformation describe-stacks \
  --stack-name "$STACK_NAME" | \
jq -r '[.Stacks[0].Outputs[] | 
    {key: .OutputKey, value: .OutputValue}] | from_entries' > cfn-output.json

EOF

# Create EKS configuration file
cat > ~/environment/scripts/eks-configuration <<-"EOF"

#!/bin/bash -ex

STACK_NAME=appmesh-workshop
PRIVSUB1_ID=$(jq < cfn-output.json -r '.PrivateSubnetOne')
PRIVSUB1_AZ=$(aws ec2 describe-subnets --subnet-ids $PRIVSUB1_ID | jq -r '.Subnets[].AvailabilityZone')
PRIVSUB2_ID=$(jq < cfn-output.json -r '.PrivateSubnetTwo')
PRIVSUB2_AZ=$(aws ec2 describe-subnets --subnet-ids $PRIVSUB2_ID | jq -r '.Subnets[].AvailabilityZone')
PRIVSUB3_ID=$(jq < cfn-output.json -r '.PrivateSubnetThree')
PRIVSUB3_AZ=$(aws ec2 describe-subnets --subnet-ids $PRIVSUB3_ID | jq -r '.Subnets[].AvailabilityZone')
AWS_REGION=$(curl -s 169.254.169.254/latest/dynamic/instance-identity/document | grep region | cut -d\" -f4)

cat > /tmp/eks-configuration.yml <<-EKS_CONF
  apiVersion: eksctl.io/v1alpha5
  kind: ClusterConfig
  metadata:
    name: $STACK_NAME
    region: $AWS_REGION
  vpc:
    subnets:
      private:
        $PRIVSUB1_AZ: { id: $PRIVSUB1_ID }
        $PRIVSUB2_AZ: { id: $PRIVSUB2_ID }
        $PRIVSUB3_AZ: { id: $PRIVSUB3_ID }
  nodeGroups:
    - name: appmesh-workshop-ng
      labels: { role: workers }
      instanceType: t2.medium
      desiredCapacity: 3
      ssh: 
        allow: true
      privateNetworking: true
      iam:
        withAddonPolicies:
          imageBuilder: true
          albIngress: true
          autoScaler: true
          appMesh: true
          xRay: true
          cloudWatch: true
          externalDNS: true
EKS_CONF

EOF

# Create the EKS building script
cat > ~/environment/scripts/build-eks <<-"EOF"

#!/bin/bash -ex

EKS_CLUSTER_NAME=$(jq < cfn-output.json -r '.EKSClusterName')

if [ -z "$EKS_CLUSTER_NAME" ]
then
  
  if ! aws sts get-caller-identity --query Arn | \
    grep -q 'assumed-role/AppMesh-Workshop-Admin/i-'
  then
    echo "Your role is not set correctly for this instance"
    exit 1
  fi

  sh -c ~/environment/scripts/eks-configuration
  eksctl create cluster -f /tmp/eks-configuration.yml
else

  NODES_IAM_ROLE=$(jq < cfn-output.json -r '.NodeInstanceRole')

  aws eks --region $AWS_REGION update-kubeconfig --name $EKS_CLUSTER_NAME
  cat > /tmp/aws-auth-cm.yml <<-EKS_AUTH
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: aws-auth
      namespace: kube-system
    data:
      mapRoles: |
        - rolearn: $NODES_IAM_ROLE 
          username: system:node:{{EC2PrivateDNSName}}
          groups:
            - system:bootstrappers
            - system:nodes
EKS_AUTH
  kubectl apply -f /tmp/aws-auth-cm.yml
fi

EOF

# build-containers script
cat > ~/environment/scripts/build-containers <<-"EOF"

#!/bin/bash -ex

CRYSTAL_ECR_REPO=$(jq < cfn-output.json -r '.CrystalEcrRepo')
NODEJS_ECR_REPO=$(jq < cfn-output.json -r '.NodeJSEcrRepo')

$(aws ecr get-login --no-include-email)

docker build -t crystal-service ecsdemo-crystal
docker tag crystal-service:latest $CRYSTAL_ECR_REPO:vanilla
docker push $CRYSTAL_ECR_REPO:vanilla

docker build -t nodejs-service ecsdemo-nodejs
docker tag nodejs-service:latest $NODEJS_ECR_REPO:latest
docker push $NODEJS_ECR_REPO:latest

EOF

# create-ecs-service script
cat > ~/environment/scripts/create-ecs-service <<-"EOF"

#!/bin/bash -ex

CLUSTER=$(jq < cfn-output.json -r '.EcsClusterName')
TASK_DEF=$(jq < cfn-output.json -r '.CrystalTaskDefinition')
TARGET_GROUP=$(jq < cfn-output.json -r '.CrystalTargetGroupArn')
SUBNET_ONE=$(jq < cfn-output.json -r '.PrivateSubnetOne')
SUBNET_TWO=$(jq < cfn-output.json -r '.PrivateSubnetTwo')
SUBNET_THREE=$(jq < cfn-output.json -r '.PrivateSubnetThree')
SECURITY_GROUP=$(jq < cfn-output.json -r '.ContainerSecurityGroup')

aws ecs create-service \
  --cluster $CLUSTER \
  --service-name crystal-service-lb \
  --task-definition $TASK_DEF \
  --load-balancer targetGroupArn=$TARGET_GROUP,containerName=crystal-service,containerPort=3000 \
  --desired-count 3 \
  --launch-type FARGATE \
  --network-configuration \
      "awsvpcConfiguration={
        subnets=[$SUBNET_ONE,$SUBNET_TWO,$SUBNET_THREE],
        securityGroups=[$SECURITY_GROUP],
        assignPublicIp=DISABLED}"

EOF

chmod +x ~/environment/scripts/*
```

* Run them!

```bash
~/environment/scripts/bootstrap
```

___

Take a moment to familiarize with the resources that were just created. At this point our microservices should be reacheable via Internet. You can get the External Load Balancer DNS and paste it in a browser to access the frontend service using the following command:

```bash
echo "http://$(jq -r '.ExternalLoadBalancerDNS' cfn-output.json)/"
```

{{% notice info %}}
At this point, the EC2 instances are being registered behind the Load Balancer. It might take a while for them to be available, so if you are seeing any error, just wait a little and try to open the URL again. 
{{% /notice %}} 


What's going on?

Here is what's happening. You have deployed the following components:

* _External Application Load Balancer_ - The ALB is forwarding the HTTP requests it receives to a group of EC2 instances in a target group. Inside the EC2 instances, there is an Ruby app running on port 3000 across 3 public subnets.

* _Ruby Frontend_ - The Ruby microservice is responsible for assembling the UI. It runs on a group of EC2 instances that are configured in a target group and receive requests from the ALB mentioned above. To assemble the UI, the Ruby app has a dependency on two backend microservices (described below).

* _Internal Application Load Balancer_ - This ALB is configured as part of a service running on ECS Fargate. Additionally, there is an A Alias record in a private hosted zone in Route53 whose key is crystal.appmeshworkshop.hosted.local pointing to the ALB FQDN assigned by AWS while creating the load balancer. This allows the Ruby app to connect to crystal.appmeshworkshop.hosted.local regardless of the supporting traffic distribution mechanism. 

* _Crystal / NodeJS Backends_ - These backend microservices run on ECS Fargate and EKS respectively. They listen on port 3000 and provide clients with some internal metadata, like IP address and AZ where they are currently running on.

There is a client side script running on the web app that reloads the page every few seconds. The diagram shows a graphical representation of the flow of each request, as it traverses all the components of the solution.
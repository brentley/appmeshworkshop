---
title: "Run the bootstrap scripts"
chapter: false
weight: 22
---

The following bootstrap scripts will do the following:

 - Install the required tools (kubectl, jq and gettext)
 - Build the Docker images
 - Push them to the ECR repository
 - Create the ECS services

#### Create the bootstrap scripts
```bash
cd ~/environment
mkdir scripts

# bootstrap script
cat > scripts/bootstrap <<-"EOF"

#!/bin/bash -ex

echo 'Installing tools'
~/environment/scripts/install-tools
echo 'Fetching CloudFormation outputs'
~/environment/scripts/fetch-outputs
echo 'Building Docker Containers'
~/environment/scripts/build-containers
echo 'Creating the ECS Services'
~/environment/scripts/create-ecs-service

EOF

# tools script
cat > scripts/install-tools <<-"EOF"

#!/bin/bash -ex

sudo yum install -y jq gettext

sudo wget -O /usr/local/bin/yq https://github.com/mikefarah/yq/releases/download/2.4.0/yq_linux_amd64
sudo chmod +x /usr/local/bin/yq
sudo curl --silent --location -o /usr/local/bin/kubectl https://storage.googleapis.com/kubernetes-release/release/v1.13.7/bin/linux/amd64/kubectl
sudo chmod +x /usr/local/bin/kubectl

curl --silent --location "https://github.com/weaveworks/eksctl/releases/download/latest_release/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv -v /tmp/eksctl /usr/local/bin

if ! [ -x "$(command -v jq)" ] || ! [ -x "$(command -v envsubst)" ] || ! [ -x "$(command -v kubectl)" ] || ! [ -x "$(command -v eksctl)" ]; then
  echo 'ERROR: tools not installed.' >&2
  exit 1
fi

EOF

# fetch-outputs script
cat > scripts/fetch-outputs <<-"EOF"

#!/bin/bash -ex

STACK_NAME="$(echo $C9_PROJECT | sed 's/^Project-//' | tr 'A-Z' 'a-z')"
aws cloudformation describe-stacks \
  --stack-name "$STACK_NAME" | jq -r '[.Stacks[0].Outputs[] | {key: .OutputKey, value: .OutputValue}] | from_entries' > cfn-output.json

EOF

# build eks script
cat > scripts/build-eks <<-"EOF"


#!/bin/bash

set -ex

if ! aws sts get-caller-identity --query Arn | \
  grep -q 'assumed-role/AppMesh-Workshop-Admin/i-'
then
  echo "Your role is not set correctly for this instance"
  exit 1
fi

~/environment/scripts/fetch-outputs

STACK_NAME="$(echo $C9_PROJECT | sed 's/^Project-//' | tr 'A-Z' 'a-z')"
PRIVSUB1=$(jq < cfn-output.json -r '.PrivateSubnetOne')
PRIVSUB2=$(jq < cfn-output.json -r '.PrivateSubnetTwo')
PRIVSUB3=$(jq < cfn-output.json -r '.PrivateSubnetThree')
eksctl create cluster -n $STACK_NAME \
  --vpc-private-subnets $PRIVSUB1,$PRIVSUB2,$PRIVSUB3 \
  --node-private-networking \
  --ssh-access \
  --alb-ingress-access \
  --appmesh-access \
  --external-dns-access \
  --full-ecr-access \
  --asg-access \
  --nodes 3

EOF

# build-containers script
cat > scripts/build-containers <<-"EOF"

#!/bin/bash -ex

CRYSTAL_ECR_REPO=$(jq < cfn-output.json -r '.CrystalEcrRepo')
NODEJS_ECR_REPO=$(jq < cfn-output.json -r '.NodeJSEcrRepo')

$(aws ecr get-login --no-include-email)

docker build -t crystal-service ecsdemo-crystal
docker tag crystal-service:latest $CRYSTAL_ECR_REPO:blue
docker push $CRYSTAL_ECR_REPO:blue

docker build -t nodejs-service ecsdemo-nodejs
docker tag nodejs-service:latest $NODEJS_ECR_REPO:latest
docker push $NODEJS_ECR_REPO:latest

EOF

# create-ecs-service script
cat > scripts/create-ecs-service <<-"EOF"

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
  --service-name crystal-service-lb-blue\
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

chmod +x scripts/*
```

#### Run them!
```bash
cd ~/environment
scripts/bootstrap
```

Take a moment to familiarize with the resources that were just created. At this point our microservices should be reacheable via Internet. Locate the key ExternalLoadBalancerDNS entry in the cfn_outputs.json file and replace the URL below with its value:

http://\<ExternalLoadBalancerDNS\>/

What's going on?

Here is what's happening. You have deployed the following components:

* External Application Load Balancer. The ALB is forwarding the HTTP requests it receives to a group of EC2 instances in a target group.
Inside the EC2 instances, there is an Ruby app running on port 3000 across 3 public subnets. 

* Ruby Frontend. The Ruby microservice is responsible for assembling the UI. It runs on a group of EC2 instances that are configured in a target group and receive requests from the ALB mentioned above. To assemble the UI, the Ruby app requires some help from other microservices, which we will call the Crystal and the NodeJS backends and we will describe below.

* Internal Application Load Balancer. This ALB is configured as part of a service running on ECS Fargate. Additionally, there is an A Alias record in a private hosted zone in Route53 whose key is crystal.appmeshworkshop.hosted.local pointing to the ALB FQDN assigned by AWS while creating the load balancer. This allows the Ruby app to connect to crystal.appmeshworkshop.hosted.local regardless of the **TODO: Finish writing content**

* Crystal / NodeJS backends - The backend microservices run on ECS Fargate and EKS respectively. They listen on port 3000 and provide clients with some internal metadata, like IP address and AZ where they are currently running.

There is a client side script running on the web app that reloads the page every fes seconds. The diagram shows a graphical representation of the flow of each request, as it traverses all the components of the solution, that is an external ALB fronting a target group of EC2 instances running the microservice implemented in Ruby. This microservice has a dependency on the microservice sitting behing the FQDN.
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
```
cd ~/environment
mkdir scripts

# bootstrap script
cat > scripts/bootstrap <<-"EOF"

  #! /bin/bash -ex

  echo 'Installing tools'
  scripts/install-tools
  echo 'Fetching CloudFormation outputs'
  scripts/fetch-outputs
  echo 'Building Docker Containers'
  scripts/build-containers
  echo 'Creating the ECS Services'
  scripts/create-ecs-service

EOF

# tools script
cat > scripts/install-tools <<-"EOF"

  #! /bin/bash -ex

  sudo yum install -y jq gettext
  sudo curl --silent --location -o /usr/local/bin/kubectl https://storage.googleapis.com/kubernetes-release/release/v1.13.7/bin/linux/amd64/kubectl
  sudo chmod +x /usr/local/bin/kubectl
  if ! [ -x "$(command -v jq)" ] || ! [ -x "$(command -v envsubst)" ] || ! [ -x "$(command -v kubectl)" ]; then
    echo 'ERROR: tools not installed.' >&2
    exit 1
  fi

EOF

# fetch-outputs script
cat > scripts/fetch-outputs <<-"EOF"

  #! /bin/bash -ex

  STACK_NAME="$(echo $C9_PROJECT | sed 's/^Project-//')"
  aws cloudformation describe-stacks \
    --stack-name "$STACK_NAME" | jq -r '[.Stacks[0].Outputs[] | {key: .OutputKey, value: .OutputValue}] | from_entries' > cfn-output.json

EOF

# build-containers script
cat > scripts/build-containers <<-"EOF"

  #! /bin/bash -ex

  CRYSTAL_ECR_REPO=$(jq < cfn-output.json -r '.CrystalEcrRepo')
  NODEJS_ECR_REPO=$(jq < cfn-output.json -r '.NodeJSEcrRepo')

  $(aws ecr get-login --no-include-email)

  docker build -t frontend-service ecsdemo-crystal
  docker tag frontend-service:latest $CRYSTAL_ECR_REPO:latest
  docker push $CRYSTAL_ECR_REPO:latest

  docker build -t nodejs-service ecsdemo-nodejs
  docker tag nodejs-service:latest $NODEJS_ECR_REPO:latest
  docker push $NODEJS_ECR_REPO:latest

EOF

# create-ecs-service script
cat > scripts/create-ecs-service <<-"EOF"

  #! /bin/bash -ex

  CLUSTER=$(jq < cfn-output.json -r '.EcsClusterName')
  TASK_DEF=$(jq < cfn-output.json -r '.CrystalTaskDefinition')
  TARGET_GROUP=$(jq < cfn-output.json -r '.CrystalTargetGroupArn')
  SUBNET_ONE=$(jq < cfn-output.json -r '.PrivateSubnetOne')
  SUBNET_TWO=$(jq < cfn-output.json -r '.PrivateSubnetTwo')
  SECURITY_GROUP=$(jq < cfn-output.json -r '.ContainerSecurityGroup')

  aws ecs create-service \
    --cluster $CLUSTER \
    --service-name CrystalService \
    --task-definition $TASK_DEF \
    --load-balancer targetGroupArn=$TARGET_GROUP,containerName=crystal-service,containerPort=3000 \
    --desired-count 2 \
    --launch-type FARGATE \
    --network-configuration \
        "awsvpcConfiguration={
          subnets=[$SUBNET_ONE,$SUBNET_TWO],
          securityGroups=[$SECURITY_GROUP],
          assignPublicIp=DISABLED}"

EOF

chmod +x scripts/*
```

#### Run them!
```
cd ~/environment
scripts/bootstrap
```


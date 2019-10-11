---
title: "Install the envoy proxy"
date: 2018-09-18T17:40:09-05:00
weight: 20
---

Time to install the envoy proxy. We will use **AWS Systems Manager** (SSM) to configure the EC2 instances that serve the frontend microservice. We will have to create an SSM document to define the actions that SSM will perform on the managed instances. Documents use JavaScript Object Notation (JSON) or YAML.

* Execute the following script in your Cloud9 environment to create the SSM document.

```bash
# Create YAML file #
cat <<-"EOF" > /tmp/install_envoy.yml
---
schemaVersion: '2.2'
description: Install Envoy Proxy
parameters: 
      region:
        type: String
      meshName:
        type: String
      vNodeName:
        type: String 
      ignoredUID:
        type: String
        default: '1337'
      proxyIngressPort:
        type: String
        default: '15000'
      proxyEgressPort:
        type: String
        default: '15001'
      appPorts:
        type: String
      egressIgnoredIPs:
        type: String
        default: '169.254.169.254,169.254.170.2'
      egressIgnoredPorts:
        type: String 
        default: '22'
mainSteps:
- action: aws:configureDocker
      name: configureDocker
      inputs:
        action: Install
- action: aws:runShellScript
      name: installEnvoy
      inputs:
        runCommand: 
          - |
            #! /bin/bash -ex

            sudo yum install -y jq

            EC2_METADATA="http://169.254.169.254/latest"
            AWS_ROLE=$(curl $EC2_METADATA/meta-data/iam/security-credentials/)
            AWS_CREDENTIALS=$(curl $EC2_METADATA/meta-data/iam/security-credentials/$AWS_ROLE)

            AWS_ACCESS_KEY_ID=$(echo $AWS_CREDENTIALS | jq -r .AccessKeyId)
            AWS_SECRET_ACCESS_KEY=$(echo $AWS_CREDENTIALS | jq -r .SecretAccessKey)
            AWS_SESSION_TOKEN=$(echo $AWS_CREDENTIALS | jq -r .Token)

            $(aws ecr get-login --no-include-email --region {{region}} --registry-ids 111345817488)

            # Install and run envoy
            sudo docker run --detach \
                --env APPMESH_VIRTUAL_NODE_NAME=mesh/{{meshName}}/virtualNode/{{vNodeName}} \
                --env AWS_ACCESS_KEY_ID=$AWS_ACCESS_KEY_ID \
                --env AWS_SECRET_ACCESS_KEY=$AWS_SECRET_ACCESS_KEY \
                --env ENABLE_ENVOY_XRAY_TRACING=1 \
                --env AWS_SESSION_TOKEN=$AWS_SESSION_TOKEN \
                --log-driver="awslogs" \
                --log-opt awslogs-create-group=true \
                --log-opt awslogs-region={{region}} \
                --log-opt awslogs-stream=envoy \
                --log-opt awslogs-group=appmesh-workshop-frontend-envoy \
                -u 1337 --network host \
                111345817488.dkr.ecr.{{region}}.amazonaws.com/aws-appmesh-envoy:v1.11.1.1-prod
- action: aws:runShellScript
      name: enableRouting
      inputs:
        runCommand: 
          - |
            #! /bin/bash -ex

            APPMESH_LOCAL_ROUTE_TABLE_ID="100"
            APPMESH_PACKET_MARK="0x1e7700ce"

            # Initialize chains
            sudo iptables -t mangle -N APPMESH_INGRESS
            sudo iptables -t nat -N APPMESH_INGRESS
            sudo iptables -t nat -N APPMESH_EGRESS
            sudo ip rule add fwmark "$APPMESH_PACKET_MARK" lookup $APPMESH_LOCAL_ROUTE_TABLE_ID
            sudo ip route add local default dev lo table $APPMESH_LOCAL_ROUTE_TABLE_ID

            # Enable egress routing
              # Ignore egress redirect based UID, ports, and IPs
              sudo iptables -t nat -A APPMESH_EGRESS \
                      -m owner --uid-owner {{ignoredUID}} \
                      -j RETURN
              sudo iptables -t nat -A APPMESH_EGRESS \
                      -p tcp \
                      -m multiport --dports "{{egressIgnoredPorts}}" \
                      -j RETURN
              sudo iptables -t nat -A APPMESH_EGRESS \
                      -p tcp \
                      -d "{{egressIgnoredIPs}}" \
                      -j RETURN
              # Redirect everything that is not ignored
              sudo iptables -t nat -A APPMESH_EGRESS \
                      -p tcp \
                      -j REDIRECT --to {{proxyEgressPort}}
              # Apply APPMESH_EGRESS chain to non-local traffic
              sudo iptables -t nat -A OUTPUT \
                      -p tcp \
                      -m addrtype ! --dst-type LOCAL \
                      -j APPMESH_EGRESS

            # Enable ingress routing
              # Route everything arriving at the application port to Envoy
              sudo iptables -t nat -A APPMESH_INGRESS \
                      -p tcp \
                      -m multiport --dports "{{appPorts}}" \
                      -j REDIRECT --to-port "{{proxyIngressPort}}"
              # Apply APPMESH_INGRESS chain to non-local traffic
              sudo iptables -t nat -A PREROUTING \
                      -p tcp \
                      -m addrtype ! --src-type LOCAL \
                      -j APPMESH_INGRESS
- action: aws:runShellScript
      name: installXRay
      inputs:
        runCommand: 
          - |
            #! /bin/bash -ex

            XRAY_HOST=https://s3.dualstack.{{region}}.amazonaws.com
            XRAY_PATH=aws-xray-assets.{{region}}/xray-daemon/aws-xray-daemon-3.x.rpm

            # Install and run xray daemon
            sudo curl $XRAY_HOST/$XRAY_PATH -o /tmp/xray.rpm
            sudo yum install -y /tmp/xray.rpm

EOF
# Create ssm document #
aws ssm create-document \
      --name appmesh-workshop-InstallEnvoy \
      --document-format YAML \
      --content file:///tmp/install_envoy.yml \
      --document-type Command
```

* Create an association using State Manager. An association is a binding of the intent (described in the document) to a target specified by either a list of instance IDs or a tag query. Use the following command to create an association. Note that you are specifying a tag query in the target parameter and using the Frontend Auto Scaling group name.

```bash
# Define variables #
AUTOSCALING_GROUP=$(jq < cfn-output.json -r '.RubyAutoScalingGroupName'); \
# Create ssm association #
aws ssm create-association \
      --name appmesh-workshop-InstallEnvoy \
      --targets "Key=tag:aws:autoscaling:groupName,Values=$AUTOSCALING_GROUP" \
      --max-errors 0 \
      --max-concurrency 50% \
      --parameters \
          "region=$AWS_REGION,
            meshName=appmesh-workshop,
            vNodeName=frontend-v1,
            appPorts=3000"
```
---
title: "Add service discovery"
date: 2018-09-18T17:39:30-05:00
weight: 10
---

App Mesh supports microservice applications that use service discovery naming for their components. 
In our baseline infrastructure, neither the frontend nor the crystal microservices have service discovery support.

We will be using **AWS Cloud Map** to discover our resources. In some way, our baseline infrastructure is already using the service. Cloud Map enables you to name your application resources with custom names, and it automatically updates the locations of these dynamically changing resources. 

* Let's create a service in Cloud Map for our frontend microservice
```
NAMESPACE_ID=$(jq < cfn-output.json -r '.NamespaceId'); \
aws servicediscovery create-service \
      --name frontend \
      --namespace-id $NAMESPACE_ID \
      --description 'Discovery service for the frontend service' \
      --dns-config 'RoutingPolicy=MULTIVALUE,DnsRecords=[{Type=SRV,TTL=60}]' \
      --health-check-custom-config 'FailureThreshold=1'
```

{{% notice note %}}
Check the json output, and write down the value of the **Id** that AWS Cloud Map assigned to the service you just created.
{{% /notice %}}

* We will use **Systems Manager State Manager** to configure the EC2 instances that serve the frontend microservice. State Manager uses documents to specify configuration. Create a YAML file in your Cloud9 environment named **register-instance.yml** with the following content:
```
---
schemaVersion: '2.2'
description: Register service instances
parameters: 
      serviceid:
        type: String
        description: The ID of the Cloud Map service
mainSteps:
- action: aws:runShellScript
      name: configureServer
      inputs:
        runCommand:
        - export AWS_DEFAULT_REGION=$(curl -s  http://169.254.169.254/latest/dynamic/instance-identity/document | jq -r '.region')
        - INSTANCE_ID=$(curl http://169.254.169.254/latest/meta-data/instance-id);
        - INSTANCE_IP=$(curl -s http://169.254.169.254/latest/meta-data/local-ipv4);
        - aws servicediscovery register-instance --service-id {{serviceid}} --instance-id $INSTANCE_ID --attributes AWS_INSTANCE_IPV4=$INSTANCE_IP,AWS_INSTANCE_PORT=3000
```

* Call the SSM API to create the document
```
aws ssm create-document \
      --name AppMesh-Workshop-RegisterInstance \
      --document-format YAML \
      --content file://register-instance.yml \
      --document-type Command
```

* Create an association using State Manager. An association is a binding of the intent (described in the document) to a target specified by either a list of instance IDs or a tag query. Use the following command to create an association. Note that you are specifying a tag query in the target parameter and using the Frontend Auto Scaling group name. **Replace {serviceid} with the service id you wrote down previousy**.

```
AUTOSCALING_GROUP=$(jq < cfn-output.json -r '.RubyAutoScalingGroupName'); \
aws ssm create-association \
      --name AppMesh-Workshop-RegisterInstance \
      --targets "Key=tag:aws:autoscaling:groupName,Values=$AUTOSCALING_GROUP" \
      --schedule-expression "cron(0 0/30 * 1/1 * ? *)" \
      --parameters "serviceid={serviceid}"
```
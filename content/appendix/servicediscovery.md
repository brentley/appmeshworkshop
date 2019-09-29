---
title: "Adding service discovery to EC2 backends"
date: 2018-09-18T17:39:30-05:00
weight: 5
---

**AWS Cloud Map** is a cloud resource discovery service. Cloud Map enables you to name your application resources with custom names, and it automatically updates the locations of these dynamically changing resources.

The following SSM Document, lets you register / deregister EC2 instances belonging to an Auto Scaling Group in Cloud Map service using servicetl.

Create a YAML file named **configure-cloudmap.yml** and copy/paste the following content:
```yaml
---
schemaVersion: '2.2'
description: Configure Cloud Map
parameters: 
      serviceId:
        type: String
      instancePort:
        type: String
mainSteps:
- action: aws:runShellScript
      name: configureCloudMap
      inputs:
        runCommand: 
          - |
            #! /bin/bash
          
            EC2_METADATA=http://169.254.169.254/latest
            REGION=$(curl -s $EC2_METADATA/dynamic/instance-identity/document | jq -r '.region')
            INSTANCE_ID=$(curl -s $EC2_METADATA/meta-data/instance-id);
            INSTANCE_IP=$(curl -s $EC2_METADATA/latest/meta-data/local-ipv4);

            cat > /etc/init.d/cloudmap-register <<-EOF
            #! /bin/bash -ex
            aws servicediscovery register-instance \
              --region $REGION \
              --service-id {{serviceId}} \
              --instance-id $INSTANCE_ID \
              --attributes AWS_INSTANCE_IPV4=$INSTANCE_IP,AWS_INSTANCE_PORT={{instancePort}}
            exit 0
            EOF
            chmod a+x /etc/init.d/cloudmap-register
        
            cat > /etc/init.d/cloudmap-deregister <<-EOF
            #! /bin/bash -ex
            aws servicediscovery deregister-instance \
              --region $REGION \
              --service-id {{serviceid}} \
              --instance-id $INSTANCE_ID
            exit 0
            EOF
            chmod a+x /etc/init.d/cloudmap-deregister
        
            cat > /usr/lib/systemd/system/cloudmap.service <<-EOF
            [Unit]
            Description=Run CloudMap service
            Requires=network-online.target network.target
            DefaultDependencies=no
            Before=shutdown.target reboot.target halt.target

            [Service]
            Type=oneshot
            KillMode=none
            RemainAfterExit=yes
            ExecStart=/etc/init.d/cloudmap-register
            ExecStop=/etc/init.d/cloudmap-deregister
        
            [Install]
            WantedBy=multi-user.target
            EOF
        
            systemctl enable cloudmap.service
            systemctl start  cloudmap.service
```

Call the SSM API to create the document

```bash
aws ssm create-document \
      --name Configure-CloudMap \
      --document-format YAML \
      --content file://configure-cloudmap.yml \
      --document-type Command
```
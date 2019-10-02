---
title: "Retrieve the SSH key"
chapter: false
weight: 31
---

Please run this command to retrieve the SSH Key and store it in Cloud9. This key will be used on the ec2 and worker node instances to allow ssh access if necessary.

```bash
# Retrieve private key
aws ssm get-parameter \
  --name /appmeshworkshop/keypair/id_rsa \
  --with-decryption | jq .Parameter.Value --raw-output > ~/.ssh/id_rsa

# Set appropriate permission on private key
chmod 600 ~/.ssh/id_rsa

# Store public key separately from private key
ssh-keygen -y -f ~/.ssh/id_rsa > ~/.ssh/id_rsa.pub
```

---
title: "Retrieve the SSH key"
chapter: false
weight: 22
---

Run the following command to retrieve the SSH Key and store it in Cloud9. This key will be used on the ec2 and worker node instances to allow ssh access if necessary.

```bash
# Retrieve private key
KEY_NAME=$(aws ssm describe-parameters | jq -r 'select(.Parameters[].Name | contains("/appmeshworkshop/keypair/")).Parameters[].Name')
aws ssm get-parameter \
  --name "$KEY_NAME" \
  --with-decryption | jq .Parameter.Value --raw-output > ~/.ssh/id_rsa

# Set appropriate permission on private key
chmod 600 ~/.ssh/id_rsa

# Store public key separately from private key
ssh-keygen -y -f ~/.ssh/id_rsa > ~/.ssh/id_rsa.pub
```

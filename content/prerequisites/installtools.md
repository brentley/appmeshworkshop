---
title: "Install the required tools"
chapter: false
weight: 16
---
Before deploying the baseline stack, let's install the required tools (kubectl, jq and gettext) to you Cloud9 environment. To do so, start creating an install script with the following commands:

```bash
# create a folder for the scripts
mkdir ~/environment/scripts

# tools script
cat > ~/environment/scripts/install-tools <<-"EOF"

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

chmod +x ~/environment/scripts/install-tools
```

Now, run it with the following command:

```bash
~/environment/scripts/install-tools
```

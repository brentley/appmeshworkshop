---
title: "Install the required tools"
chapter: false
weight: 16
---

{{% notice info %}}
Starting from here, when you see command to be entered such as below, you will enter these commands into Cloud9 IDE. You can use the **Copy to clipboard** feature (right hand upper corner) to simply copy and paste into Cloud9. In order to paste, you can use Ctrl + V for Windows or Command + V for Mac.
{{% /notice %}}

* Before deploying the baseline stack, let's install the required tools (kubectl, jq, gettext and the latest awscli version) to you Cloud9 environment. To do so, start creating an install script with the following commands:

```bash
# create a folder for the scripts
mkdir ~/environment/scripts

# tools script
cat > ~/environment/scripts/install-tools <<-"EOF"

#!/bin/bash -ex

sudo yum install -y jq gettext bash-completion

sudo curl --silent --location "https://s3.amazonaws.com/session-manager-downloads/plugin/latest/linux_64bit/session-manager-plugin.rpm" -o "session-manager-plugin.rpm"
sudo yum install -y session-manager-plugin.rpm

sudo curl --silent --location -o /usr/local/bin/kubectl https://storage.googleapis.com/kubernetes-release/release/v1.16.8/bin/linux/amd64/kubectl
sudo chmod +x /usr/local/bin/kubectl
echo 'source <(kubectl completion bash)' >>~/.bashrc
source ~/.bashrc

curl --silent --location "https://github.com/weaveworks/eksctl/releases/download/latest_release/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv -v /tmp/eksctl /usr/local/bin

if ! [ -x "$(command -v jq)" ] || ! [ -x "$(command -v envsubst)" ] || ! [ -x "$(command -v kubectl)" ] || ! [ -x "$(command -v eksctl)" ] || ! [ -x "$(command -v ssm-cli)" ]; then
  echo 'ERROR: tools not installed.' >&2
  exit 1
fi

pip install awscli --upgrade --user

EOF

chmod +x ~/environment/scripts/install-tools
```

* Now, run it with the following command:

```bash
~/environment/scripts/install-tools
```

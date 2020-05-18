---
layout:     post
title:      "SSH into private ec2-instances"
description: "Using the SSH over SSM to connect to any ec2-instance in a private subnet"
date:       2020-05-18 18:00:00
author:     "Xander Verheij"
header-img: assets/img/posts/aws-mfa-subaccount/1200px-Amazon_Web_Services_Logo.svg.png

categories:
  - Infrastructure as code
---
## Premise
Theoretically you start you deploy your elasticbeanstalk using your terraform infrastructure code and everything works out of the box.
But more often then not, something doesn't go as planned; and to speed up the debugging it would be really helpfull to just ssh into the underlying ec2-instance.

However that ec2-instance is running in a private subnet, thus not accessible via the internet. First solution that comes to mind is setting up a bastion server, but with AWS ssm we have a different option.

Again thanks to my colleague [Joep Joosten](https://www.linkedin.com/in/joepjoosten/) for doing the heavy lifting.

## Prerequisites 
* AWS Session Manager plugin [check here](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-working-with-install-plugin.html) for install guide
* To easily access instances from a suborganization check this [blogpost](https://xrv.nl/infrastructure%20as%20code/2020/05/18/aws-mfa-subaccount/)


## Install ssh over ssm
Install the library that makes the magic happen, all credits go to [elpy1](https://github.com/elpy1)!

```bash
git clone https://github.com/elpy1/ssh-over-ssm.git ~/ssh-over-ssm
```

## Configure ssh 
Add the following to your ~/.ssh/config. 
Make sure to check the ProxyCommand it should point to ssh-over-ssm you just cloned from github.

```bash
Match host i-*
StrictHostKeyChecking no
IdentityFile ~/.ssh/ssm-ssh-tmp
PasswordAuthentication no
GSSAPIAuthentication no
ProxyCommand ~/ssh-over-ssm/ssh-ssm.sh %h %r
```

## Optional configure aws to use suborganization using aws-mfa

```bash
aws-mfa {suborganization} {mfa_code}
```

## Find the instance you want to connect to

The following command can be used to find all the instances running in the account you have access to

```bash
aws ec2 describe-instances --query "Reservations[*].Instances[*].{name: Tags[?Key=='Name'] | [0].Value, instance_id: InstanceId, ip_address: PrivateIpAddress, state: State.Name}"  --output table
```

To make it a little bit easier you can add it to your .zshrc, .bashrc, [othershell]rc
```bash
alias listec2="aws ec2 describe-instances --query \"Reservations[*].Instances[*].{name: Tags[?Key=='Name'] | [0].Value, instance_id: InstanceId, ip_address: PrivateIpAddress, state: State.Name}\"  --output table"
```

## Connect using ssh!
Simply connect to the instance you want by using the following command:
```bash
ssh ec2-user@i-xyz123456789
```
And check whichever log you want, or use your tool of choice to check if you can connect to where you want to connect :-)
 
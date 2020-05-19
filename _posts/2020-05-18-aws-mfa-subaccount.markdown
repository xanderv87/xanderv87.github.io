---
layout:     post
comments: true
title:      "Log in as a role using AWS CLI (with MFA!)"
description: "Using the AWS CLI to gain access to a role while using MFA"
date:       2020-05-18 17:00:00
author:     "Xander Verheij"
header-img: assets/img/posts/aws-mfa-subaccount/1200px-Amazon_Web_Services_Logo.svg.png

categories:
  - Infrastructure as code
---
At [ihomer](https://ihomer.nl) we have project for multiple clients. Each client has there own organization under our main ihomer AWS account.
The initial approach to accessing these subaccounts was using a IAM account on each subaccount. But thanks to my collegue [Joep Joosten](https://www.linkedin.com/in/joepjoosten/) we now have a better solution!
Using the [aws-mfa](https://github.com/joepjoosten/aws-cli-mfa-oh-my-zsh) zsh plugin we can use our (ofcource mfa secured) main account credentials to hop to the subaccount.

Follow the next steps to make the magic happen!


## Prerequesites:
* [awscli](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html)
* [zsh](https://github.com/ohmyzsh/ohmyzsh/wiki/Installing-ZSH)

## Install zsh plugin
```bash
git clone --depth=1 https://github.com/joepjoosten/aws-cli-mfa-oh-my-zsh.git “$ZSH/custom/plugins/aws-mfa”
```
Enable the aws-mfa plugin in your .zshrc
```bash
plugins=(
  ...
  aws-mfa
)
```

## Configure AWS CLI
Make sure to add the credentials for your main account to your ~/.aws/credentials:
```bash
[{main_account}]
aws_access_key_id = {access_key_id}
aws_secret_access_key = {secret_access_key}
```

Update your ~/.aws/config with the following for the [main-account]:
```bash
[profile {main_account}]
output = json
region = eu-central-1
```

Add the following to your ~/.aws/config for every organization/role combo you want to use for your terraform execution:
```bash
[profile {sub_account}]
role_arn = arn:aws:iam::{organisation_id}:role/{role_name}
source_profile = {main_account}
mfa_serial = arn:aws:iam::{main_account_id}:mfa/{user_name}
region = {region}
 ```

## Login using aws-mfa
When you have configured everything you can call the following command, it will ask for your MFA code. Or you can optional provide your MFA code directly
```bash
aws-mfa {sub_account}
aws-mfa {sub_account} {mfa_code}
 ```
 
## Practical example (terraform)
Now we can for example use this to run terraform by removing the specific profile/credentials from the terraform script.

### Alter terraform scripts
Normally you might have defined a profile to be used in your terraform scripts. Using this method the profile used will be the one you logged in to using aws-mfa.
```hcl
provider "aws" {
  version = "~> 2.0"
  region = "eu-central-1"
  profile = "sub_acount" REMOVE_ME
}
```
 

### Run terraform as usual
When you have altered the terraform scripts and called the ```aws-mfa``` script you should be able to use terraform as usual.
```bash
terraform plan
```
 
 
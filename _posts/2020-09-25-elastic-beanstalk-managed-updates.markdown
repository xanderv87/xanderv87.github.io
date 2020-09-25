---
layout:     post
comments: true
title:      "Managed updates for elasticbeanstalk with terraform"
description: "How to enable managed updates for elasticbeanstalk with terraform"
date:       2020-09-25 18:00:00
author:     "Xander Verheij"
header-img: assets/img/posts/elasticbeanstalk/iu.jpeg

categories:
  - Infrastructure as code
---
Using ElasticBeanstalk can be a blessing and a curse when using Terraform to deploy it.

A lot of the automagic feature's you get when clicking in the AWS web interface can be daunting to reproduce when using a Infrastructure as Code tool like terraform.

Hopefully the following code snippets can help someone who is searching how to enable managed updates on elastic beanstalk.



```
data "aws_iam_policy" "AWSElasticBeanstalkService" {
  arn = "arn:aws:iam::aws:policy/service-role/AWSElasticBeanstalkService"
}
resource "aws_iam_role_policy_attachment" "eb_service_role" {
  role       = aws_iam_role.eb_service_role.name
  policy_arn = data.aws_iam_policy.AWSElasticBeanstalkService.arn
}
data "aws_iam_policy" "AWSElasticBeanstalkEnhancedHealth" {
  arn = "arn:aws:iam::aws:policy/service-role/AWSElasticBeanstalkEnhancedHealth"
}
resource "aws_iam_role_policy_attachment" "eb_service_role_2" {
  role       = aws_iam_role.eb_service_role.name
  policy_arn = data.aws_iam_policy.AWSElasticBeanstalkEnhancedHealth.arn
}
resource "aws_iam_role" "eb_service_role" {
  name = "${var.application_name}-${terraform.workspace}-beanstalk-service-role"
  tags = {
    Environment = terraform.workspace
  }

  assume_role_policy = <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Action": "sts:AssumeRole",
      "Principal": {
        "Service": "elasticbeanstalk.amazonaws.com"
      },
      "Effect": "Allow",
      "Sid": ""
    }
  ]
}
EOF
}
```
Here we created the needed service-roles to allow elastic beanstalk to do the managed updates and to enable enhanced health. Enhanced health is a prerequisit for managed updates.

```

  setting {
    namespace = "aws:elasticbeanstalk:healthreporting:system"
    name = "SystemType"
    value = "enhanced"
    resource = ""
  }
  setting {
    namespace = "aws:elasticbeanstalk:managedactions"
    name = "ManagedActionsEnabled"
    value = "true"
    resource = ""
  }
  setting {
    namespace = "aws:elasticbeanstalk:environment"
    name = "ServiceRole"
    value = aws_iam_role.eb_service_role.arn
    resource = ""
  }
  setting {
    namespace = "aws:elasticbeanstalk:managedactions"
    name = "PreferredStartTime"
    value = "MON:02:00"
    resource = ""
  }
  setting {
    namespace = "aws:elasticbeanstalk:managedactions"
    name = "ServiceRoleForManagedUpdates"
    value = aws_iam_role.eb_service_role.arn
    resource = ""
  }
  setting {
    namespace = "aws:elasticbeanstalk:managedactions:platformupdate"
    name = "UpdateLevel"
    value = "Patch"
    resource = ""
  }
  setting {
    namespace = "aws:elasticbeanstalk:managedactions:platformupdate"
    name = "InstanceRefreshEnabled"
    value = "true"
    resource = ""
  }
  ```

These are the actual settings to set on the elastic beanstalk instance. A quick roundup:
### SystemType: enhanced
Enables enhanced health monitoring
### ManagedActionsEnabled
Enables managed updates
### PreferredStartTime
The time at which the managed updates are done
### UpdateLevel
Can be either Patch or Minor; do you update from 1.2.1 to 1.2.2 or 1.3.1
### InstanceRefreshEnabled
Create a new instance even if there is no update to be performed.
### ServiceRole & ServiceRoleForManagedUpdates
The arn for the roles you just created, making sure the elasticbeanstalk has the appropriate rights.
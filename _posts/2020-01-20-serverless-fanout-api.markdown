---
layout:     post
comments: true
title:      "Serverless fanout API"
description: "How to create a serverless API that can fanout to multiple services/environments"
date:       2020-01-20 17:00:00
author:     "Xander Verheij"
header-img: assets/img/posts/serverless-fanout-api/terraform.png

categories:
  - Infrastructure as code
---
That's a lot of buzzwords in the title..
##Intro
Imagine you are creating an application which relies heavily on live data from an external party.
It is a relatively small project, but still you need some sort of Test or Acceptance environment. 
However the external party doesn't have anything in place that can be used, no Test or Acceptance environment, nor the ability to send requests to multiple endpoints.

Using some AWS magic we can get what we want without bothering the other party and gaining some other nicities in the process.

### Used technology:
First and foremost we will make use of all the goodies that AWS has to offer. This enables us to create this Fanout API without any physical or virtual servers and thus no maintance on that part.

To be more specific the AWS services we will be using are the following:
* AWS [Api Gateway](https://aws.amazon.com/api-gateway/)
* AWS [SNS](https://aws.amazon.com/sns)
* AWS [SQS](https://aws.amazon.com/sqs)

To automate the deployment of these services and to make sure we do not create a snowflake we will be using
* [Terraform](https://www.terraform.io)

### Assumed knowledge
Some basic knowledge about AWS is assumed i.e. you have an account and know how to login.
The same goes for Terraform; you can run a terraform file.

### Architecture
Below the bird's eye view of what we are going to create:

![IMAGE](/assets/img/posts/serverless-fanout-api/1C335F82C1DA7BD673AFF75C439F6811.jpg)


## Terraform scripts

So let's get down to business and go through the different parts of the terraform scripts

### Configuring the AWS provider

```
provider "aws" {
  profile = "blue-playground"
  region = "${var.region}"
}
```

### Creating the SNS Topic
We will need a SNS topic to which we can forward the calls to the api endpoint.

```
resource "aws_sns_topic" "fanout_sns" {
  name = "fanout-topic"
}
```

### Creating the IAM policy
We need to define some IAM policies to make sure that Api gateway is allowed to publish to the SNS topic.

```
resource "aws_iam_role" "apigateway_sns" {
  name = "fanout-apigateway-sns-role"

  assume_role_policy = <<EOF
{
 "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {"Service": "apigateway.amazonaws.com"},
      "Action": "sts:AssumeRole"
    }
    ]
}
EOF
}
resource "aws_iam_role_policy" "apigateway_sns_policy" {
  name = "test_policy"
  role = "${aws_iam_role.apigateway_sns.id}"

  policy = <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "sns:Publish",
      "Resource": "${aws_sns_topic.fanout_sns.arn}"
    }
  ]
}
EOF
}
```

### Creating the API
Here we are creating the Api Gateway and creating a deployment. 
The deployment is dependent on the integration_response, to make sure that the deployment is only triggered when the API is completly ready.

```
resource "aws_api_gateway_rest_api" "api" {
  name = "demo-api-fanout"
}
resource "aws_api_gateway_deployment" "api_deployment" {
  depends_on = [
    aws_api_gateway_integration_response.integration_response]
  rest_api_id = aws_api_gateway_rest_api.api.id
  stage_name = "demo"
}
```

### Creating the endpoint
Creating endpoints using terraform (without available modules that make your life easier) is quite tedious.

For the sake of this post I typed out all the resources required. Most of this is straightforward except for the `aws_api_gateway_integration`. This is where the 'magic' happens. 
We are forwarding the message body from the call to the api gateway endpoint in the querystring towards SNS.

```
resource "aws_api_gateway_resource" "resource" {
  depends_on = [
    aws_api_gateway_rest_api.api]
  rest_api_id = aws_api_gateway_rest_api.api.id
  parent_id = aws_api_gateway_rest_api.api.root_resource_id
  path_part = "fanout"
}

resource "aws_api_gateway_method_response" "response-200" {
  depends_on = [
    aws_api_gateway_resource.resource,
    aws_api_gateway_method.method]
  rest_api_id = aws_api_gateway_rest_api.api.id
  resource_id = aws_api_gateway_resource.resource.id
  http_method = aws_api_gateway_method.method.http_method
  status_code = "200"
}

resource "aws_api_gateway_method" "method" {
  depends_on = [
    aws_api_gateway_resource.resource]
  rest_api_id = aws_api_gateway_rest_api.api.id
  resource_id = aws_api_gateway_resource.resource.id
  http_method = "POST"
  authorization = "NONE"
  request_parameters = {
    "method.request.path.proxy" = true
  }
}

resource "aws_api_gateway_integration" "integration" {
  depends_on = [
    aws_api_gateway_resource.resource,
    aws_api_gateway_method.method]
  rest_api_id = aws_api_gateway_rest_api.api.id
  resource_id = aws_api_gateway_resource.resource.id
  http_method = aws_api_gateway_method.method.http_method
  integration_http_method = "POST"
  type = "AWS"
  uri = "arn:aws:apigateway:${var.region}:sns:path//"
  credentials = aws_iam_role.apigateway_sns.arn
  request_parameters = {
    "integration.request.querystring.Action" = "'Publish'"
    "integration.request.querystring.TopicArn" = "'${aws_sns_topic.fanout_sns.arn}'"
    "integration.request.querystring.Message" = "method.request.body"
  }
  request_templates = {
    "application/json" = ""
  }
}
resource "aws_api_gateway_integration_response" "integration_response" {
  depends_on = [
    aws_api_gateway_resource.resource,
    aws_api_gateway_method.method,
    aws_api_gateway_integration.integration]
  rest_api_id = aws_api_gateway_rest_api.api.id
  resource_id = aws_api_gateway_resource.resource.id
  http_method = "POST"
  status_code = "200"
  selection_pattern = ""

  response_templates = {
    "application/json" = <<EOF
      {"body": "Message received."}
    EOF
  }
}
```

When we run `terraform apply` using this code we will have created:
* api gateway
* a api gateway endpoint
* a sns topic towards which the POST body is forwarded

So we don't have anything yet.
The easiest way to test if everything is working in order is to create an e-mail subscription to the SNS-topic.
Unfortunatly (but logically) this can not be done using terraform, because there is a async approval to enable the e-mail notifcations.


Now that we have coupled api gateway with SNS we can notify almost every service on AWS, for the sake of this example we will use SQS:

```
resource "aws_sqs_queue" "fanout_queue" {
  name = "fanout-queue"
}

resource "aws_sns_topic_subscription" "enervalis_fanout_input_sqs" {
  topic_arn = aws_sns_topic.fanout_sns.arn
  protocol = "sqs"
  endpoint = aws_sqs_queue.fanout_queue.arn
}
```

The beauty is that we can create multiple queues and thus multiple consumers to read from this input. So in this example case with an external party without multiple environments we are able to supply multiple environments with an influx of data to make testing just a little bit easier.


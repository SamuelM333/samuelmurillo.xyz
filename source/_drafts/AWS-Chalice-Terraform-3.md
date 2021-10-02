---
title: "AWS Chalice + Terraform Part 3: Testing your app"
date: 2021-09-09 12:33:25
tags:
  - aws
  - chalice
  - lambda
  - python
  - serverless
  - datadog
  - testing
categories:
  - "AWS Chalice + Terraform"
---

This is the third part of the Chalice + Terraform series. You can check part 1 and 2 here:

{% post_link AWS-Chalice-Terraform %}
{% post_link AWS-Chalice-Terraform-2 %}


## Writing unit tests

We can go a step further and write unit tests

https://aws.github.io/chalice/topics/testing.html

```
mkdir tests
touch tests/{__init__.py,test_app.py}
```

http, 200 and payload
sns/sqs, payload

https://medium.com/craftsmenltd/a-journey-to-aws-lambda-integration-testing-with-python-localstack-and-terraform-2f17043c7dda

## Writing end-to-end tests

boto3 pytest

## What's next

Up next, let's get ready for production release, with tips for bigger codebases, logging, tracing, testing and more. You can check the fourth part of this series here:

{% post_link AWS-Chalice-Terraform-4 %}

---
title: "AWS Chalice + Terraform Part 2: Local development"
date: 2021-09-09 12:33:25
tags:
  - aws
  - chalice
  - lambda
  - python
  - serverless
  - localstack
  - docker
---

## Testing

## Blueprints

## AWS Lambda Powertools 

## Datadog

## CI/CD

## Cookiecutter template
You can find my cookiecutter at ... that includes Datadog, AWS Lambda Powertools

## Local development with LocalStack

So far we have deployed our infrastructure against the cloud. This could be a slow process during development, even more if debugging a pesky error.

It is possible to deploy our infrastructure against [LocalStack](https://github.com/localstack/localstack), a project that aims to emulate AWS resources and API calls locally, using Docker as its backend.

The LocalStack team also provide [chalice-local](https://github.com/localstack/chalice-local), a tool that will be useful for checking logs and invoking our functions.

One more tool that we are going to use is [awscli-local](https://github.com/localstack/awscli-local), which is simply the AWS CLI with some configuration to run against LocalStack, instead of actual AWS servers.

We can install all these tools using `pip`:

```sh
pip install localstack chalice-local awscli-local
```

Follow https://github.com/localstack/localstack#running to get up and running with LocalStack.

These are the steps followed to run the sample app we have built so far locally:

1. Configure and start LocalStack:

```sh
export SERVICES=sqs,sns,ssm,sts,logs,iam,apigateway,lambda,events,kms,ec2,cloudwatch,s3
localstack start
```
2. Configure the AWS Terraform provider to point to LocalStack:

```terraform
provider "aws" {
  access_key                  = "mock_access_key"
  secret_key                  = "mock_secret_key"
  region                      = "us-east-1"
  s3_force_path_style         = true
  skip_credentials_validation = true
  skip_metadata_api_check     = true
  skip_requesting_account_id  = true
  insecure                    = true

  endpoints {
    apigateway       = "http://localhost:4566"
    cloudformation   = "http://localhost:4566"
    cloudwatch       = "http://localhost:4566"
    cloudwatchevents = "http://localhost:4566"
    cloudwatchlogs   = "http://localhost:4566"
    dynamodb         = "http://localhost:4566"
    ec2              = "http://localhost:4566"
    es               = "http://localhost:4566"
    firehose         = "http://localhost:4566"
    iam              = "http://localhost:4566"
    kinesis          = "http://localhost:4566"
    lambda           = "http://localhost:4566"
    route53          = "http://localhost:4566"
    redshift         = "http://localhost:4566"
    s3               = "http://localhost:4566"
    secretsmanager   = "http://localhost:4566"
    ses              = "http://localhost:4566"
    sns              = "http://localhost:4566"
    sqs              = "http://localhost:4566"
    ssm              = "http://localhost:4566"
    stepfunctions    = "http://localhost:4566"
    sts              = "http://localhost:4566"
  }
}

output "local_url" {
  value       = "http://localhost:4566/restapis/${aws_api_gateway_rest_api.rest_api.id}/local/_user_request_/"
  description = "API Gateway URL for LocalStack"
}
```

More info here:
https://registry.terraform.io/providers/hashicorp/aws/latest/docs/guides/custom-service-endpoints#localstack

3. Add a local stage to the Chalice config file

LocalStack isn't perfect. Sometimes their API fails to retrieve VPC data (among other errors). One way of getting around these errors is disabling VPC locally:

```json
{
  "version": "2.0",
  "app_name": "chalice-tf",
  "subnet_ids": "${data.aws_subnet_ids.public.ids}",
  "security_group_ids": ["${module.security_group_service.security_group_id}"],
  "stages": {
    "local": {
      "api_gateway_stage": "local",
      "subnet_ids": [],
      "security_group_ids": [],
    },
    "dev": {
      "api_gateway_stage": "dev"
    }
  }
}
```

Here we keep our Security group and Subnet configuration for all of our functions on all stages but local, where we manually disable them.

4. Package the app with the local stage configuration

```sh
chalice package --stage local --pkg-format terraform .
```

5. Apply the Terraform code against LocalStack

```sh
# Refreshing against LocalStack can be unstable. We don't really need it here so we can disable it
terraform apply -refresh=false
```

If everything goes right, you should have the same application you had in AWS up and running locally. You may have also noticed that applying against LocalStack is blazing fast compared to AWS.

Let's test our functions:

```sh
curl http://localhost:4566/restapis/ur5mwyfjy6/local/_user_request_/
{"hello":"world"}
```

Hit the local SNS and SQS server:

```sh
awslocal sns publish --topic-arn arn:aws:sns:us-east-1:000000000000:chalice-tf-topic --message "local SNS"
awslocal sqs send-message --queue-url http://localhost:4566/000000000000/chalice-tf-queue --message-body "local SQS"
```

See the logs of the local functions:

```sh
chalice-local logs --stage local --name handle_sns_message

2021-09-06 20:09:48.453000 57bc1a START RequestId: 08bf7fcb-a441-126b-ffcf-61aec76763fb Version: $LATEST
2021-09-06 20:09:48.457000 57bc1a
2021-09-06 20:09:48.461000 57bc1a Received message with subject: None, message: local SNS
2021-09-06 20:09:48.469000 57bc1a END RequestId: 08bf7fcb-a441-126b-ffcf-61aec76763fb
2021-09-06 20:09:48.473000 57bc1a
2021-09-06 20:09:48.481000 57bc1a REPORT RequestId: 08bf7fcb-a441-126b-ffcf-61aec76763fb	Init Duration: 923.26 ms	Duration: 4.44 ms	Billed Duration: 5 ms	Memory Size: 1536 MB	Max Memory Used: 46 MB
2021-09-06 20:09:48.485000 57bc1a


chalice-local logs --stage local --name handle_sqs_message

2021-09-06 20:10:02.453000 4c64e0 START RequestId: 08bf7fcb-a441-126b-ffcf-61aec76763fb Version: $LATEST
2021-09-06 20:10:02.457000 4c64e0
2021-09-06 20:10:02.461000 4c64e0 Received message with contents: local SQS
2021-09-06 20:10:02.469000 4c64e0 END RequestId: 08bf7fcb-a441-126b-ffcf-61aec76763fb
2021-09-06 20:10:02.473000 4c64e0
2021-09-06 20:10:02.481000 4c64e0 REPORT RequestId: 08bf7fcb-a441-126b-ffcf-61aec76763fb	Init Duration: 923.26 ms	Duration: 5.14 ms	Billed Duration: 6 ms	Memory Size: 1536 MB	Max Memory Used: 46 MB
2021-09-06 20:10:02.485000 4c64e0
```


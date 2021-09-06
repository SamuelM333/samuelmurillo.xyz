---
title: AWS Chalice + Terraform
date: 2021-07-27 08:33:25
tags:
  - aws
  - chalice
  - lambda
  - python
  - serverless
  - terraform
---


[AWS Chalice](https://aws.github.io/chalice/) is yet another Python serverless framework, like [Zappa](https://github.com/zappa/Zappa) and the [Serverless Framework](serverless.com) (what a confusing name).

What makes Chalice special is that it has [Terraform Support](https://aws.github.io/chalice/topics/tf), meaning that it is able to translate all its infrastructure to Terraform code, ready to be applied to AWS. This provides all the benefits of the Serverless Framework, like configure your Lambda triggers and setup API Gateway, without fragmenting your infrastructure in CloudFormation and Terraform.

Having just one Infrastructure-as-Code tool in your project provides simplicity, more control over your application, and being able to reference serverless values directly in your Terraform without having to use a middleware data storage, like SSM.

## How-to

Let's start by creating a sample Chalice project:

```sh
pyenv virtualenv chalice-tf
pyenv activate chalice-tf
pip install chalice
chalice new-project chalice-tf
cd chalice-tf
```

This creates a simple REST API with a hello world endpoint.

Now the fun part: let's package and export this simple application to Terraform:

```sh
chalice package --pkg-format terraform .
```

This does two main things:
1. Package your Python code and requirements into a zip, located at `deployment.zip`
2. Generate the Terraform code with all the infra required to run your app, located at `chalice.tf.json`

You will have to run this command every time you change your code, so make sure you add it to your CI/CD pipeline.

Now, let's test it:
```sh
terraform init && terraform plan
Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with the following symbols:
  + create
 <= read (data resources)

... (omit plan output)

Plan: 6 to add, 0 to change, 0 to destroy.
```

We see a few notable resources here:
`aws_api_gateway_deployment.rest_api` and `aws_api_gateway_rest_api.rest_api`:
API Gateway resources. This is one of the helpful parts of Chalice.

`aws_lambda_function.api_handler`:
The lambda function itself, with its code in a zip file.

`aws_iam_role.default-role` and `aws_iam_role_policy.default-role`:
The default IAM role, with a generated policy that allows accessing the resources created. This is helpful for a quick POC, but in a production environment you might want to customize the IAM policy yourself. More on this later.

Let's apply this code:
```
terraform apply
  ... (omit plan output)

Plan: 6 to add, 0 to change, 0 to destroy.

Changes to Outputs:
  + EndpointURL = (known after apply)
  + RestAPIId   = (known after apply)
```

## Add your own Terraform code

Let's create an SQS queue and an SNS topic so we can test those triggers as well.

Create a Terraform file with the following code:

```terraform
resource "aws_sqs_queue" "this" {
  name                       = "chalice-tf-queue"
  visibility_timeout_seconds = "120"
}

resource "aws_sns_topic" "this" {
  name = "chalice-tf-topic"
}

output "queue_url" {
    value = aws_sqs_queue.this.url
}

output "topic_arn" {
    value = aws_sns_topic.this.arn
}
```

Now open `app.py` and add these two functions:

```python
@app.on_sns_message(topic='chalice-tf-topic')
def handle_sns_message(event):
    print(f"Received message with subject: {event.subject,}, message: {event.message}")


@app.on_sqs_message(queue='chalice-tf-queue', batch_size=1)
def handle_sqs_message(event):
    for record in event:
        print(f"Received message with contents: {record.body}")
```

Package the new Chalice code:

`chalice package --pkg-format terraform .`

Apply the new terraform code:

With this, we've just created two new lambda functions, with SNS and SQS triggers.
Now publish messages to your newly created topic and queue:

`aws sns publish --topic-arn arn:aws:sns:us-east-1:123456789123:chalice-tf-topic --message "Hello from SNS!"`
`aws sqs send-message --queue-url https://sqs.us-east-1.amazonaws.com/123456789123/chalice-test-queue --message-body "Hello from SQS"`

Check the CloudWatch logs of your functions using Chalice itself:
`chalice logs --name handle_sns_message`
`chalice logs --name handle_sqs_message`

## Referencing Terraform values inside Chalice

At the time of writting this is not properly documented, but it is possible to reference Terraform values on the Chalice config file:

```json
{
  "version": "2.0",
  "app_name": "chalice-tf",
  "subnet_ids": "${data.aws_subnet_ids.public.ids}",
  "security_group_ids": ["${module.security_group_service.security_group_id}"],
}
```

More info:
https://aws.github.io/chalice/topics/configfile.html
https://github.com/aws/chalice/issues/1533

## Fixing the duplicated provider error

If you try to add your own AWS Terraform provider, you will run into the following error:

```
terraform init
╷
│ Error: Duplicate provider configuration
│
│   on providers.tf line 9:
│    9: provider "aws" {
│
│ A default (non-aliased) provider configuration for "aws" was already given at chalice.tf.json:122,12-13. If multiple configurations are required, set the "alias" argument for alternative configurations.
```

If we peek at the auto-generated Terraform code at `chalice.tf.json`, looking for the provider key, we see a simple AWS provider definition with just a version constraint:
```json
{
  ...
  "provider": {
    "template": {
      "version": "~> 2"
    },
    "aws": {
      "version": ">= 2, < 4"
    },
    "null": {
      "version": ">= 2, < 4"
    }
  }
  ...
}
```

To fix this, we have two options:
1. Add an alias like the error suggests. This is annoying to deal with, since we will have to add an alias to all of our Terraform resources
1. Remove the provider defined by Chalice. This gives us flexibility, since we are able to define our provider as we see fit. Here's a `jq` script that removes the provider:

```sh
cat <<< $(jq 'del(.provider.aws)' chalice.tf.json) > chalice.tf.json
```

## Local development with LocalStack

`pip install localstack`

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

https://registry.terraform.io/providers/hashicorp/aws/latest/docs/guides/custom-service-endpoints#localstack

```terraform
terraform {
  required_version = ">= 0.12"

  backend "local" {
    path = "terraform.localstack.tfstate"
  }
}

provider "aws" {
  access_key                  = "mock_access_key"
  secret_key                  = "mock_secret_key"
  region                      = var.region
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
  value = "http://localhost:4566/restapis/${aws_api_gateway_rest_api.rest_api.id}/local/_user_request_/"
}
```

`terraform apply -auto-approve -refresh=false`

`chalice-local logs --stage local --include-lambda-messages -n handle_sns_message`

## Handle bigger codebases

https://aws.github.io/chalice/topics/blueprints.html

## Cookiecutter template
<!-- You can find my cookiecutter at ... that includes Datadog, AWS Lambda Powertools -->

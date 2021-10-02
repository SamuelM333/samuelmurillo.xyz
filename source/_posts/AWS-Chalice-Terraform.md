---
title: "AWS Chalice + Terraform: A serverless codebase that makes sense"
date: 2021-07-27 08:33:25
description: Chalice is yet another Python serverless framework, like Zappa and the Serverless Framework. What makes Chalice special is the fact that it has Terraform Support
tags:
  - aws
  - chalice
  - lambda
  - python
  - serverless
  - sns
  - sqs
  - terraform
categories:
  - "AWS Chalice + Terraform"
---


[AWS Chalice](https://aws.github.io/chalice/) is yet another Python serverless framework, like [Zappa](https://github.com/zappa/Zappa) and the [Serverless Framework](serverless.com) (what a confusing name).

What makes Chalice special is the fact that it has [Terraform Support](https://aws.github.io/chalice/topics/tf), meaning that it is able to translate all of its infrastructure to Terraform code, ready to be applied to AWS. This provides all the benefits of the Serverless Framework, like configure your Lambda triggers and set up API Gateway, without fragmenting your infrastructure in CloudFormation and Terraform.

Having just one Infrastructure-as-Code tool in your project provides simplicity, more control over your application, and being able to reference serverless values directly in your Terraform without having to use a middleware data storage, like SSM.

Chalice also handles event subscriptions and HTTP routing elegantly using Python decorators defined in the code itself, instead of unnecessarily verbose YAML files.

## How-to

Let's start by creating a sample Chalice project:

```sh
# create a virtualenv
pyenv virtualenv chalice-tf
pyenv activate chalice-tf

# install chalice
pip install chalice

# set up a new chalice project
chalice new-project chalice-tf
cd chalice-tf
```

This creates a simple REST API with a hello world endpoint.

Now the fun part: let's package and export this simple application to Terraform:

```sh
chalice package --pkg-format terraform .
```

This does two main things:
1. Package your Python code and requirements into a zip file, located at `deployment.zip`
2. Generate the Terraform code with all the infra required to deploy your app, located at `chalice.tf.json`

You will have to run this command every time you change your code, so make sure you add it to your CI/CD pipeline.

Now, let's test it:
```sh
terraform init && terraform plan

Initializing the backend...
... (omit init output)

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

  ... (omit apply output)

Apply complete! Resources: 6 added, 0 changed, 0 destroyed.

Outputs:

EndpointURL = "https://ht0npswgm8.execute-api.us-east-1.amazonaws.com/api"
RestAPIId = "ht0npswgm8"
```

If we hit the endpoint url with `curl`:

```sh
curl https://ht0npswgm8.execute-api.us-east-1.amazonaws.com/api
{"hello":"world"}
```

This is a minimal example of AWS Chalice, but we can do better.

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
    print(f"Received message with subject: {event.subject}, message: {event.message}")


@app.on_sqs_message(queue='chalice-tf-queue', batch_size=1)
def handle_sqs_message(event):
    for record in event:
        print(f"Received message with contents: {record.body}")
```

Package the new Chalice code:

```sh
chalice package --pkg-format terraform .
```

Apply the new terraform code:

```sh
terraform apply
  ... (omit plan output)
Plan: 8 to add, 3 to change, 1 to destroy.

Changes to Outputs:
  ~ EndpointURL = "https://ht0npswgm8.execute-api.us-east-1.amazonaws.com/api" -> (known after apply)
  + queue_url   = (known after apply)
  + topic_arn   = (known after apply)

  ... (omit apply output)

Apply complete! Resources: 8 added, 2 changed, 1 destroyed.

Outputs:

EndpointURL = "https://ht0npswgm8.execute-api.us-east-1.amazonaws.com/api"
RestAPIId = "ht0npswgm8"
queue_url = "https://sqs.us-east-1.amazonaws.com/123456789123/chalice-tf-queue"
topic_arn = "arn:aws:sns:us-east-1:123456789123:chalice-tf-topic"
```

With this, we've just created two new lambda functions, with SNS and SQS triggers.

You can read more about Lambda Event Sources supported by Chalice here: https://aws.github.io/chalice/topics/events.html

Now publish messages to your newly created topic and queue:

```sh
aws sns publish --topic-arn arn:aws:sns:us-east-1:123456789123:chalice-tf-topic --message "Hello from SNS!"
aws sqs send-message --queue-url https://sqs.us-east-1.amazonaws.com/123456789123/chalice-test-queue --message-body "Hello from SQS"
```

Check the CloudWatch logs of your functions using Chalice itself:
```sh
chalice logs --name handle_sns_message
chalice logs --name handle_sqs_message
```
At the time of writing, this doesn't seem to work.
See https://github.com/aws/chalice/issues/1665

We can use the AWS CLI to get our CloudWatch logs:

```
LOG_GROUP_NAME="/aws/lambda/chalice-tf-dev-handle_sns_message"
LOG_STREAM_NAME=$(aws logs describe-log-streams --log-group-name "${LOG_GROUP_NAME}" | jq -r '.logStreams | sort_by(.creationTime) | .[-1].logStreamName')
aws logs get-log-events --log-group-name "${LOG_GROUP_NAME}" --log-stream-name "${LOG_STREAM_NAME}" | jq -r '.events[] | select(has("message")) | .message'

START RequestId: 0fea9a54-5d59-4e95-9422-b315e0393a51 Version: $LATEST

Received message with subject: (None,), message: Hello from SNS

END RequestId: 0fea9a54-5d59-4e95-9422-b315e0393a51

REPORT RequestId: 0fea9a54-5d59-4e95-9422-b315e0393a51	Duration: 1.18 ms	Billed Duration: 2 ms	Memory Size: 128 MB	Max Memory Used: 54 MB	Init Duration: 146.71 message
```

```
LOG_GROUP_NAME="/aws/lambda/chalice-tf-dev-handle_sqs_message"
LOG_STREAM_NAME=$(aws logs describe-log-streams --log-group-name "${LOG_GROUP_NAME}" | jq -r '.logStreams | sort_by(.creationTime) | .[-1].logStreamName')
aws logs get-log-events --log-group-name "${LOG_GROUP_NAME}" --log-stream-name "${LOG_STREAM_NAME}" | jq -r '.events[] | select(has("message")) | .message'

START RequestId: 8e5ab039-16cb-56e8-8b8b-e989affd04dd Version: $LATEST

Received message with contents: Hello from SQS

END RequestId: 8e5ab039-16cb-56e8-8b8b-e989affd04dd

REPORT RequestId: 8e5ab039-16cb-56e8-8b8b-e989affd04dd	Duration: 1.48 ms	Billed Duration: 2 ms	Memory Size: 128 MB	Max Memory Used: 54 MB	Init Duration: 160.67 ms
```

After we are all done testing, we can clean after ourselves by running:

```sh
terraform destroy
```

## Referencing Terraform values inside Chalice

Chalice can be configured using its own config file, located at `.chalice/config.json`. See https://aws.github.io/chalice/topics/configfile.html for more information about the available settings.

At the time of writing this is not properly documented, but it is possible to reference Terraform values on the Chalice config file, like this:

```json
{
  "version": "2.0",
  "app_name": "chalice-tf",
  "subnet_ids": "${data.aws_subnet_ids.public.ids}",
  "security_group_ids": ["${module.security_group_service.security_group_id}"],
}
```

As you can see, we are using Terraform syntax, since these keys as passed as literals to `chalice.tf.json` during packaging.

In this example, we are setting a Security group and Subnet to all of our functions. These IDs are retrieved using Terraform, without the need of a middleware data storage, like SSM.

More info here: https://github.com/aws/chalice/issues/1533

You could also set here the `iam_role_arn` of a pre-existing IAM role, instead of letting Chalice generate one for you. This is a good approach for production environments.
See https://aws.github.io/chalice/topics/configfile.html#iam-roles-and-policies for a practical example.

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

## What's next?

These are the basics you need to know to get started with Chalice and how to integrate it with Terraform. This post is the first part of a series about Chalice and how to run a solid and maintainable app. Check part 2, dedicated to local development and how to speed up the process of adding new features and bug squashing:

{% post_link AWS-Chalice-Terraform-2 %}

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

What makes Chalice special is that it has [Terraform Support](https://aws.github.io/chalice/topics/tf), meaning that it is able to translate all its infrastructure to Terraform code, ready to be applied to AWS. This provides has all the benefits of the Serverless Framework, like configure your Lambda triggers and setup API Gateway, without fragmenting your infrastructure in CloudFormation and Terraform.

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

This creates a simple REST API with a hello world endpoint

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
API Gateway resources. This is one of the helpful parts of Chalice

`aws_lambda_function.api_handler`:
The lambda function itself, with its code in a zip file

`aws_iam_role.default-role` and `aws_iam_role_policy.default-role`: 
The default IAM role, with a generated policy that allows accessing the resources created. This is helpful for a quick POC, but in a production environment you might want to customize the IAM policy yourself. More on this later.

## Add your own Terraform code

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

If we peek at the auto-generated Terraform code at `chalice.tf.json`, looking for the provider key, we see a simple AWS provider definition with just a version constraint.
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

To fix this, we have two options: add an alias like the error suggests, this brings issues, or remove the provider defined by Chalice. This gives us flexibility

```sh
cat <<< $(jq 'del(.provider.aws)' chalice.tf.json) > chalice.tf.json
```

## Referencing Terraform values inside Chalice

## Local development with LocalStack

## Cookiecutter template
You can find my cookiecutter at ... that includes Datadog, AWS Lambda Powertools

---
title: AWS Chalice + Terraform
date: 2021-07-27 08:33:25
tags:
- aws 
- chalice
- python
- serverless
- terraform
---

## Background

Terraform Support
https://aws.github.io/chalice/topics/tf

## How-to

```sh
chalice package --pkg-format terraform terraform
```

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

If we peek at the auto-generated Terraform code at `chalice.tf.json`, looking for the provider key, we see a simple AWS provider definition with a version constraint.
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

## Cookiecutter template
You can find my cookiecutter at ... that includes Datadog, AWS Lambda Powertools

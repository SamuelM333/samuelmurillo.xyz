---
title: Should you use AWS Chalice for your next project?
date: 2021-09-19 22:23:45
description:
tags:
  - aws
  - chalice
  - lambda
  - python
  - serverless
  - terraform
---

{% post_link AWS-Chalice-Terraform %}
{% post_link AWS-Chalice-Terraform-2 %}
{% post_link AWS-Chalice-Terraform-3 %}
{% post_link AWS-Chalice-Terraform-4 %}

After that deep dive on Chalice, let's actually consider if you should use it the next time you have a start-up idea or you get a loose enough Jira story.

Chalice is not the most popular serverless frameworks around, that's a given. When people think about serverless as an infrastructure concept, they'll most likely think about the Serverless Framework as well. Although Chalice can be consider a niche tool, it is backed up by AWS, and using robust tooling like the [AWS CDK](https://aws.amazon.com/cdk/). It is also implementing declarative infrastructure, similar to Terraform, making it very clean and maintainable. 

## Chalice is simple

[The Zen of Python](https://www.python.org/dev/peps/pep-0020/) said it best: Simple is better than complex. Chalice has less moving parts compared to the Serverless Framework, which runs on Node.js (a bloated and unreliable platform in my opinion) and depends on plugins, often unmaintained. The Serverless Framework also has a key disadvantage, being that in order to support multiple runtimes and languages, it has to implement convoluted config files for generic scenarios that are a nightmare to maintain and scale. Chalice is written on pure Python and focused on a concrete use case, making it less prone to break, easier to maintain and less scary to update.

## Familiarity

People familiar with Flask or FastAPI should find Chalice easy to work with, as it follows the well established pattern across Python microframeworks of using decorators to define routes. We also saw that local development is possible and not that terrible.

## Terraform support

## Cons

Sadly, Chalice is limited to Python projects, so if you are planning to write your next project on Go or JavaScript, look somewhere else. Another possible deal breaker for you is that Chalice is exclusive to AWS, without support for Azure or GCP.

## So? Should you?

If you are thinking about integrating Terraform with your next Python serverless project, with a simple and scalable codebase, you should consider using Chalice.

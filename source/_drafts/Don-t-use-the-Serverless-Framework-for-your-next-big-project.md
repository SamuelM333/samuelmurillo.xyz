---
title: Don't use the Serverless Framework for your next big project
tags:
  - aws
  - lambda
  - rant
  - serverless
---
For the past few years as a developer and as an SRE, I have worked with the [Serverless Framework](serverless.com) running on [AWS Lambda](https://aws.amazon.com/lambda/), integrated with [SQS](https://aws.amazon.com/sqs/), [SNS](https://aws.amazon.com/sns), [Kinesis](https://aws.amazon.com/kinesis) and [DynamoDB](https://aws.amazon.com/dynamodb).

With this experience, I have a few things to say about the Serverless Framework and why you shouldn't use it for big projects.

## YAML chaos

Let's start with the biggest pain point: Serverless requires writing a lot of YAML.

YAML sucks, but that's a whole different story. You can read a bit more on the subject here:
- [YAML sucks](https://github.com/cblp/yaml-sucks) by Yuriy Syrovetskiy 
- [Hate YAML!](https://kula.blog/posts/yaml/) by Park Sehun
- [Why YAML is used for configuration when it's so bad and what can you do about it?](https://parksehun.medium.com/why-you-hate-yaml-and-how-to-tackle-it-3e8471cf1ca8) by Kris Kula 

The average length of a `serverless.yml` file from a project from my team is around 300 lines, and that's with **a lot** of file import calls (`${file(./environment.yml)}` for example).

https://www.serverless.com/framework/docs/providers/aws/guide/serverless.yml

Let's see a few examples of YAML nonsense:

## Are you writing Serverless or CloudFormation code?

## Node.js

Node.js has high entropy. That's my nice way of saying that it breaks often.

## Plugin hell

## Dead community

## Is all hope lost?

{% post_link AWS-Chalice-Terraform %}

{#
You can also check how a pure Terraform implementation of a serverless stack looks like
{% post_link serverless-tf %}
#}

https://knative.dev/
https://fission.io/

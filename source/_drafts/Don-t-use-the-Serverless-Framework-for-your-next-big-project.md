---
title: Don't use the Serverless Framework for your next big project
tags:
  - aws
  - lambda
  - rant
  - serverless
---
{# 
- The Serverless Framework, Serverless (capitalized), serverless.com: The actual [Serverless Framework](serverless.com)
- serverless (lowercase), serverless infrastructure: The serverless paradigm, not the framework    
#}

For the past few years as a developer and as an SRE, I have worked with the [Serverless Framework](serverless.com) running on [AWS Lambda](https://aws.amazon.com/lambda/), integrated with [SQS](https://aws.amazon.com/sqs/), [SNS](https://aws.amazon.com/sns), [Kinesis](https://aws.amazon.com/kinesis) and [DynamoDB](https://aws.amazon.com/dynamodb).

With this experience, I have a few things to say about the Serverless Framework and why you shouldn't use it for big projects.

## YAML chaos

Let's start with the biggest pain point: Serverless requires writing a lot of YAML.

YAML sucks, but that's a whole different story. You can read a bit more on the subject here:
- [YAML sucks](https://github.com/cblp/yaml-sucks) by Yuriy Syrovetskiy 
- [Hate YAML!](https://kula.blog/posts/yaml/) by Park Sehun
- [Why YAML is used for configuration when it's so bad and what can you do about it?](https://parksehun.medium.com/why-you-hate-yaml-and-how-to-tackle-it-3e8471cf1ca8) by Kris Kula 
- [YAML - Anchors, References, Extend](https://blog.daemonl.com/2016/02/yaml.html) by daemonl
  - This post has a more positive note on YAML, even calling it "pretty cool", but I'm sharing it anyways to showcase the mess that anchors in YAML are.

The average length of a `serverless.yml` file from a project from my team is around 300 lines, and that's with **a lot** of file import calls (`${file(./environment.yml)}` for example).

This is the `serverless.yml` reference for the AWS provider: https://www.serverless.com/framework/docs/providers/aws/guide/serverless.yml

**It is over 600 lines long**. Yes, you are not going to use all of the features described on the reference, as it is supposed to be a complete example. But my point is that it has a lot to configure at just a single file. You will probably also call some keys multiple times, making the file even longer.

{# Let's see a few examples of YAML nonsense: #}

{# ## Defaults: the root of all evil #}

## Are you writing Serverless or CloudFormation code?

To continue on the topic of YAML, let's talk about how the Serverless Framework handles non-serverless infrastructure.

[CloudFormation](https://aws.amazon.com/cloudformation/) is an Infrastructure-as-Code tool provided by AWS, that enables you create any infrastructure resource the AWS CLI allows. But here's the thing: It is also configured on YAML! The Serverless Framework developers saw this opportunity to ~make things even worse~ integrate Serverless with 

CloudFormation is the underlying technology that Serverless uses to deploy its resources

## Node.js

Node.js has high entropy. That's my nice way of saying that it breaks often.

## Plugin hell

KISS

https://github.com/sbstjn/serverless-stack-output

https://wb.serverless.com/plugins/serverless-offline
https://www.npmjs.com/package/serverless-offline-sqs
https://www.npmjs.com/package/serverless-offline-sns
https://serverless.com/plugins/serverless-localstack/

## Dead community

https://github.com/CoorpAcademy/serverless-plugins/pull/166

## Poor development experience

## Is all hope lost?

{% post_link AWS-Chalice-Terraform %}

{#
You can also check how a pure Terraform implementation of a serverless stack looks like
{% post_link serverless-tf %}
#}

https://knative.dev/
https://fission.io/

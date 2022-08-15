---
title: 'End-to-end distributed tracing for AWS Lambda, SNS and SQS using Datadog'
date: 2021-06-04 08:33:25
description: ''
draft: true
tags:
  - aws
  - datadog
  - lambda
  - python
  - serverless
---


## Background

According to Datadog’s Serverless distributed tracing documentation:

{% blockquote Datadog https://docs.datadoghq.com/serverless/distributed_tracing/ Distributed Tracing %}
Tracing non-HTTP requests made through SNS, Kinesis, EventBridge, MQTT and more (requires additional instrumention outlined [here](https://docs.datadoghq.com/serverless/distributed_tracing/serverless_trace_propagation/?tab=python)).
{% endblockquote %}


Following that documentation, we learned that Datadog is not capable of propagating trace IDs through SNS automatically, and requires extra instrumentation, documented here: 

One interesting challenge of this additional instrumentation is that most of the SNS topic subscribers are not Lambda functions directly, but rather SQS queues that then trigger functions. This means that the documentation provided by Datadog doesn’t work out of the box for us, and required extra investigation. This investigation pointed us in the direction of changing the way SQS subscribed to SNS topics, specifically enabling RawMessageDelivery 

After this was enabled, we started to see end to end tracing in Datadog (Lambda → SNS → SQS → Lambda)

## How-to

We encapsulated Datadog’s trace ID injecting logic on a Python decorator. Use this decorator to easily add the trace ID to a SNS publish call:

```python
"""
Code inspired on Datadog documentation. Available at:
https://docs.datadoghq.com/serverless/distributed_tracing/serverless_trace_propagation/?tab=sns#passing-trace-context-to-outgoing-events
"""
import json
from functools import wraps

from datadog_lambda.tracing import get_dd_trace_context

def inject_trace_id_to_sns_topic(func):
    """Includes trace context in outgoing message to SNS topic."""
    @wraps(func)
    def wrapper(*args, **kwargs):
        message_attributes = kwargs.get("message_attributes", {})
        message_attributes["_datadog"] = {
            'DataType': 'String',
            'StringValue': json.dumps(get_dd_trace_context())
        }
        kwargs["message_attributes"] = message_attributes
        return func(*args, **kwargs)
    return wrapper
```

Then, on the subscribing side, enable `RawMessageDelivery`:

```yaml
  QueueSNSSubscription:
    Type: AWS::SNS::Subscription
    Properties:
      Endpoint:
        Fn::GetAtt: [, Arn]
      Protocol: sqs
      RawMessageDelivery: true  # Here!
      Region: ${self:provider.region}
      TopicArn:
        Fn::Join:
          - ""
          - - "arn:aws:sns:"
            - Ref: AWS::Region
            - ":"
            - Ref: AWS::AccountId
            - ":"
            - ${self:custom.}
```

Enabling this requires a code change as the payload format changes. Refer to 

If everything was set up correctly, you should see end-to-end tracing when publishing to a topic:

SNS publish call, with the topic ARN in the metadata
![](1.png)

Lambda function triggered by SNS -> SQS event, with the SQS queue ARN in the metadata
![](2.png)

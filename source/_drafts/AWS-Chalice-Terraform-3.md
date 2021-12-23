---
title: "AWS Chalice + Terraform Part 3: Testing your app"
date: 2021-09-09 12:33:25
tags:
  - aws
  - chalice
  - docker
  - lambda
  - localstack
  - python
  - serverless
  - sns
  - sqs
  - testing
categories:
  - "AWS Chalice + Terraform"
---

This is the third part of the Chalice + Terraform series. You can check part 1 and 2 here:

{% post_link AWS-Chalice-Terraform %}
{% post_link AWS-Chalice-Terraform-2 %}

This part is focused on testing our application, focusing on unit testing and end-to-end testing. Let's get started! 

## Writing unit tests

We will follow [Chalice's testing guide](https://aws.github.io/chalice/topics/testing.html) for the basics, then add additional tests on top of it.

First, let's install `pytest`, our test runner of choice:

```sh
pip install pytest
```

You can add these to `requirements-ci.txt` for CI/CD tooling purposes:
```plain requirements-ci.txt
-r requirements.txt
pytest
```

Then, let's create our test folders and the files we will need:

```sh
mkdir -p tests/unit/
touch tests/unit/{__init__.py,test_app.py}
```

As you can see, we created a `unit` subfolder within `tests`, that way we can separate our end-to-end tests later on.

### The Chalice test client

Chalice has a built-in test suite, located at [`chalice.test.Client`](https://aws.github.io/chalice/api#Client), capable of calling the lambda functions created within the project in multiple ways:

- Bare lambdas: [https://aws.github.io/chalice/api#TestLambdaClient](TestLambdaClient)
- HTTP routes: [https://aws.github.io/chalice/api#TestHTTPClient](TestHTTPClient)
- Events: [https://aws.github.io/chalice/api#TestEventsClient](TestEventsClient)

To use the configuration of a specific stage during tests, use:

```python
from chalice.test import Client

def test_foo_function():
    with Client(app, stage_name='production') as client:
        result = client.lambda_.invoke('foo')
        assert result.payload == {'value': 'TOP LEVEL'}
```

See https://aws.github.io/chalice/topics/testing.html#environment-variables for more

### Testing REST functions

The following test checks the status code and payload of a REST enabled lambda:

```python tests/unit/test_app.py
from chalice.test import Client

from app import app

def test_index_endpoint():
    with Client(app) as client:
        response = client.http.get('/')
        assert response.status_code == 200
        assert response.json_body == {'hello': 'world'}
```

### Test a bare lambda function

In case you have a simple lambda without triggers, you can test it using `client.lambda_.invoke`:

```python app.py
@app.lambda_function()
def bar(event, context):
    return {'event': event}
```

```python tests/unit/test_app.py
def test_foo_function():
    with Client(app) as client:
        result = client.lambda_.invoke('bar', {'my': 'event'})
        assert result.payload == {'event': {'my': 'event'}}
```

Note: If you run into an error like the following:
```
FAILED tests/unit/test_app.py::test_index_function - TypeError: index() takes 0 positional arguments but 2 were given
```

It could mean that you are trying to test a function triggered by `@app.route` using `client.lambda_.invoke`. This is reserved to functions declared using `@app.lambda_function()`. Use `client.http` instead.

### Test SNS and SQS lambdas

Let's test the [SNS and SQS functions created on part 1 of the series](/2021/07/27/AWS-Chalice-Terraform/#Add-your-own-Terraform-code), like so:


```python tests/unit/test_app.py
def test_sns_handler():
    with Client(app) as client:
        response = client.lambda_.invoke(
            "handle_sns_message",
            client.events.generate_sns_event(subject="test", message="hello from sns")
        )
        assert response.payload == {'subject':'test', 'message': 'hello from sns'}


def test_sqs_handler():
    with Client(app) as client:
        response = client.lambda_.invoke(
            "handle_sqs_message",
            client.events.generate_sqs_event(message_bodies=["hello from sqs"])
        )
        assert response.payload == {'message': 'hello from sqs'}
```

### See also

Check [Chalice's documentation on testing](https://aws.github.io/chalice/topics/testing.html) for more topics about testing, like mocking `boto3` and using Pytest fixtures.

## Writing integration tests with LocalStack

Let's go one step further: using LocalStack, we can trigger our functions like we would in a real environment, then check that the function actually ran. If you need a refresher on how to set up your Chalice project to run against LocalStack, check [Part 2 of the series](/2021/10/02/AWS-Chalice-Terraform-2/).

Let's start by creating a separate folder for our integration tests:

```sh
mkdir tests/integration
touch tests/integration/{__init__.py,test_app.py}
```

### Testing strategy

AWS Lambda doesn't provide a straight-forward way of checking that a Lambda has ran, so we need to automatize the process you would normally do with the AWS Console: trigger your function, then going to CloudWatch to see if there are any execution logs. In this case, we will send a random UUID and then check that same payload was logged in our output, but feel free to adjust the assertion for your use case.

### boto3 clients for LocalStack

We will be manipulating AWS resources created locally in LocalStack, so we need to tell `boto3` that we are not hitting AWS servers but our own instead. We also need to mock AWS access keys:

```python tests/integration/test_app.py
import boto3

localstack_url = "http://localhost:4566"

# Common kwargs for boto3 client init
boto3_kwargs = {
    "region_name": "us-east-1",
    "aws_access_key_id": "aws_access_key_id",
    "aws_secret_access_key": "aws_secret_access_key",
    "endpoint_url": localstack_url
}

sns = boto3.client('sns', **boto3_kwargs)
sqs = boto3.client('sqs', **boto3_kwargs)
logs = boto3.client('logs', **boto3_kwargs)
```

We will be using the SNS and SQS clients to publish content to our topics and queues respectively, and the CloudWatch logs client to check the content of our log groups.

### Triggering our Lambdas within tests

We are all set to write our integration tests. This is how we would do it:

```python tests/integration/test_app.py
import json
from time import sleep
from uuid import uuid4

import boto3

localstack_url = "http://localhost:4566"

# We can get these from the Terraform output
topic_arn = "arn:aws:sns:us-east-1:000000000000:chalice-tf-topic"
queue_url = "http://localhost:4566/000000000000/chalice-tf-queue"

# Common kwargs for boto3 client init
boto3_kwargs = {
    "region_name": "us-east-1",
    "aws_access_key_id": "aws_access_key_id",
    "aws_secret_access_key": "aws_secret_access_key",
    "endpoint_url": localstack_url
}

sns = boto3.client('sns', **boto3_kwargs)
sqs = boto3.client('sqs', **boto3_kwargs)
logs = boto3.client('logs', **boto3_kwargs)


def get_log_events_from_log_group(log_group_name):
    """This helper function returns the latest event of the specified log group"""

    response = logs.describe_log_streams(
        logGroupName=log_group_name,
        orderBy='LastEventTime',
        descending=True,
    )

    log_stream_name = response["logStreams"][0]["logStreamName"]

    response = logs.get_log_events(
        logGroupName=log_group_name,
        logStreamName=log_stream_name,
    )

    return response


def test_sns_execution():
    _id = str(uuid4())  # Generate an unique ID to assert the execution of the function
    response = sns.publish(
        TargetArn=topic_arn,
        Message=_id,
        MessageStructure='json'
    )
    sleep(3)  # Wait for LocalStack to execute the function and log to CloudWatch

    response = get_log_events_from_log_group("/aws/lambda/chalice-tf-local-handle_sns_message")
    log_message = '\n'.join(event["message"] for event in response["events"])  # Join all log events into one string

    assert _id in log_message  # Look for our previously generated unique payload in the execution logs


def test_sqs_execution():
    _id = str(uuid4())  # Generate an unique ID to assert the execution of the function
    sqs.send_message(
        QueueUrl=queue_url,
        MessageBody=_id,
    )
    sleep(3)  # Wait for LocalStack to execute the function and log to CloudWatch

    response = get_log_events_from_log_group("/aws/lambda/chalice-tf-local-handle_sqs_message")
    log_message = '\n'.join(event["message"] for event in response["events"])  # Join all log events into one string

    assert _id in log_message  # Look for our previously generated unique payload in the execution logs
```

Note: These test expect that LocalStack is up and running, and that the Terraform infrastructure has been applied.

### See also

https://github.com/localstack/localstack-python-client for a boto3 client pre-configured to run against LocalStack

https://github.com/beelit94/python-terraform
See [A journey to AWS Lambda integration testing with Python, Localstack, and Terraform](https://medium.com/craftsmenltd/a-journey-to-aws-lambda-integration-testing-with-python-localstack-and-terraform-2f17043c7dda) by [@melon.ruet](https://medium.com/@melon.ruet) for reference on how to init, apply and destroy Terraform resources using plain Python and the [`subprocess`](https://docs.python.org/3/library/subprocess.html) library.

If you want to test the infrastructure itself, check out [Terratest](https://terratest.gruntwork.io).

## What's next

Up next, let's get ready for production release, with tips for bigger codebases, logging, tracing, testing and more. You can check the fourth part of this series here:

{% post_link AWS-Chalice-Terraform-4 %}

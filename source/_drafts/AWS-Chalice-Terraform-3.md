---
title: "AWS Chalice + Terraform Part 3: Testing your app"
date: 2021-09-09 12:33:25
tags:
  - aws
  - chalice
  - lambda
  - python
  - serverless
  - testing
  - localstack
  - docker
categories:
  - "AWS Chalice + Terraform"
---

This is the third part of the Chalice + Terraform series. You can check part 1 and 2 here:

{% post_link AWS-Chalice-Terraform %}
{% post_link AWS-Chalice-Terraform-2 %}


## Writing unit tests

We can go a step further and write unit tests

```plain requirements-ci.txt
-r requirements.txt
pytest
```

We will follow [Chalice's testing guide](https://aws.github.io/chalice/topics/testing.html) for the basics, then add additional tests on top of it.

```
mkdir tests
touch tests/{__init__.py,test_app.py}
```

### The Chalice test client

...
https://aws.github.io/chalice/api#Client
- Bare lambdas: https://aws.github.io/chalice/api#TestLambdaClient
- HTTP: https://aws.github.io/chalice/api#TestHTTPClient
- Events: https://aws.github.io/chalice/api#TestEventsClient

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

```python tests/test_app.py
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

```python tests/test_app.py
def test_foo_function():
    with Client(app) as client:
        result = client.lambda_.invoke('bar', {'my': 'event'})
        assert result.payload == {'event': {'my': 'event'}}
```

Note: If you run into an error like the following:
```
FAILED tests/test_app.py::test_index_function - TypeError: index() takes 0 positional arguments but 2 were given
```

It could mean that you are trying to test a function triggered by `@app.route` using `client.lambda_.invoke`. This is reserved to functions declared using `@app.lambda_function()`. Use `client.http` instead.

### Test SNS and SQS lambdas

Let's test the [SNS and SQS functions created on part 1 of the series](/2021/07/27/AWS-Chalice-Terraform/#Add-your-own-Terraform-code), like so:


```python tests/test_app.py
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

https://github.com/beelit94/python-terraform
https://github.com/localstack/localstack-python-client

Note that both of these packages are entirely optional and may be replaced easily with our custom code.

See [A journey to AWS Lambda integration testing with Python, Localstack, and Terraform](https://medium.com/craftsmenltd/a-journey-to-aws-lambda-integration-testing-with-python-localstack-and-terraform-2f17043c7dda) by [@melon.ruet](https://medium.com/@melon.ruet) for reference on how to init, apply and destroy Terraform resources using plain Python and the [`subprocess`](https://docs.python.org/3/library/subprocess.html) library.

localstack-python-client Boto3

```python
endpoint_url = "http://localhost:4566"
client = boto3.client("lambda", endpoint_url=endpoint_url)
```

```plain requirements-ci.txt
-r requirements.txt
pytest
python-terraform
localstack
localstack-client
```

**Don't forget to remove `localstack` from `requirements-dev.txt`**

If you want to test the infrastructure itself, you can check out [Terratest](https://terratest.gruntwork.io).

## What's next

Up next, let's get ready for production release, with tips for bigger codebases, logging, tracing, testing and more. You can check the fourth part of this series here:

{% post_link AWS-Chalice-Terraform-4 %}

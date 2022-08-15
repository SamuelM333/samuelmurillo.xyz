---
title: "AWS Chalice + Terraform Part 4: Best practices"
date: 2021-12-29 12:33:25
description: ''
draft: true
tags:
  - aws
  - chalice
  - lambda
  - python
  - serverless
  - datadog
categories:
  - "AWS Chalice + Terraform"
---

This is the fourth part of the Chalice + Terraform series. You can check previous parts here:

{% post_link AWS-Chalice-Terraform %}
{% post_link AWS-Chalice-Terraform-2 %}
{% post_link AWS-Chalice-Terraform-3 %}

This time we will focus on getting production ready by following best practices. That's a very broad objective, so we will narrow it down to:
1. Clean and maintainable codebase
1. Improve observability by introducing(?) logging and tracing
1. Secrets management
1. Continuos integration and delivery (CI/CD)

## Handle bigger codebases using blueprints

Chalice has the concept of blueprints, similar to [Flask's blueprints](https://exploreflask.com/en/latest/blueprints.html). It is a handy way of organizing more complex projects with multiple files and namespaces.

Read more about blueprints here https://aws.github.io/chalice/topics/blueprints.html

Keep in mind that Chalice has the requirement of including custom files inside the `chalicelib` folder. More info here: https://aws.github.io/chalice/topics/packaging

With this knowledge, let's expand our project using blueprints:

```sh
mkdir chalicelib
touch chalicelib/__init__.py
touch chalicelib/api.py
touch chalicelib/events.py
```

Now, let's add HTTP events to `chalicelib/api.py`

```python chalicelib/api.py
from chalice import Blueprint

extra_routes = Blueprint(__name__)


@extra_routes.route('/foo')
def foo():
    return {'foo': 'bar'}
```

Add some subscription events to `chalicelib/events.py`

```python chalicelib/events.py
from chalice import Blueprint

extra_events = Blueprint(__name__)

@extra_events.on_sns_message(topic='chalice-tf-topic')
def handle_sns_message_blueprint(event):
    print(f"Received message with subject: {event.subject}, message: {event.message} from blueprint")
```

To easily register all blueprints, let's create a set with all of them inside `__init__.py`

```python chalicelib/__init__.py
from chalicelib.api import extra_routes
from chalicelib.events import extra_events

BLUEPRINTS = (
    extra_routes,
    extra_events,
)
```

Lastly, let's elegantly register our blueprints on `app.py`:

```python app.py
from chalice import Chalice

from chalicelib import BLUEPRINTS

app = Chalice(app_name='chalice-tf')

for blueprint in BLUEPRINTS: app.register_blueprint(blueprint)
```

Read more about blueprint registration here: https://aws.github.io/chalice/topics/blueprints.html#blueprint-registration

### Test functions inside a blueprint

In case you are wondering, this is how you write tests for functions declared on a blueprint:

```python tests/unit/test_blueprint.py
from chalice.test import Client

from chalicelib.api import extra_routes

def test_foo_function():
    with Client(extra_routes) as client:
        result = client.lambda_.invoke('bar', {'my': 'event'})
        assert result.payload == {'event': {'my': 'event'}}
```

## Observability

### AWS Lambda Powertools

https://awslabs.github.io/aws-lambda-powertools-python/
https://aws.github.io/chalice/topics/middleware.html#integrating-with-aws-lambda-powertools

```python app.py
from aws_lambda_powertools import Logger
from aws_lambda_powertools import Tracer
from chalice import Chalice, ConvertToMiddleware

app = Chalice(app_name='chalice-test')

# AWS powertools
# https://aws.github.io/chalice/topics/middleware.html?highlight=handler#integrating-with-aws-lambda-powertools
# https://aws.amazon.com/blogs/developer/following-serverless-best-practices-with-aws-chalice-and-lambda-powertools/
logger = Logger(service=app.app_name)
tracer = Tracer(service=app.app_name)

app.register_middleware(ConvertToMiddleware(logger.inject_lambda_context))
app.register_middleware(ConvertToMiddleware(tracer.capture_lambda_handler(capture_response=False)))

@app.middleware('http')
def inject_route_info(event, get_response):
    logger.structure_logs(append=True, request_path=event.path)
    return get_response(event)
```

### Datadog

## Secrets management

### Hashicorp Vault

### AWS Secrets manager

## CI/CD

### Deployment

### Run your tests

## See also

- [Parsers](https://awslabs.github.io/aws-lambda-powertools-python/latest/utilities/parser/)

## Cookiecutter template

You can find my cookiecutter at ... that includes Datadog, AWS Lambda Powertools
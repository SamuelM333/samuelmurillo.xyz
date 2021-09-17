---
title: "AWS Chalice + Terraform Part 3: Getting to production"
date: 2021-09-09 12:33:25
tags:
  - aws
  - chalice
  - lambda
  - python
  - serverless
  - datadog
---

This is the third part of the Chalice + Terraform series. You can check part 1 and 2 here:

{% post_link AWS-Chalice-Terraform %}
{% post_link AWS-Chalice-Terraform-2 %}

This time we will focus on getting production ready. That's a very broad objective, so we will narrow it down to:
- Clean and maintainable codebase
- Secrets management
- Observability
- Testing
- Continuos integration and delivery (CI/CD)

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

```python
from chalice import Blueprint

extra_routes = Blueprint(__name__)


@extra_routes.route('/foo')
def foo():
    return {'foo': 'bar'}
```

Add some subscription events to `chalicelib/events.py`

```python
from chalice import Blueprint

extra_events = Blueprint(__name__)

@extra_events.on_sns_message(topic='chalice-tf-topic')
def handle_sns_message_blueprint(event):
    print(f"Received message with subject: {event.subject}, message: {event.message} from blueprint")
```

To easily register all blueprints, let's create a set with all of them inside `__init__.py`

```python
from chalicelib.api import extra_routes
from chalicelib.events import extra_events

BLUEPRINTS = (
    extra_routes,
    extra_events,
)
```

Lastly, let's elegantly register our blueprints on `app.py`:

```python
from chalice import Chalice

from chalicelib import BLUEPRINTS

app = Chalice(app_name='chalice-tf')

for blueprint in BLUEPRINTS: app.register_blueprint(blueprint)
```

Read more about blueprint registration here: https://aws.github.io/chalice/topics/blueprints.html#blueprint-registration

## AWS Lambda Powertools

### Logging and tracing

https://awslabs.github.io/aws-lambda-powertools-python/latest/#features

```python
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

### See also

- [Parsers](https://awslabs.github.io/aws-lambda-powertools-python/latest/utilities/parser/)

## Datadog

## Testing

## CI/CD

## Cookiecutter template

You can find my cookiecutter at ... that includes Datadog, AWS Lambda Powertools
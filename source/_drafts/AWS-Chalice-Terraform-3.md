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

## Testing

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

Lastly, let's register these blueprints on `app.py`:

```python
from chalicelib.blueprints import extra_events, extra_routes

app.register_blueprint(extra_events)
app.register_blueprint(extra_routes)
```

## AWS Lambda Powertools 

## Datadog

## CI/CD

## Cookiecutter template

You can find my cookiecutter at ... that includes Datadog, AWS Lambda Powertools
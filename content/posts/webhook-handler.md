---
title: "Webhook handlers via decorators"
date: 2021-05-27T00:03:38+03:00
draft: true
---

### Problem
You have an external system(e.g payment gateway) sending events onto your backend http endpoint.
There are several types of events have to be processed in different ways.

![test](images/webhook_handlers/external_system.png)

How to organize the code?

### Possible solution
Move processing of particular types into separate functions and register them via decorators.


Create a decorator that consumes parameter(e.g event) and stores a function in the dict or other container.

```python
from functools import wraps

event_handlers = dict()  # Dict  to store event -> func mappings


def handler(event):

    def decorator(func):
        event_handlers[event] = func

        @wraps(func)
        def wrapper(data):

            result = func(data)

            return result

        return wrapper

    return decorator

```

Write functions responsible for processing specific events and hang decorators on them.
Write a function getting event handlers functions and call the function.

```python


@handler(event=PayPalEvent.PAYMENT_SALE_COMPLETED)
def activate_subscription(data):
    resource = data['resource']
    ...
    # Subscription activation logic
    pass


@handler(event=PayPalEvent.CUSTOMER_DISPUTE_CREATED)
def create_dispute(data):
    resource = data['resource']
    ...
    # Create dispute logic
    pass

def process_not_handled(data):
    write_logs(...)
    ...
    # Processing not handled event logic

def process(event: str, data: dict) -> None:
    func = event_handlers.get(event, process_not_handled)
    func(data)

```


Parse event in request and call processing function.

```python

class Webhook(APIView):
    permission_classes = [permissions.AllowAny]

    def post(self, request):
        validate_request(request)
        event = request.data.get('event_type')
        handlers.process(event=event, data=request.data)

        return Response(status=status.HTTP_200_OK)
```

### Additional links
Open source in-library use example: [dj-stripe](https://github.com/dj-stripe/dj-stripe/blob/master/djstripe/event_handlers.py)


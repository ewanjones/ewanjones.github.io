---
layout: post
title:  "Implementing an event driven architecture"
date:   2020-03-04 20:32:00 +0000
categories: architecture
---

## Why should I use event sourcing?

Event sourcing is an excellent technique for businesses processes where events occur and the system should react in some way. The idea behind this architecture is that we record events and subscribe actions to them.


## Processes and Events


We can model a process by storing any events which happen to a process object. Any data we think is important to the event and the rest of the process (for tracking and monitoring) can be stored in a json format or similar. Let's take an order tracking system:

```python
class Order:
    id = 12345
    customer_name = "Ewan"
    item_ids = [12. 34, 55]

    created_at = datetime.datetime.now()


class OrderEvent:
    order_id = 12345
    event_type: OrderEventType = OrderEventType.
    data = {}

    occurred_at = datetime.datetime.now()

```

Event types should be as granular as possible, recording when things start and finish. They provide a detailed audit of everything that happens, so if something goes wrong, you want to know exactly where to find the point of failure, e.g. between an order and the packaging actually starting. Naming is important, and should capture exactly what that event was. We use a convention of always keeping them in the past tense and include a verb.

```python
class OrderEventType(Enum):
    CUSTOMER_REQUESTED = "CUSTOMER_REQUESTED"
    PAYMENT_RECEIVED = "CUSTOMER_REQUESTED"
    PAYMENT_SUCCEEDED = "PAYMENT_SUCCEEDED"
    PAYMENT_FAILED = "PAYMENT_FAILED"
    WAREHOUSE_RECEIVED = "WAREHOUSE_RECEIVED"
    PACKAGING_STARTED = "PACKAGING_STARTED"
    ORDER_PACKAGED = "ORDER_PACKAGED"
    COURIER_COLLECTED = "COURIER_COLLECTED"
    PACKAGE_DELIVERED = "PACKAGE_DELIVERED"
```

This format allows you to not only easily track one process through it's events, you can aggregate events easily for getting whole system reports.


## Commands


You can create a top level module for all `commands` that will be triggered from outside (interfaces). The purpose of commands is to validate the request, fetch the process and record the event happening.

The only errors that should occur are if there is a legitimate problem. This can be caught in the interface layer (a REST API) and returned to the user. Remember, any unexpected errors will mean the event won't be recorded, so it's best to keep this module free of any complex logic.


## Actions

Actions are things which are triggered when we create an event. These functions should only do one thing. This means that if something goes wrong whilst running that action, it won't affect the other actions which need to happen. We also wrap the action call in a try/except in order to catch the error message and attach to the event.

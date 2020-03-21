---
layout: post
title:  "Implementing an event driven architecture"
date:   2020-03-04 20:32:00 +0000
categories: architecture
---

At work (Octopus Energy), we recently implemented an event-sourced architecture for industry market messaging system, involving conversations with various industry participants through file transfer. Each file we send or receive contains data about your meter and consumption and marks a new event in the process.

Our previous solution involved handlers which recieve the file and update the system synchronously. The trouble is that if a handler encountered an error the system  would get stuck. Any further events coming in also failed, because they rely on data provided by those previous files. Debugging this often involved trawling through the code to work out where it failed and then reprocess each file in turn.

With event sourcing, our main aim is to record the event happening and all the data required to give us information about the switch. No business logic or trying to respond at all. We subscribe actions to the events and then try running them asynchronously. Any errors are caught and stored with the event, so we can reprocess them after fixing the bug. This method also ensures idempotency, as once the event is recorded we won't add the actions again unless we're force adding them.

To give you a taster of the pros and cons, let's go through a simple example of an order tracking system.


## Processes and Events


We can model a process by storing any events which happen. Any data we think is important to the event and the rest of the process (for tracking and monitoring) can be stored in a json format or similar. The models can be stored in any way - we use postgres but object storage, like mongodb,  may suit this kind of architecture beter.

```python
class Order:
    id = 12345
    customer_name = "Ewan"
    item_ids = [12. 34, 55]
    delivery_date = datetime.date(2020, 3, 1)
    events = []

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
    ORDER_SUCCESS_EMAIL_GENERATED = "ORDER_SUCCESS_EMAIL_GENERATED"
    ORDER_SUCCESS_EMAIL_SENT = "ORDER_SUCCESS_EMAIL_SENT"
    PAYMENT_FAILED = "PAYMENT_FAILED"
    WAREHOUSE_RECEIVED = "WAREHOUSE_RECEIVED"
    PACKAGING_STARTED = "PACKAGING_STARTED"
    ORDER_PACKAGED = "ORDER_PACKAGED"
    COURIER_COLLECTED = "COURIER_COLLECTED"
    PACKAGE_DELIVERED = "PACKAGE_DELIVERED"
```


I use the built-in python enum package here, which helps with type checking and validation. Recording events in this format allows you to not only easily track one process through it's events, you can aggregate events easily for creating system-wide reports, although this is easier through a relational database.


## Commands


You can create a top level module for all `commands` that will be triggered from outside interfaces. The purpose of commands is to validate the request, fetch the process and record the event happening. They're like the entrypoint to your application.


```python
def handle_payment_received(customer_name, payment_id):
    process = data.Order.get(customer_name=customer_name)
    if not process:
        raise OrderError(f"Cannot find order for customer {customer_name}")
    process.record_payment_received(payment_id)

```

Commands can be triggered by a user doing something in your application or can be triggered by external event (e.g. webhooks). The naming convention we use is `handle_<something_happened>` for external events and use verbs for direct actions the user has requested to take, e.g. `do_something`

The only errors that should occur are if there is a legitimate problem which should prevent the user executing that command. This can be caught in the interface layer (a REST API) and returned to the user. Remember, any unexpected errors will mean the event won't be recorded, so it's best to keep this module free of any complex logic.


## Actions


Actions are things which are triggered when we create an event. These functions should only do one thing. This means that if something goes wrong whilst running that action, it won't affect of the others that need to happen. We also wrap the action call in a try/except in order to catch the error message and attach to the event.

Actions are subscribed to events when they are recorded. We went for a simple pub/sub architecture that allows us to invert dependencies between our data and domain layer (see Uncle Bob's hexagonal architecture). In our `build` module (which I'll explain later), we subscribe a handler to an event and then the actions to our event, like so:

```python
# build.py

@subscribe_event(OrderEventType.PAYMENT_SUCCEEDED)
@has_action(actions.send_payment_success_email, conditions=[customer_emails_accepted])
def handle_order_package(order_process, event):
    # do some build steps here
```

```python
# actions.py

def send_payment_success_email(order_process, event):
    email = Email(subject="Order successful", body="Some text")
    order_process.record_order_success_email(email.id)
```

We decided to write a module of `conditions` which return a boolean of whether the action should be added. If any don't pass then the action is ignored. It's useful to log this for debugging purposes, otherwise you can have actions which seemingly disappear.


## Building the process


It's easy to think of the process object as something we can attach mutator methods onto in order to make updates. The trouble with this is that there's no audit of this happening, which leads to hard-to-fix bugs.

Instead, we imagine that we're starting from scratch and loop through all the events chronologically and mutate the event as we go. A `build_process` function may look like:


```python
def build_process(process):
    for event in process.events:
        handler = data.event_handlers.get(event.event_type)
        # Store attributes of the process that you need to base decisions on
        context = {
            "ordered": False,
            "packaged": False,
            "delivered": False,
        }

        if handler:
            # Mutate the process and update context
            handler(process, event, context)

            # Run any actions still pending in the events
            # Passing in the context allows us to reduce the amount of DB queries (like an in-memory cache)
            for action in event.data.get("actions"):
                action(process, event, context)

    # You can figure out the status once we know everything that's happened
    process.status = _get_status_from_context(process, context)
    process.save()

    return process
```


And our event handler from earlier might do something like this:


```python
# build.py

@subscribe_event(OrderEventType.PAYMENT_SUCCEEDED)
@has_action(actions.send_payment_success_email, conditions=[customer_emails_accepted])
def handle_order_package(order_process, event):
    context["ordered"] = True
```


The benefit of this is that we never lose any state. If, for example, and event happened which changed the state erroneously, we just need to delete the event and rebuild and our process will be back in the state it should be in. Conversely, if we want to change a value stored in the process, we just add an event which describes the nature of the change, e.g. `CUSTOMER_CHANGED_DELIVERY_DATE` and then the build process will overwrite the original date, but the original date will be stored there in case we need to know what the customer initially picked.

We can also rebuild the events up to a certain time to see what state the process was at a certain point in time. Useful for finding out how a process changed over time.


## Downsides of event sourcing


As with any architectural pattern, we need to consider the possible downfalls. It might be that one of them might be a deal breaker.


This pattern is very database-query heavy for relational databases, as if you want to find if an event happened you have to query a table of potentially millions of events. This pattern may benefit from object storage, as the events are nested inside of the process and only requires one query. On the other hand, object storage would not be as efficient for aggregating events accross processes (for business analysis) so you may end up having to implement some kind of indexing (e.g. elasticsearch).


Forgetting to log data to an event can be an asolute killer, and means that you need to support events which may not contain that data, either by making assumptions about a default value or backfilling all of the events. With the `data` part of the event not having a schema, it makes it hard to track these kind of changes.


## Summary


Event sourcing is extremely powerful in situations where a system reacts to react to incoming events. In order to increase your success, you need to thoroughly plan what information you're going to need throughout the process and whether it should be stored in events or on the process.

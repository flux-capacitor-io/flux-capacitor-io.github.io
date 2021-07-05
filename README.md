<a href="flux-capacitor.io">
    <img src="https://flux-capacitor.io/assets/brand/flux-capacitor-white.svg" alt="Flux Capacitor logo" title="Flux Capacitor" align="right" height="60" />
</a>


Flux Capacitor
======================
Flux Capacitor is a service that tackles one of the biggest problems in today's microservice jungle:
how to let your services communicate reliably and easily without the need for complex infrastructure like message
queues, service registries, api gateways etc.

Once your apps have connected to Flux Capacitor they can publish and subscribe to messages. Messages come in different
flavors:

* Queries: stuff you want to ask, e.g. get me a user profile
* Commands: stuff you want to do, e.g. update a password
* Events: stuff you want to announce, e.g. an email address was changed

Because your apps connect to Flux Capacitor and not to each other it is super easy to add, remove or modify services.
And if a service is getting too busy it is trivial to scale the number of service instances: Flux Capacitor
automatically takes care of the load balancing between instances. All you need to do is deploy and run your instances.

Aside from routing messages between applications, Flux Capacitor also does the following:

* _Event sourcing:_ published events about your domain models end up in their own event log. To rebuild a domain model
  simply replay its events; no need to store your model in a database.
* _Search:_ index anything you want to search for later; no dedicated search engine necessary.
* _Auditing:_ all messages get stored in message logs by design. This makes it easy to see exactly what happened in your
  application, from events to queries and even errors: everything is available.
* _Scheduling:_ aside from sending messages immediately you can also schedule messages for the future.
* _Application metrics:_ you can trace the performance of all connected services from one place.
* _Kick-ass performance:_ you can publish and read at speeds well over millions of messages per second.

[comment]: <> (As the demand for more advanced software has kept growing, so has its complexity. At first this complexity was mainly)

[comment]: <> (felt at the application level. To combat this, developers transitioned from monolith to microservices. However, this)

[comment]: <> (just meant that much of the complexity shifted from the application to the infrastructure layer. As each microservice)

[comment]: <> (typically requires load balancing, a security layer, a data store, and so on, lots of infrastructure typically gets)

[comment]: <> (duplicated. Moreover, each request needs to find its way to the right microservice. This has led to the invention of)

[comment]: <> (service registries, api gateways and more advanced message queues, to name just a few.)

[comment]: <> (Needless to say, all of this has not helped to make software development easier. Developers today need to be well-versed)

[comment]: <> (in the development and maintenance of both software and the underlying infrastructure. However, on a more positive note,)

[comment]: <> (the actual demands on the infrastructure don't seem to be that many. In short, this is what we would want our infra)

[comment]: <> (layer to handle for us:)

[comment]: <> (* Message routing to and from services)

[comment]: <> (* Load balance messages to our service instances)

[comment]: <> (* Allow services to dynamically scale up and down)

[comment]: <> (* Data persistence &#40;mostly key-based, event-sourcing, or for search&#41;)

[comment]: <> (Obviously, we need all of this . )

[comment]: <> (evolves has evolved the need)

# Table of contents

- [Flux Capacitor](#flux-capacitor)
- [Overview](#overview)
- [Why this product exists](#why-this-product-exists)
- [Core concepts](#core-concepts)
    * [Message routing](#message-routing)
        + [Commands and Queries](#commands-and-queries)
        + [Tracking](#tracking)
            - [Flux Capacitor makes time travel possible](#flux-capacitor-makes-time-travel-possible)
        + [Consumers](#consumers)
        + [Load balancing](#load-balancing)
        + [Single threaded](#single-threaded)
        + [Message Functions](#message-functions)
            - [Queries](#queries)
            - [Commands](#commands)
            - [Events](#events)
            - [Results](#results)
            - [Errors](#errors)
            - [Notifications](#notifications)
            - [Metrics](#metrics)
            - [Schedules](#schedules)
    * [Event sourcing](#event-sourcing)
        + [Event sourcing in Flux Capacitor](#event-sourcing-in-flux-capacitor)
        + [Upcasting](#upcasting)
    * [Scheduling messages](#scheduling-messages)
    * [Full behavior testing](#full-behavior-testing)

# Overview

# Why this product exists

[comment]: <> (-- even in de vrieskast)

[comment]: <> (Building software must be the greatest job in the world. You can be very creative when coding. )

[comment]: <> (And when you make a great design, you can have a big impact on a large number of people, )

[comment]: <> (more impact than you can have with most other work. )


[comment]: <> (Most enjoyable are permanent solutions. For example the Jackson library for Java, )

[comment]: <> (which does it job so well, we never have to worry about mapping JSON to Java and back.)

[comment]: <> (When coding yourself, most effort goes to solving problems as permanent as possible, which allows you to  )

[comment]: <> (Now in 2021, most of the time developers spend on their programs is not functional or creative, but technical. )

[comment]: <> (The scaling and performance demands increase, core features and structures have to be changed.)

# Core concepts

## Message routing

### Commands and Queries

Most applications communicate by sending API calls to eachother. Applications can send a query and get an answer back by
doing a GET call, for instance "Give me all shipped products". Application can also send a command to do something and
get back whether it succeeded, for instance "Add a new order", "Delete an order". These queries and commands are the
main body of communication between services.

Direct communication, like API calls, does not scale well for rapidly-changing applications with high performance
demands. For robustness, you for instance need to balance load across multiple service instances, you have to set up api
gateways and load balancers to reroute these direct calls. And service registries are needed to tell you which services
are available to be routed to. Besides this infrastructure, you also have direct exposed endpoints, which need to be
secured with even more infrastructure, like a firewall, DDoS protection, some authentication mechanism, etc. You would
have to hire some cloud infrastructure engineers to deal with all these concerns before being able to launch your
product.

What if your could send queries and commands completely indirect, and without exposed endpoint? The above mentioned
infrastructure would not be needed. An application would simply post a query into a single place, and receive the answer
from that place, without having any technical dependency on any of the applications that answered. It would not known
anything about "how" the question ends up in the right place, or "how" the answer is linked back to the application. The
only thing your application would need to know is "what" question can be answered. Queries and commands between
applications would be as easy as within the application.

With Flux Capacitor we have created this indirect, endpointless way for your services to communicate. We provide your
applications a single endpoint, where your services can post messages and can track messages.

If you need to ask a question to another application (or to yourself), you will be able to achieve it with for example
this Java code:

[comment]: <> (@formatter:off)
```java
List<Order> orders = FluxCapacitor.queryAndWait(new GetOrders(...));
```
Or if you want to post something, telling an application to do something:

```java
FluxCapacitor.sendCommandAndWait(new AddOrder(...));
```

The results of your action will be returned to you, as if you called a method directly! An error would be thrown the same as well.
Whether the question is director indirect, the interface for your application to "ask" remains the same.

If you want to handle a request, you will create a **Handler**, which is a function that takes a request as input, and
gives a result as output. For instance:

```java
@HandleQuery
List<Order> handle(GetOrders query){
    return ...;
}
```
[comment]: <> (@formatter:on)
You still need to create the business specific behavior your application needs to perform, but all the technicalities of
connecting the right requester with the right responder are no longer your problem.

### Tracking

**Tracking** gives us the ability to get data from us to your application as fast as possible, without requiring your
application to supply an endpoint to us.

Tracking is superior to pushing messages or queues. It works like this:

1. Your application tells us to get the next X messages
2. Your application processes the messages
3. Your application sends us back its new position (the index of the **last message it processed**).

When there are no messages waiting, your connection will hold until we receive a new message, and instantly send the
message to you. When there are lots of messages waiting, you will immediately get a batch with a size of your own
choosing.

The request-response interaction with tracking looks like this:

![alt text](https://github.com/flux-capacitor-io/flux-capacitor-io.github.io/raw/master/dist/img/Tracking.jpg "Basics")

With this mechanism, it is not possible to overwhelm your application with too much pushing, like a DDoS attack. There
is no endpoint to overload, and you will only get a next batch of messages once you are done processing. Your
application is in control of what it receives.

Also it is not possible to lose a message. With queues, messages often disappear once read, or there are certain
irritating limitations set for how long a read message is available. With tracking however, your will always start where
you left off.

Suppose your application crashes during processing using tracking (during step 2). In that case, the application has not
updated its position yet. Once your application has rebooted, the application will start at the same position and
receive the same set of messages again. Or if you were running multiple nodes, we simply shift the messages to your
other nodes. [More on that in the chapter about load balancing](#load-balancing).

#### Flux Capacitor makes time travel possible

you can go back in time with trackers. Great scot!

You can reset a tracker to any previous point in time. Or when you add a new tracker, you can tell us to start tracking
from the beginning of time.

Resetting a tracker can be very useful. Suppose you are creating a new application, and created a bug that made you
process a bunch of messages wrongly. You deploy your fix, and then simply reset your tracker to the moment you deployed
the bug. All messages will again be processed, now with the fixed message handlers. This has often been our saving grace
during new projects.

You can wait with certain secondary features, like billing customers based on usage. You will always be able to use your
old data.

### Consumers

The parts of your application that processes messages are called **Consumers**. A consumer consists of:

* A filter of the types of messages it consumes (e.g. only events, or all queries called "GetOrders")
* A filter of handlers that belong to this consumer (e.g. all handlers in this package, or only handler methods called
  X)

Each consumer is tracking separately. Within the same application, you can have multiple separate consumers that track
messages at completely different speeds.

The beauty of these consumers, is that whether consumers live in the same application or in separately running
applications, their behavior and interaction remains exactly the same. In a sense, consumers are small separate
applications.

![alt text](https://github.com/flux-capacitor-io/flux-capacitor-io.github.io/raw/master/dist/img/moveconsumersfreely.jpg "Consumers can be moved freely")

With separate consumers, you can divide parts of your application that do not really belong together, before you even
moved them to a separate application.

Suppose you have a shop application, with orders and deliveries, and your billing department needs you to count all
orders for specific categories of things. You don't want that code influencing your core code. With a separate consumer,
you can build the billing part as if it is completely separate.

More often than not, programmers will link separate concerns directly that should never be linked at all. Some concerns
could touch every part of your core code. These are called **cross-cutting concerns**,
[more on this here.](https://en.wikipedia.org/wiki/Cross-cutting_concern#:~:text=Cross%2Dcutting%20concerns%20are%20parts,oriented%20programming%20or%20procedural%20programming.)
With a separate consumer, you can easily listen to a whole bunch of messages separately, and often remove these
cross-cutting concerns from the core code.

Creating a consumer quite easy, here is an example in Java using Spring and our client library:

``` java
@Configuration
@ComponentScan
class Config {
    @Autowired
    void configure(FluxCapacitorBuilder builder) {
        builder.addConsumerConfiguration(ConsumerConfiguration.builder()
                        .messageType(QUERY)
                        .name(...)
                        .handlerFilter(h -> h.getClass().getPackage().getName()
                            .startsWith("com.example")).build());
    }
}
```

### Load balancing

A must for communication between services is high availability.

With API communication this is often done by running multiple nodes of a service, and having incoming traffic
distributed over these nodes. The load is distributed by passing all traffic through an API gateway, where a load
balancer divides the traffic between nodes.

With our asynchronous setup, we give you load balancing by default, without requiring any infrastructure like load
balancers. Load is automatically balanced between consumers **with the same name**.

The load balancing works with message **Segments**. Messages are divided across a set of segments (right now 1024
segments). When two consumers with the same name are tracking Flux Capacitor, segments are divided 50-50. Consumers only
get messages from their assigned segments, and consumers only update positions on their assigned segments.

When you place a new node, for instance to deploy a new release, the consumers in the service will often start tracking
with none of the segments. Once one of the other consumers is done processing, it will get a smaller piece of the
segments, to make room for the new node. When you remove a node, an message is sent to Flux Capacitor to disconnect the
consumers, releasing those segments for other consumers to pick up. When a node fatally crashed, we automatically
release the segments after a certain time.

![alt text](https://github.com/flux-capacitor-io/flux-capacitor-io.github.io/raw/master/dist/img/Loadbalancer.jpg "Loadbalancing")

Messages are given random segments by default based on a message id. But you can set a **Routing key** (@RoutingKey in
our client library) to base the segments on your data in the message. Two messages with the same routing key will be in
the same segment, and thus be processed in the same consumer. Therefore, messages with the same routing key are always
processed in order.

Messages in order are very useful for **event-sourcing**, since we guarantee we are not out of order, and thus are
working with the latest situation. Also, we were able to create efficient local caching for aggregate in our client
library. [For more see the chapter on event-sourcing](#event-sourcing).

### Single threaded

Every consumer is given a single thread by default.

### Message Functions

Now that you know about our asynchronous messaging, it is easy to explain why we have defined several message types. We
will discuss them one by one.

#### Queries

Queries are similar to GET API requests, they are questions that result in an answer or error. We provide you methods to
input a query, and receive this result or error easily and in the same manner, whether the answer came locally or from a
completely different application.

A query is published when you call ```FluxCapacitor.queryAndWait(...)```, which lets the thread wait for the answer.
There is also an async version ```FluxCapacitor.query(...)``` using CompletableFutures. Query handlers are annotated
with ```@HandleQuery```.

We automatically

* track result messages, pick out the result belonging to your query, and return it.
* track error messages, pick out the error belonging to your query, and throw it.

#### Commands

Commands are similar to POST API requests, they are commands to perform an action, that result in a success or error
response.

A command is published when you call ```FluxCapacitor.sendCommandAndWait(...)```. There is also an async version, and
the method  ```FluxCapacitor.sendAndForgetCommand(...)``` for when you are not interested in the result. Command
handlers are annotated with ```@HandleCommand```.

We automatically track results and errors for commands as well. The results are void, but they do stop the waiting,
indicating successful processing.

#### Events

Events are things that happened. Events do not result in an answer or error.

An event is published when you apply an event to an aggregate: ```FluxCapacitor.loadAggregate(...).apply(...)```
. [More about this in the chapter about event-sourcing](#event-sourcing). Events not associated with an aggregate are
published with ```FluxCapacitor.publishEvent(...)```. Event handlers are annotated with ```@HandleEvent```.

#### Results

Results are answers to other messages. The value you return in for instance a query or command handler, is automatically
published as a result. Results are returned to the calling method, but can also be handled like any other message with
the handler annotation ```@HandleResult```.

#### Errors

An uncaught exception within for instance a query or command handler, is automatically published as an error. Errors are
thrown in the calling method, but can also be handled like any other message with the handler
annotation ```@HandleError```.

#### Notifications

Notifications are messages that ignore segments. They are sent to every node your have. Notifications are events and
cannot be published separately. But they can be consumed and tracked separately, and can be handled
with ```@HandleNotification```.

#### Metrics

Metrics are messages concerning the technical operation of your application. It has a separate log from events, since
you do not want technical events mixing with functional events. Every communication between your application and Flux
Capacitor is automatically published to this log (e.g. ApplyEvents, GetEvents, ReadResults), as are a few internal
events (e.g. DisconnectClient for when we disconnect an unresponsive client).

You can add your own metrics by calling ```FluxCapacitor.publishMetrics(...)```. Metrics can be handled
with ```@HandleMetrics```. Very useful for creating your own audit

#### Schedules

Schedules are messages that are handled in the
future. [More about this in the chapter about scheduling messages](#scheduling-messages)

## Event sourcing

Most applications use a database to keep track of the latest state of the application. Getting to that latest state
probably involved lots of tiny updates, e.g: a user signed up, an order got shipped, a complaint was filed, and so on.

What if we simply stored all these changes as events? Wouldn't we then be able to reconstruct the state of the
application at any point in time by replaying those events? Well yes: that's exactly the idea behind event sourcing.

Here's a chain of events for a given webshop order:

<img src="https://flux-capacitor.io/assets/Event%20sourcing.jpg" alt="Event sourcing" title="Event sourcing" />

After the dust has settled it almost appears like nothing happened: the customer paid but got refunded, the shipped item
is back in inventory, and the webshop got paid but later reimbursed the customer. In reality a lot did happen, but it's
hard to squeeze this entire timeline into a database that only keeps track of the current state.

To fetch the order using event sourcing you would load these events and apply them one by one to recreate the order,
i.e: no need to store the order in a database. Event sourcing offers a lot of advantages compared to the traditional way
of storing data. For more in-depth explanations please refer to
<a href="https://martinfowler.com/eaaDev/EventSourcing.html" target="_blank">these</a>
<a href="https://docs.microsoft.com/en-us/azure/architecture/patterns/event-sourcing" target="_blank">excellent</a>
<a href="https://www.eventstore.com/blog/what-is-event-sourcing" target="_blank">articles</a>.

### Event sourcing in Flux Capacitor

Flux Capacitor doesn't force you to use event sourcing in any way. It simply makes it easy for you if you do. Here's how
you can load an event sourced entity (or aggregate) using our Java client:

```java
class OrderCommandHandler {
    @HandleCommand
    void handle(ShipOrder command) {
        FluxCapacitor.loadAggregate(command.getOrderId(), Order.class) //load the order entity
                .apply(new OrderShipped(...)); //apply a new event
    }
}
```

In this example we ask the Flux Capacitor client to load an order entity when a handler receives a command to ship an
order. We then apply a new event to this entity, called OrderShipped. That event will be appended to the event log of
the order entity. The next time the order is event sourced the new event will be returned as well. Flux Capacitor will
also add the event to the *global* event log, so any event consumers will also get this event.

How does the Flux Capacitor client know how to apply the OrderShipped event on the Order? Also this part is very easy:

```java

@Aggregate
class Order {
    String orderId;
    OrderDetails details;
    boolean shipped;
    ...

    @ApplyEvent
    Order handle(OrderShipped event) {
        return this.toBuilder().shipped(true).build;
    }
    
    ...
}
```

As you see we leave all the business logic up to you. We tackle generic technical challenges like event sourcing, so you
can focus on what's actually important in your business.

What if you don't want to event source this entity but store and load this entity the traditional way? That's easy too:
just change `@Aggregate` to `@Aggregate(eventSourced=false)` and we'll automatically store the latest state of the
entity in Flux Capacitor's key value store. Boom!

### Upcasting

Event sourcing your entities is very powerful, but it comes with a challenge: your stored events need to stand the test
of time. They need to keep up with all the changes you make to your event classes, like changing a field name. Flux
Capacitor makes it easy to deal with these sorts of changes.

Before deserializing any stored value Flux Capacitor will first attempt to _upcast_ the value using a chain of
upcasters. Upcasters are so called because they transform serialized values to be compatible with the latest revision of
the value class.

Say you changed the name of a field in your event class. An upcaster can modify the serialized event payload before the
data is deserialized. This all happens in your application at the client side; the messages stored in Flux Capacitor
Service are not modified.

To mark a change in the revision of a message payload simply annotate its class:

```java

@Revision(1)
class UserCreated {
    String userId; //renamed from id
    ...
}
```

Assuming that you’re using the default `JacksonSerializer`, here’s how you would write an upcaster for the change:

```java
class UserUpcaster {
    @Upcast(type = "com.example.UserCreated", revision = 0)
    ObjectNode upcastUserCreatedTo1(ObjectNode json) {
        return json.rename(...);
    }
}
```

This upcaster will be applied to all revision 0 events of the UserCreated event. After upcasting, the revision of the
serialized event will automatically be incremented by 1.

## Scheduling messages

Another notoriously tricky problem in a distributed application is the scheduling of future events. Typically this would
involve an intricate master-slave setup with synchronization on a database, but with Flux Capacitor it is as easy as
sending and handling any other message. Scheduled messages are stored and read like other messages except that they are
not released before their deadline and can be cancelled.

Here's an example of a handler in Java that asks a customer if they are satisfied with their order 2 days after it got
shipped:

```java
import java.time.Duration;

class OrderFeedbackHandler {
    @HandleEvent
    void handle(ShipOrder event) {
        FluxCapacitor.scheduler().schedule("OrderFeedback-" + event.getOrderId(), Duration.ofDays(2),
                new AskForFeedback(...));
    }

    @HandleSchedule
    void handle(AskForFeedback schedule) {
        ... //send an email to the customer
    }

    @HandleEvent
    void handle(ReturnOrder event) {
        FluxCapacitor.scheduler().cancelSchedule("OrderFeedback-" + event.getOrderId());
    }
}
```

In the example above you can see that the schedule can be easily cancelled, in this case if the customer returns the
order before those 2 days are up.

## Full behavior testing

Full backend behavior tests, especially tests across multiple different services, are normally quite difficult to
achieve. Often a local environment has to be created with complete service setup. Databases, full running services and
networking that at least partially resembles the real setup. These "full" tests are often very slow to start, and
because of this, most often not used as the primary means of testing. They are most often used to detect problems with
your message routing, the functional interaction only being covered in separate unit tests.

With Flux Capacitor we support fast and easy full backend behavior tests. Ofcourse, you don't need to test our message
routing, you should only test functional behavior that you created. In the previous chapters we have explained
how our message routing works, and especially
how [asking a question to your own application and to another application is now exactly the same](#commands-and-queries)
. Well, here we utilize its full potential, for we can run all handlers locally with minimal message traffic, and in
parallel.

An example from our [example bank project, check it out to see the test speed and for more cleancut examples]():

```java
class BankAccountTest {
    private static final CreateAccount createAccount = CreateAccount.builder().accountId("a").userId("user1").build();
    private static final CreateAccount createAnotherAccount = CreateAccount.builder().accountId("b").userId("user2").build();
    private static final TransferMoney transferMoney = new TransferMoney("a", "b", BigDecimal.TEN);
    private final TestFixture testFixture = TestFixture.create(new AccountCommandHandler(), new TransferEventHandler(),
            new AccountLifecycleHandler());

    @Test
    void testCreateAccountTwiceNotAllowed() {
        testFixture.givenCommands(createAccount).whenCommand(createAccount).expectException(IllegalCommandException.class);
    }

    @Test
    void testTransferNotAllowedWithInsufficientFunds() {
        testFixture.givenCommands(createAccount, createAnotherAccount)
                .whenCommand(transferMoney).expectException(IllegalCommandException.class).expectNoEvents();
    }
}
```

If you want to be stubborn and still test including our message routing, you are ofcourse able to do this. We provide a
test-server [here](https://hub.docker.com/repository/docker/fluxcapacitorio/flux-capacitor-test).
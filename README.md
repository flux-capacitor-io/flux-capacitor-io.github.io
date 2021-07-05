<a href="flux-capacitor.io">
    <img src="https://flux-capacitor.io/assets/brand/flux-capacitor-white.svg" alt="Flux Capacitor logo" title="Flux Capacitor" align="right" height="60" />
</a>

# Table of contents

- [Flux Capacitor](#flux-capacitor)
- [Messaging as a Service](#messaging-as-a-service)
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

# Flux Capacitor

Building software is great. You can literally create something out of nothing and change the world.

Some ideas can be realised very quickly too. If your idea needs a website and basic webshop, you could be up and running
in a day. But if your needs are a little more demanding, the time required to launch a new product, feature or even bug
fix tends to be significant. Most of that time is not spent on problems unique to your company. In fact, most companies
struggle with the exact same technical problems.

We believe it doesn’t have to be this way. At Flux Capacitor we take on some of the biggest challenges plaguing
architects and developers today so you don't have to. This way, you can focus on the actual challenges in your domain
and launch your idea in a month instead of a year. We believe in solving problems at their roots, not by remedying their
symptoms.

# Messaging as a Service

So what problems do we tackle? Well, our biggest objective is to make it easy, fast and reliable for your services to
communicate with each other and the outside world. We don't want you to worry about building the next microservice
platform like so many companies have before you, just so you can launch your next big thing.

We've developed a specialized service that enables highly performant, reliable messaging between applications. It makes
it a breeze to launch new services. Once a service is connected to Flux Capacitor it can talk to any other connected
service. Your service doesn't need any proxy, load balancer, firewall, service registry, api gateway, circuit breaker,
message queue, etc. to accomplish that. And if your service is getting too busy to handle all requests you can just add
a second instance; Flux Capacitor will automatically balance the request load.

# Core concepts

## Message routing

### Commands and Queries

Most applications communicate by sending API calls to each other. The majority of these calls are either queries or
commands. A query is a request for information, like: "Give me all orders that shipped in the last month". In web APIs a
query is typically performed via a GET request. A command is a request for the application to do something, for
instance "Place a new order", "Delete an order". In web APIs these are usually the POST, PUT and DELETE requests.

Web API calls and other forms of direct communication, do not scale well for rapidly-changing applications with high
performance demands. For high availability and scalability, services are typically deployed more than once. A load
balancer is used to balance load and handle failover. Proxies and things like a content delivery network or api gateway
is often used to route or modify specific API requests. As the numbers of endpoints grow you'll also need something like
a service registry. Besides all this infrastructure, you'll also have directly exposed endpoints, which need to be
secured with even more infrastructure, like a firewall, DDoS protection, some authentication mechanism, etc. You will
need a dedicated team of cloud engineers to deal with all these concerns before you are even ready to launch your
product.

All this additional infrastructure is the result of the fact that queries and commands are being pushed to your
services. What if we would turn this around? What if your services would pull in their queries and commands from a and
handle them at their own pace? That simple change would render all the mentioned infrastructure obsolete. Moreover, we
would break the link between the producer of a request and its responder. The producer does not need to know anything
about the consumer (location transparency) and vice versa. The only thing your application needs to know is what queries
or commands it wants to handle. Passing requests to another application would become just as easy as passing it to a
class within your application.

With Flux Capacitor this indirect form of communication based on pull instead of push is made very easy. We provide your
applications a single endpoint, to which your services can both publish and subscribe various types of messages.

For instance, if you need to ask a question to another application (or your own), you will be able to do it with this
single line of code:

[comment]: <> (@formatter:off)

```java
List<Order> orders = FluxCapacitor.queryAndWait(new GetOrders(...));
```

Note: examples here are in Java, but Flux Capacitor does not care what language your applications are written
in of course.

To send a command you can simply do:

```java
FluxCapacitor.sendCommand(new PlaceOrder(...));
```

The results of your action will be returned to you, so it would be just as if you invoked a method within your
application! And if your action causes an exception somewhere the exception will be thrown to you as well.

To handle a request, all you need to do is add a **Handler**, which is a function that takes a request as input, and gives
a result as output. For instance:

```java
@HandleQuery
List<Order> handle(GetOrders query) {
    return...;
}
```

[comment]: <> (@formatter:on)

As you see you will still need to implement all the behavior that's specific to your business, but all the
technicalities of connecting services are no longer your concern.

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

The things doing the tracking we call **Trackers**. Queries with tracking requires two trackers, and looks like this:

![alt text](https://github.com/flux-capacitor-io/flux-capacitor-io.github.io/raw/master/dist/img/Tracking.jpg "Basics")

With tracking, it is not possible to overwhelm your application with too much pushing, like a DDoS attack. There is no
endpoint to overload, and you will only get a next batch of messages once you are done processing. Your application is
in control of what it receives.

Also it is not possible to lose a message. With queues, messages often disappear once read, or there are certain
limitations set for how long a read message is available. With tracking however, your will always start where you left
off. Suppose your application crashes during processing using tracking (during step 2). In that case, the application
has not updated its position yet. Once your application has rebooted, the application will start at the same position
and receive the same set of messages again. Or if you were running multiple nodes, we simply shift the messages to other
trackers in your consumer.

### Consumers

The parts of your application that processes messages are called **Consumers**. A consumer consists of:

* A filter of the types of messages it consumes (e.g. only events, or all queries called "GetOrders")
* A filter of handlers that belong to this consumer (e.g. all handlers in this package, or only handler methods called
  X)
* A set of trackers

The trackers belonging to a single consumer can be seen as separate **Threads** of the consumer. The main function of
separate trackers is load balancing and redundancy. [More on how we load balance with consumers here](#load-balancing)

In a sense, consumers are small separate applications. The beauty of these consumers, is that whether consumers live in
the same application or in separately running applications, their behavior and interaction remains exactly the same.

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

An example of a consumer configuration:

``` java
@Configuration
@ComponentScan
class Config {
    @Autowired
    void configure(FluxCapacitorBuilder builder) {
        builder.addConsumerConfiguration(ConsumerConfiguration.builder()
                        .messageType(QUERY).threads(2)
                        .name(...)
                        .handlerFilter(h -> h.getClass().getPackage().getName()
                            .startsWith("com.example")).build());
    }
}
```

If a handler is not covered by any of your custom consumers, it is assigned to a default consumer.

### Load balancing

A must for communication between services is high availability.

With API communication this is often done by running multiple nodes of a service, and having incoming traffic
distributed over these nodes. The load is distributed by passing all traffic through an API gateway, where a load
balancer divides the traffic between nodes.

With our asynchronous setup, we give you load balancing by default, without requiring any infrastructure like load
balancers. Load is automatically balanced within consumers.

The load balancing works with message **Segments**. Messages are divided across a set of segments (right now 1024
segments). When a consumer consists of two trackers, segments are divided 50-50. Trackers only
get messages from their assigned segments, and trackers only update positions on their assigned segments.

![alt text](https://github.com/flux-capacitor-io/flux-capacitor-io.github.io/raw/master/dist/img/segments.jpg "Loadbalancing")

When you place a new node, for instance to deploy a new release, the new consumers will start tracking with none of the
segments. Once one of the pre-existing consumers is done processing, it will continue with a smaller piece of the
segments, to make room for the new consumers.

When you remove a node, a message is sent to Flux Capacitor to disconnect the consumers immediately, releasing those
segments for other consumers to pick up. In case a node fatally crashes without sending a disconnect, we automatically
disconnect after a certain time and thus release the segments.

Messages are given random segments by default based on a message id. But you can set a **Routing key** (@RoutingKey in
our client library) to base the segments on your data in the message. Two messages with the same routing key will be in
the same segment, and thus be processed in the same consumer. Therefore, messages with the same routing key are always
processed in order.

Messages in order are very useful for **event-sourcing**, since we guarantee we are not out of order, and thus are
working with the latest situation. Also, we were able to create efficient local caching for aggregate in our client
library. [For more see the chapter on event-sourcing](#event-sourcing).

#### Flux Capacitor makes time travel possible

You can go back in time with consumers.

![alt text](https://github.com/flux-capacitor-io/flux-capacitor-io.github.io/raw/master/dist/img/greatscott.gif "Great scott!")

You can reset a consumer to any previous point in time. Or when you add a new consumer, you can tell us to start
tracking from the beginning of time.

Resetting a consumer can be very useful. Suppose you are creating a new application, and created a bug that made you
process a bunch of messages wrongly. You deploy your fix, and then simply reset your tracker to the moment you deployed
the bug. All messages will again be processed, now with the fixed message handlers. This has often been our saving grace
during new projects.

You can wait with certain secondary features, like billing customers based on usage. You will always be able to use your
old data.

### Single threaded

Every consumer is given a single thread by default. You can add a thread to a consumer by adding ```.threads(2)```.
Message routing within these threads are also separated based on segments. In a sense, every tracker in a consumer can
be seen as a different thread.

With the s

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
annotation ```@HandleError```. For a real example, check TransferEventHandler in our bank example.

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
routing, you should only test functional behavior that you created. In the previous chapters we have explained how our
message routing works, and especially
how [asking a question to your own application and to another application is now exactly the same](#commands-and-queries)
. Well, here we utilize its full potential, for we can run all handlers locally with minimal message traffic, and in
parallel.

An example from our [example bank project, check it out to see the test speed and for more cleancut examples]():

```java
class BankAccountTest {
    private static final CreateAccount createAccount = CreateAccount.builder().accountId("a").userId("user1").build();
    private static final CreateAccount createAnotherAccount =
            CreateAccount.builder().accountId("b").userId("user2").build();
    private static final TransferMoney transferMoney = new TransferMoney("a", "b", BigDecimal.TEN);
    private final TestFixture testFixture = TestFixture.create(new AccountCommandHandler(), new TransferEventHandler(),
                                                               new AccountLifecycleHandler());

    @Test
    void testCreateAccountTwiceNotAllowed() {
        testFixture.givenCommands(createAccount).whenCommand(createAccount)
                .expectException(IllegalCommandException.class);
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
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

## 1. Message routing

### 1.1 Indirect request-response

Most applications communicate by sending API calls to eachother. Applications can send questions and get an answer back,
or they can send a command to do something and get back whether it succeeded, e.g: Give me all shipped products, Add a
new order, Delete an order, etc. These are request-response interactions.

What if your could do request-response completely indirect, asynchronously? An application would simply post a question
into the world, and receive the answer from the world, without having any technical dependency on the other
applications. The only thing your application needs to know is "what" question can be asked. It would not known anything
about "how" the question ends up in the right place, or "how" the answer is linked back to the application. That is
something that we support with Flux Capacitor.

We provide your applications a single endpoint, where they can post messages and can track messages.

If you need to ask a question to a another application, you will be able to achieve it with for example this Java code:

```java
List<Order> orders=FluxCapacitor.queryAndWait(new GetOrders(...));
```

Or if you want to post something, telling an application to do something:

```java
FluxCapacitor.sendAndForgetCommand(new AddOrder(...));
```

If you want to handle a request, you will create a **Handler**, which is a function that takes a request as input, and
gives a result as output. For instance:

```java
@HandleQuery
List<Order> handle(GetOrders query){
        return...
        }
```

You still need to createthe business specific behavior your application needs to perform, but all the technicalities of
connecting the right requester with the right responder are no longer your problem.

### 1.2 Tracking

**Tracking** gives us the ability to get data from us to your application as fast as possible, without requiring your
application to supply an endpoint to us.

Tracking is superior to pushing messages or queues. It works like this:

* Your application tells us to get the next X messages
* Your application processes the messages
* Your application sends us back the index of the **last message it processed**.

When there are no messages waiting, your connection will hold until we receive a new message, and instantly send the
message to you. When there are lots of messages waiting, you will immediately get a batch with a size of your own
choosing.

The request-response interaction with tracking looks like this:

![alt text](https://github.com/flux-capacitor-io/flux-capacitor-io.github.io/raw/master/dist/img/Tracking.jpg "Basics")

With this mechanism, it is not possible to overwhelm your application with too much pushing, like a DDoS attack. There
is no endpoint to overload, and you will only get a next batch of messages once you are done processing. It is on-demand
messaging, your application is in control of what it receives.

Also it is not possible to lose a message. With queues, messages often disappear once read, or there are certain
irritating limitations set for how long a read message is available.

Suppose your application crashes during processing using tracking. Your application was still processing, so did not
send a new position or index to us. Once your application has rebooted, you will receive the messages again that it was
processing. We keep an index for where your application was, and you start there, where you left off. If you were
running multiple nodes, we can easily shift the messages to your other nodes. More on that in the chapter about **Load
balancing**.

### Flux Capacitor makes time travel possible

The crème de la crème of our tracking, is that you can go back in time. Great scot!

You can reset a tracker to any previous point in time. When you add a new tracker, you can tell us to start tracking
from the beginning of time.

Suppose you are creating a new application, and created a bug that made you process a whole bunch of messages wrongly.
You deploy your fix, and then simply reset your tracker to the moment you deployed the bug. This has often been our
saving grace during new projects.

Another benefit is that you can wait with certain secondary features, like billing customers based on usage. You will
always be able to use your old data.


[comment]: <> (```java)

[comment]: <> (@Configuration)

[comment]: <> (@ComponentScan)

[comment]: <> (public class Config {)

[comment]: <> (  @Bean)

[comment]: <> (  public Client fluxCapacitorClient&#40;&#41; {)

[comment]: <> (    return WebSocketClient.newInstance&#40;WebSocketClient.Properties.builder&#40;&#41;)

[comment]: <> (            .serviceBaseUrl&#40;"..."&#41;.build&#40;&#41;&#41;;)

[comment]: <> (  })

[comment]: <> (})

[comment]: <> (```)

### 1.3 Consumers

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

More often than not, programmers will link separate concerns directly that should never be linked at all. Some concern
will touch every part of your core code if it is not easy to create separate consumers. These are called **cross-cutting
concerns**.
[More on that here.](https://en.wikipedia.org/wiki/Cross-cutting_concern#:~:text=Cross%2Dcutting%20concerns%20are%20parts,oriented%20programming%20or%20procedural%20programming.)

We have made this process quite easy, here is an example in Java using Spring and our client library:

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
                            .startsWith(...)).build());
    }
}
```

### 1.5 Message Functions

* Queries: Processed once, returns result or error
* Commands: Processed once, returns result or error, can lead to events (see event-sourcing chapter)
* Events: Processed many times

### 1.6 Handlers are strong

We already talked a bit about Handlers, but some of their possibilities are important to know.

-- two handlers in separate classes -- passive

```
@Component
public class PetstoreEmailSender {
    @HandleEvent
    void handle(CatReceivedForAdoption event) {
        if(event.isCuteEnough()) {
            FluxCapacitor.sendCommand(new SendEmailToCustomers(createEmail(event)));
        }
    }
    
    @HandleEvent
    void handle(BlackCatReceivedForAdoption event) {
        // do nothing
    }
}
```

### 1.7 Load balancing

Index per consumer (3 orderService, inventory service). Een service 2x deployed om te koppelen aan segmenten

### 1.8 Single threaded

## 2. Event sourcing

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

### 2.1 Event sourcing in Flux Capacitor

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

### 2.2 Upcasting

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

## 3. Scheduling messages

Another notoriously tricky problem in a distributed application is the scheduling of future events. Typically this 
would involve an intricate master-slave setup with synchronization on a database, but with Flux Capacitor
it is as easy as sending and handling any other message. Scheduled messages are stored and read like other messages
except that they are not released before their deadline and can be cancelled.

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
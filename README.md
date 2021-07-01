<a href="flux-capacitor.io">
    <img src="https://flux-capacitor.io/assets/brand/flux-capacitor-white.svg" alt="Flux Capacitor logo" title="Flux Capacitor" align="right" height="60" />
</a>


Flux Capacitor
======================
Flux Capacitor is a service that tackles one of the biggest problems in today's microservice jungle:
how to let your services communicate reliably and easily without the need for complex infrastructure like message queues,
service registries, api gateways etc.

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

In the following section we will show you how message routing is organised in Flux Capacitor service, and why it is the best way to communicate in software.

### 1.1 The essence

In its barest essence, Flux Capacitor contains message logs, and services can publish to or listen to these logs.

Microservices lose the direct dependency on one another. All communication is indirect and asynchronous.
When you would normally perform POST or GET API calls to a microservice, you now publish a message to Flux Capacitor that represents this POST or GET.
The response to this message is a message as well, which finds it way back indirect and asynchronous too.

![alt text](https://github.com/flux-capacitor-io/flux-capacitor-io.github.io/raw/master/dist/img/basics2.jpg "Basics")

### 1.2 Consumers

Messages are sent to Flux Capacitor and parts of your application consume/process these messages. 
We call the individual parts that do this **Consumers**.

Each message log can have infinite consumers listening to the log. 
Each consumer consumes new messages at its own pace.
Two consumers will not process messages in order. One might be faster than the other.

For each consumer we store the position of the last message it has consumed.
When a consumer receives a new batch of messages, the batch starts at this position. 
When the consumer is done processing the batch, it will let us know the new position of the last message it has successfully consumed.
With this mechanism, we guarantee a consumer consumes all messages only once, and in order.

The beauty of these consumers, is that whether two consumers live in the same application or are separately running applications,
their behavior and interaction remains exactly the same. In a sense, consumers are small separate applications.

![alt text](https://github.com/flux-capacitor-io/flux-capacitor-io.github.io/raw/master/dist/img/moveconsumersfreely.jpg "Consumers can be moved freely")

Making a consumer is easy, here is an example in Java using Spring and our client library:

```
@Configuration
@ComponentScan
public class PetStoreConfig {
    @Autowired
    public void configure(FluxCapacitorBuilder builder) {
        builder.addConsumerConfiguration(ConsumerConfiguration.builder()
                        .messageType(EVENT)
                        .name("petstore-consumer")
                        .handlerFilter(h -> h.getClass().getPackage().getName()
                            .startsWith("com.fluxcapacitor.petstore")).build());
    }
}
```
In plain English: We give the consumer a unique name (petstore-consumer), said which message log we want to consume (events)
and we point to the **Handlers** that belong to this consumer. More on Handlers next.

[comment]: <> (Two consumers will not process messages in order. One might be faster than the other.)
[comment]: <> (When a consumer terminates unexpectedly, it will not have updated its position. When the consumer is restarted, it will continue where it left off.)
[comment]: <> (More on this in the chapter on Load balancing.)

### 1.2 Handlers

Consumers consist of a set of **Handlers**, the smallest unit when working with Flux Capacitor.

When a message is received from Flux Capacitor service, the message is given to all handlers.

Here is an example of a handler for the event "CatReceivedForAdoption", which in some cases will publish a command "SendEmail".

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


### 1.2 Connection simplicity

![alt text](https://github.com/flux-capacitor-io/flux-capacitor-io.github.io/raw/master/dist/img/simplicity2.jpg "Connection simplicity")

Because your services only connect with Flux Capacitor, normal microservice setups become much more simple. 
Same functionality, a lot less overhead.
"How" you share data is fully covered by our service and client library, you only need to think of "what" to share.

This way of sharing data also removes the need for:
* a highly-available security service to verify that incoming traffic is authenticated
* load balancers and service registries, to guarantee availability
* ddos protection, firewall


* circuit-breakers and retry-mechanisms

We are able to fix these concerns once in the place where they belong, in a central service.

### 1.3 Load balancing

![alt text](https://github.com/flux-capacitor-io/flux-capacitor-io.github.io/raw/master/dist/img/Loadbalancer.jpg "Loadbalancing")

We balance load by spreading messages over segments. 
When you scale a service up to increase processing speed, we automatically rebalance the segments of messages given to each service.
Rebalancing is done asynchronous when a service is asking for a next batch of messages.

Most messages are given a semi-random segment to spread load. 
Some message however need to be processed by the exact same service. To get this done, you can specify a custom routing key in the message.
An example of this are messages related to event-sourcing, where messages about the same aggregate must be processed in the same order (See chapter 2 ).
Our client library makes these configurations as convenient as possible with simple annotations.
### 1.3 Consumer driven

Index per consumer (3 orderService, inventory service). Een service 2x deployed om te koppelen aan segmenten

### 1.4 Single threaded

elke tracker eigen thread

### 1.5 "The Flux Capacitor makes time travel possible"

### 1.6

## 1.1 Microclients

A microclient is what remains of a traditional microservices, once we migrated them to use a message service like Flux
Capacitor.

![alt text](https://github.com/flux-capacitor-io/flux-capacitor-io.github.io/raw/master/dist/img/Microservice%20vs%20client.jpg "Microservice vs client")

Compared to a microservice, a microclient does not need:

* A database
* Mapping to a database format
* Load balancing
* Api gateways
* Service registries
* Endpoints
* Security checks
* Direct connections to other services
* Circuit breakers or retry loops

Every microservice can be turned into a microclient, there is no loss of functionality. The only thing microclients do,
is read from and post to a message service. Getting messages from A to B, load balancing and persisting is done through
the message service with a fraction of the code.

Most companies we have seen, each dev team is responsible for a few up to 10 microservices. A small company with 10
teams will approach 100 separate microservices quick. This will be 100 separate databases and infrastructure setups, all
slightly different, all made by slightly different teams at slightly different times.

The most value you get from a move to microclients, is the removal of al these extra technical components to maintain,
and less technical code in the core.

## Event sourcing

Most applications use a database to keep track of the latest state of the application. Getting to that latest state
probably involved lots of tiny updates, e.g: a user signed up, an order got shipped, a complaint was filed, and so on.

What if we simply stored all these changes as events? Wouldn't we then be able to reconstruct the state of the
application at any point in time by replaying those events? Well yes: that's exactly the idea behind event sourcing.

Here's a chain of events for a given webshop order:

<img src="/assets/Event%20sourcing.jpg" alt="Event sourcing" title="Event sourcing" />

After the dust has settled it almost appears like nothing happened: the customer paid but got refunded, the shipped item
is back in inventory, and the webshop got paid but later reimbursed the customer. In reality a lot did happen, but it's
hard to squeeze this entire timeline into a database that only keeps track of the current state.

To fetch the order using event sourcing you would load these events and apply them one by one to recreate the order,
i.e: no need to store the order in a database. Event sourcing offers a lot of advantages compared to the traditional way
of storing data. For more in-depth explanations please refer to
[these](https://martinfowler.com/eaaDev/EventSourcing.html)
[excellent](https://docs.microsoft.com/en-us/azure/architecture/patterns/event-sourcing)
[articles](https://www.eventstore.com/blog/what-is-event-sourcing).

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
also add the event to the *global* event log, so any event consumers will be able to handle this event as well.

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

As you see we leave all the business logic up to you. We only tackle generic technical challenges like event sourcing,
so you can focus on what's actually important in your business.

What if you don't to event source this entity? That's easy too: just change `@Aggregate`
to `@Aggregate(eventSourced=false)` and we'll automatically store the latest state of the entity in Flux's key value
store. Boom! 
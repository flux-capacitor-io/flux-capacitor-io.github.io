<a href="flux-capacitor.io">
    <img src="https://flux-capacitor.io/assets/brand/flux-capacitor-white.svg" alt="Flux Capacitor logo" title="Flux Capacitor" align="right" height="60" />
</a>


Flux Capacitor service
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

Message routing is one of the core feature of Flux Capacitor. It consists of the following:

### 1.1 The essence

In its barest essence, Flux Capacitor contains message logs, and services can publish to or listen to these logs.

Microservices lose the direct dependency on one another. All communication is indirect and asynchronous.
When you would normally perform POST or GET API calls to a microservice, you now publish a message to Flux Capacitor that represents this POST or GET.
The response to this message is a message as well, and published in a similar manner.

![alt text](https://github.com/flux-capacitor-io/flux-capacitor-io.github.io/raw/master/dist/img/basics1.jpg "Basics")

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

##1.1 Microclients

A microclient is what remains of a traditional microservices, once we migrated them to use a message service like Flux Capacitor.

![alt text](https://github.com/flux-capacitor-io/flux-capacitor-io.github.io/raw/master/dist/img/Microservice%20vs%20client.jpg "Microservice vs client")

Compared to a microservice, a microclient does not need:
*  A database
*  Mapping to a database format   
*  Load balancing
*  Api gateways
*  Service registries
*  Endpoints
*  Security checks
*  Direct connections to other services
*  Circuit breakers or retry loops

Every microservice can be turned into a microclient, there is no loss of functionality. 
The only thing microclients do, is read from and post to a message service. 
Getting messages from A to B, load balancing and persisting is done through the message service with a fraction of the code.

Most companies we have seen, each dev team is responsible for a few up to 10 microservices. 
A small company with 10 teams will approach 100 separate microservices quick. 
This will be 100 separate databases and infrastructure setups, all slightly different, all made by slightly different teams at slightly different times.

The most value you get from a move to microclients, is the removal of al these extra technical components to maintain, 
and less technical code in the core.


## Event sourcing

## Microfunctions

Eventually we might be able to have you upload only your microfunctions, without even a need for a runnable microclient

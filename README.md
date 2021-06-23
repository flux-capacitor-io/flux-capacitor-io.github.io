<a href="flux-capacitor.io">
    <img src="https://flux-capacitor.io/assets/brand/flux-capacitor-white.svg" alt="Flux Capacitor logo" title="Flux Capacitor" align="right" height="60" />
</a>

Flux Capacitor service
======================

Flux Capacitor is a service that tackles one of the biggest problems in today's microservice jungle:
how to let applications communicate reliably and easily without the need for complex infrastructure
like message queues, service registries, api gateways etc.

Once your apps are connected to Flux Capacitor they can publish and subscribe to messages.
Messages come in distinct flavors:

* Queries: stuff you want to know, e.g. get me a user profile
* Commands: stuff you want do, e.g. update a password
* Events: stuff you want to announce, e.g. an email address was changed

Flux Capacitor distinguishes between these flavors and let's you easily

Because your apps are connected to Flux Capacitor and not to each other it is super easy
to add, remove or modify services. And if a service is getting too busy it is trivial to
scale the number of service instances: Flux Capacitor automatically takes care of the load balancing
between instances. All you need to do is deploy and run your instances.

Aside from routing messages between applications, Flux Capacitor also supports the following:

* _Event sourcing:_ published events about your domain models end up in their own event log.
To rebuild a domain model simply replay its events; no need to store your model in a database.
* _Auditing:_ all messages are stored in message logs by design. This makes it easy to see exactly what
happened in your application, from events to queries and even errors: everything is
available.
* _Scheduling:_ aside from sending messages immediately you can also schedule messages for the future.
* _Application metrics:_
* _Kick-ass performance:_ you can publish and read at speeds well over a million messages per second.
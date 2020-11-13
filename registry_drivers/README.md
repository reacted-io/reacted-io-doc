# Registry Drivers

ReActed allows you to use one or more service registries at the same time. 
In this way, ReActed can be used as hub or as a glue between different parts of your infrastructure. 
A service registry is used to make a node active and searchable in the cluster and to publish `Services`. 
Strictly speaking, any `ReActorRef` can be published for being contacted from a remote host, 
a [service](../services.md) is a nice abstraction that provides you out of the box a set extra features.

A [`ReActorSystem`](../reactor_system.md) name **must** be unique within a cluster. This means that its reactor system name
must be unique among all the reactor systems connected to the same service registries.

If a `Service` is marked as *remote* but no service registries are detected within the system, it will be automatically
made locally discoverable. If a service registry driver detects that the `ReActorSystem` to which it belongs to is not unique
within the cluster, it will deactivate itself. This is to prevent radical actions if, during a network outage, some other
misconfigured reactor system should register with the same name. This event can be intercepted with a [typed subscription](../subscriptions.md)
to the `DuplicatedPublicationError.class` message type.

On network failure, the driver deactivates all the [`active gates`](../channel_drivers/README.md) for the reactor systems gates managed by that driver.
This means that it will not be possible communicating anymore through **any new** `ReActorRef` pointing towards the deactivated gate. Every communication
attempt will result into a `DeliveryStatus.NOT_DELIVERED`.

!> For example, let's assume that the service registry suddenly goes offline. It will be possible keep using a `ReActorRef` to
a `Service` previously discovered, but if we should receive a message coming from one reactor system that was managed by the
offline service registry, it will not be possible replying.  




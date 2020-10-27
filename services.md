# Services

A `Service` is a `ReActor` with some out of the box behaviors that allow to easily and reliably a [published](registry_drivers/README.md) functionality.
This means that when publishing a `Service`, it will be automatically published on all the registered [service registries](channel_drivers/README.md), 
it will be automatically made [discoverable](#Service Discovery), [searchable](#Service Discovery), [updated](#Search Criteria) and [vertically scalable](#How a Service works).
A `Service` acts like a one-stop-shop a certain `ReActor` has to be published, *locally* or [*remotely*](services.md#Remote Services).
**Elastic horizontal scalability is planned, but not yet implemented.**  

## How a Service works

A `Service` acts as a message router. On creation, it [spawns](reactor.md) the configured number of **routees**. A routee is a `ReActor` whose
behavior has been defined by the user while configuring the `Service` itself. So, a `Service` does not have any business related behavior,
it only manages system tasks such as publication, [system statistics refresh](reactor_system.md#System Monitor), [discovery](#Service Discovery), [load balancing](#Logical load balancing) 
and routee manteinance. Once a message is sent to a `Service`, it *routes* the message towards one if its routees, according to the
configured load balancing policies.
A `Service` always tries to keep its configuration enforced. If it has been configured for having **N** routees, they are spawned on `Service` creation.
If any of those routees should stop for any reason, the `Service` will recognize such a situation and will attempt to re-spawn as many routee as required
to be consistent with its configuration. 

### Logical load balancing

Currently two logical load balancing policies are available: `Service.LoadBalancingPolicy.ROUND_ROBIN` and `Service.LoadBalancingPolicy.LOWEST_LOAD`.

`ROUND_ROBIN` policy will route every new message towards a different *routee* from the one chosen for the previous message sent to the `Service`.
`LOWEST_LOAD` policy will route every new messages towards the routee with the lowest number of messages in its mailbox.

### Selection policies

Given the described structure, where does a `Service` `ReActorRef` point to? To the container, the `Service` or to one of the *routees*?
It depends on the `SelectionType`. Currently supported only for [local services](#Local services), `SelectionType` allows you to
specify if the `ReActorRef` that should be returned by a successful [service discovery](#Service Discovery) should be a direct reference
to a routee (`SelectionType.DIRECT`) if you want to skip the extra hop with the risk to hammer on a overloaded routee, or instead a reference
to the router (`SelectionType.ROUTED`) with the benefits of [load balancing](#Logical load balancing), [resiliency](#How a Service works) and
[backpressuring](mailboxes.md#Backpressuring Mailbox) in case the `Service` has been configured to provide it. 

## Service Discovery

A service can be found because it reacts to `ServiceDiscoveryRequest` message. A [reactor system](reactor_system.md) offers out of the box
two commodity methods for locating a service.

With the first one 
```java
public CompletionStage<Try<ServiceDiscoveryReply>> serviceDiscovery(ServiceDiscoverySearchFilter searchFilter)
```
On completion, if successful, a `ServiceDiscoveryReply` will be returned, containing all the `ReActorRef` that have
been found that matched the specified research criteria.

With the second one
```java
public CompletionStage<Try<DeliveryStatus>> serviceDiscovery(ServiceDiscoverySearchFilter searchFilter, ReActorRef requester)
```
it's possible to specify a *destination* `ReActor` for the reception of the `ServiceDiscoveryReply`.

### Search Criteria

It's possible to define search criteria providing a `ServiceDiscoverySearchFilter`. This interface comes with the `BasicServiceDiscoverySearchFilter`
concrete and extension oriented implementation.

With a `BasicServiceDiscovertSearchFilter` is possible to set the following constraints while looking for a service:

```java
public final Builder setServiceName(String serviceName)
```  
Mandatory. The service name. It is going to be an exact match with the service name that has been exported on service creation/publication

```java
public final Builder setIpAddress(@Nullable InetAddress ipAddress)
```
Ip address where the service is runnin on

```java
public final Builder setHostNameExpression(@Nullable Pattern hostNameExpr)
```
A regexp for matching the service hostname

```java
public final Builder setCpuLoad(@Nullable Range<Double> cpuLoad)
```
A [Range](https://guava.dev/releases/30.0-jre/api/docs/com/google/common/collect/Range.html) of the average cpu load of the machine where the service is running on.
This value is periodically refreshed by the [system monitor](reactor_system.md#System Monitor)

```java
public final Builder setChannelId(Set<ChannelId> channelIdSet)
public final Builder setChannelId(ChannelId channelId)
```
The [channel id](channel_drivers/README.md) on which the service has to be exported. In case a `Service` has been exported over more than one *channel*,
with this fillter is possible specifiying the preferred one.

```java
public final Builder setSelectionType(SelectionType selectionType)
```

*Currently* supported only on [local services](#Local Services). Allows to specify if the returned `ReActorRef` should be [direct or routed references](#Selection policies).

## Local Services

A local `Service` is a service that is going to expose its functionalities only whithin the [reactor system](reactor_system.md) where it has been created.


## Backpressured service

## Remote Services
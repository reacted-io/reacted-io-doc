# Quickstart

If you are in a rush to be productive with ReActed and you do not need any detailed explanation, this section is for you. 
I will show a quick setup that will allow you to start experimenting.

As first thing, download the latest version of ReActed from [maven repository](https://mvnrepository.com/artifact/io.reacted/reacted-framework)

## Creating a ReActorSystem

ReActors live within a ReActorSystem, so the first thing to do is creating one. The very least thing that you can specify
for a [reactor system](reactor_system.md) is its name. Remember, a reactor system name **must be unique** in a cluster.

```java
import io.reacted.core.config.reactorsystem.ReActorSystemConfig;
import io.reacted.core.reactorsystem.ReActorSystem;

public class Quickstart {
    public static void main(String[] args) {
        ReActorSystem showOffSystem = new ReActorSystem(ReActorSystemConfig.newBuilder()
                                                                           .setReactorSystemName("ShowOffreActorSystemName")
                                                                           .build()).initReActorSystem();
    }
}
```
Always remember to init your reactor system once created.

## Creating a ReActor

Now you might want to create a reactor. From ReActed perspective a `ReActor` is not a special entity, it's not even
an entity to be fair. All what concerns ReActed are which `ReActions` should be called for a given type and some other
information that are nicely modeled by `ReActorConfig` regarding the runtime behavior. The `ReActor` or `ReActiveEntity`
interfaces are just a comfortable way to guide the user. You can *spawn* a reactor using the API call you prefer, but
as long as you can provide some `ReActions` and a `ReActorConfig` you are fine.

For this quickstart guide, let's create a `Hello World` reactor that greets the caller when receives a `GreetingsRequest` message.

```java
    private static final class Greeter implements ReActiveEntity {
        @Nonnull
        @Override
        public ReActions getReActions() {
            return ReActions.newBuilder()
                            .reAct(ReActorInit.class,
                                   ((reActorContext, init) -> reActorContext.logInfo("A reactor was born")))
                            .reAct(GreetingsRequest.class,
                                   ((raCtx, greetingsRequest) -> raCtx.reply("Hello from " +
                                                                             Greeter.class.getSimpleName())))
                            .build();
        }
    }
    private static final class GreetingsRequest implements Serializable { }
``` 

Putting it all together and *spawning* the above `ReActivEntity` then becomes like this:

```java
import io.reacted.core.config.reactors.ReActorConfig;
import io.reacted.core.config.reactorsystem.ReActorSystemConfig;
import io.reacted.core.messages.reactors.ReActorInit;
import io.reacted.core.reactors.ReActions;
import io.reacted.core.reactors.ReActiveEntity;
import io.reacted.core.reactorsystem.ReActorRef;
import io.reacted.core.reactorsystem.ReActorSystem;

import javax.annotation.Nonnull;
import java.io.Serializable;

public class Quickstart {
    public static void main(String[] args) {
        ReActorSystem showOffSystem = new ReActorSystem(ReActorSystemConfig.newBuilder()
                                                                           .setReactorSystemName("ShowOffreActorSystemName")
                                                                           .build()).initReActorSystem();
        try {
            ReActorRef greeter = showOffSystem.spawn(new Greeter(), 
                                                     ReActorConfig.newBuilder()
                                                                  .setReActorName("Greeter")
                                                                  .build()).orElseSneakyThrow();
        } catch (Exception anyException) {
            anyException.printStackTrace();
        }
        showOffSystem.shutDown();
    }

    private static final class Greeter implements ReActiveEntity {
        @Nonnull
        @Override
        public ReActions getReActions() {
            return ReActions.newBuilder()
                            .reAct(ReActorInit.class,
                                   ((reActorContext, init) -> reActorContext.logInfo("A reactor was born")))
                            .reAct(GreetingsRequest.class,
                                   ((raCtx, greetingsRequest) -> raCtx.reply("Hello from " +
                                                                             Greeter.class.getSimpleName())))
                            .build();
        }
    }
    private static final class GreetingsRequest implements Serializable { }
}
```
If we should try to run the above example, we would get the following output

```text
[ReActed-Dispatcher-Thread-ReactorSystemDispatcher-2] INFO io.reacted.core.reactors.systemreactors.SystemLogger - A reactor was born

Process finished with exit code 0
```
## Querying a ReActor

We said that we wanted our reactor to answer to some requests. We can query a reactor *asking* for something or letting
it talking freely with another `ReActor`. Let's `ask` for a greet as first thing:

```java
greeter.ask(new GreetingsRequest(), String.class, "Introduction request")
       .toCompletableFuture()
       .join()
       .ifSuccessOrElse(showOffSystem::logInfo, Throwable::printStackTrace);
```
[Ask](messaging.md#Ask) allows to receive a message from a `ReActor` from outside the scope of a [ReAction](reactor.md)

Adding the above request just after the raction creation and running the program, the output becomes:

```text
[ReActed-Dispatcher-Thread-ReactorSystemDispatcher-2] INFO io.reacted.core.reactors.systemreactors.SystemLogger - A reactor was born
[ReActed-Dispatcher-Thread-ReactorSystemDispatcher-2] INFO io.reacted.core.reactors.systemreactors.SystemLogger - Hello from Greeter

Process finished with exit code 0
```

## Greetings service

Now let's transform this `Greeter` into a [Service](services.md). As a service it can be [discovered](services.md#Service-Discovery)
and queried without previously having its `ReActorRef`, like we did in the example above.

```java
import io.reacted.core.config.reactors.ReActorConfig;
import io.reacted.core.config.reactors.TypedSubscriptionPolicy;
import io.reacted.core.config.reactorsystem.ReActorSystemConfig;
import io.reacted.core.messages.reactors.ReActorInit;
import io.reacted.core.messages.services.ServiceDiscoveryRequest;
import io.reacted.core.reactors.ReActions;
import io.reacted.core.reactors.ReActor;
import io.reacted.core.reactorsystem.ReActorSystem;
import io.reacted.core.reactorsystem.ServiceConfig;

import javax.annotation.Nonnull;
import java.io.Serializable;

public class Quickstart {
    public static void main(String[] args) {
        ReActorSystem showOffSystem = new ReActorSystem(ReActorSystemConfig.newBuilder()
                                                                           .setReactorSystemName("ShowOffreActorSystemName")
                                                                           .build()).initReActorSystem();
        try {
            String serviceName = "Greetings";
            showOffSystem.spawnService(ServiceConfig.newBuilder()
                                                    .setReActorName(serviceName)
                                                    .setRouteesNum(2)
                                                    .setRouteeProvider(Greeter::new)
                                                    .build()).orElseSneakyThrow();
        } catch (Exception anyException) {
            anyException.printStackTrace();
        }
        showOffSystem.shutDown();
    }

    private static final class Greeter implements ReActor {

        @Nonnull
        @Override
        public ReActorConfig getConfig() {
            return ReActorConfig.newBuilder()
                                .setReActorName("Worker")
                                .build();
        }

        @Nonnull
        @Override
        public ReActions getReActions() {
            return ReActions.newBuilder()
                            .reAct(ReActorInit.class,
                                   ((reActorContext, init) -> reActorContext.logInfo("A reactor was born")))
                            .reAct(GreetingsRequest.class,
                                   ((raCtx, greetingsRequest) -> raCtx.reply("Hello from " +
                                                                             Greeter.class.getSimpleName())))
                            .build();
        }
    }
    private static final class GreetingsRequest implements Serializable { }
}
```

Notice that we changed the [service replica factor](services.md#How-a-Service-works) to `2`. For this reason, the
output of the above program is

```text
[ReActed-Dispatcher-Thread-ReactorSystemDispatcher-0] INFO io.reacted.core.reactors.systemreactors.SystemLogger - A reactor was born
[ReActed-Dispatcher-Thread-ReactorSystemDispatcher-0] INFO io.reacted.core.reactors.systemreactors.SystemLogger - A reactor was born

Process finished with exit code 0
```

## Querying Greetings Service

For querying a `Service` a `ReActorRef` is required. We must then make it *discoverable* allowing it to receive to [service discovery](services.md#Service-Discovery) requests.

```java
.setTypedSubscriptions(TypedSubscriptionPolicy.LOCAL.forType(ServiceDiscoveryRequest.class))
```
Just adding a [local subscription](subscriptions.md#Local-Subscription) for the `ServiceDiscoveryRequest` message type is enough.

For discovering it we simply define a [search filter](services.md#Search-Criteria) and perform a search:

```java
    public static void main(String[] args) {
        ReActorSystem showOffSystem = new ReActorSystem(ReActorSystemConfig.newBuilder()
                                                                           .setReactorSystemName("ShowOffreActorSystemName")
                                                                           .build()).initReActorSystem();
        try {
            String serviceName = "Greetings";
            showOffSystem.spawnService(ServiceConfig.newBuilder()
                                                    .setReActorName(serviceName)
                                                    .setRouteesNum(2)
                                                    .setTypedSubscriptions(TypedSubscriptionPolicy.LOCAL.forType(ServiceDiscoveryRequest.class))
                                                    .setRouteeProvider(Greeter::new)
                                                    .build())
                         .orElseSneakyThrow();

            var searchFilter = BasicServiceDiscoverySearchFilter.newBuilder()
                                                                .setServiceName(serviceName)
                                                                .build();
            var serviceDiscoveryReply = showOffSystem.serviceDiscovery(searchFilter)
                                                     .toCompletableFuture()
                                                     .join()
                                                     .orElseSneakyThrow();
            
            if (serviceDiscoveryReply.getServiceGates().isEmpty()) {
                showOffSystem.logInfo("No services found, exiting");
            } else {
                var serviceGate = serviceDiscoveryReply.getServiceGates().iterator().next();
                serviceGate.ask(new GreetingsRequest(), String.class, "Request to service")
                           .toCompletableFuture()
                           .join()
                           .ifSuccessOrElse(showOffSystem::logInfo, Throwable::printStackTrace);
            }
        } catch (Exception anyException) {
            anyException.printStackTrace();
        }
        showOffSystem.shutDown();
    }
``` 

We made it synchronous for clarity, but could have been done with an async pipeline.
Using the `Greeter` `ReActor` from above,  output of this above program is:

```text
[ReActed-Dispatcher-Thread-ReactorSystemDispatcher-3] INFO io.reacted.core.reactors.systemreactors.SystemLogger - A reactor was born
[ReActed-Dispatcher-Thread-ReactorSystemDispatcher-3] INFO io.reacted.core.reactors.systemreactors.SystemLogger - A reactor was born
[ReActed-Dispatcher-Thread-ReactorSystemDispatcher-0] INFO io.reacted.core.reactors.systemreactors.SystemLogger - Hello from Greeter

Process finished with exit code 0
```
## Publishing Greetings Service on a cluster

The only thing that ReActed needs for clustering is a [service registry driver](registry_drivers/README.md) and a [channel](channel_drivers/README.md) for
communicating with other [reactor systems](reactor_system.md). Let's publish `Greetings` service on a cluster over Kafka.

You can quickly setup a Kafka instance using [this tutorial](https://kafka.apache.org/quickstart).

!> NOTE: ReActed [ZooKeeper driver](registry_drivers/zookeeper/zookeeper_main.md) is compatible ONLY with ZooKeeper 3.6+ 

Once Kafka it's up and running we need to:
 - setup [ZooKeeper server registry driver](registry_drivers/zookeeper/zookeeper_main.md). 
 - setup [kafka channel driver](channel_drivers/kafka/kafka_main.md)
 - marke the `Service` as remotely discoverable

```java
    public static void main(String[] args) {
        var zooKeeperDriverConfig = ZooKeeperDriverConfig.newBuilder()
                                                         .setReActorName("LocalhostCluster")
                                                         .build();
        var kafkaDriverConfig = KafkaDriverConfig.newBuilder()
                                                 .setChannelName("KafkaQuickstartChannel")
                                                 .setTopic("ReactedTopic")
                                                 .setGroupId("QuickstartGroupServer")
                                                 .setMaxPollRecords(1)
                                                 .setBootstrapEndpoint("localhost:9092")
                                                 .build();
        ReActorSystem showOffSystem = new ReActorSystem(ReActorSystemConfig.newBuilder()
                                                                           .addServiceRegistryDriver(new ZooKeeperDriver(zooKeeperDriverConfig))
                                                                           .addRemotingDriver(new KafkaDriver(kafkaDriverConfig))
                                                                           .setReactorSystemName("ShowOffreActorSystemName")
                                                                           .build()).initReActorSystem();
        try {
            String serviceName = "Greetings";
            showOffSystem.spawnService(ServiceConfig.newBuilder()
                                                    .setReActorName(serviceName)
                                                    .setRouteesNum(2)
                                                    .setRouteeProvider(Greeter::new)
                                                    .setIsRemoteService(true)
                                                    .build())
                         .orElseSneakyThrow();
            TimeUnit.SECONDS.sleep(10);
        } catch (Exception anyException) {
            anyException.printStackTrace();
        }
        showOffSystem.shutDown();
    }
```

Launching it after all the log lines from Zookeeper and Kafka drivers, we get

```text
[ReActed-Dispatcher-Thread-ReactorSystemDispatcher-3] INFO io.reacted.core.reactors.systemreactors.SystemLogger - Service Greetings published
```
It's perfectly normal seeing more lines like this, it's the [system monitor](reactor_system.md#System-Monitor) that is refreshing the service statistics
and the [remoting root reactor](reactor_system.md#System-Remoting-Root) propagates these information towards the [service registry drivers](registry_drivers/README.md) that
*react* refreshing the publications.

## Querying a published service

The code in the previous section, publish a `Service` for ten seconds, then it shuts down the system. Now let's create
a simple reacted cluster over kafka. First, we need to extract the `Service` from the *server* class.

```java
import io.reacted.core.config.reactors.ReActorConfig;
import io.reacted.core.messages.reactors.ReActorInit;
import io.reacted.core.reactors.ReActions;
import io.reacted.core.reactors.ReActor;
import io.reacted.patterns.NonNullByDefault;

import javax.annotation.Nonnull;
import java.io.Serializable;

@NonNullByDefault
final class GreeterService implements ReActor {

    @Nonnull
    @Override
    public ReActorConfig getConfig() {
        return ReActorConfig.newBuilder()
                            .setReActorName("Worker")
                            .build();
    }

    @Nonnull
    @Override
    public ReActions getReActions() {
        return ReActions.newBuilder()
                        .reAct(ReActorInit.class, 
                               (raCtx, init) ->
                                       raCtx.logInfo("{} was born", raCtx.getSelf().getReActorId().getReActorName()))
                        .reAct(GreetingsRequest.class,
                               (raCtx, greetingsRequest) -> raCtx.reply("Hello from " + 
                                                                        GreeterService.class.getSimpleName()))
                        .build();
    }
    static final class GreetingsRequest implements Serializable { }
}
```

Then, we remove the shutdown from the server:

```java
import io.reacted.core.config.reactorsystem.ReActorSystemConfig;
import io.reacted.core.drivers.local.SystemLocalDrivers;
import io.reacted.core.reactorsystem.ReActorSystem;
import io.reacted.core.reactorsystem.ServiceConfig;
import io.reacted.drivers.channels.kafka.KafkaDriver;
import io.reacted.drivers.channels.kafka.KafkaDriverConfig;
import io.reacted.drivers.serviceregistries.zookeeper.ZooKeeperDriver;
import io.reacted.drivers.serviceregistries.zookeeper.ZooKeeperDriverConfig;
import io.reacted.patterns.NonNullByDefault;

@NonNullByDefault
public class QuickstartServer {
    public static void main(String[] args) {
        var zooKeeperDriverConfig = ZooKeeperDriverConfig.newBuilder()
                                                         .setReActorName("LocalhostCluster")
                                                         .build();
        var kafkaDriverConfig = KafkaDriverConfig.newBuilder()
                                                 .setChannelName("KafkaQuickstartChannel")
                                                 .setTopic("ReactedTopic")
                                                 .setGroupId("QuickstartGroupServer")
                                                 .setMaxPollRecords(1)
                                                 .setBootstrapEndpoint("localhost:9092")
                                                 .build();
        var showOffServerSystem = new ReActorSystem(ReActorSystemConfig.newBuilder()
                                                                       .addServiceRegistryDriver(new ZooKeeperDriver(zooKeeperDriverConfig))
                                                                       .addRemotingDriver(new KafkaDriver(kafkaDriverConfig))
                                                                       .setReactorSystemName("ShowOffServerActorSystemName")
                                                                       .build()).initReActorSystem();
        try {
            String serviceName = "Greetings";
            showOffServerSystem.spawnService(ServiceConfig.newBuilder()
                                                          .setReActorName(serviceName)
                                                          .setRouteesNum(2)
                                                          .setRouteeProvider(GreeterService::new)
                                                          .setIsRemoteService(true)
                                                          .build())
                               .orElseSneakyThrow();
        } catch (Exception anyException) {
            anyException.printStackTrace();
            showOffServerSystem.shutDown();
        }
    }
}
```
In the end, we have the client application

```java
import io.reacted.core.config.reactorsystem.ReActorSystemConfig;
import io.reacted.core.drivers.local.SystemLocalDrivers;
import io.reacted.core.messages.services.BasicServiceDiscoverySearchFilter;
import io.reacted.core.reactorsystem.ReActorSystem;
import io.reacted.drivers.channels.kafka.KafkaDriver;
import io.reacted.drivers.channels.kafka.KafkaDriverConfig;
import io.reacted.drivers.serviceregistries.zookeeper.ZooKeeperDriver;
import io.reacted.drivers.serviceregistries.zookeeper.ZooKeeperDriverConfig;
import io.reacted.patterns.NonNullByDefault;

@NonNullByDefault
public class QuickstartClient {
    public static void main(String[] args) {
        var zooKeeperDriverConfig = ZooKeeperDriverConfig.newBuilder()
                                                         .setReActorName("LocalhostCluster")
                                                         .build();
        var kafkaDriverConfig = KafkaDriverConfig.newBuilder()
                                                 .setChannelName("KafkaQuickstartChannel")
                                                 .setTopic("ReactedTopic")
                                                 .setGroupId("QuickstartGroupClient")
                                                 .setMaxPollRecords(1)
                                                 .setBootstrapEndpoint("localhost:9092")
                                                 .build();
        var showOffClientSystem = new ReActorSystem(ReActorSystemConfig.newBuilder()
                                                                       .addServiceRegistryDriver(new ZooKeeperDriver(zooKeeperDriverConfig))
                                                                       .addRemotingDriver(new KafkaDriver(kafkaDriverConfig))
                                                                       .setReactorSystemName("ShowOffClientReActorSystemName")
                                                                       .build()).initReActorSystem();
        String serviceName = "Greetings";
        var searchFilter = BasicServiceDiscoverySearchFilter.newBuilder()
                                                            .setServiceName(serviceName)
                                                            .build();
        var serviceDiscoveryReply = showOffClientSystem.serviceDiscovery(searchFilter)
                                                       .toCompletableFuture()
                                                       .join()
                                                       .orElseSneakyThrow();

        if (serviceDiscoveryReply.getServiceGates().isEmpty()) {
            showOffClientSystem.logInfo("No services found, exiting");
        } else {
            var serviceGate = serviceDiscoveryReply.getServiceGates().iterator().next();
            serviceGate.ask(new GreeterService.GreetingsRequest(), String.class, "Request to service")
                       .toCompletableFuture()
                       .join()
                       .ifSuccessOrElse(showOffClientSystem::logInfo, Throwable::printStackTrace);
        }
        showOffClientSystem.shutDown();
    }
}
```
Assuming a working Kafka setup, we launch the server and then the client. In client's logs we can see

```text
[ReActed-Dispatcher-Thread-ReactorSystemDispatcher-0] INFO io.reacted.core.reactors.systemreactors.SystemLogger - Hello from GreeterService
```

## Backpressuring

Since everyone would obviously be interested in such a service, we must think about backpressuring.
Changing the mailbox provider of the `Service` like this

```java
.setMailBoxProvider(BackpressuringMbox::newDefaultMailBox)
```

would automatically start [dropping requests](mailboxes.md#Backpressuring-Mailbox) if the service cannot keep up with them.

## Remote message subscription

In our infrastructure now we have a client that creates requests and sends them over Kafka, and a server that replies.
Now we want to expand our microservices oriented infrastructure because suddenly we found out that keeping statistics
about the `GreetingsRequets` is crucial. More precisely, we want to be able to react every time a new `GreetingsRequest`
is sent, but without touching the previously deployed `QuickstartService`. 
With ReActed we can easily add to our choreography a *passive listener*, or with ReActed terminology, a [typed subscriber](subscriptions.md).
In this scenario the new microservice is not going to be directly involved into the communication, it simply has to be triggered
whenever a new message is sent. All what we have to do is to connect our [subscriber](subscriptions.md#Full-Subscription-Use-Case)
to the same [channel](channel_drivers/kafka/kafka_main.md) and setup a `ReActor` with a `TypedSubscriptionPolicy.FULL` for 
the message we are interested in. Alternatively, we can create an unpublished *subscribed [service](services.md)* to automatically parallelize
the handling of the intercepted messages.

```java
import io.reacted.core.config.reactors.ReActorConfig;
import io.reacted.core.config.reactors.ServiceConfig;
import io.reacted.core.reactors.ReActor;
import io.reacted.core.typedsubscriptions.TypedSubscription;
import io.reacted.core.config.reactorsystem.ReActorSystemConfig;
import io.reacted.core.reactors.ReActions;
import io.reacted.core.reactorsystem.ReActorSystem;
import io.reacted.drivers.channels.kafka.KafkaDriver;
import io.reacted.drivers.channels.kafka.KafkaDriverConfig;
import io.reacted.patterns.NonNullByDefault;

import javax.annotation.Nonnull;

@NonNullByDefault
public class QuickstartSubscriber {
    public static void main(String[] args) {
        var kafkaDriverConfig = KafkaDriverConfig.newBuilder()
                                                 .setChannelName("KafkaQuickstartChannel")
                                                 .setTopic("ReactedTopic")
                                                 .setGroupId("QuickstartGroupSubscriber")
                                                 .setMaxPollRecords(1)
                                                 .setBootstrapEndpoint("localhost:9092")
                                                 .build();
        var showOffSubscriberSystem = new ReActorSystem(ReActorSystemConfig.newBuilder()
                                                                           .addRemotingDriver(new KafkaDriver(kafkaDriverConfig))
                                                                           .setReactorSystemName("ShowOffSubscriberReActorSystemName")
                                                                           .build()).initReActorSystem();
        showOffSubscriberSystem.spawnService(ServiceConfig.newBuilder()
                                                          .setRouteeProvider(GreetingsRequestSubscriber::new)
                                                          .setReActorName("DataCaptureService")
                                                          .setRouteesNum(4)
                                                          .setTypedSubscriptions(TypedSubscription.FULL.forType(GreeterService.GreetingsRequest.class))
                                                          .build())
                               .peekFailure(Throwable::printStackTrace)
                               .ifError(error -> showOffSubscriberSystem.shutDown());
    }

    private static class GreetingsRequestSubscriber implements ReActor {
        @Nonnull
        @Override
        public ReActorConfig getConfig() {
            return  ReActorConfig.newBuilder()
                                 .setReActorName("Worker")
                                 .build();
        }

        @Nonnull
        @Override
        public ReActions getReActions() {
            return ReActions.newBuilder()
                            .reAct(GreeterService.GreetingsRequest.class,
                                   (raCtx, greetingsRequest) ->
                                           raCtx.logInfo("{} intercepted {}",
                                                         raCtx.getSelf().getReActorId().getReActorName(),
                                                         greetingsRequest))
                            .build();
        }
    }
}
```
Running all together and performing some requests from the client, will generate this output from the subscriber

```text
[ReActed-Dispatcher-Thread-ReactorSystemDispatcher-1] INFO io.reacted.core.reactors.systemreactors.SystemLogger - [DataCaptureService-Worker-2] intercepted io.reacted.examples.quickstart.GreeterService$GreetingsRequest@2b7542d2
[ReActed-Dispatcher-Thread-ReactorSystemDispatcher-1] INFO io.reacted.core.reactors.systemreactors.SystemLogger - [DataCaptureService-Worker-3] intercepted io.reacted.examples.quickstart.GreeterService$GreetingsRequest@311368b7
[ReActed-Dispatcher-Thread-ReactorSystemDispatcher-1] INFO io.reacted.core.reactors.systemreactors.SystemLogger - [DataCaptureService-Worker-0] intercepted io.reacted.examples.quickstart.GreeterService$GreetingsRequest@3071cf95
[ReActed-Dispatcher-Thread-ReactorSystemDispatcher-1] INFO io.reacted.core.reactors.systemreactors.SystemLogger - [DataCaptureService-Worker-1] intercepted io.reacted.examples.quickstart.GreeterService$GreetingsRequest@2c084893
[ReActed-Dispatcher-Thread-ReactorSystemDispatcher-1] INFO io.reacted.core.reactors.systemreactors.SystemLogger - [DataCaptureService-Worker-2] intercepted io.reacted.examples.quickstart.GreeterService$GreetingsRequest@314cb92a
[ReActed-Dispatcher-Thread-ReactorSystemDispatcher-1] INFO io.reacted.core.reactors.systemreactors.SystemLogger - [DataCaptureService-Worker-3] intercepted io.reacted.examples.quickstart.GreeterService$GreetingsRequest@1b302964
[ReActed-Dispatcher-Thread-ReactorSystemDispatcher-1] INFO io.reacted.core.reactors.systemreactors.SystemLogger - [DataCaptureService-Worker-0] intercepted io.reacted.examples.quickstart.GreeterService$GreetingsRequest@250350f0
[ReActed-Dispatcher-Thread-ReactorSystemDispatcher-1] INFO io.reacted.core.reactors.systemreactors.SystemLogger - [DataCaptureService-Worker-1] intercepted io.reacted.examples.quickstart.GreeterService$GreetingsRequest@4b21d813
```
The messages have been successfully intercepted and processed by routees in round robin
                                                          


 

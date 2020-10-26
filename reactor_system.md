# ReActorSystem

## What is a ReActorSystem

A `ReActorSystem` is the runtime container of a `ReActor`. A `ReActor` exists only within a certain reactor system and
cannot be moved. A `ReActorSystem` contains everything is related to a `ReActor` and to the manteinance of the running
system. [Drivers](channel_drivers/README.md), [dispatchers](dispatcher.md) and everything else is registered in and local
to a reactor system. ReActors on different reactor systems can talk to each other using [messages](messaging.md)

## ReActorSystem structure

```mermaid
graph System Hierarchy Structure
    Init --> ReActor System Root
    ReActor System Root --> System Remoting Root
    ReActor System Root --> System ReActors Root
    System ReActors Root --> DeadLetters
    System ReActors Root --> System Monitor
    System ReActors Root --> System Logger
    ReActor System Root --> User ReActors Root
    Note right of User ReActors Root: Every user defined reactor is going to be created below this point
```

### System reactors

## Configure a ReActorSystem

A `ReActorSystem` can be created using a `ReActorSystemConfig` as an argument. 

```java
 public Builder setReactorSystemName(String reactorSystemName) 
```
Mandatory. Specifies the name of the `ReActorSsystem` in a [cluster](clustering.md). Each `ReActorSystem` name **must**
be unique within a cluster.

```java
 public Builder setMsgFanOutPoolSize(int msgFanOutPoolSize)
```
Every time a message is sent to a `ReActor` within the `ReActorSystem` a fan out process take place. This fan out process
is responsible for delivering messages to [typed subscribers](subscriptions.md). This parameter define how many threads
there are in the thread pool used for the fan out.

```java
 public Builder setRecordExecution(boolean shallRecordExecution)
```
Enable or disables the generation of the execution information required for [replaying](replaying.md).

```java
 public Builder setLocalDriver(LocalDriver<? extends ChannelDriverConfig<?, ?>> localDriver)
```
Defines which [local driver](channel_drivers/README.md) should be used for the local delivery of messages.

```java
 public Builder setSystemMonitorRefreshInterval(Duration refreshInterval)
```
[Services](services.md) by default receive information about the status of the current system. This because in this way
they can publish such statistics on a [service registry](registry_drivers/README.md) and service discovery queries based
on those values can be performed. This option specifies how often those statistics should be refreshed.

```java
 public Builder addDispatcherConfig(DispatcherConfig dispatcherConfig)
```  
Allows you to define a new [dispatcher](dispatcher.md). Up to `ReActorSystemConfig.MAX_DISPATCHERS_CONFIGS` are allowed.
Defining extra dispatchers is an effective way to [separate the load type](patterns.md#Managing blocking operations) within a reactor system.

```java
 public Builder addRemotingDriver(RemotingDriver<? extends ChannelDriverConfig<?, ?>> remotingDriver)
```
Allows you to register a new [remoting channel](channel_drivers/README.md). A new driver should explicitly registered for each
*channel* that should be supported.

```java
 public Builder addServiceRegistryDriver(ServiceRegistryDriver<? extends ServiceRegistryConfig.Builder<?, ?>,
                                         ? extends ServiceRegistryConfig<?, ?>> serviceRegistryDriver)
```
Allows to register a new [service registry](registry_drivers/README.md) driver. A new driver should be registered for
each new configuration of every different [service registry type](registry_drivers/zookeeper/zookeeper_main.md)
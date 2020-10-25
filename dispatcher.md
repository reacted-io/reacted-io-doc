# Dispatcher 

A dispatcher is a dedicated thread pool on which `ReActions` are executed. It performs *reactors scheduling* and 
*messages load balancing*. Once a reactor has been scheduled, messages are sequentially processed from its [mailbox](mailboxes.md).
A reactor is guaranteed to be scheduled at maximum on one dispatcher at a time. This means that its `ReActions` could
be executed on different threads. A dispatcher takes care of ensuring memory consistency for reactions executed within
its scope. When a `ReAction` throws an exception, the dispatcher intercepts it and triggers the termination of the reactor
that caused it. Dispatchers process a configurable amount of messages from a given [mailbox](mailboxes.md) before yielding the
cpu to another reactor. Changing this value can result in increasing the throughput or increasing the reactiveness
of the system.
Since the dispatcher is the computing core component of **ReActed** it is **mandatory** that it never blocks.
All the operations that could block should be done asynchronously from the [reactors perspective](patterns.md#Managing blocking operations).
**ReActed** provides a default dispatcher called `ReactorSystemDispatcher`. A `ReActorSystem` can be configured to hold
different dispatchers with different configurations and through a `ReActiveEntityConfig` we can specify which dispatcher
should be used for a given reactor or [service](services.md). A dispatcher name must be unique within a `ReActorSystem`.

## Configuration

```java
DispatcherConfig.newBuilder()
                .setDispatcherName("Dispatcher")
```
The dispatcher name. It has to be unique within the `ReActorSystem`. A `ReActiveEntity` can be specified to run within
a specific dispatcher specifying this name.
```java

                .setBatchSize(1_000)
```
Batch size, how many messages should be processed at maximum per each reactor before yielding the dispatcher
```java
                .setDispatcherThreadsNum(1)
                .build()
```
How many threads should be included in the dispatcher




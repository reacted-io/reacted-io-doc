# ReActor

A replayable actor or `reactor`, is a reactive entity that has a state, a [mailbox](mailboxes.md) and some 
*reactions* that map a received *message* type to a certain action. ReActors communicate with each other using messages:
when a message is received by a reactor, it is saved within its mailbox waiting for the proper `reaction` to be called. 
The messages are processed strictly sequentially and in the order specified by the mailbox. From the programmer perspective,
this cycle is completely single threaded, removing any need for synchronization or thread safe operations.

## Hierarchy and Life Cycle

Every reactor is part of a hierarchy: it has a single and immutable father during all its lifecycle and can have as many
children as required. If a child terminates it has not impact on the father reactor, but if a father terminates then
the termination will be propagated to all its children, covering the whole hierarchy.

Every reactor has a lifecycle divided into three phases: Init, Main, Stop. 

### Init Phase

On reactor creation, the ReActed automatically sends a `ReActorInit` message to the newborn. This message is always
the first message that will be received by any reactor. The behavior associated with the `ReActorInit` message
can be specified within the reactor's reactions.

### Main Phase

Once that a reactor has been inited, the main phase begins. In this phase the reactions for any message that is found
in the mailbox are sequentially called. 

### Stop Phase

In any of the above phases, a reactor can be requested to stop. The stop request is *dispatched* immediately,
without processing any further message present in the mailbox. On termination `ReActorStop` message is processed by the 
terminating reactor, so the appropriate behavior can be customized providing the appropriate reaction in the reactor's
configuration 

>NOTE: A reactor termination has a top down behavior: once a reactor has been terminated, all its children will be
>recursively terminated as well. The ratio behind this is that a child can be terminated when there is nothing waiting 
>for it, otherwise we could kill something required by a still operating father reactor. *The termination will be 
>considered and marked as completed when the whole hierarchy has been terminated*.  

## Creating a ReActor

Let's see the first example of a reactor.

```java
import io.reacted.core.config.reactors.ReActorConfig;
import io.reacted.core.messages.reactors.ReActorInit;
import io.reacted.core.messages.reactors.ReActorStop;
import io.reacted.core.reactors.ReActions;
import io.reacted.core.reactors.ReActor;
import io.reacted.core.reactorsystem.ReActorContext;
import org.jetbrains.annotations.NotNull;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import java.io.Serializable;
import java.time.Instant;

public class FirstReActor implements ReActor {
    private static final Logger LOGGER = LoggerFactory.getLogger(FirstReActor.class);
    private int receivedMsgs;

    @NotNull
    @Override
    public ReActions getReActions() {
        return ReActions.newBuilder()
                        .reAct(ReActorInit.class, this::onReActorInit)
                        .reAct(ReActorStop.class, this::onReActorStop)
                        .reAct(this::onAnyOtherMessageType)
                        .build();
    }

    @NotNull
    @Override
    public ReActorConfig getConfig() {
        return ReActorConfig.newBuilder()
                            .setReActorName(FirstReActor.class.getSimpleName())
                            .build();
    }

    private void onReActorInit(ReActorContext raCtx, ReActorInit init) {
        raCtx.getReActorSystem().logDebug("Init received");
    }

    private void onReActorStop(ReActorContext raCtx, ReActorStop stop) {
        LOGGER.debug("Termination requested after {} messages", receivedMsgs);
    }

    private void onAnyOtherMessageType(ReActorContext raCtx, Serializable message) {
        this.receivedMsgs++;
        System.out.printf("Received message type %s%n", message.getClass().getSimpleName());
        if (Instant.now().toEpochMilli() % this.receivedMsgs == 0) {
            raCtx.stop()
                 .toCompletableFuture()
                 .thenAcceptAsync(noVal -> raCtx.getReActorSystem()
                                                .logDebug("ReActor %s hierarchy is terminated",
                                                          raCtx.getSelf().getReActorId()
                                                                         .getReActorName()));
        }
    }
}
```
The above reactor prints a statement every time a message is processed.  Every time a message is received, we check
if the time is a multiple of the already received messages num and in that case we request a termination of the
reactor. Once the termination is completed, a line is printed containing the name of the reactor that was just terminated.
 
From the above code, we can quickly notice that:
- Any class can become a reactor simply implementing the `ReActor` interface. It's not mandatory for a class to
implement this interface to be used as a reactor: as long as the `ReActions` and a `ReActorConfig` are provided on
creation, a reactor can be created.
- It's possible to setup reactions for a message type or providing a wildcard reaction. Pattern matching for message
types are done on the exact type.
- Messages **must** implement the `Serializable` interface. This because the communication between reactors is location
and technology agnostic, so a message can always virtually go over some other media than shared memory
- It's mandatory *naming* a reactor. We can have default values for the other configuration properties, but
a reactor always needs a unique name among its alive siblings. If interested in using the [replay](replay.md) feature,
a reactor name should always be non randomic
- ReActed offers a [centralized logging system](centralized_logger.md)
- No synchronization primitives are required to ensure the correct updated of the counter, regardless of how many threads 
are sending messages as the same time
- `ReActorContext` provides a set of method useful to control the reactor behavior and to access its internal properties
- We can attach an async operation to be executed after that the whole hierarchy has been terminated chaining to the
completion stage returned by the stop operation

## ReActors communication

ReActors talk to each other through [messaging](messaging.md). Let's create now a couple of reactors that talk to each
other performing a simple ping pong.

```java
import io.reacted.core.config.reactors.ReActorConfig;
import io.reacted.core.config.reactorsystem.ReActorSystemConfig;
import io.reacted.core.messages.reactors.DeliveryStatus;
import io.reacted.core.messages.reactors.ReActorInit;
import io.reacted.core.reactors.ReActions;
import io.reacted.core.reactors.ReActiveEntity;
import io.reacted.core.reactorsystem.ReActorContext;
import io.reacted.core.reactorsystem.ReActorRef;
import io.reacted.core.reactorsystem.ReActorSystem;
import org.jetbrains.annotations.NotNull;

import java.util.concurrent.TimeUnit;

public class PingPongExample {
    public static void main(String[] args) throws InterruptedException {
        ReActorSystem exampleReActorSystem = new ReActorSystem(ReActorSystemConfig.newBuilder()
                                                                                  .setReactorSystemName("ExampleSystem")
                                                                                  .build()).initReActorSystem();

        ReActorRef pongReactor = exampleReActorSystem.spawnReActor(ReActions.newBuilder()
                                                                            .reAct(String.class,
                                                                                   PingPongExample::onPing)
                                                                            .reAct(ReActions::noReAction)
                                                                            .build(),
                                                                   ReActorConfig.newBuilder()
                                                                                .setReActorName("Pong")
                                                                                .build())
                                                     .peekFailure(error -> exampleReActorSystem.shutDown())
                                                     .orElseSneakyThrow();

        exampleReActorSystem.spawnReActor(new ReActiveEntity() {
                                            private long pingNum = 0;
                                            @NotNull
                                            @Override
                                            public ReActions getReActions() {
                                                return ReActions.newBuilder()
                                                                .reAct(ReActorInit.class, 
                                                                       ((ractx, init) -> pongReactor.tell(ractx.getSelf(),
                                                                                                          "FirstPing")))
                                                                .reAct(String.class, 
                                                                       (raCtx, pongPayload) -> onPong(raCtx, pongPayload, 
                                                                                                      ++pingNum))
                                                                .reAct(ReActions::noReAction)
                                                                .build();
                                                
                                            };
                                          },
                                          ReActorConfig.newBuilder()
                                                       .setReActorName("Ping")
                                                       .build())
                            .peekFailure(error -> exampleReActorSystem.shutDown())
                            .orElseSneakyThrow();

        TimeUnit.MILLISECONDS.sleep(3);
        exampleReActorSystem.shutDown();
    }

    private static void onPing(ReActorContext raCtx, String pingMessage) {
        raCtx.getSender().tell(raCtx.getSelf(), "Pong " + pingMessage)
                         .toCompletableFuture()
                         .thenAcceptAsync(deliveryStatus -> deliveryStatus.filter(DeliveryStatus::isDelivered)
                                                                          .ifError(deliveryError -> raCtx.getReActorSystem()
                                                                                                         .logError("Delivery Error",
                                                                                                                   deliveryError)));
    }

    private static void onPong(ReActorContext raCtx, String pongMessage, long pingId) {
        raCtx.getReActorSystem()
             .logDebug("%s from reactor %s", pongMessage, raCtx.getSender().getReActorId().getReActorName());
        raCtx.getSender().tell(raCtx.getSelf(), "PingRequest " + pingId);
    }
}
```
This is complete fully working client/server ping pong application. 
ReActors exist within a [ReActorSystem](reactor_system.md). As JVM is the runtime environment for a java program,
a `ReActorSystem` is the runtime environment for a reactor. Through a `ReActorSystem` we can **spawn** a new reactor.
Once a reactor has been *spawned* a `ReActorInit` message will be immediately delivered to trigger the *Init* phase.

In the above example we did not use a *class* as a reactor, instead we provided on once case just the behaviors and 
the config for the reactor, in the other the config and a convenience inline interface implementation. 
The effect of using `ReActions::noReAction` as argument for the wildcard reaction, is to silently  ignore all the messages but the ones for which has been specified an explicit reaction. In the above example it means
that the `ReActorInit` and the `ReActorStop` messages will be silently ignored.

Inside a reaction we saw that we can interact with a reactor through the `ReActorContext` object that is always passed
as an argument. Outside the scope of a reaction, we can do that sending a **message**. 
It's possible [sending a message](messaging.md) to a reactor using do that its  `ReActorRef`. 
A `ReActorRef` is a **location and technology agnostic** reference that uniquely address a reactor across a ReActed [cluster](clustering.md).

In the `onPing` method we obtain the `ReActorRef` of the reactor that sent the current message and we [tell](messaging.md#tell)
another message to reply. `tell` returns a completion stage that is going to be completed with the outcome of the operation.
It's not mandatory providing a check for the outcome of the operation, it highly depends on the logic of the application.
ReActed guarantees that no message can be lost on `tell`, but this topic is covered in [messaging](messaging.md) and
[drivers](drivers.md) chapters. 

The `Ping` reactor has a specific `ReActorInit` reaction. It takes `Pong`'s `ReActorRef` and `tell`s the first message.
This will initiate a ping-pong of messages between the two reactors for some milliseconds and after that we [shutdown](reactor_system.md#Shutdown) the
`ReActorSystem`.
The output of the above program is the following:
```text
[ReActed-Dispatcher-Thread-ReactorSystemDispatcher-3] INFO io.reacted.core.reactors.systemreactors.SystemLogger - Pong FirstPing from reactor Pong
[ReActed-Dispatcher-Thread-ReactorSystemDispatcher-3] INFO io.reacted.core.reactors.systemreactors.SystemLogger - Pong PingRequest 1 from reactor Pong
[ReActed-Dispatcher-Thread-ReactorSystemDispatcher-2] INFO io.reacted.core.reactors.systemreactors.SystemLogger - Pong PingRequest 2 from reactor Pong
[ReActed-Dispatcher-Thread-ReactorSystemDispatcher-1] INFO io.reacted.core.reactors.systemreactors.SystemLogger - Pong PingRequest 3 from reactor Pong
[ReActed-Dispatcher-Thread-ReactorSystemDispatcher-0] INFO io.reacted.core.reactors.systemreactors.SystemLogger - Pong PingRequest 4 from reactor Pong
[ReActed-Dispatcher-Thread-ReactorSystemDispatcher-3] INFO io.reacted.core.reactors.systemreactors.SystemLogger - Pong PingRequest 5 from reactor Pong
[ReActed-Dispatcher-Thread-ReactorSystemDispatcher-2] INFO io.reacted.core.reactors.systemreactors.SystemLogger - Pong PingRequest 6 from reactor Pong
[ReActed-Dispatcher-Thread-ReactorSystemDispatcher-1] INFO io.reacted.core.reactors.systemreactors.SystemLogger - Pong PingRequest 7 from reactor Pong
[ReActed-Dispatcher-Thread-ReactorSystemDispatcher-0] INFO io.reacted.core.reactors.systemreactors.SystemLogger - Pong PingRequest 8 from reactor Pong
[ReActed-Dispatcher-Thread-ReactorSystemDispatcher-3] INFO io.reacted.core.reactors.systemreactors.SystemLogger - Pong PingRequest 9 from reactor Pong
[ReActed-Dispatcher-Thread-ReactorSystemDispatcher-2] INFO io.reacted.core.reactors.systemreactors.SystemLogger - Pong PingRequest 10 from reactor Pong
[ReActed-Dispatcher-Thread-ReactorSystemDispatcher-2] INFO io.reacted.core.runtime.Dispatcher - Dispatcher Thread ReActed-Dispatcher-Thread-ReactorSystemDispatcher-2 is terminating. Processed: 9
[ReActed-Dispatcher-Thread-ReactorSystemDispatcher-3] INFO io.reacted.core.runtime.Dispatcher - Dispatcher Thread ReActed-Dispatcher-Thread-ReactorSystemDispatcher-3 is terminating. Processed: 9
[ReActed-Dispatcher-Thread-ReactorSystemDispatcher-1] INFO io.reacted.core.runtime.Dispatcher - Dispatcher Thread ReActed-Dispatcher-Thread-ReactorSystemDispatcher-1 is terminating. Processed: 11
[ReActed-Dispatcher-Thread-ReactorSystemDispatcher-0] INFO io.reacted.core.runtime.Dispatcher - Dispatcher Thread ReActed-Dispatcher-Thread-ReactorSystemDispatcher-0 is terminating. Processed: 9
```

## ReActors Hierarchies











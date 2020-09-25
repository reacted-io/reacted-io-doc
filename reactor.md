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
import javax.annotation.Nonnull;
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
                                                     //To be more compact and redundant we could simply flatMap here
                                                     .peekFailure(error -> exampleReActorSystem.shutDown())
                                                     .orElseSneakyThrow();

        exampleReActorSystem.spawnReActor(new ReActiveEntity() {
                                            private long pingNum = 0;
                                            @Nonnull
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


```java
import io.reacted.core.config.ConfigUtils;
import io.reacted.core.config.reactors.ReActorConfig;
import io.reacted.core.config.reactorsystem.ReActorSystemConfig;
import io.reacted.core.messages.reactors.ReActorInit;
import io.reacted.core.messages.reactors.ReActorStop;
import io.reacted.core.reactors.ReActions;
import io.reacted.core.reactors.ReActiveEntity;
import io.reacted.core.reactors.ReActor;
import io.reacted.core.reactorsystem.ReActorContext;
import io.reacted.core.reactorsystem.ReActorRef;
import io.reacted.core.reactorsystem.ReActorSystem;
import io.reacted.patterns.NonNullByDefault;
import javax.annotation.concurrent.Immutable;
import java.io.Serializable;
import java.util.concurrent.TimeUnit;
import java.util.stream.LongStream;

public class FamilyExample {
    public static void main(String[] args) {

        ReActorSystem exampleReActorSystem = new ReActorSystem(ReActorSystemConfig.newBuilder()
                                                                                  .setReactorSystemName("ExampleSystem")
                                                                                  .build()).initReActorSystem();
        try {
            var father = exampleReActorSystem.spawnReActor(new Father(), ReActorConfig.newBuilder()
                                                                                      .setReActorName("Father")
                                                                                      .build()).orElseSneakyThrow();
            var uncle = exampleReActorSystem.spawnReActor(new Uncle(), ReActorConfig.newBuilder()
                                                                                    .setReActorName("Uncle")
                                                                                    .build()).orElseSneakyThrow();
            father.tell(uncle, new BreedRequest(3));
            TimeUnit.SECONDS.sleep(1);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        } finally {
            exampleReActorSystem.shutDown();
        }
    }

    private static void onStop(ReActorContext raCtx, ReActorStop stop) {
        raCtx.getReActorSystem().logDebug(raCtx.getSelf().getReActorId().getReActorName() + " is terminating");
    }

    @NonNullByDefault
    private static class Father implements ReActiveEntity {
        private final ReActions fatherReactions;
        private long requestedChildren;

        private Father() {
            this.fatherReactions = ReActions.newBuilder()
                                            .reAct(BreedRequest.class, this::onBreedRequest)
                                            .reAct(ThankYouFather.class, this::onThankYou)
                                            .reAct(ReActorStop.class, FamilyExample::onStop)
                                            .reAct(ReActions::noReAction)
                                            .build();
        }

        @Override
        public ReActions getReActions() { return fatherReactions; }

        private void onBreedRequest(ReActorContext raCtx, BreedRequest breedRequest) {
            this.requestedChildren = breedRequest.getRequestedChildren();

            raCtx.getReActorSystem()
                 .logDebug("%s received a %s for %s from %s",
                           raCtx.getSelf().getReActorId().getReActorName(),
                           breedRequest.getClass().getSimpleName(), breedRequest.getRequestedChildren(),
                           raCtx.getSender().getReActorId().getReActorName());

            LongStream.range(0, breedRequest.getRequestedChildren())
                      .forEachOrdered(childNum -> raCtx.spawnChild(new Child(childNum, raCtx.getSender())));
        }

        private void onThankYou(ReActorContext raCtx, ThankYouFather thanks) {
            if (--this.requestedChildren == 0) {
                raCtx.stop()
                     .thenAcceptAsync(voidVal -> raCtx.getSender().tell(ReActorRef.NO_REACTOR_REF, new ByeByeUncle()));
            }
        }
    }

    @NonNullByDefault
    private static class Uncle implements ReActiveEntity {
        private static final ReActions UNCLE_REACTIONS = ReActions.newBuilder()
                                                                  .reAct(Greetings.class, Uncle::onGreetingsFromChild)
                                                                  .reAct(ByeByeUncle.class, Uncle::onByeByeUncle)
                                                                  .reAct(ReActorStop.class, FamilyExample::onStop)
                                                                  .reAct(ReActions::noReAction)
                                                                  .build();
        @Override
        public ReActions getReActions() { return UNCLE_REACTIONS; }

        private static void onGreetingsFromChild(ReActorContext raCtx, Greetings greetingsMessage) {
            raCtx.getReActorSystem().logDebug("%s received %s. Sending thank you to %s",
                                              raCtx.getSelf().getReActorId().getReActorName(),
                                              greetingsMessage.getGreetingsMessage(),
                                              raCtx.getSender().getReActorId().getReActorName());
            raCtx.getSender().tell(raCtx.getSelf(), new ThankYouFather());
        }

        private static void onByeByeUncle(ReActorContext raCtx, ByeByeUncle timeToDie) { raCtx.stop(); }
    }

    @NonNullByDefault
    private static class Child implements ReActor {
        private final ReActorRef breedRequester;
        private final ReActorConfig childConfig;
        private Child(long childId, ReActorRef breedRequester) {
            this.breedRequester = breedRequester;
            this.childConfig = ReActorConfig.newBuilder()
                                            .setReActorName(Child.class.getSimpleName() + "-" + childId)
                                            .build();
        }

        @Override
        public ReActorConfig getConfig() { return childConfig; }

        @Override
        public ReActions getReActions() {
            return ReActions.newBuilder()
                            .reAct(ReActorInit.class, this::onInit)
                            .reAct(ReActorStop.class, FamilyExample::onStop)
                            .reAct(ReActions::noReAction)
                            .build();
        };

        private void onInit(ReActorContext raCtx, ReActorInit init) {
            this.breedRequester.tell(raCtx.getParent(),
                                     new Greetings("Hello from " + childConfig.getReActorName()));
        }
    }

    @Immutable
    private static final class BreedRequest implements Serializable {
        private final long requestedChildren;
        private BreedRequest(long requestedChildren) {
            this.requestedChildren = ConfigUtils.requiredInRange(requestedChildren, 1L, Long.MAX_VALUE,
                                                                 IllegalArgumentException::new);
        }
        private long getRequestedChildren() { return requestedChildren; }
    }

    @NonNullByDefault
    @Immutable
    private static final class Greetings implements Serializable {
        private final String greetingsMessage;
        private Greetings(String greetingsMessage) { this.greetingsMessage = greetingsMessage; }
        private String getGreetingsMessage() { return greetingsMessage; }
    }

    @Immutable
    private static final class ThankYouFather implements Serializable { private ThankYouFather() { } }

    @Immutable
    private static final class ByeByeUncle implements Serializable { private ByeByeUncle() { } }
}
```


```text
[ReActed-Dispatcher-Thread-ReactorSystemDispatcher-2] INFO io.reacted.core.reactors.systemreactors.SystemLogger - Father received a BreedRequest for 3 from Uncle
[ReActed-Dispatcher-Thread-ReactorSystemDispatcher-3] INFO io.reacted.core.reactors.systemreactors.SystemLogger - Uncle received Hello from Child-1. Sending thank you to Father
[ReActed-Dispatcher-Thread-ReactorSystemDispatcher-1] INFO io.reacted.core.reactors.systemreactors.SystemLogger - Uncle received Hello from Child-0. Sending thank you to Father
[ReActed-Dispatcher-Thread-ReactorSystemDispatcher-3] INFO io.reacted.core.reactors.systemreactors.SystemLogger - Uncle received Hello from Child-2. Sending thank you to Father
[ReActed-Dispatcher-Thread-ReactorSystemDispatcher-0] INFO io.reacted.core.reactors.systemreactors.SystemLogger - Father is terminating
[ReActed-Dispatcher-Thread-ReactorSystemDispatcher-0] INFO io.reacted.core.reactors.systemreactors.SystemLogger - Child-0 is terminating
[ReActed-Dispatcher-Thread-ReactorSystemDispatcher-1] INFO io.reacted.core.reactors.systemreactors.SystemLogger - Child-1 is terminating
[ReActed-Dispatcher-Thread-ReactorSystemDispatcher-2] INFO io.reacted.core.reactors.systemreactors.SystemLogger - Child-2 is terminating
[ReActed-Dispatcher-Thread-ReactorSystemDispatcher-1] INFO io.reacted.core.reactors.systemreactors.SystemLogger - Uncle is terminating
[ReActed-Dispatcher-Thread-ReactorSystemDispatcher-3] INFO io.reacted.core.runtime.Dispatcher - Dispatcher Thread ReActed-Dispatcher-Thread-ReactorSystemDispatcher-3 is terminating. Processed: 4
[ReActed-Dispatcher-Thread-ReactorSystemDispatcher-2] INFO io.reacted.core.runtime.Dispatcher - Dispatcher Thread ReActed-Dispatcher-Thread-ReactorSystemDispatcher-2 is terminating. Processed: 7
[ReActed-Dispatcher-Thread-ReactorSystemDispatcher-1] INFO io.reacted.core.runtime.Dispatcher - Dispatcher Thread ReActed-Dispatcher-Thread-ReactorSystemDispatcher-1 is terminating. Processed: 7
[ReActed-Dispatcher-Thread-ReactorSystemDispatcher-0] INFO io.reacted.core.runtime.Dispatcher - Dispatcher Thread ReActed-Dispatcher-Thread-ReactorSystemDispatcher-0 is terminating. Processed: 6
```







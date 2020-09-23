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

## Example

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
            raCtx.stop();
        }
    }
}
```
The above reactor prints a statement every time a message is processed. 
From the above code, we can quickly notice that:
- Any class can become a reactor simply implementing the `ReActor` interface
- It's possible to setup reactions for a message type or providing a wildcard reaction
- Messages **must** implement the `Serializable` interface
- It's mandatory *naming* a reactor. We can have default values for the other configuration properties, but
a reactor always needs a unique name among its alive siblings
- ReActed offers a [centralized logging reactor](centralized_logger.md)
- No synchronization primitives are required to ensure the correct updated of the counter, regardless of how many threads 
are sending messages as the same time
- `ReActorContext` provides a set of method useful to control the reactor behavior and to access its internal properties

Given a [ReActorSystem](reactor_system.md) we can *spawn* the reactor defined above with a single line:

```java
exampleReActorSystem.spawnReActor(new FirstReActor())
                    .ifError(Throwable::printStackTrace);
```

## 




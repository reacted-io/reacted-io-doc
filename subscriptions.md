# Message Subscriptions

Every `ReActiveEntity` can statically or dynamically set some *typed subscriptions*.
Being subscribed for a type means that the subscribing `ReActiveEntity` will receive a copy of any message matching
the specified types. A [choreography oriented system](patterns.md#Reactors-choreography) can use this feature to set
reactive hooks and implementing features without touching the other codebase.

Typed subscriptions can be of two types: `TypedSubscriptionPolicy.LOCAL` or `TypedSubscriptionPolicy.FULL`

## Local Subscription

A `ReActiveEntity` locally type-subscribed will receive copies of messages sent from within the `ReActorSystem` in 
which the subscriber runs into

## Full Subscription

A full typed subscription allows the reactor to receive a copy of the messages for the subscribed types regardless of
the reactorsystem of the destination. This includes messages meant for another reactor within the current
[reactor system](reactor_system.md) also if generated in some other `ReActorSystem` of messages not meant for any reactor
within the current reactor system, but since due to the nature of the [channel](channel_drivers) they have been intercepted.

## Use Cases

### Local Subscription Use Case

Let's say that we just set up our log processing stack and we want to automatically push every log entry to a centralized
remote log collector. With a local typed subscription we can easily implement a [change data capture](patterns.md) for
the log entries. 

```java
public static class LogInterceptor implements ReActor {
    @Nonnull
    @Override
    public ReActorConfig getConfig() {
        return ReActorConfig.newBuilder()
                            .setReActorName("RemoteLogPusher")
                            .setTypedSubscriptions(TypedSubscriptionPolicy.LOCAL.forType(ReActedInfo.class),
                                                   TypedSubscriptionPolicy.LOCAL.forType(ReActedError.class))
                            .build();
    }

    @Nonnull
    @Override
    public ReActions getReActions() {
        return ReActions.newBuilder()
                        .reAct(ReActedInfo.class, LogInterceptor::onNewLog)
                        .reAct(ReActedError.class, LogInterceptor::onNewLog)
                        .build();
    }
    private static void onNewLog(ReActorContext raCtx, LogMessage logMessage) {
        //push it remotely...
    }
}
```

### Full Subscription Use Case

Let's say that we still have to push every log entry inside our centralized log collector, but we have no way to
alter the code that is running on the machines or it's simply too much. The good thing is that these [reactor systems](reactor_system.md)
talk to each other using [Kafka](channel_drivers/kafka/kafka_main.md). 
In this scenario, we can simply *decorate* our infrastructure adding a reactor that reacts to log messages seen over
the medium.

Changing from `TypedSubscriptionPolicy.LOCAL` to `TypedSubscriptionPolicy.FULL` and connecting the reactor system
with an appropriate [channel](channel_drivers/README.md) will give the same effect
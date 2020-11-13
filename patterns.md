# Patterns

This page is meant to be a collection of ideas and patterns that can make you reacted-life more easy and productive.

## Managing blocking operations

[Dispatchers](dispatcher.md) are the ticking core of ReActed and for this reason they cannot be stopped. Unfortunately,
some operations are potentially blocking, such as i/o. A blocked dispatcher blocks **all** the reactors that are currently
scheduled on it, so such situation must be avoided at all costs. 
Blocking operations can be demanded to a different thread(pool) and the result of such operation can just be sent to 
the requesting reactor through a message. Alternatively, the same concept can be applied differently defining and
using a [dedicated dispatcher](reactor.md#ReActor-Configuration) for all those reactor that we know that can block. 

## Change Data Capture

Even if ReActed is not a database, it can provide such a functionality through [subscriptions](subscriptions.md). 
Let's take as an example [dead letters](reactor_system.md#DeadLetters) or [system logger](centralized_logger.md). Those two system
reactors receive respectively messages for which a destination was not available at the moment of delivery and
the logging requests. What if we would like to intercept these messages for statistics or log monitoring?
We have to simply [spawn](reactor.md) a `ReActor` [subscribed](subscriptions.md) to the message types that are generated for communicating.

A way to distribute to all the subscribers a given message would be this: 

```java
  reActorSystemIntance.getSystemSink().tell(anyMessageSender, payload);    
```
`getSystemSynk()` returns an instance to [`Init`](reactor_system.md#ReActorSystem-structure) that by default
just swallows all messages that receives. An alternative would be using a `ReActor` configured with a [null mailbox](mailboxes.md#Null-Mailbox)
or using `reActorSystemInstance.broadcastToLocalSubscribers(ReActorRef msgSender, PayLoadT payload)`

## ReActor Factory

Building a system *fully replayable* can be tricky. One of the tricky parts is that if you have to spawn a reactor as
part of the handling of user input, in a replay phase you do not have such input from user, so you cannot react, you cannot
spawn the reactor and without a valid destination the [replay driver](channel_drivers/replay/replay_main.md) cannot deliver
messages and it just suspend waiting for a valid destination to appear. To overcome this issue you can use a reactor factory.
Instead of spawning a `ReActor` as a direct reaction to use input, spawn a `ReActor` as a reaction of a message sent after
the reception of use input. This snippet is taken from the **webappbackend** in examples directory:

```java
    @Override
    public void handle(HttpExchange exchange) {
        var requestId = exchange.getRequestURI().toASCIIString() + "|" + Instant.now().toString();
        this.requestIdToHttpExchange.put(requestId, exchange);
        thisCtx.selfTell("GET".equals(exchange.getRequestMethod())
                         ? new SpawnGetHandler(requestId)
                         : "POST".equals(exchange.getRequestMethod())
                            ? new SpawnPostHandler(requestId)
                            : new SpawnUnknownHandler(requestId));
    }
```

Whenever a new request arrives, the handling reactor `thisCtx` sends itself a spawn request instead of directly
spawning a new `ReActor`. While replaying the log even without user input `thisCtx` will receive the replayed
*spawn request* and the whole flow can be successfully replicated.
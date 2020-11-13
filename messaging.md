# Communicating with a ReActor

**ReActed** offers a set of primitives for communicating with a `ReActiveEntity`. A valid message destination is
*always* defined by a `ReActorRef`. A `ReActorRef` uniquely identifies a `ReActiveEntity` within a [cluster](clustering.md).
From the programmer's perspective there is no difference in communicating with a local or remote `ReActiveEntity` or
no difference regardless of the technology used for doing that. Every message has at least 3 fields: sender, destination, payload.
Payload is any `Serializable` object. Sender and destination are two `ReActorRef` defining the source and the destination
of that send attempt. All communication attempts return a `CompletionStage<Try<DeliveryStatus>>`.
The `CompletionStage` is going to be completed when the attempt is terminated. On failure Try will containing the
exception that cause the failure and on success `DeliveryStatus` will contain the result of the attempt.
Using *tell* as an example

```
Destination.tell(sender, payload)
           .thenAcceptAsync(result -> result.filter(DeliveryStatus::isDelivered)
                                            .ifSuccessOrElse(success -> ANY SUCCESS ACTION,
                                                             throwable -> ANY ERROR ACTION)
```
The above snipped will execute **SUCCESS ACTION** once the delivery attempt has been successfully terminated,
otherwise in case of I/O error, backpressure of whatever, *ANY ERROR ACTION* will be executed.

The `thenAcceptAsync` method has been used purposely: the generating thread of the completion stage can be a [dispatcher](dispatcher.md),
so we do not want to chain potentially blocking actions on its thread.

A failed communication will not be automatically reattempted.
 
## Tell

Tells a message to a destination. Its returned `CompletionStage` is going to be completed once the message has been
*written* on the [communication channel](channel_drivers/README.md).

!> NOTE: Implementation detail: on a [local direct channel](channel_drivers/README.md#Direct-Channels) a `tell` is always going to be
!> completed with the outcome of the delivery into the target's mailbox. This because there is no other medium than memory
!> between source and destination. A delivery can fail (i.e. a [bounded mailbox](mailboxes.md#Bounded-Mailbox) refuses the message,
!> but message cannot be lost on *local direct channels*. If you are looking for performances this is the way to go

## Atell

`atell` stands for *acked tell*. With `atell` delivery attempts will be marked as completed once the message has been
acked by the destination. The destination acks a delivery request once a result for the operation is known. This means
that if the destination is performing some kind of backpressuring on the delivery requests and the request is suspended,
an ack will not be generated until the operation is completed, regardless of its result.
So, if the returned `Try<DeliveryStatus>` is a success and the status is `DELIVERED`, it provides delivery guarantee within the
target's mailbox. `atell` is automatically transformed into a `tell` on [local direct channels](channel_drivers/README.md#Direct-Channels),
so using it provides delivery guarantee or explicit failure regardless on the location or the [channel type](channel_drivers/README.md) used to reach
the recipient. Since a remote peer may never complete the operation due to a crash, `atell` requests may be automatically
completed with a `Failure` carrying a `TimeoutException`. Please note that `atell` provides you with more detailed information
regarding the status of a delivery, but it comes with the price of increased overhead and latency due to the acking machanism.
Since `atell` purpouse is to provide safety and that is dependent on the medium technology, a [driver](channel_drivers/README.md) can be
configured as reliable to automatically transform all the `atell` into `tell`.  

## Reply
`ReActorContext.reply` Tells a message to the sender of the last message received by the used `ReActorContext`. 

## Areply

`ReActorContext.areply` is the same as `ReActorContext.reply` but an `atell` is used instead of a `tell` for communication

## SelfTell

`ReActorContext.selfTell` tells a message to the same reactor identified by the `ReActorContext` using itself as a
source `ReActorRef`

## Ask

All the above primitives can be used from within a `ReAction` scope. With `tell` and `atell` we can communicate with a
`ReActiveEntity` just having its reference, but it's a one way communication because unless we setup some `ReAction` we
cannot get a reply.

`ask` overcomes this limitation, allowing you to send one message to any `ReActiveEntity` and to (a)synchronously get a reply.

```java <ReplyT extends Serializable, RequestT extends Serializable>
        CompletionStage<Try<ReplyT>> ask(RequestT request, Class<ReplyT> expectedReply, String requestName)
```
The return semantic is the same of the other methods, but with `ask` we must specify the expected reply type and a
*name* for the request. In any given time, two `ask` with the same request type, the same expected return type,
the same destination and the same name, cannot cohexist. We will see in detail the ratio behind the naming in **ReActed**
in the [replay](channel_drivers/replay/replay_main.md) section.  

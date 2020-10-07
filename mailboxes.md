# Mailbox

Actors communicate with each other sending messages. These messages are going to be saved in a mailbox, waiting for the
[dispatcher](dispatcher.md) to schedule the reactor for processing them. **ReActed** provides out of the box different mailbox 
implementations, allowing you to choose for the best one given the problem that has to be solved.

## Basic Mailbox

Unbounded list of messages. Messages are returned FIFO

## Bounded Mailbox

Bounded list of messages. Messages are returned FIFO. If a message delivery cannot be completed because the mailbox is
full, the `DeliveryStatus` of the operation will be `BACKPRESSURED`

## Priority Mailbox

Unbounded mailbox that behaves like a heap. Messages are ordered according to comparison order

## Inflatable Mailbox

[Bounded mailbox](mailboxes.md#Bounded Mailbox) that can have its maximum size programmatically adjusted 

## Null Mailbox

Drops all messages

## Backpressuring Mailbox

This special mailbox is used to provide automatic *backpressuring* to any other mailbox. It can be used to automatically
provide backpressuring to [reactors](reactor.md), [services](services.md) or [streams](reacted_streams.md).
`BackpressuringMailbox` behaves exactly like a [submission publisher](https://docs.oracle.com/en/java/javase/14/docs/api/java.base/java/util/concurrent/SubmissionPublisher.html)
with the difference that any ```java public CompletionStage<Try<DeliveryStatus>> asyncDeliver(Message message)``` is
*always* asynchronously completed. Chaining actions on the returned `CompletionStage` has the effect to execute them
once the delivery attempt has been completed, so after that the backpressure had taken place.

### Backpressuring Mailbox Configuration

This mailbox is pretty flexible, let's see an example taken from [reacted streams](reacted_streams.md) implementation:

```java
BackpressuringMbox.newBuilder()
                  .setRealMbox(new BoundedBasicMbox(subscription.getBufferSize()))
```
As we said this is a wrapper mailbox, so we have to specify which the backing up (real) mailbox is
```java
                  .setBackpressureTimeout(subscription.getBackpressureTimeout())
```
Specify for how much time an update should be kept on hold waiting for a dispatcher to consume some messages befor
resulting in a delivery error. Specifying as value `BackpresusringMailbox.BEST_EFFORT_TIMEOUT` a message will be
dropped immediately is a delivery is not possible, while `BackpressuringMailbox.RELIABLE_DELIVERY_TIMEOUT` will cause
to *wait* indefinitely until a delivery is possible. Please remember that if the `delivery` interface is *never*
backpressured neither for this mailbox, the `asyncDelivery` is *always* async. Chaining on the delivery results
of a `BackpressuringMailbox` configured with `BackpressuringMailbox.RELIABLE_DELIVERY_TIMEOUT` means that the chained
actions will be executed after the delivery of the message, in case of delivery failure (that can be checked with 
the Try<DeliveryStatus>) or will wait forever.
```java
                  .setBufferSize(subscription.getBufferSize())
```
Specify how big the buffer is. This number will be round up to the next power of 2. Defines how many messages can be
sent to this mailbox before any backpressuring action takes place
```java
                  .setRequestOnStartup(0)
```
As we said, `BackpressuringMailbox` behaves like a `Flow` publisher, so we can specify how many messages can pass through
once it is created. If used as a mailbox for a [reactor](reactor.md) or a [service](services.md) a sane value for this
parameter is `1` 
```java
                  .setAsyncBackpressurer(subscription.getAsyncBackpressurer())
```
Thread pool that is going to be used to perform the async-backpressured delivery to the real mailbox
```java
                  .setNonDelayable(Set.of(ReActorInit.class, ReActorStop.class,
                                          SubscriptionRequest.class,
                                          SubscriptionReply.class,
                                          UnsubscriptionRequest.class,
                                          SubscriberError.class,
                                          PublisherInterrupt.class))
```
Some message types cannot be backpressured and not even delayed (such as system messages). A delivery for any of the
`NonDelayable` message types will not be lost or delayed, instead the delivery will be attempted immediately
```java
                  .setNonBackpressurable(Set.of(PublisherComplete.class))
```
Message types that regardless of the `BackpressuringMailbox` timeout configuration should be reliably delivered, so
any delivery attempt of one of the specified message types will be converted into a `BackpressuringMailbox.RELIABLE_DELIVERY_TIMEOUT`
delivery.
```java
                  .setSequencer(subscription.getSequencer())
```
Since avery `asyncDelivery` *cannot* be blocking also in case of producer-slowdown backpressuring, the *sequencer* is
a specific thread that is used to sequentially attempt the async deliveries. If not specified, a thread will be
automatically created by the mailbox for this task. A sequencer can be safely shared among different
 `BackpressuringMailbox` if required.
```java
                  .setRealMailboxOwner()
                  .build();
``` 
The `ReActorContext` of the `ReActiveEntity` that is using this this mailbox.

> NOTE: **ReActed** and `BackpressuringMailbox` are completely agnostic about the location of the sender of the messages.
> This means that it can be used to automatically backpressure **remote** producers too


# ReActed Streams

A *reacted stream* is a *distributed*, *multi subscriber*, [*replayable*](replaying.md), [reactive-streams full compliant](https://www.reactive-streams.org/) messages stream with customizable automatic backpressure
out of the box. A *reacted stream publisher* is a `Serializable` entity, this means that the publisher can flow within
a reacted cluster and any interested reactor can simply subscribe to it to start receiving its data.

## Submission publisher

`ReactedSubmissionPublisher` is a [Java Flow](https://docs.oracle.com/javase/9/docs/api/java/util/concurrent/Flow.html) compliant local and remote backpressured publisher.
It can be created with ```java public ReactedSubmissionPublisher(ReActorSystem localReActorSystem, String feedName)``` where `feedName` is a unique
publisher name. No publishers with the same name can coexist at the same moment.
`ReactedSubmissionPublisher` is a `Serializable` entity, this means that can be sent through one of the [messaging primitives](messaging.md) to local or remote systems. 
When messages are sent towards a remote subscriber, communication is always Point-to-Point also if two subscribers are within the same reactor system.

## Subscription

Subscription is possible through one of the many Java Flow-compatible subscribe overloads. Fine tuning the subscription
parameters can hugely impact the behaviour and the performances of your application. The most fine tunable overload,
accepts a `ReActedSubscription` that gives you full control over the subscription behavior.

> NOTE: Also subscriptions can be **named**. Using an unnamed subscription will result in a [non replayable](replaying.md) stream.

In ReActed, backpressuring policies and behaviours are controlled by the subscriber since subscribers might be *many* with different
requirements or resource constraints. Every backpressured subscriber, behaves like a [Java Submission Publisher subscriber](https://docs.oracle.com/javase/9/docs/api/java/util/concurrent/SubmissionPublisher.html).
For this reason, all the parameters that can be configured within `ReActedSubscription` will affect the subscriber exactly like if it was a Java Submission Publisher subscriber.

Submitting a new `Message` to a subscriber *must* always be an asynchronous operation. For this reason with  

```java public Builder<PayloadT> setSequencer(@Nullable ThreadPoolExecutor sequencer)``` an helper single thread can be specified for ensuring this. Of none is provided,
one will be automatically created *per subscriber base*. If you are in a situation with many or frequent short lived subscribers, you might consider sharing this thread among
different subscription instances if you are willing to pay the price of some delay if one of the subscriptions should [**block**](#Backpressuring) a lot due to the excessive load.

### Backpressuring

We said that backpressuring is controlled by the subscriber, but this is not entirely true. A subscriber can define the policies for dropping or delaying 
the delivery of a message in case of high load. With ```java public Builder<PayloadT> setBackpressureTimeout(Duration backpressureTimeout)``` a delivery
delay timeout can be specified. If we are willing to drop new messages in case of congestion, `ReactedSubmissionPublisher.BEST_EFFORT_SUBSCRIPTION` can be specified.
Otherwise, if a message should not dropped no matter what, then `ReactedSubmissionPublisher.RELIABLE_SUBSCRIPTION` will cause the delivery attempt to never expire.
For all the other scenarios a simple `Duration` can be use to define for how long a delivery should be attempted before expiring and dropping the message.
  
A `ReactedSubmissionPublisher` instead, can control how messages are submitted. A submission can be done completely ignoring the backpressure information
or taking them in account. In case of message dropping backpressure strategy there isn't much difference, but in case of producer slowdown it changes
completely. With ```java public CompletionStage<Void> backpressurableSubmit(PayloadT message)``` is possible to publish a message towards all the
subscribers and getting in return a `CompletionStage` that is going to be completed **only** when the message delivery has been done or dropped by
all the subscribers.

This means that every non-`ReactedSubmissionPublisher.BEST_EFFORT_SUBSCRIPTION` subscription has the power to slowdown the producer if this method and
the returned `CompletionStage` are checked before sending the next message over the stream.

## Examples

Creating a `ReacteStreamPublisher` where `Integer` are going to be streamed:

```java 
var streamPublisher = new ReactedSubmissionPublisher<Integer>(reactorSystem, "Test-Integer-Streamer-Publisher");
```
Then, given a `Flow.Subscriber<Integer>` named `SUBSCRIBER-$(N)` with N integer, we can subscribe for best effort updates:

```java streamPublisher.subscribe(SUBSCRIBER-1);```

Note that as said above, with this overload the stream is not going to be [replayable](replaying.md) and a new *sequencer* thread will be created per subscription.

Let's create a *replayable*, *best effort* subscription now:

```java streamPublisher.subscribe(SUBSCRIBER-2, "Best effort replayable subscription 2);```

While for creating a *reliable*, *replayable* subscription:

```java streamPublisher.subscribe(SUBSCRIBER-3, ReactedSubmissionPublisher.RELIABLE_SUBSCRIPTION, "Reliable and replayable subscription")```

Or if we are picky, we can specify anything:

```java
 var asyncDeliveryService = Executors.newFixedThreadPool(2);
 var sharedSequencer = new ThreadPoolExecutor(0, 1, 10, TimeUnit.SECONDS, new LinkedBlockingDeque<>());

 streamPublisher.subscribe(ReactedSubmissionPublisher.ReActedSubscription.<Integer>newBuilder()
                                          //Attempt delivery for 1 maximum 1 minute before giving up and dropping the update
                                          .setBackpressureTimeout(Duration.ofMinutes(1))
                                          //Before blocking on submission for lack of space, 256 entries can be temporarily stored into the internal buffer 
                                          .setBufferSize(256)
                                          .setSubscriber(SUBSCRIBER-4)
                                          .setSubscriberName("Picky, almost best effort subscription")
                                          //This is for performing the async backpressure  
                                          .setAsyncBackpressurer(asyncDeliveryService)
                                          //This is for ensuring that a submission request is never blocking even if the buffer is full and the 
                                          //submission order is not scrambled  
                                          .setSequencer(sharedSequencer)
                                          .build())
```
Using a shared sequencer and a shared async backpressurer executor service (such as `ForkJoinPool.commonPool()`) will expose the implementation 
to cross local subscribers delays, but will not create any additional overhead due to excessive thread creation. This could be a strategy
to use when the number of subscribers within a single ReActed node should be high.

Given all the bove subscriptions, we can publish towards all of them regardless of the location using the `ReactedStreamPublisher`.

If we want to use the standard interface that does not take in account the data producing slowdown backpressure strategy, we can do like this

```java streamPublisher.submit(ANY_INTEGER);```

Otherwise, if we want our flow to be automatically regulated according to the speed of the non-best effort consumers, we can do something like this:

```java
 AtomicInteger counter = new AtomicInteger(1);
 CompletaionStage<Void operationComplete = AsyncUtils.asyncLoop(noValue -> streamPublisher.backpressurableSubmit(counter.getAndIncrement()), 
                                                                null, (Void)null, REQUESTS_NUM);
```
A new submission will be *sequentially* attempted once a result (delivery or drop) for the previous one from all the subscribers, is known. `REQUESTS_NUM` iteration will take
place. The call is non blocking and the returned `CompletionStage` will be marked as completed once all the attempts have taken place.

## Stream Graphs

Yet to be implemented

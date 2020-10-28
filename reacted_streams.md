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



## Stream Graphs

Yet to be implemented

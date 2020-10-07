# Message Subscriptions

Every `ReActiveEntity` can statically or dynamically set some *typed subscriptions*.
Being subscribed for a type means that the subscribing `ReActiveEntity` will receive a copy of any message matching
the specified types. A [choreography oriented system](patterns.md#Reactors choreography) can use this feature to set
reactive hooks and implementing features without touching the other codebase.

Typed subscriptions can be of two types: `SubscriptionPolicy.LOCAL` or `SubscriptionPolicy.REMOTE`

## Local Subscription

A `ReActiveEntity` locally type-subscribed will receive copies of messages sent from within the `ReActorSystem` in 
which the subscriber runs into

## Remote Subscription

A remote typed subscription 
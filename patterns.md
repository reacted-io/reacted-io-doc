# Patterns

## Orchestrating reactors
## Managing blocking operations
## Change Data Capture

Even if ReActed is not a database, it can provide such a functionality through [subscriptions](subscriptions.md). 
Let's take as an example [dead letters](reactor_system.md#DeadLetters)-or [system logger](centralized_logger.md). Those two system
reactors receive respectively messages for which a destination was not available at the moment of delivery and
the logging requests. What if we would like to intercept these messages for statistics or log monitoring?
We would have simply to [spawn](reactor.md) a `ReActor` subscribed to the message types that are generated for communicating
with the above two reactors:

```java
    
```

## ReActor Factory
# Replay Driver

Replay driver allows you to read the messages from a [recorded session](../../reactor_system.md#Session Recording)
saved with the [chronicle queue](../../channel_drivers/cq/cq_main.md) [local driver](../../channel_drivers/README.md) and to feed with them your [reactor system](../../reactor_system.md).

## Replay driver replay semantic

The word choice of the previous section has not been casual: ***replay driver* is about re-delivering the messages that were 
recorded during a session recorded execution, is *not* about providing the same input and checking if the same output 
is obtained.**. The only messages that are going to be delivered within a replayed session with the default *replay driver* 
are the ones that were recorded. What *replay driver* does is delivering messages in an appropriate order to actors living 
within the reactor system. It does not create reactors or reset the state of the system, it feeds messages to your
application logic and prevents from delivering un-recorded messages. A realtime diff during replay is envisioned, but not yet implemented.  

## How it works

ReActed by design is a highly concurrent system with the meaning that every reactor has it own execution flow and all of them
advance all together. There isn't a clearly defined total ordering of the executions within a reactor system: if you
think about the dispatchers that can be multiple and multi threaded a direct consequence is that some actions can actually take place
at the *same* time. Other than that, without some centralized and sequential control of the executed actions it would
not possible defining an exact and correct execution total order.

> For this reason, *replay driver* does not embrace total ordering but a **causal** ordering. 
> ReActed *replay driver* guarantees that a message is not going to be *replayed* before its place in the cause-effect
> **flow** and is going to be replayed in the exact execution order of the reactor that receives it.
> **The replay is going to be *consistent* per flow basis and *exact* per reactor basis.**

Execution flows that are not entwined might receive messages in a different **total** order from the original session,
but the replayed execution order is always going to be consistent per flow and exact per reactor.

### Example

Let's see a quick example of the above rules. Let's say that we recorded an execution with 6 reactors. Since we are
brave and we do not fear © infrigiments, let's call them with the name of some Åvengers.

In the first flow, Iron Man sends a message to Hulk, THEN Hulk sends a message to the poor Loki.

> Iron Man -> Hulk -> Puny Loki. Loki receives the message and says "Yeeaaargh"

In the second flow, Thor sends a message to Hulk, THEN Hulk sends a message to Black Widow.

> Thor -> Hulk -> Black Widow. Black widow receives the message and says "Thanks!"

The following two replayed executions are all valid:

Execution 1:

> Iron Man -> Hulk  
> Hulk -> Loki  
> Loky says "Yeeaaargh"  
> Thor -> Hulk  
> Hulk -> Black Widow  
> Black Widow says "Thanks!"   

Execution 2:

> Iron Man -> Hulk  
> Thor -> Hulk  
> Hulk -> Loki  
> Hulk -> Black widow  
> Black Widow says "Thanks!"  
> Loki says "Yeeaaargh"  

Black widow can receive a message only after that it has been told to Hulk by Thor.
Hulk processes the messages in the same order they were executed while recording the session.
Black Widow and Loki executions are independent once they received the messages they were waiting for.

## Naming

*Replay driver* behavior is about delivering messages in an appropriate order to reactors that are waiting for them.
This means that it must be able to find the exact destination of a message within the [local reactor system](../../reactor_system.md).

In the [reactors section](../../reactor.md) we learnt that a *spawned reactor* is uniquely targettable within a cluster
through the location agnostic `ReActorRef`. One of the requirements for *spawning* a reactor, is providing a *reactor name*.

*Replay driver* is the reason behind the advice of not choosing randomly generated reactor names: reactors with a randomly
generated name have a different `ReActorRef` every time they are spawned, and in such a case the *replay driver* would not
know where to send the messages from the log. The destination would not match.



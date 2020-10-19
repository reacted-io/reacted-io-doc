# Replaying a recorded session

[Replay](channel_drivers/replay/replay_main.md) driver allows you to read the messages from a [recorded session](reactor_system.md#Session Recording)
saved with the [chronicle queue](channel_drivers/cq/cq_main.md) [local driver](channel_drivers/README.md) and to feed with them your [reactor system](reactor_system.md).

ReActed by design is a highly concurrent system with the meaning that every reactor has it own execution flow and all of them
advance all together. There isn't a clearly defined total ordering of the executions within a reactor system: if you
think about the dispatchers that can be multiple and multi threaded it's trivial that some actions can actually take place
at the *same* time, but a recorded session globally grants the causal order and the total order per reactor.

This means that ReActed can replicate the exact sequence of messages that a reactor **received** and only after that in
the original execution they have been generated. 

That said, ReActed can sequentially feed the reactors if your system with the very same messages that were received
during the recording phase and with the exact execution sequence that took place. Replaying a reactor system does not mean
reproducing the exact execution flow that lead to a certain state, but delivering to all the reactors the messages that they
received allowing them to reconstruct their state *indipendently* from the others' behavior. 
It's not about recomputing the global state, but leaving all the reactors to recompute their own state using the input
from messages and converging to a state globally equivalent to the original execution in a given moment in time.

  
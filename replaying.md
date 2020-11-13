# Session Replay Basics

A [reactor system](reactor_system.md) can be configured to enable what is called *session recording*
Session recording it's not about saving data in a specific format, it's about generating additional information 
regarding the execution flow. This information will be emitted by the system under the form of `Message`. This means
that all the extra information will pass through the [local channel](channel_drivers/README.md#Local-Channels) that
has been configured. Deciding what to do with those information it's completely up to the [local driver](channel_drivers/README.md).

ReActed offers out of the box a [chronicle queue local driver](channel_drivers/cq/cq_main.md) that saves every message
that passes through in the [chronicle queue format](https://github.com/OpenHFT/Chronicle-Queue). This low latency, low garbage
and replay prone format allows to save all the binary data contained in the messages. 
[Enabling session recording](reactor_system.md#Session-Recording) and selecting the [chronicle queue local driver](channel_drivers/cq/cq_main.md) will result
in saving all the messages and the information regarding their execution. 

## How to replay a session

What a [reaction](reactor.md) actually does is completely up to the provided code, so ReActed cannot replicate what
took place within a *reaction*. For this reason  **ReActed does not have a fixed replay engine**, but instead it has flexible driver model. 
From ReActed perspective replaying a session is about providing to the `ReActors` the messages that were executed during a recorded session. 
Decoding a message and providing it to the destination `ReActor` is exactly what a [channel driver](channel_drivers/README.md) does.
For this reason, ReActed offers the [local replay driver](channel_drivers/replay/replay_main.md). The *default* replay driver
takes as input a recorded session and sends the recorded messages to the appropriate destinations in the appropriate order
using the extra information that were generated while recording the session. 
Since the provided *replay driver* is just a driver, if such approach should not be appropriate or should be customized,
simply registering another replay driver with the custom logic instead of the default one will do the work.

From ReActed perspective it does not matter how or from where the messages are coming from, as long as a communication
happens through messages it can be fully recorded from a local driver. This implies that also a communication with a database
or with a remote web service can be logged and replayed whether [properly implemented](patterns.md#ReActor-Factory).

!> NOTE: What ReActed can do is replaying all the messages that were exchanged within a session. More your application
is written in a event sourced style, more the replay feature becomes useful. More parts do not interact with ReActed
messaging, more replay coverage you loose. You can find a showoff implementation of a web app backend that interacts 
with web clients, microservices and a database in a fully replayable way in the examples [package](https://github.com/reacted-io/reacted/tree/master/examples/src/main/java/io/reacted/examples/webappbackend)

  
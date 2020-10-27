# Channel Drivers

In ReActed communications are done over *channels*. A channel is defined as a pair of `ChannelId` and channel `Properties`.
A `ChannelId` provides information about the `ChannelType` of the channel and its name, while channel `Properties` contain details
about how to contact the reactor system over the defined channel.

ReActed abstracts the communication over the channel concept, so as soon as there is a *channel driver* it means that
any communication schema or technology can be used. A *channel driver* provides a specific implementation about how to
read or write from a certain medium type.

## Local Channels

Also local communication within a [reactor system](../reactor_system.md) is abstracted over the channel concept, so
this can be configured as well. Since a local channel defines how local reactors talk to each other, it is possibile
to specify only one local driver per reactor system

### Direct Channels

Allow direct communication with the reactor's mailbox. These are the most lightweight and fast channels available for
local communication. These come in 3 flavours:

- Direct Channel: speed oriented channel that uses memory only
- Direct Logger Channel: in memory channel that saves dumps all the messages sent into a human readable format
- Direct Simplified Logger Channel: in memory channel that saves a subset of the message fields, useful for quick debugging

### Indirect Channels

With this channel family a message is not delivered directly into the destination reactor's mailbox, but it is 
first saved on a medium and then read from the medium for being delivered. No message can be delivered if it's not on
the medium then, so it provides a full and realiable binary log of all the communication within a reactor system.
[Chronicle Queue](cq/cq_main.md) is an example of this type of channel.
Please note that having this kind of information allow us to perform a [replay](replay/replay_main.md) of all the messages
that were exchanged during a *recorded* session.

## Remote Channels

A remote channel is a channel towards any other reactor system other than the current one. This means that if two reactor
systems are on the same machine and within the same jvm instance, they can talk to each other using a remote channel.

*Remoting* does not allow any special capability because ReActed aims to be *location agnostic*, it is a transparent layer
that allows communication towards/from reactors without taking care of the details.

Differently from local channels, a reactor system can have as many remoting channels as wanted. Although this may be
reconsidered in future, **a remoting driver instance defines how to communicate with a specific remote channel.**

This means that if a reactor system wishes to talk over two different channels using the same underlying technology,
two channel(driver) instances should be configured. Please note that a driver/channel configuration does **not** define
how to talk with *other* reactor systems over that channel, but allows you to provide details about how the current reactor system
can be contacted by **other** reactor systems over that `ChannelId`
 



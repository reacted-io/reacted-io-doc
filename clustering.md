# Clustering 

Clustering is automatically enabled specifying one or more [remoting](channel_drivers/README.md#Remote Channels) and
one or more [service registries](registry_drivers/README.md)
ReActed automatically joins and discovers other nodes of the cluster using one or more [service registries](../registry_drivers/README.md).
Every ReActed node publishes itself and the details about the supported [ChannelIds](channel_drivers/README.md) on the known service registries.
When any ReActed node wants to talk with a remote reactor, the `ChannelId`s supported by the remote peer are checked and
if there is a match (a match means that also the local reactor system is registered for that `ChannelId`) the proper driver
and the remote channel `Properties` are used to generate a route and communicate.

## Remote Channels

A remote channel identifies a logical channel. 

## Service Discovery
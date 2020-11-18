# Chronicle Queue Driver

Chronicle Queue driver allows ReActed nodes to talk with each other using a [Chronicle Queue](https://github.com/OpenHFT/Chronicle-Queue) file.
comes in two flavours, *local* and *remote*.

A *Local Chronicle Queue Driver* is a [local channel driver](/channel_drivers/README.md#Local-Channels) and is used to store messages
exchanged within the local [reactor system](/reactor_system.md).

A *Remote Chronicle Queue Driver* is a [remote channel driver](/channel_drivers/README.md#Remote-Channels) and allows two [reactor systems](/reactor_system.md)
running under one or more different jvm instances to communicate with each other on the same machine.

## Chronicle Queue Driver Configuration

Other than the [common options](/channel_drivers/README.md#Channel-driver-configuration) the supported Chronicle Queue specific options are
the Chronicle Queue *topic* and the Chronicle Queue directory name.





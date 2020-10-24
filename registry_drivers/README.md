# Registry Drivers

ReActed allows you to use one or more service registries at the same time. It can be used as hub or as a glue between
different parts of your infrastructure. Service Registries are used to publish information about the nodes currently
active in the cluster and the `Services` they publish. Strictly speaking, any `ReActorRef` can be published for being
contacted from a remote host, a [service](../services.md) is a nice abstraction that provides you out of the box some extra feature.


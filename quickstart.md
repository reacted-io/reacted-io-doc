# Quickstart

If you are in a rush to be productive with ReActed and you do not need any detailed explanation, this section is for you. 
I will show a quick setup that will allow you to start experimenting.

## Creating a ReActorSystem

ReActors live within a ReActorSystem, so the first thing to do is creating one. The very least thing that you can specify
for a [reactor system](reactor_system.md) is its name. Remember, a reactor system name **must be unique** in a cluster.

```java
ReActorSystem showOffSystem = new ReActorSystem(ReActorSystemConfig.newBuilder()
                                                                   .setReactorSystemName("ShowOffReActorSystenName")
                                                                   .build()).initReActorSystem();
```
Always remember to init your reactor system once created.

## Creating a ReActor

Now you might want to create a reactor. From ReActed perspective a `ReActor` is not a special entity, it's not even
an entity to be fair. All what concerns ReActed are which `ReActions` should be called for a given type and some other
information that are nicely modeled by `ReActorConfig` regarding the runtime behavior. The `ReActor` or `ReActiveEntity`
interfaces are just a comfortable way to guide the user. You can *spawn* a reactor using the API call you prefer, but
as long as you can provide some `ReActions` and a `ReActorConfig` you are fine.



### Configuring a ReActor
### ReActions


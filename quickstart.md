# Quickstart

If you are in a rush to be productive with ReActed and you do not need any detailed explanation, this section is for you. 
I will show a quick setup that will allow you to start experimenting

## Creating a ReActorSystem

ReActors live within a ReActorSystem, so the first thing to do is creating one. The very least thing that you can specify
for a [reactor system](reactor_system.md) is its name. Remember, a reactor system name **must be unique** in a cluster.

```java
ReActorSystem showOffSystem = new ReActorSystem(ReActorSystemConfig.newBuilder()
                                                                   .setReactorSystemName("ShowOffReActorSystenName")
                                                                   .build()).initReActorSystem();
```
Always remember to init your reactor system once created

## Creating a ReActor

Now you might want to create a reactor. We will do more: we will create a local [service](services.md) that provides
the exact time and 

### Configuring a ReActor
### ReActions


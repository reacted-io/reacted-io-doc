# ZooKeeper Driver

[ZooKeeper](https://zookeeper.apache.org/) allows you to create a ReActed cluster. It is a `ReActor` and can be configured
through a `ZooKeeperDriverConfig` object. A new ZooKeeper driver must be registered for any ZooKeeper instance ReActed should
connect to. 

It can be easily configured using its configuration builder and added at the creation of a `ReActorSystem` like this

```java
 ZooKeeperDriverConfig zkDriverConfig = = ZooKeeperDriverConfig.newBuilder()
                                                               .setReActorName("TestClusterServiceRegistryDriver")
                                                               .build();
  new ReActorSystem(ReActorSystemConfig.newBuilder()
                                       .addServiceRegistryDriver(new ZooKeeperDriver(zooKeeperCfg))
```

!> NOTE: ZooKeeper Driver only supports ZooKeeper 3.6+ 
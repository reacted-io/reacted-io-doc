# Kafka Driver

Kafka Driver allows ReActed nodes to talk each other using [Kafka](https://kafka.apache.org/).
Other than the [common options](/channel_drivers/README.md#Channel-driver-configuration) the supported kafka specific options are:

```java
public Builder setBootstrapEndpoint(String bootstrapEndpoint)
```
*host*:*port* of the Kafka bootstrap endpoint 

```java
public Builder setTopic(String topic)
```
Kafka topic where messages will be read and written

```java
public Builder setGroupId(String groupId)
```
Kafka Group identifier.

!> If you setup two Kafka Driver instances to have the same group id, remember that the messages from the topic will be
once per group id 
 
```java
public Builder setMaxPollRecords(int maxPollRecords)
```
How many records will be attempted to be read at maximum at once
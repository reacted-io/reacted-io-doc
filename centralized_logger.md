# Centralized logger

ReActed offers a centralized logging system. It is backed up by [SL4J](http://www.slf4j.org/) and 
[Java Logging](https://docs.oracle.com/javase/10/core/java-logging-overview.htm#JSCOR-GUID-B83B652C-17EA-48D9-93D2-563AE1FF8EDA), 
so the syntax and the log levels come from the first and the behavior can be controlled configuring the second.

>Centralized logging works sending log messages to a `SystemLogger` reactor. This means that you will find evidence
>of the logging within the logged messages if a [persisting driver is used](channel_drivers/cq/cq_main.md), but also
>means that during a [replayed](channel_drivers/replay/replay_main.md) session if some extra logging statement should
>be put while debugging, this will have no effect.

Centralized logging can be [easily exploited](patterns.md#Change Data Capture) and used to automatically transform and
send all your logs towards a database or a log processing framework.
# PipelineBuilder.java

**Role:** Functional interface with a single `addLast(ChannelHandler)` method. Used as a lambda target in `BasePipelineFactory.initChannel` and in `addProtocolHandlers` to abstract away the actual `ChannelPipeline.addLast` call.
**Fits in:** `BasePipelineFactory.addTransportHandlers` and `addProtocolHandlers` accept a `PipelineBuilder` parameter; implementations call `pipeline::addLast`. The factory wraps each call to inject Guice members or wrap in `WrapperInboundHandler` as needed.
**Read next:** [[BasePipelineFactory.java]] (uses this interface), [[TrackerServer.java]] (anonymous subclass calls `addProtocolHandlers(PipelineBuilder, Config)`)

## Public API

```java
void addLast(ChannelHandler handler);
```

Single method — implementations are usually method references (`pipeline::addLast`) or lambdas that intercept for injection.

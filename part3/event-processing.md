Event Publishing & Processing
=============================

应用程序生成的事件需要分派给更新查询数据库，搜索引擎或任何其他需要它们的资源的组件：事件处理程序。 事件总线负责将事件消息分发给所有感兴趣的组件。 在接收端，事件处理器负责处理这些事件，其中包括调用相应的事件处理程序。

Publishing Events
-----------------

在绝大多数情况下，Aggregates将通过应用它们来发布事件。 但是，有时需要将事件（可能来自另一个组件内）直接发布到事件总线。 要发布事件，只需在“EventMessage”中包装描述事件的有效负载。 `GenericEventMessage.asEventMessage（Object）`方法允许你将任何对象包装到`EventMessage`中。 如果传递的对象已经是一个`EventMessage`，它就会返回。

Event Bus
---------
`EventBus`是将事件分派给订阅事件处理程序的机制。 Axon提供了Event Bus的两个实现：`SimpleEventBus`和`EmbeddedEventStore`。 虽然这两个实现都支持订阅和跟踪处理器（请参阅[事件处理器]（＃事件处理器）），但EmbeddedEventStore会保留事件，这允许您在稍后阶段重播它们。 SimpleEventBus具有易失性存储，并且一旦将它们发布到订阅组件，就会“忘记”事件。

使用配置API时，默认情况下使用`SimpleEventBus`。 为了配置`EmbeddedEventStore`，你需要提供一个StorageEngine的实现，它实际上存储了Events。

```java
    Configurer configurer = DefaultConfigurer.defaultConfiguration();
    configurer.configureEmbeddedEventStore(c -> new InMemoryEventStorageEngine());
```

Event Processors
----------------
事件处理程序定义接收事件时要执行的业务逻辑。 事件处理器是处理该处理的技术方面的组件。 他们启动一个工作单元，可能还有一个事务，还要确保相关数据可以正确地附加到在事件处理期间创建的所有消息。

事件处理器大致有两种形式：订阅和跟踪。 订阅事件处理器将自己订阅到事件源，并由发布机制管理的线程调用。 跟踪事件处理器，另一方面，使用它自己管理的线程从源头提取它们的消息。

### Assigning handlers to processors

所有处理器都有一个名称，用于标识跨JVM实例的处理器实例。 具有相同名称的两个处理器可以被视为同一个处理器的两个实例。

所有事件处理程序都附加到名称为事件处理程序的类的程序包名称的处理器上。

For example, the following classes:

- `org.axonframework.example.eventhandling.MyHandler`,
- `org.axonframework.example.eventhandling.MyOtherHandler`, and
- `org.axonframework.example.eventhandling.module.MyHandler`

will trigger the creation of two Processors:

- `org.axonframework.example.eventhandling` with 2 handlers, and 
- `org.axonframework.example.eventhandling.module` with a single handler

配置API允许您配置其他策略以将类分配给处理器，甚至可以将特定实例分配给特定处理器。

### Ordering Event Handlers within a single Event Processor

To order Event Handlers within an Event Processor, the ordering in which Event Handlers are registered (as described in the [Registering Event Handlers](../part2/event-handling.md#registering-event-handlers) section) is guiding. Thus, the ordering in which Event Handlers will be called by an Event Processor for Event Handling is their insertion ordering in the configuration API. 

If Spring is selected as the mechanism to wire everything, the ordering of the Event Handlers can be specified by adding the `@Order` annotation. This annotation should be placed on class level of your Event Handler class, adding a `integer` value to specify the ordering.

Do note that it is not possible to order Event Handlers which are not a part of the same Event Processor.

### Configuring processors

Processors take care of the technical aspects of handling an event, regardless of the business logic triggered by each event. However, the way "regular" (singleton, stateless) event handlers are Configured is slightly different from Sagas, as different aspects are important for both types of handlers.

#### Event Handlers ####

By default, Axon will use Subscribing Event Processors. It is possible to change how Handlers are assigned and how processors are configured using the `EventHandlingConfiguration` class of the Configuration API.

The `EventHandlingConfiguration` class defines a number of methods that can be used to define how processors need to be configured.

- `registerEventProcessorFactory` allows you to define a default factory method that creates Event Processors for which no explicit factories have been defined.

- `registerEventProcessor(String name, EventProcessorBuilder builder)` defines the factory method to use to create a Processor with given `name`. Note that such Processor is only created if `name` is chosen as the processor for any of the available Event Handler beans.

- `registerTrackingProcessor(String name)` defines that a processor with given name should be configured as a Tracking Event Processor, using default settings. It is configured with a TransactionManager and a TokenStore, both taken from the main configuration by default.

- `registerTrackingProcessor(String name, Function<Configuration, TrackingEventProcessorConfiguration> processorConfiguration, Function<Configuration, SequencingPolicy<? super EventMessage<?>>> sequencingPolicy)` defines that a processor with given name should be configured as a Tracking Processor, and use the given `TrackingEventProcessorConfiguration` to read the configuration settings for multi-threading. The `SequencingPolicy` defines which expectations the processor has on sequential processing of events. See [Parallel Processing](#parallel-processing) for more details.

- `usingTrackingProcessors()` sets the default to Tracking Processors instead of Subscribing ones.

#### Sagas ####

Sagas配置使用`SagaConfiguration`类。 它提供静态方法来初始化一个实例，用于跟踪处理或订阅。

要配置Saga在订阅模式下运行，只需执行以下操作：

```java
SagaConfiguration<MySaga> sagaConfig = SagaConfiguration.subscribingSagaManager(MySaga.class);
```

If you don't want to use the default EventBus / Store as source for this Saga to get its messages from, you can define another source of messages as well:

```java
SagaConfiguration.subscribingSagaManager(MySaga.class, c -> /* define source here */);
```
Another variant of the `subscribingSagaManager()` method allows you to pass a (builder for an) `EventProcessingStrategy`. By default, Sagas are invoked in synchronously. This can be made asynchronous using this method. However, using Tracking Processors is the preferred way for asynchronous invocation.

To configure a Saga to use a Tracking Processor, simply do:
```java
SagaConfiguration.trackingSagaManager(MySaga.class);
``` 

This will set the default properties, meaning a single Thread is used to process events. To change this:
```java
SagaConfiguration.trackingSagaManager(MySaga.class)
                 // configure 4 threads
                 .configureTrackingProcessor(c -> TrackingProcessingConfiguration.forParallelProcessing(4)) 
```
The `TrackingProcessingConfiguration` has a few methods allowing you to specify how many segments will be created and which ThreadFactory should be used to create Processor threads. See [Parallel Processing](#parallel-processing) for more details.

Check out the API documentation (JavaDoc) of the `SagaConfiguration` class for full details on how to configure event handling for a Saga.


### Token Store ###

跟订阅处理器不同，跟踪处理器需要一个令牌存储器来存储他们的进度。跟踪处理器通过其事件流接收的每条消息都伴随有一个令牌。 该令牌允许处理器在稍后的时间点重新打开该流，从最后一个事件中提取它离开的位置。

The Configuration API takes the Token Store, as well as most other components Processors need from the Global Configuration instance. If no TokenStore is explicitly defined, an `InMemoryTokenStore` is used, which is *not recommended in production*.

To configure a different Token Store, use `Configurer.registerComponent(TokenStore.class, conf -> ... create token store ...)`

Note that you can override the TokenStore to use with Tracking Processors in the respective `EventHandlingConfiguration` or `SagaConfiguration` that defines that Processor. Where possible, it is recommended to use a Token Store that stores tokens in the same database as where the Event Handlers update the view models. This way, changes to the view model can be stored atomically with the changed tokens, guaranteeing exactly once processing semantics.

### Parallel Processing ###

As of Axon Framework 3.1, Tracking Processors can use multiple threads to process an Event Stream. They do so, by claiming a so-called segment, identifier by a number. Normally, a single thread will process a single Segment.

The number of Segments used can be defined. When a Processor starts for the first time, it can initialize a number of segments. This number defines the maximum number of threads that can process events simultaneously. Each node running of a TrackingProcessor will attempt to start its configured amount of Threads, to start processing these.

Event Handlers may have specific expectations on the ordering of events. If this is the case, the processor must ensure these events are sent to these Handlers in that specific order. Axon uses the `SequencingPolicy` for this. The `SequencingPolicy` is essentially a function, that returns a value for any given message. If the return value of the `SequencingPolicy` function is equal for two distinct event messages, it means that those messages must be processed sequentially. By default, Axon components will use the `SequentialPerAggregatePolicy`, which makes it so that Events published by the same Aggregate instance will be handled sequentially.

A Saga instance is never invoked concurrently by multiple threads. Therefore, a Sequencing Policy for a Saga is irrelevant. Axon will ensure each Saga instance receives the Events it needs to process in the order they have been published on the Event Bus.

> ** Note **
>
> Note that Subscribing Processors don't manage their own threads. Therefore, it is not possible to configure how they should receive their events. Effectively, they will always work on a sequential-per-aggregate basis, as that is generally the level of concurrency in the Command Handling component.

#### Multi-node processing ####

对于跟踪处理器来说，处理事件的线程是否都运行在同一节点上，或者在托管相同（逻辑）TrackingProcessor的不同节点上运行并不重要。 当具有相同名称的两个TrackingProcessor实例在不同机器上处于活动状态时，它们被视为同一个逻辑处理器的两个实例。 他们将“竞争”事件流的各个部分。 每个实例将“声明”一个段，防止分配给该段的事件在其他节点上处理。

The `TokenStore` instance will use the JVM's name (usually a combination of the host name and process ID) as the default `nodeId`. This can be overridden in `TokenStore` implementations that support multi-node processing.

Distributing Events
-------------------

In some cases, it is necessary to publish events to an external system, such as a message broker.

### Spring AMQP

Axon provides out-of-the-box support to transfer Events to and from an AMQP message broker, such as Rabbit MQ.

#### Forwarding events to an AMQP Exchange

The `SpringAMQPPublisher` forwards events to an AMQP Exchange. It is initialized with a `SubscribableMessageSource`, which is generally the `EventBus` or `EventStore`. Theoretically, this could be any source of Events that the publisher can Subscribe to.

To configure the SpringAMQPPublisher, simply define an instance as a Spring Bean. There is a number of setter methods that allow you to specify the behavior you expect, such as Transaction support,  publisher acknowledgements (if supported by the broker), and the exchange name.
 
The default exchange name is 'Axon.EventBus'. 

> **Note**
>
> Note that exchanges are not automatically created. You must still declare the Queues, Exchanges and Bindings you wish to use. Check the Spring documentation for more information.

#### Reading Events from an AMQP Queue

Spring has extensive support for reading messages from an AMQP Queue. However, this needs to be 'bridged' to Axon, so that these messages can be handled from Axon as if they are regular Event Messages.

The `SpringAMQPMessageSource` allows Event Processors to read messages from a Queue, instead of the Event Store or Event Bus. It acts as an adapter between Spring AMQP and the `SubscribableMessageSource` needed by these processors.

The easiest way to configure the SpringAMQPMessageSource, is by defining a bean which overrides the default `onMessage` method and annotates it with `@RabbitListener`, as follows:

```java
@Bean
public SpringAMQPMessageSource myMessageSource(Serializer serializer) {
    return new SpringAMQPMessageSource(serializer) {
        @RabbitListener(queues = "myQueue")
        @Override
        public void onMessage(Message message, Channel channel) throws Exception {
            super.onMessage(message, channel);
        }
    };
}
```

Spring's `@RabbitListener` annotation tells Spring that this method needs to be invoked for each message on the given Queue ('myQueue' in the example). This method simply invokes the `super.onMessage()` method, which performs the actual publication of the Event to all the processors that have been subscribed to it.

To subscribe Processors to this MessageSource, pass the correct `SpringAMQPMessageSource` instance to the constructor of the Subscribing Processor:
```java
// in an @Configuration file:
@Autowired
public void configure(EventHandlingConfiguration ehConfig, SpringAmqpMessageSource myMessageSource) {
    ehConfig.registerSubscribingEventProcessor("myProcessor", c -> myMessageSource);
}
```

Note that Tracking Processors are not compatible with the SpringAMQPMessageSource.

Asynchronous Event Processing
-----------------------------

The recommended approach to handle Events asynchronously is by using a Tracking Event Processor. This implementation can guarantee processing of all events, even in case of a system failure (assuming the Events have been persisted).

However, it is also possible to handle Events asynchronously in a `SubscribingProcessor`. To achieve this, the `SubscribingProcessor` must be configured with an `EventProcessingStrategy`. This strategy can be used to change how invocations of the Event Listeners should be managed.

The default strategy (`DirectEventProcessingStrategy`) invokes these handlers in the thread that delivers the Events. This allows processors to use existing transactions.
 
The other Axon-provided strategy is the `AsynchronousEventProcessingStrategy`. It uses an Executor to asynchronously invoke the Event Listeners.

Even though the `AsynchronousEventProcessingStrategy` executes asynchronously, it is still desirable that certain events are processed sequentially. The `SequencingPolicy` defines whether events must be handled sequentially, in parallel or a combination of both. Policies return a sequence identifier of a given event. If the policy returns an equal identifier for two events, this means that they must be handled sequentially by the event handler. A `null` sequence identifier means the event may be processed in parallel with any other event.

Axon provides a number of common policies you can use:

* The `FullConcurrencyPolicy` will tell Axon that this event handler may handle all events concurrently. This means that there is no relationship between the events that require them to be processed in a particular order.

* The `SequentialPolicy` tells Axon that all events must be processed sequentially. Handling of an event will start when the handling of a previous event is finished.

* `SequentialPerAggregatePolicy` will force domain events that were raised from the same aggregate to be handled sequentially. However, events from different aggregates may be handled concurrently. This is typically a suitable policy to use for event listeners that update details from aggregates in database tables.

Besides these provided policies, you can define your own. All policies must implement the `SequencingPolicy` interface. This interface defines a single method, `getSequenceIdentifierFor`, that returns the sequence identifier for a given event. Events for which an equal sequence identifier is returned must be processed sequentially. Events that produce a different sequence identifier may be processed concurrently. For performance reasons, policy implementations should return `null` if the event may be processed in parallel to any other event. This is faster, because Axon does not have to check for any restrictions on event processing.

It is recommended to explicitly define an `ErrorHandler` when using the `AsynchronousEventProcessingStrategy`. The default `ErrorHandler` propagates exceptions, but in an asynchronous execution, there is nothing to propagate to, other than the Executor. This may result in Events not being processed.
Instead, it is recommended to use an `ErrorHandler` that reports errors and allows processing to continue. The `ErrorHandler` is configured on the constructor of the `SubscribingEventProcessor`, where the `EventProcessingStrategy` is also provided.

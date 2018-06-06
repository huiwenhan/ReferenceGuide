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

从Axon Framework 3.1开始，跟踪处理器可以使用多个线程来处理事件流。 他们这样做，通过一个数字为标识符的声称所谓的分段。 通常，一个线程将处理单个段。

可以定义使用的段的数量。 当处理器第一次启动时，它可以初始化多个段。 此数字定义可以同时处理事件的最大线程数。 TrackingProcessor的每个节点都将尝试启动配置的线程数量，以开始处理这些线程。

事件处理程序可能对事件的排序有特定的期望。 如果是这种情况，处理器必须确保这些事件以特定的顺序发送给这些处理程序。 Axon为此使用`SequencingPolicy`。 SequencingPolicy本质上是一个函数，它为任何给定的消息返回一个值。 如果`SequencingPolicy`函数的返回值对于两个不同的事件消息相等，则意味着这些消息必须按顺序处理。 默认情况下，Axon组件将使用`SequentialPerAggregatePolicy`，它使得由相同的聚集实例发布的事件将被顺序处理。

一个Saga实例永远不会被多个线程同时调用。 因此，佐贺的排序策略是无关紧要的。 Axon将确保每个Saga实例都按它们在事件总线上发布的顺序接收它需要处理的事件。

> ** Note **
>
> 请注意，订阅处理器不管理他们自己的线程。 因此，不可能配置他们应该如何接收他们的事件。 实际上，它们将始终按每个聚合按顺序进行工作，因为这通常是命令处理组件中的并发级别。

#### Multi-node processing ####

对于跟踪处理器来说，处理事件的线程是否都运行在同一节点上，或者在托管相同（逻辑）TrackingProcessor的不同节点上运行并不重要。 当具有相同名称的两个TrackingProcessor实例在不同机器上处于活动状态时，它们被视为同一个逻辑处理器的两个实例。 他们将“竞争”事件流的各个部分。 每个实例将“声明”一个段，防止分配给该段的事件在其他节点上处理。

The `TokenStore` instance will use the JVM's name (usually a combination of the host name and process ID) as the default `nodeId`. This can be overridden in `TokenStore` implementations that support multi-node processing.

Distributing Events
-------------------

在某些情况下，有必要将事件发布到外部系统，如消息代理。

### Spring AMQP

Axon提供了开箱即用的支持，可以将事件传送到AMQP消息代理（如Rabbit MQ）和从AMQP消息代理传送事件。

#### Forwarding events to an AMQP Exchange

`SpringAMQPPublisher`将事件转发给AMQP Exchange。 它使用`SubscribableMessageSource`进行初始化，通常是`EventBus`或`EventStore`。 理论上，这可能是发布者可以订阅的任何事件的来源。

要配置SpringAMQPPublisher，只需将一个实例定义为Spring Bean。 有许多setter方法允许您指定您期望的行为，例如事务支持，发布者确认（如果代理支持）以及交换名称。
 
默认交换名称是'Axon.EventBus'。

> **Note**
>
> 请注意，交换不会自动创建。 您仍然必须声明您希望使用的队列，交换和绑定。 查看Spring文档以获取更多信息。

#### Reading Events from an AMQP Queue

Spring对从AMQP队列读取消息提供了广泛的支持。 但是，这需要与Axon'桥接'，以便这些消息可以从Axon处理，就好像它们是常规的事件消息一样。

`SpringAMQPMessageSource`允许事件处理器从队列中读取消息，而不是事件存储或事件总线。 它充当Spring AMQP和这些处理器所需的`SubscribableMessageSource`之间的适配器。

配置SpringAMQPMessageSource最简单的方法是定义一个bean，它覆盖默认的`onMessage`方法并用`@ RabbitListener`注释它，如下所示：

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

异步处理事件的推荐方法是使用跟踪事件处理器。 这个实现可以保证处理所有事件，即使在系统故障的情况下（假设事件已被持续）。

但是，也可以在“订阅处理器”中异步处理事件。 为了达到这个目的，'SubscribingProcessor`必须配置一个`EventProcessingStrategy`。 这个策略可以用来改变如何管理事件监听器的调用。

默认策略（`DirectEventProcessingStrategy`）在传递事件的线程中调用这些处理程序。 这允许处理器使用现有的事务。
 
另一个Axon提供的策略是`AsynchronousEventProcessingStrategy`。 它使用Executor异步调用事件监听器。

即使“AsynchronousEventProcessingStrategy”异步执行，仍然需要按顺序处理某些事件。 SequencingPolicy定义事件是否必须按顺序，并行或两者结合来处理。 策略返回给定事件的序列标识符。 如果策略为两个事件返回相同的标识符，这意味着它们必须由事件处理程序按顺序处理。 “空”序列标识符表示该事件可以与任何其他事件并行处理。

Axon提供了许多可以使用的通用策略：

* FullConcurrencyPolicy将告诉Axon该事件处理程序可以同时处理所有事件。 这意味着要求按特定顺序处理的事件之间没有关系。

* SequentialPolicy告诉Axon所有事件必须按顺序处理。 事件的处理将在前一个事件的处理完成时开始。

* “SequentialPerAggregatePolicy”将强制从相同聚合中引发的域事件按顺序处理。 但是，来自不同聚合的事件可能会同时处理。 这通常是用于更新数据库表中聚合的细节的事件侦听器的合适策略。

除了这些提供的政策，您可以定义自己的。 所有策略都必须实现`SequencingPolicy`接口。 这个接口定义了一个方法，`getSequenceIdentifierFor`，它返回给定事件的序列标识符。 必须按顺序处理返回了相同序列标识符的事件。 产生不同序列标识符的事件可以同时处理。 出于性能原因，如果事件可能与任何其他事件并行处理，则策略实现应返回“null”。 这是更快的，因为Axon不必检查事件处理的任何限制。

议在使用“AsynchronousEventProcessingStrategy”时明确定义一个`ErrorHandler`。 默认的`ErrorHandler`传播异常，但在异步执行中，除Executor之外没有任何东西可传播。 这可能会导致事件未被处理。

相反，建议使用报告错误并允许处理继续的`ErrorHandler`。 `ErrorHandler`配置在`SubscribingEventProcessor`的构造函数中，其中还提供了`EventProcessingStrategy`。

Command Dispatching
===================

使用明确的命令调度机制具有许多优点。 首先，有一个对象清楚地描述了客户的意图。 通过记录命令，您可以存储意图数据和相关数据以备将来参考。 命令处理还可以很容易地通过Web服务将命令处理组件公开给远程客户端。 测试也变得容易很多，您可以通过定义开始的情况（给定），命令执行（时机）和预期结果（然后）列出一些事件和命令来定义测试脚本（参见[测试]（../part2/testing.md））。 最后一个主要优势是，在同步和异步以及本地和分布式命令处理之间切换非常容易。

这并不意味着使用显式命令对象进行命令分派是实现它的唯一方法。 Axon的目标不是规定特定的工作方式，而是支持您按自己的方式进行操作，同时提供最佳做法作为默认行为。 仍然可以使用可以调用的服务层来执行命令。 该方法只需启动一个工作单元（请参见[工作单元]（../ part1 / messaging-concepts.md＃工作单元）），并在方法完成时对其执行提交或回滚。

接下来的部分概述了与使用Axon框架设置Command调度基础架构有关的任务。

The Command Gateway
===================

Command Gateway是Command调度机制的便捷接口。 虽然您不需要使用网关来分派命令，但通常这是最简单的选择。

有两种使用Command Gateway的方法。 首先是使用Axon提供的CommandGateway接口和DefaultCommandGateway实现。 命令网关提供了许多方法，允许您发送命令并同步等待结果，具有超时或异步。

另一种选择也许是最灵活的。 你可以使用`CommandGatewayFactory`将几乎任何接口变成一个命令网关。 这使您可以使用强类型定义应用程序的界面，并声明自己的（已检查）业务异常。 Axon会在运行时自动为该接口生成一个实现。

Configuring the Command Gateway
-------------------------------

您的自定义网关和Axon提供的自定义网关都必须配置为至少可以访问Command Bus。 另外，Command Gateway可以配置一个`RetryScheduler`，`CommandDispatchInterceptor`s和`CommandCallback`s。

当命令执行失败时，`RetryScheduler`能够安排重试。 IntervalRetryScheduler是一个实现，它将以设定的时间间隔重试给定的命令，直到它成功为止，或者重试次数达到最大值。 当一个命令因显式非瞬态异常而失败时，根本不会执行重试。 请注意，只有当命令由于RuntimeException而失败时，才会调用重试调度程序。 检查异常被视为“业务异常”，并且永远不会触发重试。 “RetryScheduler”的典型用法是在分布式Command Bus上分派命令时。 如果某个节点发生故障，则重试调度程序将导致将命令分派给能够处理该命令的下一个节点（请参阅[分配命令总线]（＃distribute-the-command-bus））。

CommandDispatchInterceptor允许在将CommandMessage发送到Command Bus之前进行修改。 与在CommandBus上配置的CommandDispatchInterceptor相比，只有在通过此网关发送消息时才会调用这些拦截器。 例如，拦截器可用于将元数据附加到命令或进行验证。

为每个发送的命令调用CommandCallback。 这允许通过此网关发送的所有命令的一些通用行为，而不管它们的类型如何。

Creating a Custom Command Gateway
---------------------------------

Axon允许将自定义界面用作命令网关。 在接口中声明的每个方法的行为都基于参数类型，返回类型和声明的异常。 使用这个网关不仅方便，而且允许您在需要的地方模拟您的界面，从而使测试变得更加简单。

这是参数如何影响CommandGateway的行为：

-   预计第一个参数是要分派的实际命令对象。

-   用@ MetaDataValue注解的参数将赋值给元数据字段，其中标识符作为注释参数传递

-   “MetaData”类型的参数将与CommandMessage上的“MetaData”合并。 如果它们的密钥相同，则由后面的参数定义的元数据将覆盖较早参数的元数据。

-   CommandCallback类型的参数将在命令处理后调用onSuccess或onFailure。 您可能传入多个回调，并可能与返回值组合。 在这种情况下，回调的调用将始终与返回值（或异常）相匹配。

-   最后两个参数的类型可以是`long`（或`int`）和`TimeUnit`。 在这种情况下，只要这些参数指示，该方法将最多阻止。 该方法如何在超时上作出反应取决于方法中声明的异常（请参见下文）。 请注意，如果方法的其他属性完全阻止阻塞，则永远不会发生超时。

方法的声明返回值也会影响其行为：

-   A `void` return type will cause the method to return immediately, unless there are other indications on the method that one would want to wait, such as a timeout or declared exceptions.

-   Return types of `Future`, `CompletionStage` and `CompletableFuture` will cause the method to return immediately. You can access the result of the Command Handler using the `CompletableFuture` instance returned from the method. Exceptions and timeouts declared on the method are ignored.

-   Any other return type will cause the method to block until a result is available. The result is cast to the return type (causing a ClassCastException if the types don't match).

Exceptions have the following effect:

-   Any declared checked exception will be thrown if the Command Handler (or an interceptor) threw an exception of that type. If a checked exception is thrown that has not been declared, it is wrapped in a `CommandExecutionException`, which is a `RuntimeException`.

-   When a timeout occurs, the default behavior is to return `null` from the method. This can be changed by declaring a `TimeoutException`. If this exception is declared, a `TimeoutException` is thrown instead.

-   When a Thread is interrupted while waiting for a result, the default behavior is to return null. In that case, the interrupted flag is set back on the Thread. By declaring an `InterruptedException` on the method, this behavior is changed to throw that exception instead. The interrupt flag is removed when the exception is thrown, consistent with the java specification.

-   Other Runtime Exceptions may be declared on the method, but will not have any effect other than clarification to the API user.

Finally, there is the possibility to use annotations:

-   As specified in the parameter section, the `@MetaDataValue` annotation on a parameter will have the value of that parameter added as meta data value. The key of the meta data entry is provided as parameter to the annotation.

-   Methods annotated with `@Timeout` will block at most the indicated amount of time. This annotation is ignored if the method declares timeout parameters.

-   Classes annotated with `@Timeout` will cause all methods declared in that class to block at most the indicated amount of time, unless they are annotated with their own `@Timeout` annotation or specify timeout parameters.

``` java
public interface MyGateway {

    // fire and forget
    void sendCommand(MyPayloadType command);

    // method that attaches meta data and will wait for a result for 10 seconds
    @Timeout(value = 10, unit = TimeUnit.SECONDS)
    ReturnValue sendCommandAndWaitForAResult(MyPayloadType command,
                                             @MetaDataValue("userId") String userId);

    // alternative that throws exceptions on timeout
    @Timeout(value = 20, unit = TimeUnit.SECONDS)
    ReturnValue sendCommandAndWaitForAResult(MyPayloadType command)
                         throws TimeoutException, InterruptedException;

    // this method will also wait, caller decides how long
    void sendCommandAndWait(MyPayloadType command, long timeout, TimeUnit unit)
                         throws TimeoutException, InterruptedException;
}

// To configure a gateway:
CommandGatewayFactory factory = new CommandGatewayFactory(commandBus);
// note that the commandBus can be obtained from the `Configuration` object returned on `configurer.initialize()`.
MyGateway myGateway = factory.createGateway(MyGateway.class);
```

The Command Bus
===============

命令总线是将命令调度到它们各自的命令处理程序的机制。 每个命令总是发送到一个命令处理程序。 如果没有命令处理程序可用于分派的命令，则引发`NoHandlerForCommandException`异常。 将多个命令处理程序订阅到相同的命令类型将导致订阅相互替换。 在这种情况下，最后一次订阅获胜。

Dispatching commands
--------------------

CommandBus提供了两种方法来将命令分派给它们各自的处理程序：`dispatch（commandMessage，callback）`和`dispatch（commandMessage）`。 第一个参数是包含实际分派的命令的消息。 可选的第二个参数需要一个回调，该命令允许在命令处理完成时通知调度组件。 这个回调函数有两个方法：`onSuccess（）`和`onFailure（）`，它们分别在命令处理正常返回或者抛出异常时被调用。

调用组件可能不会认为回调是在调度该命令的同一个线程中调用的。 如果调用线程在继续之前依赖于结果，则可以使用“FutureCallback”。 它是'Future'（在java.concurrent包中定义）和Axon的 'CommandCallback` 的组合。 或者，考虑使用Command Gateway。

如果应用程序不直接对Command的结果感兴趣，则可以使用`dispatch（commandMessage）`方法。

SimpleCommandBus
----------------

“SimpleCommandBus”顾名思义就是最简单的实现。 它可以直接处理调度它们的线程中的命令。 处理完命令后，修改后的聚合被保存，生成的事件将在同一个线程中发布。 在大多数情况下，例如Web应用程序，此实现将满足您的需求。 SimpleCommandBus是配置API中默认使用的实现。

像大多数`CommandBus`实现一样，`SimpleCommandBus`允许配置拦截器。 在命令总线上分派命令时调用CommandDispatchInterceptor。 CommandHandlerInterceptor在实际的命令处理器方法之前被调用，允许你修改或阻止命令。 有关更多信息，请参见[命令拦截器]（＃命令拦截器）。

由于所有命令处理都在同一个线程中完成，因此此实现仅限于JVM的边界。 这种实施的表现很好，但并不特别。 要跨越JVM边界，或为了充分利用CPU周期，请查看其他`CommandBus`实现。

AsynchronousCommandBus
----------------------

顾名思义，`AsynchronousCommandBus`实现从调度它们的线程异步执行Commands。 它使用Executor在不同的Thread上执行实际的处理逻辑。

默认情况下，`AsynchronousCommandBus`使用无限制的缓存线程池。 这意味着一个线程在分派Command时被创建。 已完成处理命令的线程将重新用于新命令。 如果线程没有处理60秒的命令，线程将被停止。

或者，可以提供“Executor”实例来配置不同的线程策略。

请注意，应该在停止应用程序时关闭“AsynchronousCommandBus”，以确保任何等待的线程都已正确关闭。 要关闭，请调用`shutdown（）`方法。 如果它实现了`ExecutorService`接口，它也会关闭任何提供的`Executor`实例。

DisruptorCommandBus
-------------------

`SimpleCommandBus`具有合理的性能特征，尤其是当您在[性能调优]（../ part4 / performance-tuning.md＃performance-tuning）中查看性能提示时。 SimpleCommandBus需要锁定以防止多个线程同时访问同一个聚合的事实会导致处理开销和锁定争用。

DisruptorCommandBus采用不同的方法来执行多线程处理。 不是每个进程使用多个线程，而是有多个线程，每个线程负责处理一部分进程。 DisruptorCommandBus使用Disruptor（<http://lmax-exchange.github.io/disruptor/>）这个小型的并发编程框架，通过采用不同的多线程方法来获得更好的性能。 不是在调用线程中进行处理，而是将任务交给两组线程，每个线程都负责处理一部分处理。 第一组线程将执行命令处理程序，更改聚合的状态。 第二组将存储事件并将其发布到事件存储。

虽然`DisruptorCommandBus`很容易比'SimpleCommandBus`高出4倍（！），但有一些限制：

-   DisruptorCommandBus只支持事件源聚合。 该命令总线还充当Disruptor处理的聚合的存储库。 要获得对存储库的引用，请使用`createRepository（AggregateFactory）`。

-   命令只能导致单个聚合实例中的状态更改。

-   使用缓存时，它只允许给定标识符的单个聚合。 这意味着不可能有两个具有相同标识符的不同类型的聚合体。

-   命令通常不会导致需要回滚工作单元的故障。 发生回滚时，DisruptorCommandBus不能保证命令按照它们的分派顺序进行处理。 此外，它需要重试其他许多命令，导致不必要的计算。

-   在创建新的聚合实例时，更新创建实例的命令可能并不全都按照提供的顺序发生。 一旦创建了聚合，所有的命令将按照它们被调度的顺序执行。 为了确保顺序，在创建命令上使用回调来等待创建的聚合。 它不应该超过几毫秒。

To construct a `DisruptorCommandBus` instance, you need an `EventStore`. This component is explained in [Repositories and Event Stores](repositories-and-event-stores.md).

Optionally, you can provide a `DisruptorConfiguration` instance, which allows you to tweak the configuration to optimize performance for your specific environment:

-   Buffer size: The number of slots on the ring buffer to register incoming commands. Higher values may increase throughput, but also cause higher latency. Must always be a power of 2. Defaults to 4096.

-   ProducerType: Indicates whether the entries are produced by a single thread, or multiple. Defaults to multiple.

-   WaitStrategy: The strategy to use when the processor threads (the three threads taking care of the actual processing) need to wait for each other. The best WaitStrategy depends on the number of cores available in the machine, and the number of other processes running. If low latency is crucial, and the DisruptorCommandBus may claim cores for itself, you can use the `BusySpinWaitStrategy`. To make the Command Bus claim less of the CPU and allow other threads to do processing, use the `YieldingWaitStrategy`. Finally, you can use the `SleepingWaitStrategy` and `BlockingWaitStrategy` to allow other processes a fair share of CPU. The latter is suitable if the Command Bus is not expected to be processing full-time. Defaults to the `BlockingWaitStrategy`.

-   Executor: Sets the Executor that provides the Threads for the `DisruptorCommandBus`. This executor must be able to provide at least 4 threads. 3 of the threads are claimed by the processing components of the `DisruptorCommandBus`. Extra threads are used to invoke callbacks and to schedule retries in case an Aggregate's state is detected to be corrupt. Defaults to a `CachedThreadPool` that provides threads from a thread group called "DisruptorCommandBus".

-   TransactionManager: Defines the Transaction Manager that should ensure that the storage and publication of events are executed transactionally.

-   InvokerInterceptors: Defines the `CommandHandlerInterceptor`s that are to be used in the invocation process. This is the process that calls the actual Command Handler method.

-   PublisherInterceptors: Defines the `CommandHandlerInterceptor`s that are to be used in the publication process. This is the process that stores and publishes the generated events.

-   RollbackConfiguration: Defines on which Exceptions a Unit of Work should be rolled back. Defaults to a configuration that rolls back on unchecked exceptions.

-   RescheduleCommandsOnCorruptState: Indicates whether Commands that have been executed against an Aggregate that has been corrupted (e.g. because a Unit of Work was rolled back) should be rescheduled. If `false` the callback's `onFailure()` method will be invoked. If `true` (the default), the command will be rescheduled instead.

-   CoolingDownPeriod: Sets the number of seconds to wait to make sure all commands are processed. During the cooling down period, no new commands are accepted, but existing commands are processed, and rescheduled when necessary. The cooling down period ensures that threads are available for rescheduling the commands and calling callbacks. Defaults to 1000 (1 second).

-   Cache: Sets the cache that stores aggregate instances that have been reconstructed from the Event Store. The cache is used to store aggregate instances that are not in active use by the disruptor.

-   InvokerThreadCount: The number of threads to assign to the invocation of command handlers. A good starting point is half the number of cores in the machine.

-   PublisherThreadCount: The number of threads to use to publish events. A good starting point is half the number of cores, and could be increased if a lot of time is spent on IO.

-   SerializerThreadCount: The number of threads to use to pre-serialize events. This defaults to 1, but is ignored if no serializer is configured.

-   Serializer: The serializer to perform pre-serialization with. When a serializer is configured, the `DisruptorCommandBus` will wrap all generated events in a `SerializationAware` message. The serialized form of the payload and meta data is attached before they are published to the Event Store.

Command Interceptors
====================

One of the advantages of using a command bus is the ability to undertake action based on all incoming commands. Examples are logging or authentication, which you might want to do regardless of the type of command. This is done using Interceptors.

There are different types of interceptors: Dispatch Interceptors and Handler Interceptors. Dispatch Interceptors are invoked before a command is dispatched to a Command Handler. At that point, it may not even be sure that any handler exists for that command. Handler Interceptors are invoked just before the Command Handler is invoked.

Message Dispatch Interceptors
-----------------------------

Message Dispatch Interceptors are invoked when a command is dispatched on a Command Bus. They have the ability to alter the Command Message, by adding Meta Data, for example, or block the command by throwing an Exception. These interceptors are always invoked on the thread that dispatches the Command.

### Structural validation

There is no point in processing a command if it does not contain all required information in the correct format. In fact, a command that lacks information should be blocked as early as possible, preferably even before any transaction is started. Therefore, an interceptor should check all incoming commands for the availability of such information. This is called structural validation.

Axon Framework has support for JSR 303 Bean Validation based validation. This allows you to annotate the fields on commands with annotations like `@NotEmpty` and `@Pattern`. You need to include a JSR 303 implementation (such as Hibernate-Validator) on your classpath. Then, configure a `BeanValidationInterceptor` on your Command Bus, and it will automatically find and configure your validator implementation. While it uses sensible defaults, you can fine-tune it to your specific needs.

> **Tip**
>
> You want to spend as few resources on an invalid command as possible. Therefore, this interceptor is generally placed in the very front of the interceptor chain. In some cases, a Logging or Auditing interceptor might need to be placed in front, with the validating interceptor immediately following it.

The BeanValidationInterceptor also implements `MessageHandlerInterceptor`, allowing you to configure it as a Handler Interceptor as well.

Message Handler Interceptors
----------------------------

Message Handler Interceptors can take action both before and after command processing. Interceptors can even block command processing altogether, for example for security reasons.

Interceptors must implement the `MessageHandlerInterceptor` interface. This interface declares one method, `handle`, that takes three parameters: the command message, the current `UnitOfWork` and an `InterceptorChain`. The `InterceptorChain` is used to continue the dispatching process.

Unlike Dispatch Interceptors, Handler Interceptors are invoked in the context of the Command Handler. That means they can attach correlation data based on the Message being handled to the Unit of Work, for example. This correlation data will then be attached to messages being created in the context of that Unit of Work.

Handler Interceptors are also typically used to manage transactions around the handling of a command. To do so, register a `TransactionManagingInterceptor`, which in turn is configured with a `TransactionManager` to start and commit (or roll back) the actual transaction.

Distributing the Command Bus
============================

The CommandBus implementations described in earlier only allow Command Messages to be dispatched within a single JVM. Sometimes, you want multiple instances of Command Buses in different JVMs to act as one. Commands dispatched on one JVM's Command Bus should be seamlessly transported to a Command Handler in another JVM while sending back any results.

That's where the `DistributedCommandBus` comes in. Unlike the other `CommandBus` implementations, the `DistributedCommandBus` does not invoke any handlers at all. All it does is form a "bridge" between Command Bus implementations on different JVM's. Each instance of the `DistributedCommandBus` on each JVM is called a "Segment".

![Structure of the Distributed Command Bus](distributed-command-bus.png)

> **Note**
>
> While the distributed command bus itself is part of the Axon Framework Core module, it requires components that you can find in one of the *axon-distributed-commandbus-...* modules. If you use Maven, make sure you have the appropriate dependencies set. The groupId and version are identical to those of the Core module.

The `DistributedCommandBus` relies on two components: a `CommandBusConnector`, which implements the communication protocol between the JVM's, and the `CommandRouter`, which chooses a destination for each incoming Command. This Router defines which segment of the Distributed Command Bus should be given a Command, based on a Routing Key calculated by a Routing Strategy. Two commands with the same Routing Key will always be routed to the same segment, as long as there is no change in the number and configuration of the segments. Generally, the identifier of the targeted aggregate is used as a routing key.

Two implementations of the `RoutingStrategy` are provided: the `MetaDataRoutingStrategy`, which uses a Meta Data property in the Command Message to find the routing key, and the `AnnotationRoutingStrategy`, which uses the `@TargetAggregateIdentifier` annotation on the Command Messages payload to extract the Routing Key. Obviously, you can also provide your own implementation.

By default, the RoutingStrategy implementations will throw an exception when no key can be resolved from a Command Message. This behavior can be altered by providing a UnresolvedRoutingKeyPolicy in the constructor of the MetaDataRoutingStrategy or AnnotationRoutingStrategy. There are three possible policies:

-   ERROR: This is the default, and will cause an exception to be thrown when a Routing Key is not available

-   RANDOM\_KEY: Will return a random value when a Routing Key cannot be resolved from the Command Message. This effectively means that those commands will be routed to a random segment of the Command Bus.

-   STATIC\_KEY: Will return a static key (being "unresolved") for unresolved Routing Keys. This effectively means that all those commands will be routed to the same segment, as long as the configuration of segments does not change.

JGroupsConnector
----------------

The `JGroupsConnector` uses (as the name already gives away) JGroups as the underlying discovery and dispatching mechanism. Describing the feature set of JGroups is a bit too much for this reference guide, so please refer to the [JGroups User Guide](http://www.jgroups.org/ug.html) for more details.

Since JGroups handles both discovery of nodes and the communication between them, the `JGroupsConnector` acts as both a `CommandBusConnector` and a `CommandRouter`.

> **Note**
> 
> You can find the JGroups specific components for the `DistributedCommandBus` in the `axon-distributed-commandbus-jgroups` module.

The JGroupsConnector has four mandatory configuration elements:

-   The first is a JChannel, which defines the JGroups protocol stack. Generally, a JChannel is constructed with a reference to a JGroups configuration file. JGroups comes with a number of default configurations which can be used as a basis for your own configuration. Do keep in mind that IP Multicast generally doesn't work in Cloud Services, like Amazon. TCP Gossip is generally a good start in such type of environment.

-   The Cluster Name defines the name of the Cluster that each segment should register to. Segments with the same Cluster Name will eventually detect each other and dispatch Commands among each other.

-   A "local segment" is the Command Bus implementation that dispatches Commands destined for the local JVM. These commands may have been dispatched by instances on other JVMs or from the local one.

-   Finally, the Serializer is used to serialize command messages before they are sent over the wire.

> **Note**
>
> When using a Cache, it should be cleared out when the `ConsistentHash` changes to avoid potential data corruption (e.g. when commands don't specify a `@TargetAggregateVersion` and a new member quickly joins and leaves the JGroup, modifying the aggregate while it's still cached elsewhere.)

Ultimately, the JGroupsConnector needs to actually connect, in order to dispatch Messages to other segments. To do so, call the `connect()` method. 

``` java
JChannel channel = new JChannel("path/to/channel/config.xml");
CommandBus localSegment = new SimpleCommandBus();
Serializer serializer = new XStreamSerializer();

JGroupsConnector connector = new JGroupsConnector(channel, "myCommandBus", localSegment, serializer);
DistributedCommandBus commandBus = new DistributedCommandBus(connector, connector);

// on one node:
commandBus.subscribe(CommandType.class.getName(), handler);
connector.connect();

// on another node, with more CPU:
commandBus.subscribe(CommandType.class.getName(), handler);
commandBus.subscribe(AnotherCommandType.class.getName(), handler2);
commandBus.updateLoadFactor(150); // defaults to 100
connector.connect();

// from now on, just deal with commandBus as if it is local...
```

> **Note**
>
> Note that it is not required that all segments have Command Handlers for the same type of Commands. You may use different segments for different Command Types altogether. The Distributed Command Bus will always choose a node to dispatch a Command to that has support for that specific type of Command.

If you use Spring, you may want to consider using the `JGroupsConnectorFactoryBean`. It automatically connects the Connector when the ApplicationContext is started, and does a proper disconnect when the `ApplicationContext` is shut down. Furthermore, it uses sensible defaults for a testing environment (but should not be considered production ready) and autowiring for the configuration.

Spring Cloud Connector
----------------------

The Spring Cloud Connector setup uses the service registration and discovery mechanism described by [Spring Cloud](http://projects.spring.io/spring-cloud/) for distributing the Command Bus. You are thus left free to choose which Spring Cloud implementation to use to distribute your commands. An example implementations is the Eureka Discovery/Eureka Server combination.
 
 > **Note**
 >
 > The `SpringCloudCommandRouter` uses the Spring Cloud specific `ServiceInstance.Metadata` field to inform all the nodes in the system of its message routing information. It is thus of importance that the Spring Cloud implementation selected supports the usage of the `ServiceInstance.Metadata` field. If the desired Spring Cloud implementation does not support the modification of the `ServiceInstance.Metadata` (e.g. Consul), the `SpringCloudHttpBackupCommandRouter` is a viable solution. See the end of this chapter for configuration specifics on the `SpringCloudHttpBackupCommandRouter`. 

Giving a description of every Spring Cloud implementation would push this reference guide to far. Hence we refer to their respective documentations for further information.

The Spring Cloud Connector setup is a combination of the `SpringCloudCommandRouter` and a `SpringHttpCommandBusConnector`, which respectively fill the place of the `CommandRouter` and the `CommandBusConnector` for the `DistributedCommandBus`.

> **Note**
>
> When using the `SpringCloudCommandRouter`, make sure that your Spring application is has heartbeat events enabled. The implementation leverages the heartbeat events published by a Spring Cloud application to check whether its knowledge of all the others nodes is up to date. Hence if heartbeat events are disabled the majority of the Axon applications within your cluster will not be aware of the entire set up, thus posing issues for correct command routing.

The `SpringCloudCommandRouter` has to be created by providing the following:

- A "discovery client" of type `DiscoveryClient`. This can be provided by annotating your Spring Boot application with `@EnableDiscoveryClient`, which will look for a Spring Cloud implementation on your classpath.
 
- A "routing strategy" of type `RoutingStrategy`. The `axon-core` module currently provides several implementations, but a function call can suffice as well. If you want to route the Commands based on the 'aggregate identifier' for example, you would use the `AnnotationRoutingStrategy` and annotate the field on the payload that identifies the aggregate with `@TargetAggregateIdentifier`.

Other optional parameters for the `SpringCloudCommandRouter`  are:

- A "service instance filter" of type `Predicate<ServiceInstance>`. This predicate is used to filter out `ServiceInstances` which the `DiscoveryClient` might encounter which by forehand you know will not handle any command messages. This might be useful if you've got several services within the Spring Cloud Discovery Service set up which you do not want to take into account for command handling, ever.  

- A "consistent hash change listener" of type `ConsistentHashChangeListener`. Adding a consistent hash change listener provides you the opportunity to perform a specific task if  new members have been added to the known command handlers set.

The `SpringHttpCommandBusConnector` requires three parameters for creation:
 
- A "local command bus" of type `CommandBus`. This is the Command Bus implementation that dispatches Commands to the local JVM. These commands may have been dispatched by instances on other JVMs or from the local one.

- A `RestOperations` object to perform the posting of a Command Message to another instance.

- Lastly a "serializer" of type `Serializer`. The serializer is used to serialize the command messages before they are sent over the wire.

> **Note**
>
> The Spring Cloud Connector specific components for the `DistributedCommandBus` can be found in the `axon-distributed-commandbus-springcloud` module.

The `SpringCloudCommandRouter` and `SpringHttpCommandBusConnector` should then both be used for creating the `DistributedCommandsBus`. In Spring Java config, that would look as follows:

```java
// Simple Spring Boot App providing the `DiscoveryClient` bean
@EnableDiscoveryClient
@SpringBootApplication
public class MyApplication {
    
    public static void main(String[] args) {
        SpringApplication.run(MyApplication.class, args);
    }

    // Example function providing a Spring Cloud Connector
    @Bean
    public CommandRouter springCloudCommandRouter(DiscoveryClient discoveryClient) {
        return new SpringCloudCommandRouter(discoveryClient, new AnnotationRoutingStrategy());
    }
    
    @Bean
    public CommandBusConnector springHttpCommandBusConnector(@Qualifier("localSegment") CommandBus localSegment,
                                                             RestOperations restOperations,
                                                             Serializer serializer) {
        return new SpringHttpCommandBusConnector(localSegment, restOperations, serializer);
    }
    
    @Primary // to make sure this CommandBus implementation is used for autowiring
    @Bean
    public DistributedCommandBus springCloudDistributedCommandBus(CommandRouter commandRouter, 
                                                                  CommandBusConnector commandBusConnector) {
        return new DistributedCommandBus(commandRouter, commandBusConnector);
    }

}
```
``` java
// if you don't use Spring Boot Autoconfiguration, you will need to explicitly define the local segment:
@Bean
@Qualifier("localSegment")
public CommandBus localSegment() {
    return new SimpleCommandBus();
}

```
> **Note**
>
> Note that it is not required that all segments have Command Handlers for the same type of Commands. You may use different segments for different Command Types altogether. The Distributed Command Bus will always choose a node to dispatch a Command to that has support for that specific type of Command.

##### Spring Cloud Http Back Up Command Router

Internally, the `SpringCloudCommandRouter` uses the `Metadata` map contained in the Spring Cloud `ServiceInstance` to communicate the allowed message routing information throughout the distributed Axon environment. If the desired Spring Cloud implementation however does not allow the modification of the `ServiceInstance.Metadata` field (e.g. Consul), one can choose to instantiate a `SpringCloudHttpBackupCommandRouter` instead of the `SpringCloudCommandRouter`. 

The `SpringCloudHttpBackupCommandRouter`, as the name suggests, has a back up mechanism if the `ServiceInstance.Metadata` field does not contained the expected message routing information. That back up mechanism is to provide an HTTP endpoint from which the message routing information can be retrieved and by simultaneously adding the functionality to query that endpoint of other known nodes in the cluster to retrieve their message routing information. As such the back up mechanism functions is a Spring Controller to receive requests at a specifiable endpoint and uses a `RestTemplate` to send request to other nodes at the specifiable endpoint.

To use the `SpringCloudHttpBackupCommandRouter` instead of the `SpringCloudCommandRouter`, add the following Spring Java configuration (which replaces the `SpringCloudCommandRouter` method in our earlier example):

```java
@Configuration
public class MyApplicationConfiguration {
    @Bean
    public CommandRouter springCloudHttpBackupCommandRouter(DiscoveryClient discoveryClient, 
                                                            RestTemplate restTemplate, 
                                                            @Value("${axon.distributed.spring-cloud.fallback-url}") String messageRoutingInformationEndpoint) {
        return new SpringCloudHttpBackupCommandRouter(discoveryClient, new AnnotationRoutingStrategy(), restTemplate, messageRoutingInformationEndpoint);
    }
}
``` 

Messaging concepts
==================

Axon的核心概念之一就是消息 组件之间的所有通信都使用消息对象完成。 这为这些组件提供了所需的位置透明度，以便在必要时扩展和分配这些组件。

尽管所有这些消息都实现了“消息”接口，但不同类型的消息之间有明显的区别，以及它们是如何处理的。

所有消息都包含有效负载，元数据和唯一标识符。 消息的有效载荷是消息含义的功能描述。 该对象的类名和它携带的数据的组合描述了应用程序的消息含义。 元数据允许您描述邮件发送的上下文。 例如，您可以存储跟踪信息，以便跟踪消息的来源或原因。 您还可以存储信息来描述正在执行命令的安全上下文。

> **Note**
>
> 请注意，所有消息都是不可变的。 将数据存储在消息中实际上意味着根据前一个消息创建一个新消息，并添加额外的信息。 这保证了消息可以安全地在多线程和分布式环境中使用。

Commands
--------

命令描述了改变应用程序状态的意图。 它们被实现为（最好是只读的）使用CommandMessage实现包装的POJO。

命令总是只有一个目的地。 虽然发件人并不关心哪个组件处理命令或组件驻留的位置，但知道它的结果可能很有趣。 这就是为什么通过命令总线发送的命令消息允许返回结果的原因。

Events
------

事件是描述应用程序中发生的事件的对象。 事件的典型来源是聚合。 在Aggregate内发生重要事件时，它将引发一个事件。 在Axon Framework中，事件可以是任何对象。 强烈建议您确保所有事件都是可序列化的。

当事件分派时，Axon将它们包装在一个`EventMessage`中。 使用的消息的实际类型取决于事件的来源。 当一个事件由一个Aggregate引发时，它被包装在一个`DomainEventMessage`中（它扩展了`EventMessage`）。 所有其他事件都包含在一个`EventMessage.`中。除了常见的`Message`属性，如唯一的标识符，`EventMessage`还包含一个时间戳。 “DomainEventMessage”还包含引发该事件的聚合的类型和标识符。 它还包含聚集事件流中事件的序列号，它允许重现事件的顺序。

> **Note**
>
> 即使`DomainEventMessage`包含对Aggregate Identifier的引用，您也应始终在实际的Event中包含标识符。 EventStore使用DomainEventMessage中的标识符来存储事件，并不总是为其他目的提供可靠的值。

原始的Event对象存储为EventMessage的Payload。 在有效负载旁边，您可以将信息存储在事件消息的元数据中。 元数据的目的是存储有关事件的附加信息，该事件主要不是作为商业信息。 审计信息就是一个典型的例子。 它允许您查看在哪些情况下引发了事件，例如触发处理的用户帐户或处理该事件的计算机的名称。

> **Note**
>
> In general, you should not base business decisions on information in the meta-data of event messages. If that is the case, you might have information attached that should really be part of the Event itself instead. Meta-data is typically used for reporting, auditing and tracing.

Although not enforced, it is good practice to make domain events immutable, preferably by making all fields final and by initializing the event within the constructor. Consider using a Builder pattern if Event construction is too cumbersome.

> **Note**
>
> Although Domain Events technically indicate a state change, you should try to capture the intention of the state in the event, too. A good practice is to use an abstract implementation of a domain event to capture the fact that certain state has changed, and use a concrete sub-implementation of that abstract class that indicates the intention of the change. For example, you could have an abstract `AddressChangedEvent`, and two implementations `ContactMovedEvent` and `AddressCorrectedEvent` that capture the intent of the state change. Some listeners don't care about the intent (e.g. database updating event listeners). These will listen to the abstract type. Other listeners do care about the intent and these will listen to the concrete subtypes (e.g. to send an address change confirmation email to the customer).
>
> ![ "Adding intent to events](state-change-intent.png)

When dispatching an Event on the Event Bus, you will need to wrap it in an Event Message. The `GenericEventMessage` is an implementation that allows you to wrap your Event in a Message. You can use the constructor, or the static `asEventMessage()` method. The latter checks whether the given parameter doesn't already implement the `Message` interface. If so, it is either returned directly (if it implements `EventMessage`,) or it returns a new `GenericEventMessage` using the given `Message`'s payload and Meta Data. If an Event is applied (published) by an Aggregate Axon will automatically wrap the Event in a `DomainEventMessage` containing the Aggregate's Identifier, Type and Sequence Number. 

Queries
-------

Queries describe a request for information or state. A query can have multiple handlers. When dispatching queries, the client indicates whether he wants a result from one or from all available query handlers.

Unit of Work
------------

The Unit of Work is an important concept in the Axon Framework, though in most cases you are unlikely to interact with it directly. The processing of a message is seen as a single unit. The purpose of the Unit of Work is to coordinate actions performed during the processing of a message (Command, Event or Query). Components can register actions to be performed during each of the stages of a Unit of Work, such as onPrepareCommit or onCleanup.

You are unlikely to need direct access to the Unit of Work. It is mainly used by the building blocks that Axon provides. If you do need access to it, for whatever reason, there are a few ways to obtain it. The Handler receives the Unit Of Work through a parameter in the handle method. If you use annotation support, you may add a parameter of type `UnitOfWork` to your annotated method. In other locations, you can retrieve the Unit of Work bound to the current thread by calling `CurrentUnitOfWork.get()`. Note that this method will throw an exception if there is no Unit of Work bound to the current thread. Use `CurrentUnitOfWork.isStarted()` to find out if one is available.

One reason to require access to the current Unit of Work is to attach resources that need to be reused several times during the course of message processing, or if created resources need to be cleaned up when the Unit of Work completes. In such case, the `unitOfWork.getOrComputeResource()` and the lifecycle callback methods, such as `onRollback()`, `afterCommit()` and `onCleanup()` allow you to register resources and declare actions to be taken during the processing of this Unit of Work.

> **Note**
>
> Note that the Unit of Work is merely a buffer of changes, not a replacement for Transactions. Although all staged changes are only committed when the Unit of Work is committed, its commit is not atomic. That means that when a commit fails, some changes might have been persisted, while others are not. Best practices dictate that a Command should never contain more than one action. If you stick to that practice, a Unit of Work will contain a single action, making it safe to use as-is. If you have more actions in your Unit of Work, then you could consider attaching a transaction to the Unit of Work's commit. Use `unitOfWork.onCommit(..)` to register actions that need to be taken when the Unit of Work is being committed.

Your handlers may throw an Exception as a result of processing a message. By default, unchecked exceptions will cause the UnitOfWork to roll back all changes. As a result, scheduled side effects are cancelled.

Axon provides a few Rollback strategies out-of-the-box:
 - `RollbackConfigurationType.NEVER`, will always commit the Unit of Work,
 - `RollbackConfigurationType.ANY_THROWABLE`, will always roll back when an exception occurs,
 - `RollbackConfigurationType.UNCHECKED_EXCEPTIONS`, will roll back on Errors and Runtime Exception
 - `RollbackConfigurationType.RUNTIME_EXCEPTION`, will roll back on Runtime Exceptions (but not on Errors)

When using Axon components to process messages, the lifecycle of the Unit of Work will be automatically managed for you. If you choose not to use these components, but implement processing yourself, you will need to programmatically start and commit (or roll back) a Unit of Work instead.

In most cases, the `DefaultUnitOfWork` will provide you with the functionality you need. It expects processing to happen within a single thread. To execute a task in the context of a Unit Of Work, simply call `UnitOfWork.execute(Runnable)` or `UnitOfWork.executeWithResult(Callable)` on a new `DefaultUnitOfWork`. The Unit Of Work will be started and committed when the task completes, or rolled back if the task fails. You can also choose to manually start, commit or rollback the Unit Of Work if you need more control.

Typical usage is as follows:

``` java
UnitOfWork uow = DefaultUnitOfWork.startAndGet(message);
// then, either use the autocommit approach:
uow.executeWithResult(() -> ... logic here);

// or manually commit or rollback:
try {
    // business logic comes here
    uow.commit();
} catch (Exception e) {
    uow.rollback(e);
    // maybe rethrow...
}
```

A Unit of Work knows several phases. Each time it progresses to another phase, the UnitOfWork Listeners are notified.

-   Active phase: this is where the Unit of Work is started. The Unit of Work is generally registered with the current thread in this phase (through `CurrentUnitOfWork.set(UnitOfWork)`). Subsequently the message is typically handled by a message handler in this phase.

-   Commit phase: after processing of the message is done but before the Unit of Work is committed, the `onPrepareCommit` listeners are invoked. If a Unit of Work is bound to a transaction, the `onCommit` listeners are invoked to commit any supporting transactions. When the commit succeeds, the `afterCommit` listeners are invoked. If a commit or any step before fails, the `onRollback` listeners are invoked. The message handler result is contained in the `ExecutionResult` of the Unit Of Work, if available.

-   Cleanup phase: This is the phase where any of the resources held by this Unit of Work (such as locks) are to be released. If multiple Units Of Work are nested, the cleanup phase is postponed until the outer unit of work is ready to clean up.

The message handling process can be considered an atomic procedure; it should either be processed entirely, or not at all. Axon Framework uses the Unit Of Work to track actions performed by the message handlers. After the handler completed, Axon will try to commit the actions registered with the Unit Of Work.

It is possible to bind a transaction to a Unit of Work. Many components, such as the CommandBus and QueryBus implementations and all asynchronously processing Event Processors, allow you to configure a Transaction Manager. This Transaction Manager will then be used to create the transactions to bind to the Unit of Work that is used to manage the process of a Message.

When application components need resources at different stages of message processing, such as a Database Connection or an EntityManager, these resources can be attached to the Unit of Work. The `unitOfWork.getResources()` method allows you to access the resources attached to the current Unit of Work. Several helper methods are available on the Unit of Work directly, to make working with resources easier.

When nested Units of Work need to be able to access a resource, it is recommended to register it on the root Unit of Work, which can be accessed using `unitOfWork.root()`. If a Unit of Work is the root, it will simply return itself.

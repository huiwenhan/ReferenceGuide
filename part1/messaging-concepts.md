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
>一般而言，您不应将业务决策基于事件消息的元数据中的信息。 如果是这样的话，你可能会得到附加的信息，应该真的成为事件本身的一部分。 元数据通常用于报告，审计和跟踪。

虽然没有强制执行，但最好通过使所有字段最终并通过在构造函数中初始化事件来使域事件不可变。 如果事件结构过于繁琐，请考虑使用Builder模式。

> **Note**
>
> 尽管领域事件在技术上表明了状态的变化，但您也应该尝试在事件中捕捉到状态的意图。 一个好的做法是使用域事件的抽象实现来捕捉某个状态已经发生变化的事实，并使用该抽象类的具体子实现来指示变化的意图。 例如，你可以有一个抽象的`AddressChangedEvent`，两个实现`ContactMovedEvent`和`AddressCorrectedEvent`来捕获状态变化的意图。 一些听众不关心意图（例如数据库更新事件监听器）。 这些将会听取抽象类型。 其他听众确实关心意图，他们会听到具体的子类型（例如，向客户发送地址更改确认电子邮件）。
>
> ![ "Adding intent to events](state-change-intent.png)

在事件总线上分派事件时，您需要将其包装在事件消息中。 `GenericEventMessage`是一个允许你在一个消息中包装你的事件的实现。 您可以使用构造函数或静态的`asEventMessage（）`方法。 后者检查给定的参数是否已经实现了`Message`接口。 如果是这样，它要么直接返回（如果它实现了`EventMessage`），要么它使用给定的`Message`的有效载荷和元数据返回一个新的`GenericEventMessage`。 如果一个事件由一个聚集Axon应用（发布），将自动将该事件包装在一个包含聚集的标识符，类型和序列号的`DomainEventMessage`中。

Queries
-------

查询描述了对信息或状态的请求。 一个查询可以有多个处理程序。 当分派查询时，客户端指示他是想要一个结果还是来自所有可用的查询处理程序。

Unit of Work
------------

工作单元是Axon框架中的一个重要概念，但在大多数情况下，您不可能直接与其交互。 消息的处理被视为一个单元。 工作单元的目的是协调处理消息（命令，事件或查询）期间执行的操作。 组件可以注册要在工作单元的每个阶段执行的操作，例如onPrepareCommit或onCleanup。

您不太可能需要直接访问工作单元。 它主要由Axon提供的构建模块使用。 如果您确实需要访问它，出于任何原因，有几种方法可以获得它。 处理程序通过句柄方法中的参数接收工作单元。 如果您使用注释支持，则可以将“UnitOfWork”类型的参数添加到注释的方法中。 在其他位置，您可以通过调用`CurrentUnitOfWork.get（）`来检索绑定到当前线程的工作单元。 请注意，如果没有工作单元绑定到当前线程，则此方法将引发异常。 使用`CurrentUnitOfWork.isStarted（）`来确定一个是否可用。

要求访问当前工作单元的一个原因是附加需要在消息处理过程中多次重复使用的资源，或者在工作单元完成时需要清理创建的资源。 在这种情况下，`unitOfWork.getOrComputeResource（）`和生命周期回调方法，比如`onRollback（）`，`afterCommit（）`和`onCleanup（）`允许你注册资源并声明在 处理这个工作单位。

> **Note**
>
> 请注意，工作单元仅仅是变化的缓冲区，而不是交易的替代物。 虽然所有分阶段的变更只是在工作单元承诺时才提交，但其承诺不是原子性的。 这意味着，如果提交失败，则可能会保留一些更改，而其他更改则不会。 最佳实践规定Command不应包含多个操作。 如果你坚持这种做法，一个工作单元将包含一个单一的行动，使它可以安全地使用。 如果您在工作单元中有更多的操作，那么您可以考虑将事务附加到工作单元的提交。 使用`unitOfWork.onCommit（..）`注册工作单元提交时需要采取的操作。

由于处理消息，您的处理程序可能会抛出异常。 默认情况下，未经检查的异常将导致UnitOfWork回滚所有更改。 结果，预定的副作用被取消。

Axon提供了一些开箱即用的回滚策略：
 - `RollbackConfigurationType.NEVER`, 将始终承诺工作单位，
 - `RollbackConfigurationType.ANY_THROWABLE`, 将在发生异常时始终回滚，
 - `RollbackConfigurationType.UNCHECKED_EXCEPTIONS`, 将回滚错误和运行时异常
 - `RollbackConfigurationType.RUNTIME_EXCEPTION`, 将回滚运行时异常（但错误不会）

使用Axon组件处理消息时，工作单元的生命周期将自动为您管理。 如果您选择不使用这些组件，而是实现自己的处理，则需要以编程方式启动并提交（或回滚）工作单元。

在大多数情况下，DefaultUnitOfWork将为您提供所需的功能。 它期望处理发生在单个线程内。 要在Unit Of Work中执行任务，只需在新的DefaultUnitOfWork上调用UnitOfWork.execute（Runnable）或UnitOfWork.executeWithResult（Callable）即可。 工作单元将在任务完成时启动并提交，或者在任务失败时回滚。 如果您需要更多控制，您也可以选择手动启动，提交或回滚工作单元。

典型用法如下：

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

工作单位有几个阶段。 每次进入另一阶段时，都会通知UnitOfWork侦听器。

-   Active phase: 这是工作单元开始的地方。 工作单元通常在此阶段通过当前线程注册（通过`CurrentUnitOfWork.set（UnitOfWork）`）。 随后，消息通常在此阶段由消息处理程序处理。

-   Commit phase: 在完成消息处理之后但在提交工作单元之前，调用`onPrepareCommit`侦听器。 如果一个工作单元绑定到一个事务，那么会调用`onCommit`监听器来提交任何支持事务。 当提交成功时，调用`afterCommit`监听器。 如果提交或任何步骤失败，则调用`onRollback`侦听器。 消息处理结果包含在Unit Of的“ExecutionResult”中（如果可用）。

-   Cleanup phase: 这是本工作单元拥有的任何资源（如锁）将被释放的阶段。 如果多个工作单元嵌套，则清理阶段将推迟到外部工作单元准备清理。

消息处理过程可以被认为是一个原子程序; 它应该完全处理，或者根本不处理。 Axon Framework使用Unit Of Work来跟踪消息处理程序执行的操作。 处理程序完成后，Axon将尝试执行在Unit Of Work中注册的操作。

可以将交易绑定到工作单元。 许多组件（如CommandBus和QueryBus实现以及所有异步处理事件处理器）都允许您配置事务管理器。 此事务管理器将用于创建事务以绑定到用于管理消息处理的工作单元。

当应用程序组件需要消息处理的不同阶段（如数据库连接或EntityManager）中的资源时，可以将这些资源连接到工作单元。 `unitOfWork.getResources（）`方法允许您访问附加到当前工作单元的资源。 直接在工作单元上提供几种帮助方法，以便更轻松地处理资源。

当嵌套的工作单元需要能够访问某个资源时，建议将其注册到工作单元的根单元中，该单元可以使用`unitOfWork.root（）`进行访问。 如果一个工作单位是根，它就会自己回归。

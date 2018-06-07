Managing complex business transactions
======================================

并非每个命令都能够在单个ACID事务中完全执行。 资金转移是一个很常见的例子，它常常作为交易的一个参数出现。 人们通常认为，将资金从一个账户转移到另一个账户绝对需要原子和一致的交易。 好吧，事实并非如此。 相反，这是不可能的。 如果资金从银行A的账户转移到银行B的另一个账户，该怎么办？ A银行是否获得了B银行数据库的锁定？ 如果转帐正在进行中，银行A已扣除金额，但银行B尚未存入吗？ 不是真的，这是“正在进行”。 另一方面，如果在将钱存入银行B的账户时出现问题，则银行A的客户会想要退还他的钱。 所以我们确实期望一定的一致性。


尽管ACID交易在某些情况下不是必需的，甚至不可能，但仍需要某种形式的交易管理。 通常，这些事务被称为BASE事务：基本可用性，软状态，最终一致性。 与ACID相反，BASE事务不能轻易回滚。 为了回滚，需要采取补偿行动来恢复作为交易一部分发生的任何事情。 在汇款的例子中，B银行存款失败将退还银行A的款项。

在CQRS中，Sagas可以用来管理这些BASE事务。 他们对事件做出响应并可能调度命令，调用外部应用程序等。在域驱动设计的背景下，萨加斯被用作几个有界上下文之间的协调机制并不罕见。


Saga
====

Saga是一种特殊类型的事件监听器：一种管理商业交易的事件监听器。 一些交易可能会持续几天甚至几周，而其他交易则会在几毫秒内完成。 在Axon中，Saga的每个实例都负责管理单个业务事务。 这意味着Saga保持管理该交易所需的状态，继续执行或采取补偿行动以回滚已采取的任何行动。 通常情况下，与常规事件监听器相反，Saga有起点和终点，都是由事件触发的。 虽然Saga的出发点通常很清楚，但Saga可以有很多方式结束。


在Axon中，Sagas是定义一个或多个@SagaEventHandler方法的类。 与常规事件处理程序不同，传奇的多个实例可能随时存在。 Sagas由一个处理器（追踪或订阅）管理，该处理器专用于处理该特定Saga类型的事件。


Life Cycle
----------

单个Saga实例负责管理单个事务。 这意味着您需要能够指示Saga的生命周期的开始和结束。

在Saga中，事件处理程序用`@ SagaEventHandler`注释。 如果一个特定的事件表示一个事务的开始，则将另一个注释添加到同一个方法：`@ StartSaga`。 此注释将创建一个新的Saga，并在发布匹配的事件时调用其事件处理函数方法。

默认情况下，只有当没有合适的现有Saga（同类型）可以找到时才会启动新的Saga。 您还可以通过将`@ StartSaga`注释中的`forceNew`属性设置为`true`来强制创建新的Saga实例。

结束Saga可以通过两种方式完成。 如果某个事件总是表示Saga的生命周期结束，请用`@ EndSaga`在该传奇中注释该事件的处理程序。 Saga的生命周期将在调用处理程序后结束。 或者，您可以从传奇内部调用“SagaLifecycle.end（）”来结束生命周期。 这可以让你有条件地结束Saga。

Event Handling
--------------

Saga中的事件处理与常规的事件监听器相当。 方法和参数解析的相同规则在这里是有效的。 但是有一个主要区别。 虽然有一个Event Listener实例可处理所有传入事件，但可能存在多个Saga实例，每个实例都对不同的事件感兴趣。 例如，管理交易订单ID为“1”的的Saga将不会对订单“2”的事件感兴趣，反之亦然。

Axon不会将所有事件发布到所有Saga实例（这将完全浪费资源），而只会发布包含Saga与之相关联的属性的Events。 这是使用`AssociationValue`完成的。 一个`AssociationValue`由一个键和一个值组成。 密钥表示所用标识符的类型，例如“orderId”或“order”。 该值表示上例中的对应值“1”或“2”。
    
评估`@ SagaEventHandler`注释方法的顺序与`@EventHandler`方法的顺序相同（请参阅[注释事件处理程序]（event-handling.md#definition-event-handlers））。 如果处理程序方法的参数与传入的事件匹配，并且该事件与处理程序方法中定义的属性关联，则方法匹配。

`@ SagaEventHandler`注解有两个属性，其中`associationProperty`是最重要的属性。 这是应该用于查找关联的Sagas的传入Event上的属性的名称。 关联值的关键是属性的名称。 该值是属性的getter方法返回的值。

例如，用一个方法“`String getOrderId（）`”考虑一个传入事件，它返回“123”。 如果接受这个事件的方法用`@SagaEventHandler（associationProperty =“orderId”）`注释，那么这个事件将被路由到所有与具有关键“orderId”和值“123”的AssociationValue关联的Sagas。 这可能只是一个，多于一个，甚至完全没有。

有时，您想要关联的属性的名称不是您想要使用的关联的名称。 例如，您有一个与卖出订单与买入订单相匹配的Saga，您可以拥有一个包含“buyOrderId”和“sellOrderId”的交易对象。 如果您希望Saga将该值与“orderId”相关联，则可以在“@ SagaEventHandler”注释中定义不同的keyName。 它会变成`@SagaEventHandler（associationProperty =“sellOrderId”，keyName =“orderId”）`

Managing associations
---------------------

当一个Saga管理跨多个领域概念的事务时，例如订单，货运，发票等，Saga需要与这些概念的实例相关联。 关联需要两个参数：标识关联类型（订单，货件等）的键和表示该概念标识符的值。

将Saga与一个概念联系起来有几种方式。 首先，当调用一个`@ StartSaga`注释的事件处理程序时新创建一个Saga时，它会自动与`@ SagaEventHandler`方法中标识的属性关联。 任何其他关联都可以使用`SagaLifecycle.associateWith（String key，String / Number value）`方法创建。 使用`SagaLifecycle.removeAssociationWith（String key，String / Number value）`方法删除特定的关联。

> Note
>
> 由于存储的关联值条目需要标识符的“字符串”表示，因此在Saga中关联域概念的API有意识地仅允许“String”或“Number”作为标识值。 在API中使用简单的标识符值以简单的`String`表示是设计的，因为数据库中的字符串列条目使数据库引擎之间的比较更简单。 因此，故意没有`associateWith（String，Object）`，因为`Object＃toString（）`调用的结果可能会提供笨拙的标识符。

想象一下，为一个围绕订单的事务创建了一个Saga。 Saga与订单自动关联，因为该方法用`@StartSaga`注释。 Saga负责为该订单创建发票，并告诉Shipping为其创建货件。 一旦货件到达且发票已付款，交易即告完成，Saga关闭。

这里是这样一个Saga的代码：

``` java
public class OrderManagementSaga {

    private boolean paid = false;
    private boolean delivered = false;
    @Inject
    private transient CommandGateway commandGateway;

    @StartSaga
    @SagaEventHandler(associationProperty = "orderId")
    public void handle(OrderCreatedEvent event) {
        // client generated identifiers
        ShippingId shipmentId = createShipmentId();
        InvoiceId invoiceId = createInvoiceId();
        // associate the Saga with these values, before sending the commands
        SagaLifecycle.associateWith("shipmentId", shipmentId);
        SagaLifecycle.associateWith("invoiceId", invoiceId);
        // send the commands
        commandGateway.send(new PrepareShippingCommand(...));
        commandGateway.send(new CreateInvoiceCommand(...));
    }

    @SagaEventHandler(associationProperty = "shipmentId")
    public void handle(ShippingArrivedEvent event) {
        delivered = true;
        if (paid) { SagaLifecycle.end(); }
    }

    @SagaEventHandler(associationProperty = "invoiceId")
    public void handle(InvoicePaidEvent event) {
        paid = true;
        if (delivered) { SagaLifecycle.end(); }
    }

    // ...
}
```
通过允许客户端生成标识符，Saga可以很容易地与一个概念关联，而不需要请求 - 响应类型命令。 在发布命令之前，我们将事件与这些概念相关联。 这样，我们可以保证捕获作为此命令一部分生成的事件。 一旦支付了发票并且货物已经到达，这将结束这个Saga。

Keeping track of Deadlines
--------------------------

当事情发生时，让Saga采取行动很容易。 毕竟，有一个事件通知Saga。 但是，如果你希望你的Saga在没有任何事情发生的时候做点什么？ 这就是截止日期所用的时间。 在发票中，这通常是几周，而信用卡支付的确认应在几秒钟内发生。

在Axon中，您可以使用`EventScheduler`来安排发布事件。 在发票的例子中，您希望发票在30天内支付。 Saga在发送`CreateInvoiceCommand`后会安排`InvoicePaymentDeadlineExpiredEvent`在30天内发布。 EventScheduler在安排事件后返回`ScheduleToken`。 此令牌可用于取消计划，例如收到发票付款时。

Axon提供了两个`EventScheduler`实现：一个纯Java，一个使用Quartz 2作为后台调度机制。

这个“EventScheduler”的纯Java实现使用“ScheduledExecutorService”来安排Event发布。 虽然这个调度程序的时间非常可靠，但它是一个纯粹的内存中实现。 一旦JVM关闭，所有日程安排都将丢失。 这使得这种实现不适合长期计划。

`SimpleEventScheduler`需要配置一个`EventBus`和一个`SchedulingExecutorService`（参见java.util.concurrent.Executors`类的静态方法以获得辅助方法）。

`QuartzEventScheduler`是一个更可靠和更有企业价值的实现。 使用Quartz作为基础调度机制，它提供了更强大的功能，如持久性，集群和失败管理。 这意味着事件发布是有保证的。 它可能有点晚，但会发布。

它需要配置一个Quartz`Scheduler`和一个`EventBus`。 或者，您可以设置Quartz作业计划的组名，默认为“AxonFramework-Events”。

一个或多个组件将监听预定的事件。 这些组件可能依赖于绑定到调用它们的线程的事务。 计划事件由`EventScheduler`管理的线程发布。 为了管理这些线程上的事务，你可以配置一个`TransactionManager`或一个`UnitOfWorkFactory`来创建一个Transaction Bound of Work。

> **Note**
>
> Spring用户可以使用`QuartzEventSchedulerFactoryBean`或`SimpleEventSchedulerFactoryBean`来简化配置。 它允许您直接设置PlatformTransactionManager。

Injecting Resources
-------------------

Sagas通常不仅仅是基于事件维护状态。 他们与外部组件进行交互。 为此，他们需要访问必要的资源来处理组件。 通常情况下，这些资源并不是Saga的一部分，不应该坚持这样。 但是一旦重构了Saga，这些资源就必须在事件发送到该事件之前注入。

为此，有`ResourceInjector`。 它被`SagaRepository`用来向Saga注入资源。 Axon提供了一个`SpringResourceInjector`，它通过Application Context中的资源注入带注释的字段和方法，以及一个`SimpleResourceInjector`，它将注册的资源注入`@Inject`注释的方法和字段。

> **Tip**
>
>由于资源不应该与Saga持久，请务必将`transient`关键字添加到这些字段。 这将阻止序列化机制尝试将这些字段的内容写入存储库。 在Saga被反序列化后，存储库将自动重新注入所需的资源。

`SimpleResourceInjector`允许注入预先指定的资源集合。 它扫描Saga的（setter）方法和字段，找到用@ @Inject注释的方法和字段。

当使用配置API时，Axon将默认为`ConfigurationResourceInjector`。 它将注入配置中可用的任何资源。 像EventBus，EventStore，CommandBus和CommandGateway这样的组件可以默认使用，但你也可以使用`configurer.registerComponent（）`注册你自己的组件。

`SpringResourceInjector`使用Spring的依赖注入机制将资源注入到聚合中。 这意味着如果需要，您可以使用setter注射或直接领域注射。 要注入的方法或字段需要注释以便Spring将其识别为依赖项，例如使用`@ Autowired`。

Saga Infrastructure
===================

事件需要重定向到适当的Saga实例。 为此，需要一些基础结构类。 最重要的组件是`SagaManager`和`SagaRepository`。

Saga Manager
------------

与处理事件的任何组件一样，处理由事件处理器完成。 但是，由于Sagas不是处理事件的单例实例，而是具有单独的生命周期，因此需要对其进行管理。

Axon supports life cycle management through the `AnnotatedSagaManager`, which is provided to an Event Processor to perform the actual invocation of handlers. It is initialized using the type of the Saga to manage, as well as a SagaRepository where Sagas of that type can be stored and retrieved. A single `AnnotatedSagaManager` can only manage a single Saga type.

When using the Configuration API, Axon will use sensible defaults for most components. However, it is highly recommended to define a `SagaStore` implementation to use. The `SagaStore` is the mechanism that 'physically' stores the Saga instances somewhere. The `AnnotatedSagaRepository` (the default) uses the `SagaStore` to store and retrieve Saga instances as they are required.

```java
Configurer configurer = DefaultConfigurer.defaultConfiguration();
configurer.registerModule(
        SagaConfiguration.subscribingSagaManager(MySagaType.class)
                         // Axon defaults to an in-memory SagaStore, defining another is recommended
                         .configureSagaStore(c -> new JpaSagaStore(...)));

// alternatively, it is possible to register a single SagaStore for all Saga types:
configurer.registerComponent(SagaStore.class, c -> new JpaSagaStore(...));
```

Saga Repository and Saga Store
------------------------------

The `SagaRepository` is responsible for storing and retrieving Sagas, for use by the `SagaManager`. It is capable of retrieving specific Saga instances by their identifier as well as by their Association Values.

There are some special requirements, however. Since concurrency handling in Sagas is a very delicate procedure, the repository must ensure that for each conceptual Saga instance (with equal identifier) only a single instance exists in the JVM.

Axon provides the `AnnotatedSagaRepository` implementation, which allows the lookup of Saga instances while guaranteeing that only a single instance of the Saga is accessed at the same time. It uses a `SagaStore` to perform the actual persistence of Saga instances.

The choice for the implementation to use depends mainly on the storage engine used by the application. Axon provides the `JdbcSagaStore`, `InMemorySagaStore`, `JpaSagaStore` and `MongoSagaStore`. 

In some cases, applications benefit from caching Saga instances. In that case, there is a `CachingSagaStore` which wraps another implementation to add caching behavior. Note that the `CachingSagaStore` is a write-through cache, which means save operations are always immediately forwarded to the backing Store, to ensure data safety.

### JpaSagaStore
The `JpaSagaStore` uses JPA to store the state and Association Values of Sagas. Sagas themselves do not need any JPA annotations; Axon will serialize the sagas using a `Serializer` (similar to Event serialization, you can use either a `JavaSerializer` or an `XStreamSerializer`).

The `JpaSagaStore` is configured with an `EntityManagerProvider`, which provides access to an `EntityManager` instance to use. This abstraction allows for the use of both application managed and container managed `EntityManager`s. Optionally, you can define the serializer to serialize the Saga instances with. Axon defaults to the `XStreamSerializer`.

### JdbcSagaStore
The `JdbcSagaStore` uses plain JDBC to store stage instances and their association values. Similar to the `JpaSagaStore`, Saga instances don't need to be aware of how they are stored. They are serialized using a Serializer.

The `JdbcSagaStore` is initialized with either a `DataSource` or a `ConnectionProvider`. While not required, when initializing with a `ConnectionProvider`, it is recommended to wrap the implementation in a `UnitOfWorkAwareConnectionProviderWrapper`. It will check the current Unit of Work for an already open database connection, to ensure that all activity within a unit of work is done on a single connection.

Unlike JPA, the JdbcSagaRepository uses plain SQL statement to store and retrieve information. This may mean that some operations depend on the Database specific SQL dialect. It may also be the case that certain Database vendors provide non-standard features that you would like to use. To allow for this, you can provide your own `SagaSqlSchema`. The `SagaSqlSchema` is an interface that defines all the operations the repository needs to perform on the underlying database. It allows you to customize the SQL statement executed for each one of them. The default is the `GenericSagaSqlSchema`. Other implementations available are `PostgresSagaSqlSchema`, `Oracle11SagaSqlSchema` and `HsqlSagaSchema`.

### MongoSagaStore
The `MongoSagaStore` stores the Saga instances and their associations in a MongoDB database. The `MongoSagaStore` stores all sagas in a single Collection in a MongoDB database. Per Saga instance, a single document is created.

The `MongoSagaStore` also ensures that at any time, only a single Saga instance exists for any unique Saga in a single JVM. This ensures that no state changes are lost due to concurrency issues.

The `MongoSagaStore` is initialized using a `MongoTemplate` and optionally a `Serializer`. The `MongoTemplate` provides a reference to the collection to store the Sagas in. Axon provides the `DefaultMongoTemplate`, which takes the `MongoClient` instance as well as the database name and name of the collection to store the sagas in. The database name and collection name may be omitted. In that case, they default to "axonframework" and "sagas", respectively. 

Caching
-------

If a database backed Saga Storage is used, saving and loading Saga instances may be a relatively expensive operation. Especially in situations where the same Saga instance is invoked multiple times within a short time span, a cache can be beneficial to the application's performance.

Axon provides the `CachingSagaStore` implementation. It is a `SagaStore` that wraps another one, which does the actual storage. When loading Sagas or Association Values, the `CachingSagaStore` will first consult its caches, before delegating to the wrapped repository. When storing information, all calls are always delegated, to ensure that the backing storage always has a consistent view on the Saga's state.

To configure caching, simply wrap any `SagaStore` in a `CachingSagaStore`. The constructor of the `CachingSagaStore` takes three parameters: the repository to wrap and the caches to use for the Association Values and Saga instances, respectively. The latter two arguments may refer to the same cache, or to different ones. This depends on the eviction requirements of your specific application.

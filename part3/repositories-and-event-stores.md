Repositories and Event Stores
=============================

存储库是提供对聚合的访问的机制。 存储库充当用于保存数据的实际存储机制的网关。 在CQRS中，存储库只需要能够根据其唯一标识符查找聚合。 对任何其他类型的查询应该执行查询数据库。

在Axon框架中，所有存储库必须实现`Repository`接口。 该接口规定了三种方法：`load（标识符，版本）`，`load（标识符）`和`newInstance（factoryMethod）`。 `load`方法允许您从存储库加载聚合。 可选的`version`参数用于检测并发修改（请参阅[高级冲突检测和解析]（＃高级冲突检测和解析））。 `newInstance`用于注册仓库中新创建的聚合。

根据您的基础持久性存储和审计需求，有大量基本实现可以提供大多数存储库所需的基本功能。 Axon框架区分保存聚集当前状态的存储库（请参阅[标准存储库]（＃标准存储库））以及存储聚合事件的存储库（请参见[事件采购存储库]（＃事件溯源-repositories））。

请注意，Repository接口不规定“删除（标识符）”方法。 通过调用聚合中的AggregateLifecycle.markDeleted（）方法来删除聚合。 删除聚合与其他任何情况一样是一种状态迁移，唯一的区别是在很多情况下它是不可逆的。 你应该在你的聚合上创建你自己有意义的方法，将聚合的状态设置为“已删除”。 这也允许您注册您想要发布的任何事件。

Standard repositories
---------------------

标准存储库存储聚合的实际状态。 每次更改后，新状态都会覆盖旧的状态。 这使得应用程序的查询组件可以使用命令组件也使用的相同信息。 根据您创建的应用程序类型，这可能是最简单的解决方案。 如果是这样的话，Axon提供了一些构建模块来帮助您实现这样一个存储库。

Axon为标准Repository提供了一个开箱即用的实现：GenericJpaRepository。 它期望Aggregate是一个有效的JPA实体。 它配置了一个EntityManagerProvider，该EntityManagerProvider提供了用于管理实际持久性的EntityManager和一个指定存储在Repository中的Aggregate实际类型的类。 当Aggregate调用静态的`AggregateLifecycle.apply（）`方法时，您还会传入将要发布Events的`EventBus`。

您也可以轻松实现自己的存储库。 在这种情况下，最好从抽象的`LockingRepository`扩展。 作为聚合包装类型，建议使用`AnnotatedAggregate`。 查看“GenericJpaRepository”的源代码。

Event Sourcing repositories
===========================

可以根据事件重建其状态的聚合根也可以配置为由事件溯源存储库加载。 这些存储库不存储聚合本身，而是存储聚合生成的一系列事件。 基于这些事件，聚合的状态可以随时恢复。

EventSourcingRepository实现提供了AxonFramework中任何事件资源库所需的基本功能。 它取决于一个`EventStore`（请参阅[Event store实现]（＃event-store-implement）），它抽象出事件的实际存储机制。

或者，您可以提供一个Aggregate Factory。 AggregateFactory指定如何创建一个聚合实例。 一旦创建了聚合，“EventSourcingRepository”可以使用它从事件存储加载的事件对其进行初始化。 Axon框架附带了许多可以使用的`AggregateFactory`实现。 如果他们不够用，创建你自己的实现是非常容易的。

*GenericAggregateFactory*

`GenericAggregateFactory`是一个特殊的`AggregateFactory`实现，可以用于任何类型的事件源聚集根。 `GenericAggregateFactory`创建一个存储库管理的Aggregate类型的实例。 Aggregate类必须是非抽象的，并声明一个默认的no-arg构造函数，它根本不会进行初始化。

GenericAggregateFactory适用于大多数场景，其中聚合不需要特别注入不可序列化的资源。

*SpringPrototypeAggregateFactory*

根据您的架构选择，使用Spring将依赖关系注入到聚合中可能会很有用。 例如，您可以将查询存储库注入到您的聚合中，以确保某些值的存在（或不存在）。

为了将依赖关系注入到你的聚合中，你需要在Spring上下文中配置一个你的聚合根的原型bean，它也定义了`SpringPrototypeAggregateFactory`。 它不是使用构造函数创建常规实例，而是使用Spring应用程序上下文实例化聚合。 这也会在你的聚合中注入任何依赖。

*Implementing your own AggregateFactory*

在某些情况下，`GenericAggregateFactory`只是不能提供你需要的东西。 例如，对于不同的场景（例如`PublicUserAccount`和`BackOfficeAccount`都可以扩展`Account`），您可以拥有一个抽象聚合类型和多个实现。 您可以使用单个存储库，并配置知道不同实现的AggregateFactory，而不是为每个聚合创建不同的存储库。

Aggregate Factory所做的大部分工作是创建未初始化的Aggregate实例。 它必须使用给定的聚集标识符和流中的第一个事件。 通常，此事件是一个创建事件，其中包含有关聚合的预期类型的提示。 您可以使用这些信息来选择实现并调用其构造函数。 确保没有事件由该构造函数应用; 聚合必须是未初始化的。

与简单资源库实施的直接集合加载相比，基于事件初始化集合可能是一项耗时的工作。 “CachingEventSourcingRepository”提供了一个缓存，如果可用，可以从中加载聚集。

Event store implementations
===========================

Event Sourcing存储库需要一个事件存储来存储和加载来自聚合的事件。 事件存储提供了事件总线的功能，除此之外，它还可以保持已发布的事件，并且能够基于聚合标识符检索事件。

Axon提供了一个开箱即用的事件存储，即EmbeddedEventStore。 它将事件的实际存储和检索委托给一个`EventStorageEngine`。

有多个`EventStorageEngine`实现可用：

`JpaEventStorageEngine`
-----------------------

The `JpaEventStorageEngine` stores events in a JPA-compatible data source. The JPA Event Store stores events in so called entries. These entries contain the serialized form of an event, as well as some fields where meta-data is stored for fast lookup of these entries. To use the `JpaEventStorageEngine`, you must have the JPA (`javax.persistence`) annotations on your classpath.

By default, the event store needs you to configure your persistence context (e.g. as defined in `META-INF/persistence.xml` file) to contain the classes `DomainEventEntry` and `SnapshotEventEntry` (both in the `org.axonframework.eventsourcing.eventstore.jpa` package).

Below is an example configuration of a persistence context configuration:

``` xml
<persistence xmlns="http://java.sun.com/xml/ns/persistence" version="1.0">
    <persistence-unit name="eventStore" transaction-type="RESOURCE_LOCAL"> (1)
        <class>org...eventstore.jpa.DomainEventEntry</class> (2)
        <class>org...eventstore.jpa.SnapshotEventEntry</class>
    </persistence-unit>
</persistence>
```

1.   In this sample, there is a specific persistence unit for the event store. You may, however, choose to add the third line to any other persistence unit configuration.
1.   This line registers the `DomainEventEntry` (the class used by the `JpaEventStore`) with the persistence context.

> **Note**
>
> Axon uses Locking to prevent two threads from accessing the same Aggregate. However, if you have multiple JVMs on the same database, this won't help you. In that case, you'd have to rely on the database to detect conflicts. Concurrent access to the event store will result in a Key Constraint Violation, as the table only allows a single Event for an aggregate with any sequence number. Inserting a second event for an existing aggregate with an existing sequence number will result in an error.
>
> The `JpaEventStorageEngine` can detect this error and translate it to a `ConcurrencyException`. However, each database system reports this violation differently. If you register your `DataSource` with the `JpaEventStore`, it will try to detect the type of database and figure out which error codes represent a Key Constraint Violation. Alternatively, you may provide a `PersistenceExceptionTranslator` instance, which can tell if a given exception represents a Key Constraint Violation.
>
> If no `DataSource` or `PersistenceExceptionTranslator` is provided, exceptions from the database driver are thrown as-is.

By default, the JPA Event Storage Engine requires an `EntityManagerProvider` implementation that returns the `EntityManager` instance for the `EventStorageEngine` to use. This also allows for application managed persistence contexts to be used. It is the `EntityManagerProvider`'s responsibility to provide a correct instance of the `EntityManager`.

There are a few implementations of the `EntityManagerProvider` available, each for different needs. The `SimpleEntityManagerProvider` simply returns the `EntityManager` instance which is given to it at construction time. This makes the implementation a simple option for Container Managed Contexts. Alternatively, there is the `ContainerManagedEntityManagerProvider`, which returns the default persistence context, and is used by default by the Jpa Event Store.

If you have a persistence unit called "myPersistenceUnit" which you wish to use in the `JpaEventStore`, this is what the `EntityManagerProvider` implementation could look like:

``` java
public class MyEntityManagerProvider implements EntityManagerProvider {

    private EntityManager entityManager;

    @Override
    public EntityManager getEntityManager() {
        return entityManager;
    }

    @PersistenceContext(unitName = "myPersistenceUnit")
    public void setEntityManager(EntityManager entityManager) {
        this.entityManager = entityManager;
    }                
```

By default, the JPA Event Store stores entries in `DomainEventEntry` and `SnapshotEventEntry` entities. While this will suffice in many cases, you might encounter a situation where the meta-data provided by these entities is not enough. Or you might want to store events of different aggregate types in different tables.

If that is the case, you can extend the `JpaEventStorageEngine`. It contains a number of protected methods that you can override to tweak its behavior.

> **Warning**
>
> Note that persistence providers, such as Hibernate, use a first-level cache on their `EntityManager` implementation. Typically, this means that all entities used or returned in queries are attached to the `EntityManager`. They are only cleared when the surrounding transaction is committed or an explicit "clear" is performed inside the transaction. This is especially the case when the Queries are executed in the context of a transaction.
>
> To work around this issue, make sure to exclusively query for non-entity objects. You can use JPA's "SELECT new SomeClass(parameters) FROM ..." style queries to work around this issue. Alternatively, call `EntityManager.flush()` and `EntityManager.clear()` after fetching a batch of events. Failure to do so might result in `OutOfMemoryException`s when loading large streams of events.

JDBC Event Storage Engine
-------------------------

The JDBC event storage engine uses a JDBC Connection to store Events in a JDBC compatible data storage. Typically, these are relational databases. Theoretically, anything that has a JDBC driver could be used to back the JDBC Event Storage Engine.

Similar to its JPA counterpart, the JDBC Event Storage Engine stores Events in entries. By default, each Event is stored in a single Entry, which corresponds with a row in a table. One table is used for Events and another for the Snapshots.

The `JdbcEventStorageEngine` uses a `ConnectionProvider` to obtain connections. Typically, these connections can be obtained directly from a DataSource. However, Axon will bind these connections to a Unit of Work, so that a single connection is used in a Unit of Work. This ensures that a single transaction is used to store all events, even when multiple Units of Work are nested in the same thread.

> **Note**
>
> Spring users are recommended to use the `SpringDataSourceConnectionProvider` to attach a connection from a `DataSource` to an existing transaction.

MongoDB Event Storage Engine
----------------------------

MongoDB is a document based NoSQL store. Its scalability characteristics make it suitable for use as an Event Store. Axon provides the `MongoEventStorageEngine`, which uses MongoDB as backing database. It is contained in the Axon Mongo module (Maven artifactId `axon-mongo`).

Events are stored in two separate collections: one for the actual event streams and one for the snapshots.

By default, the `MongoEventStorageEngine` stores each event in a separate document. It is, however, possible to change the `StorageStrategy` used. The alternative provided by Axon is the `DocumentPerCommitStorageStrategy`, which creates a single document for all Events that have been stored in a single commit (i.e. in the same `DomainEventStream`).

Storing an entire commit in a single document has the advantage that a commit is stored atomically. Furthermore, it requires only a single roundtrip for any number of events. A disadvantage is that it becomes harder to query events directly in the database. When refactoring the domain model, for example, it is harder to "transfer" events from one aggregate to another if they are included in a "commit document".

The MongoDB doesn't take a lot of configuration. All it needs is a reference to the collections to store the Events in, and you're set to go. For production environments, you may want to double check the indexes on your collections.

Event Store Utilities
---------------------

Axon provides a number of Event Storage Engines that may be useful in certain circumstances.

### Combining multiple Event Stores into one
The `SequenceEventStorageEngine` is a wrapper around two other Event Storage Engines. When reading, it returns the events from both event storage engines. Appended events are only appended to the second event storage engine. This is useful in cases where two different implementations of Event Storage are used for performance reasons, for example. The first would be a larger, but slower event store, while the second is optimized for quick reading and writing.

### Filtering Stored Events
The `FilteringEventStorageEngine` allows Events to be filtered based on a predicate. Only Events that match this predicate will be stored. Note that Event Processors that use the Event Store as a source of Events, may not receive these events, as they are not being stored. 

### In-Memory Event Storage
There is also an `EventStorageEngine` implementation that keeps the stored events in memory: the `InMemoryEventStorageEngine`. While it probably outperforms any other event store out there, it is not really meant for long-term production use. However, it is very useful in short-lived tools or tests that require an event store.

Influencing the serialization process
-------------------------------------

Event Stores need a way to serialize the Event to prepare it for storage. By default, Axon uses the `XStreamSerializer`, which uses [XStream](http://x-stream.github.io/) to serialize Events into XML. XStream is reasonably fast and is more flexible than Java Serialization. Furthermore, the result of XStream serialization is human readable. Quite useful for logging and debugging purposes.

The XStreamSerializer can be configured. You can define aliases it should use for certain packages, classes or even fields. Besides being a nice way to shorten potentially long names, aliases can also be used when class definitions of events change. For more information about aliases, visit the [XStream website](http://x-stream.github.io/).

Alternatively, Axon also provides the `JacksonSerializer`, which uses [Jackson](https://github.com/FasterXML/jackson) to serialize Events into JSON. While it produces a more compact serialized form, it does require that classes stick to the conventions (or configuration) required by Jackson.

You may also implement your own Serializer, simply by creating a class that implements `Serializer`, and configuring the Event Store to use that implementation instead of the default.

### Serializing Events vs 'the rest' ###

Since Axon 3.1, it is possible to use a different serializer for the storage of events, than all other objects that Axon needs to serializer (such as Commands, Snapshot, Sagas, etc). While the `XStreamSerializer`'s capability to serialize virtually anything makes it a very decent default, its output isn't always a form that makes it nice to share with other applications. The `JacksonSerializer` creates much nicer output, but requires a certain structure in the objects to serialize. This structure is typically present in events, making it a very suitable event serializer.

Using the Configuration API, you can simply register an Event Serializer as follows:

```java
Configurer configurer = ... // initialize
configurer.configureEventSerializer(conf -> /* create serializer here*/);
```

If no explicit `eventSerializer` is configured, Events are serialized using the main serializer that has been configured (which in turn defaults to the XStream serializer).

Event Upcasting
===============

由于软件应用程序的性质不断变化，事件定义很可能随着时间而改变。 由于事件存储被视为只读和只读数据源，因此应用程序必须能够读取所有事件，而不管它们何时添加。 这是Upcasting到来的地方。

最初是面向对象编程的概念，其中“子类在需要时自动转换为其超类”，Upcasting的概念也可以应用于事件源。 推动事件意味着将其从原来的结构转变为新的结构。 与OOP上传不同，事件上传不能完全自动完成，因为新事件的结构对旧事件是未知的。 必须提供手工写作的Upcasters来指定如何将旧结构上传到新结构。

Upcasters是一些类，它们接受一个修订版`x`的输入事件并输出零个或多个修订版`x + 1`的新事件。 而且，升级者在一个链中被处理，这意味着一个升级者的输出被发送到下一个的输入。 这允许您以增量方式更新事件，为每个新事件修订编写一个Upcaster，使它们变得小巧，独立并且易于理解。

> **Note**
>
> Perhaps the greatest benefit of upcasting is that it allows you to do non-destructive refactoring, i.e. the complete event history remains intact.

In this section we'll explain how to write an upcaster, describe the different (abstract) implementations of the Upcaster that come with Axon, and explain how the serialized representations of events affects how upcasters are written.

To allow an upcaster to see what version of serialized object they are receiving, the Event Store stores a revision number as well as the fully qualified name of the Event. This revision number is generated by a `RevisionResolver`, configured in the serializer. Axon provides several implementations of the `RevisionResolver`, such as the `AnnotationRevisionResolver`, which checks for an `@Revision` annotation on the Event payload, a `SerialVersionUIDRevisionResolver` that uses the `serialVersionUID` as defined by Java Serialization API and a `FixedValueRevisionResolver`, which always returns a predefined value. The latter is useful when injecting the current application version. This will allow you to see which version of the application generated a specific event.

Maven users can use the `MavenArtifactRevisionResolver` to automatically use the project version. It is initialized using the groupId and artifactId of the project to obtain the version for. Since this only works in JAR files created by Maven, the version cannot always be resolved by an IDE. If a version cannot be resolved, `null` is returned.

Axon's upcasters do not work with the `EventMessage` directly, but with an `IntermediateEventRepresentation`. The `IntermediateEventRepresentation` provides functionality to retrieve all necessary fields to construct an `EventMessage` (and thus a upcasted `EventMessage` too), together with the actual _upcast_ functions. These upcast functions by default only allow the adjustment of the events payload, payload type and additions to the event its metadata. The actual representation of the events in the upcast function may vary based on the event serializer used or the desired form to work with, so the upcast function of the `IntermediateEventRepresentation` allows the selection of the expected representation type. The other fields, for example the message/aggregate identifier, aggregate type, timestamp etc. are not adjustable by the `IntermediateEventRepresentation`. Adjusting those fields is not the intended work for an Upcaster, hence those options are not provided by the provided `IntermediateEventRepresentation` implementations.

The basic `Upcaster` interface for events in the Axon Framework works on a `Stream` of `IntermediateEventRepresentations` and returns a `Stream` of `IntermediateEventRepresentations`. The upcasting process thus does not directly return the end result of the introduced upcast functions, but chains every upcasting function from one revision to another together by stacking `IntermediateEventRepresentations`. Once this process has taken place and the end result is pulled from them, that is when the actual upcasting function is performed on the serialized event.

Provided abstract Upcaster implementations 
------------------------------------------

As described earlier, the `Upcaster` interface does not upcast a single event; it requires a `Stream<IntermediateEventRepresentation>` and returns one. However, an Upcaster is usually written to adjust a single event out of this stream. More elaborate upcasting set ups are also imaginable, for example from one events to multiple, or an upcaster which pulls state from an earlier event and pushes it in a later one. This section describes the currently provided abstract implementations of Event Upcasters which a user can extend to add its own desired upcast functionality.

- `SingleEventUpcaster` - This is a one to one implementation of an event Upcaster. Extending from this implementation requires one to implement a `canUpcast` and `doUpcast` function, which respectively check whether the event at hand is to be upcasted, and if so how it should be upcasted. This is most likely the implementation to extend from, as most event adjustments are based on self contained data and are one to one.
- `EventMultiUpcaster` - This is a one to many implementation of an event Upcaster. It is mostly identical to a `SingleEventUpcaster`, with the exception that the `doUpcast` function returns a `Stream` instead of a single `IntermediateEventRepresentation`. As such this upcaster allows you to revert a single event to several events. This might be useful if you for example have figured out you want more fine grained events from a _fat_ event.
- `ContextAwareSingleEventUpcaster` - This is a one to one implementation of an Upcaster, which can store context of events during the process. Next to the `canUpcast` and `doUpcast`, the context aware Upcaster requires one to implement a `buildContext` function, which is used to instantiate a context which is carried between events going through the upcaster. The `canUpcast` and `doUpcast` functions receive the context as a second parameter, next to the `IntermediateEventRepresentation`. The context can then be used within the upcasting process to pull fields from earlier events and populate other events. It thus allows you to move a field from one event to a completely different event.
- `ContextAwareEventMultiUpcaster` - This is a one to many implementation of an Upcaster, which can store context of events during the process. This abstract implementation is a combination of the `EventMultiUpcaster` and `ContextAwareSingleEventUpcaster`, and thus services the goal of keeping context of `IntermediateEventRepresentations` and upcasting one such representation to several. This implementation is useful if you not only want to copy a field from one event to another, but have the requirement to generate several new events in the process.

Writing an upcaster
-------------------
The following Java snippets will serve as a basic example of a one to one Upcaster (the `SingleEventUpcaster`).  

Old version of the event: 
```java
@Revision("1.0")
public class ComplaintEvent {
    private String id;
    private String companyName;

    // Constructor, getter, setter...
}
```

New version of the event: 
```java
@Revision("2.0")
public class ComplaintEvent {
    private String id;
    private String companyName;
    private String description; // New field

    // Constructor, getter, setter...
}
```

Upcaster: 
```java
// Upcaster from 1.0 revision to 2.0 revision
public class ComplaintEventUpcaster extends SingleEventUpcaster {
    private static SimpleSerializedType targetType = new SimpleSerializedType(ComplainEvent.class.getTypeName(), "1.0");

    @Override
    protected boolean canUpcast(IntermediateEventRepresentation intermediateRepresentation) {
        return intermediateRepresentation.getType().equals(targetType);
    }

    @Override
    protected IntermediateEventRepresentation doUpcast(IntermediateEventRepresentation intermediateRepresentation) {
        return intermediateRepresentation.upcastPayload(
                new SimpleSerializedType(targetType.getName(), "2.0"),
                org.dom4j.Document.class,
                document -> {
                    document.getRootElement()
                            .addElement("description")
                            .setText("no complaint description"); // Default value
                    return document;
                }
        );
    }
}
```

Spring boot configuration: 
```java
@Configuration
public class AxonConfiguration {

    @Bean
    public SingleEventUpcaster myUpcaster() {
        return new ComplaintEventUpcaster();
    }

    @Bean
    public JpaEventStorageEngine eventStorageEngine(Serializer serializer,
                                                    DataSource dataSource,
                                                    SingleEventUpcaster myUpcaster,
                                                    EntityManagerProvider entityManagerProvider,
                                                    TransactionManager transactionManager) throws SQLException {
        return new JpaEventStorageEngine(serializer,
                myUpcaster::upcast,
                dataSource,
                entityManagerProvider,
                transactionManager);
    }
}
``` 

Content type conversion
-----------------------

An upcaster works on a given content type (e.g. dom4j Document). To provide extra flexibility between upcasters, content types between chained upcasters may vary. Axon will try to convert between the content types automatically by using `ContentTypeConverter`s. It will search for the shortest path from type `x` to type `y`, perform the conversion and pass the converted value into the requested upcaster. For performance reasons, conversion will only be performed if the `canUpcast` method on the receiving upcaster yields true.

The `ContentTypeConverter`s may depend on the type of serializer used. Attempting to convert a `byte[]` to a dom4j `Document` will not make any sense unless a `Serializer` was used that writes an event as XML. To make sure the `UpcasterChain` has access to the serializer-specific `ContentTypeConverter`s, you can pass a reference to the serializer to the constructor of the `UpcasterChain`.

> **Tip**
>
> To achieve the best performance, ensure that all upcasters in the same chain (where one's output is another's input) work on the same content type.

If the content type conversion that you need is not provided by Axon you can always write one yourself using the `ContentTypeConverter` interface.

The `XStreamSerializer` supports Dom4J as well as XOM as XML document representations. The `JacksonSerializer` supports Jackson's `JsonNode`.

Snapshotting
============

当聚集体长期存活，并且它们的状态不断变化时，它们会产生大量事件。 不得不加载所有这些事件来重建聚合的状态可能会对性能产生巨大的影响。 快照事件是具有特殊用途的域事件：它将任意数量的事件汇总为一个事件。 通过定期创建和存储快照事件，事件存储不必返回长长的事件列表。 只是最后一个快照事件和快照创建后发生的所有事件。

例如，库存物品往往会经常变化。 每次出售物品时，事件会将库存减少一个。 每次装运新货物时，库存都会增加一些数量。 如果您每天销售一百件商品，那么您每天至少会制作100件活动。 几天后，您的系统将花费太多时间阅读所有这些事件，以确定是否应该提出“ItemOutOfStockEvent”。 一个快照事件可以替代很多这些事件，只需通过存储当前的库存数量即可。

Creating a snapshot
-------------------

快照创建可以由多种因素触发，例如自上次快照创建的事件数量，初始化聚合的时间超过某个阈值，基于时间等。目前，Axon提供了一种机制，允许您 根据事件计数阈值触发快照。

SnapshotTriggerDefinition接口提供了何时应该创建快照的定义。

'EventCountSnapshotTriggerDefinition`提供了在加载聚合所需事件数量超过特定阈值时触发快照创建的机制。 如果加载聚合所需的事件数量超过了特定的可配置阈值，触发器会通知“Snapshotter”为聚合创建快照。

The snapshot trigger is configured on an Event Sourcing Repository and has a number of properties that allow you to tweak triggering:

-   `Snapshotter` sets the actual snapshotter instance, responsible for creating and storing the actual snapshot event;

-   `Trigger` sets the threshold at which to trigger snapshot creation;

A Snapshotter is responsible for the actual creation of a snapshot. Typically, snapshotting is a process that should disturb the operational processes as little as possible. Therefore, it is recommended to run the snapshotter in a different thread. The `Snapshotter` interface declares a single method: `scheduleSnapshot()`, which takes the aggregate's type and identifier as parameters.

Axon provides the `AggregateSnapshotter`, which creates and stores `AggregateSnapshot` instances. This is a special type of snapshot, since it contains the actual aggregate instance within it. The repositories provided by Axon are aware of this type of snapshot, and will extract the aggregate from it, instead of instantiating a new one. All events loaded after the snapshot events are streamed to the extracted aggregate instance.

> **Note**
>
> Do make sure that the `Serializer` instance you use (which defaults to the `XStreamSerializer`) is capable of serializing your aggregate. The `XStreamSerializer` requires you to use either a Hotspot JVM, or your aggregate must either have an accessible default constructor or implement the `Serializable` interface.

The `AbstractSnapshotter` provides a basic set of properties that allow you to tweak the way snapshots are created:

-   `EventStore` sets the event store that is used to load past events and store the snapshots. This event store must implement the `SnapshotEventStore` interface.

-   `Executor` sets the executor, such as a `ThreadPoolExecutor` that will provide the thread to process actual snapshot creation. By default, snapshots are created in the thread that calls the `scheduleSnapshot()` method, which is generally not recommended for production.

The `AggregateSnapshotter` provides one more property:

-   `AggregateFactories` is the property that allows you to set the factories that will create instances of your aggregates. Configuring multiple aggregate factories allows you to use a single Snapshotter to create snapshots for a variety of aggregate types. The `EventSourcingRepository` implementations provide access to the `AggregateFactory` they use. This can be used to configure the same aggregate factories in the Snapshotter as the ones used in the repositories.

> **Note**
>
> If you use an executor that executes snapshot creation in another thread, make sure you configure the correct transaction management for your underlying event store, if necessary. 
>
> Spring users can use the `SpringAggregateSnapshotter`, which will automatically look up the right `AggregateFactory` from the Application Context when a snapshot needs to be created.

Storing Snapshot Events
-----------------------
当快照存储在事件存储中时，它将自动使用该快照汇总所有之前的事件并将其返回。 所有的事件存储实现允许并发创建快照。 这意味着它们允许在另一个进程为相同聚合添加事件时存储快照。 这允许快照过程作为一个单独的过程完全运行。

> **Note**
>
> Normally, you can archive all events once they are part of a snapshot event. Snapshotted events will never be read in again by the event store in regular operational scenario's. However, if you want to be able to reconstruct aggregate state prior to the moment the snapshot was created, you must keep the events up to that date.

Axon provides a special type of snapshot event: the `AggregateSnapshot`, which stores an entire aggregate as a snapshot. The motivation is simple: your aggregate should only contain the state relevant to take business decisions. This is exactly the information you want captured in a snapshot. All Event Sourcing Repositories provided by Axon recognize the `AggregateSnapshot`, and will extract the aggregate from it. Beware that using this snapshot event requires that the event serialization mechanism needs to be able to serialize the aggregate.

Initializing an Aggregate based on a Snapshot Event
---------------------------------------------------

A snapshot event is an event like any other. That means a snapshot event is handled just like any other domain event. When using annotations to demarcate event handlers (`@EventHandler`), you can annotate a method that initializes full aggregate state based on a snapshot event. The code sample below shows how snapshot events are treated like any other domain event within the aggregate.

``` java
public class MyAggregate extends AbstractAnnotatedAggregateRoot {

    // ... code omitted for brevity

    @EventHandler
    protected void handleSomeStateChangeEvent(MyDomainEvent event) {
        // ...
    }

    @EventHandler
    protected void applySnapshot(MySnapshotEvent event) {
        // the snapshot event should contain all relevant state
        this.someState = event.someState;
        this.otherState = event.otherState;
    }
}                
```

There is one type of snapshot event that is treated differently: the `AggregateSnapshot`. This type of snapshot event contains the actual aggregate. The aggregate factory recognizes this type of event and extracts the aggregate from the snapshot. Then, all other events are re-applied to the extracted snapshot. That means aggregates never need to be able to deal with `AggregateSnapshot` instances themselves.

Advanced conflict detection and resolution
==========================================

明确变化含义的主要优点之一是可以更精确地检测冲突变化。 通常，当两个用户同时（几乎）同时操作相同的数据时，会发生这些冲突的更改。 想象两个用户，都在查看特定版本的数据。 他们都决定改变这些数据。 他们都会发送一个类似于“在该聚合的X版本上”的命令，这样做，其中X是聚合的预期版本。 其中之一将实际应用于预期版本的更改。 其他用户不会。

Instead of simply rejecting all incoming commands when aggregates have been modified by another process, you could check whether the user's intent conflicts with any unseen changes. 

To detect conflict, pass a parameter of type `ConflictResolver` to the `@CommandHandler` method of your aggregate. This interface provides `detectConflicts` methods that allow you to define the types of events that are considered a conflict when executing that specific type of command.

> **Note**
>
> Note that the `ConflictResolver` will only contain any potentially conflicting events if the Aggregate was loaded with an expected version. Use `@TargetAggregateVersion` on a field of a command to indicate the expected version of the Aggregate.

If events matching the predicate are found, an exception is thrown (the optional second parameter of `detectConflicts` allows you to define the exception to throw). If none are found, processing continues as normal.

If no invocations to `detectConflicts` are made, and there are potentially conflicting events, the `@CommandHandler` will fail. This may be the case when an expected version is provided, but no `ConflictResolver` is available in the parameters of the `@CommandHandler` method. 

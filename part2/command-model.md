Command Model
===============

在基于CQRS的应用程序中，领域模型（如Eric Evans和Martin Fowler所定义的）可以是一个非常强大的机制，用于处理状态更改验证和执行过程中涉及的复杂性。 虽然典型的领域模型有大量的构建块，但是其中一个在应用于CQRS中的命令处理时扮演主导角色：聚合。

应用程序中的状态更改以Command开头。 命令是表达意图的组合（它描述了你想要做什么）以及基于该意图采取行动所需的信息。 命令模型用于处理传入的命令，以验证它并定义结果。 在这个模型中，一个Command Handler负责处理某种类型的命令，并根据其中包含的信息采取行动。

Aggregate
---------
聚合是始终保持一致状态的实体或实体组。 聚集根是负责维护此一致状态的聚合树顶部的对象。 这使得Aggregate成为在任何基于CQRS的应用程序中实现命令模型的主要构建模块。

> **Note**
>
> 术语“聚合”是指Evans在域驱动设计中定义的集合：
>
> “一组关联对象，作为数据更改目的的单位。 外部引用仅限于Aggregate的一个成员，被指定为根。 一组一致性规则适用于Aggregate的边界内。“

例如，“联系人”聚合可以包含两个实体：联系人和地址。 为了保持整个聚合状态一致，向联系人添加地址应通过联系人实体完成。 在这种情况下，联系人实体是指定的聚合根。

在Axon中，聚合由一个聚合标识标识。 这可能是任何对象，但是对于标识符的良好实现有几条准则。 标识符必须：

-  实现`equals`和`hashCode`确保与其他实例进行良好的平等比较，

-   实现一个提供一致结果的`toString（）`方法（相同的标识符应该提供一个相等的toString（）结果），以及

-   最好是`Serializable`。

测试fixtures（参见[Testing]（testing.md））将验证这些条件，并在Aggregate使用不兼容的标识符时通过测试。 “String”，“UUID”和数字类型的标识符总是合适的。 Do ** not ** 使用原始类型作为标识符，因为它们不允许延迟初始化。 在某些情况下，Axon可能会错误地将基元的默认值假定为标识符的值。

> **Note**
>
> 使用随机生成的标识符被认为是一个好习惯，而不是按顺序标识符。 使用序列会大大降低应用程序的可伸缩性，因为机器需要互相保持最新使用的序列号的最新版本。 与UUID碰撞的几率很小（如果生成8.2×10 <11> </ sup> UUID，则机会为10 <-15 </ sup>）。
>
> 此外，在为聚合使用功能标识符时要小心。 他们有改变的趋势，因此很难相应地调整你的应用程序。

Aggregate implementations
-------------------------
一个聚合总是通过一个称为聚合根的实体来访问。 通常，该实体的名称与聚合的名称完全相同。 例如，一个订单集合可以由一个订单实体组成，该实体引用多个订单行实体。 订单和订单一起，形成总和。

聚集是一个常规的对象，它包含改变状态的状态和方法。 虽然根据CQRS原则不完全正确，但也可以通过访问器方法公开聚合的状态。

聚合根必须声明包含聚合标识符的字段。 该标识符必须在第一个事件发布时最迟被初始化。 该标识符字段必须由“@ AggregateIdentifier”注释注释。
如果您使用JPA并在聚合上使用JPA批注，则Axon也可以使用JPA提供的“@ Id”批注。

聚合可以使用`AggregateLifecycle.apply（）`方法来注册要发布的事件。 与'EventBus`不同的是，消息需要被包装在一个EventMessage中，'apply（）`允许你直接传递有效载荷对象。

``` java
import static org.axonframework.commandhandling.model.AggregateLifecycle.apply;

@Entity // Mark this aggregate as a JPA Entity
public class MyAggregate {
    
    @Id // When annotating with JPA @Id, the @AggregateIdentifier annotation is not necessary
    private String id;
    
    // fields containing state...
    
    @CommandHandler
    public MyAggregate(CreateMyAggregateCommand command) {
        // ... update state
        apply(new MyAggregateCreatedEvent(...));
    }
    
    // constructor needed by JPA
    protected MyAggregate() {
    }
}
```

通过定义一个`@ EventHandler`注释的方法，一个Aggregate中的实体可以监听Aggregate发布的事件。 当发布EventMessage时（在任何外部处理程序发布之前），将调用这些方法。

Event sourced aggregates
------------------------
除了存储Aggregate的当前状态之外，还可以根据它过去发布的Events来重建Aggregate的状态。 为了这个工作，所有的状态改变必须由一个Event来表示。

对于主要部分来说，事件溯源聚合类似于'常规'聚合：它们必须声明一个标识符并且可以使用`apply`方法来发布事件。 但是，事件溯源聚合中的状态更改（即字段值的任何更改）必须*独占*在`@ EventSourcingHandler`注释方法中执行。 这包括设置聚合标识符。

请注意，聚合标识符必须在聚合发布的第一个事件的“@ EventSourcingHandler”中设置。 这通常是创建事件。

源事件的聚合根源也必须包含无参数构造函数。 Axon Framework使用此构造函数在使用过去的事件初始化它之前创建一个空的Aggregate实例。 加载聚合时，未能提供此构造函数将导致异常。

``` java
public class MyAggregateRoot {

    @AggregateIdentifier
    private String aggregateIdentifier;
    
    // fields containing state...

    @CommandHandler
    public MyAggregateRoot(CreateMyAggregate cmd) {
        apply(new MyAggregateCreatedEvent(cmd.getId()));
    }

    // constructor needed for reconstruction
    protected MyAggregateRoot() {
    }

    @EventSourcingHandler
    private void handleMyAggregateCreatedEvent(MyAggregateCreatedEvent event) {
        // make sure identifier is always initialized properly
        this.aggregateIdentifier = event.getMyAggregateIdentifier();
        
        // ... update state
    }
}                
```

使用特定规则解决`@ EventSourcingHandler`注释的方法。 这些规则与`@ EventHandler`注释方法相同，在[Annotated Event Handler]（./ event-handling.md#definition-event-handlers）中有详细解释。

> **Note**
>
>事件处理程序方法可以是私有的，只要JVM的安全设置允许Axon框架更改方法的可访问性即可。 这使您可以清楚地将聚合的公共API（公开生成事件的方法）与处理事件的内部逻辑分开。
>
> 大多数IDE都可以选择忽略具有特定注释的方法的“未使用私有方法”警告。 或者，您可以向该方法添加一个@SuppressWarnings（“UnusedDeclaration”）注释，以确保您不会意外删除事件处理程序方法。

在某些情况下，特别是当聚合结构增长超出一些实体时，对于在同一聚合的其他实体中发布的事件的反应更为清晰。 但是，由于在重新构建聚合状态时也会调用事件处理程序方法，因此必须采取特殊的预防措施。

可以在Event Sourcing Handler方法中 ` apply（）` 新事件。 这使得实体B可以应用事件来响应实体A做某事。 重放历史事件时，Axon将忽略apply（）调用。 请注意，在这种情况下，在所有实体接收到第一个事件后，内部`apply（）`调用的事件仅发布给实体。 如果需要发布更多事件，根据应用内部事件后的实体状态，使用`apply（...）。和ThenApply（...）`

你也可以使用静态的`AggregateLifecycle.isLive（）`方法来检查聚合是否是'活的'。 基本上，如果聚合完成重放历史事件，则视为聚合。 在重播这些事件时，isLive（）将返回false。 使用这个`isLive（）`方法，您可以执行仅在处理新生成的事件时完成的活动。

Complex Aggregate structures
----------------------------
复杂的业务逻辑通常需要的不仅仅是聚合根可以提供的聚合。 在这种情况下，将复杂性分散到聚合中的许多实体是很重要的。 在使用事件溯源时，不仅聚合根需要使用事件来触发状态转换，而且聚合中的每个实体也是如此。

> ** Note **
> 对Aggregates不应该暴露状态的规则的常见误解是没有任何实体应该包含任何属性访问器方法。 不是这种情况。 事实上，如果聚合中的实体 *within* 在同一总量中向其他实体公开状态，则聚合可能会受益匪浅。 但是，建议不要暴露Aggregate的状态 *outside* 。

Axon为复杂聚合结构中的事件溯源提供支持。 实体像聚合根一样是简单的对象。 声明子实体的字段必须用`@ AggregateMember`注释。 此注释告诉Axon注释的字段包含应该检查命令和事件处理程序的类。

当实体（包括聚合根）应用事件时，首先由聚合根处理，然后通过所有带有@ @ AggregateMember注释的字段向其子实体下泡。

（可能）包含子实体的字段必须用@ @ AggregateMember注解。 此注释可用于多种字段类型：

-  实体类型，在字段中直接引用;

-  包含`Iterable`（其中包括所有集合，例如`Set`，`List`等）的内部字段;

-  在包含`java.util.Map`的字段的值内

### Handling commands in an Aggregate

建议直接在包含处理此命令的状态的聚合中定义命令处理程序，因为命令处理程序不需要该聚合的状态来完成其工作。

要在Aggregate中定义Command Handler，只需使用`@ CommandHandler`注释命令处理方法。 @ CommandHandler注解方法的规则与任何处理程序方法相同。 但是，命令不仅可以通过其有效负载进行路由。 命令消息带有一个名称，该名称默认为Command对象的全限定类名称。

默认情况下，`@ CommandHandler`注解方法允许以下参数类型：

- 第一个参数是命令消息的有效载荷。 如果@ CommandHandler注解明确定义了处理程序可以处理的Command的名称，它也可以是'Message'或'CommandMessage`   类型。 默认情况下，命令名称是命令有效负载的完全限定类名称。

- 使用`@ MetaDataValue`注解的参数将使用注释中指示的键解析为元数据值。 如果`required`是`false`（默认），当元数据值不存在时传递`null`。 如果`required`是`true`，解析器将不匹配，并防止在元数据值不存在时调用该方法。

-  `MetaData`类型的参数将注入一个`CommandMessage`的整个`MetaData`。`MetaData`类型的参数将注入一个`CommandMessage`的整个`MetaData`。

- Parameters of type `UnitOfWork` get the current Unit of Work injected. This allows command handlers to register actions to be performed at specific stages of the Unit of Work, or gain access to the resources registered with it.

- Parameters of type `Message`, or `CommandMessage` will get the complete message, with both the payload and the Meta Data. This is useful if a method needs several meta data fields, or other properties of the wrapping Message.

In order for Axon to know which instance of an Aggregate type should handle the Command Message, the property carrying the Aggregate Identifier in the Command object must be annotated with `@TargetAggregateIdentifier`. The annotation may be placed on either the field or an accessor method (e.g. a getter).

Commands that create an Aggregate instance do not need to identify the target aggregate identifier, although it is recommended to annotate the Aggregate identifier on them as well. 

If you prefer to use another mechanism for routing Commands, the behavior can be overridden by supplying a custom `CommandTargetResolver`. This class should return the Aggregate Identifier and expected version (if any) based on a given command.

> **Note**
>
> When the `@CommandHandler` annotation is placed on an Aggregate's constructor, the respective command will create a new instance of that aggregate and add it to the repository. Those commands do not require to target a specific aggregate instance. Therefore, those commands do not require any `@TargetAggregateIdentifier` or `@TargetAggregateVersion` annotations, nor will a custom `CommandTargetResolver` be invoked for these commands.
>
> When a command creates an aggregate instance, the callback for that command will receive the aggregate identifier when the command executed successfully.

``` java
import static org.axonframework.commandhandling.model.AggregateLifecycle.apply;

public class MyAggregate {

    @AggregateIdentifier
    private String id;

    @CommandHandler
    public MyAggregate(CreateMyAggregateCommand command) {
        apply(new MyAggregateCreatedEvent(IdentifierFactory.getInstance().generateIdentifier()));
    }

    // no-arg constructor for Axon
    MyAggregate() {
    }

    @CommandHandler
    public void doSomething(DoSomethingCommand command) {
        // do something...
    }

    // code omitted for brevity. The event handler for MyAggregateCreatedEvent must set the id field
}

public class DoSomethingCommand {

    @TargetAggregateIdentifier
    private String aggregateId;

    // code omitted for brevity

}
```

The Axon Configuration API can be used configure the Aggregate. For example:

```
Configurer configurer = ...
// to use defaults:
configurer.configureAggreate(MyAggregate.class);

// allowing customizations:
configurer.configureAggregate(
        AggregateConfigurer.defaultConfiguration(MyAggregate.class)
                           .configureCommandTargetResolver(c -> new CustomCommandTargetResolver())
);
```

`@CommandHandler` annotations are not limited to the aggregate root. Placing all command handlers in the root will sometimes lead to a large number of methods on the aggregate root, while many of them simply forward the invocation to one of the underlying entities. If that is the case, you may place the `@CommandHandler` annotation on one of the underlying entities' methods. For Axon to find these annotated methods, the field declaring the entity in the aggregate root must be marked with `@AggregateMember`. Note that only the declared type of the annotated field is inspected for Command Handlers. If a field value is null at the time an incoming command arrives for that entity, an exception is thrown.

```java
public class MyAggregate {

    @AggregateIdentifier
    private String id;

    @AggregateMember
    private MyEntity entity;

    @CommandHandler
    public MyAggregate(CreateMyAggregateCommand command) {
        apply(new MyAggregateCreatedEvent(...);
    }

    // no-arg constructor for Axon
    MyAggregate() {
    }

    @CommandHandler
    public void doSomething(DoSomethingCommand command) {
        // do something...
    }

    // code omitted for brevity. The event handler for MyAggregateCreatedEvent must set the id field
    // and somewhere in the lifecycle, a value for "entity" must be assigned to be able to accept
    // DoSomethingInEntityCommand commands.
}

public class MyEntity {

    @CommandHandler
    public void handleSomeCommand(DoSomethingInEntityCommand command) {
        // do something
    }
}
```
> **Note**
>
> Note that each command must have exactly one handler in the aggregate. This means that you cannot annotate multiple entities (either root nor not) with @CommandHandler, that handle the same command type. In case you need to conditionally route a command to an entity, the parent of these entities should handle the command, and forward it based on the conditions that apply.
>
> The runtime type of the field does not have to be exactly the declared type. However, only the declared type of the `@AggregateMember` annotated field is inspected for `@CommandHandler` methods.

It is also possible to annotate Collections and Maps containing entities with `@AggregateMember`. In the latter case, the values of the map are expected to contain the entities, while the key contains a value that is used as their reference.

As a command needs to be routed to the correct instance, these instances must be properly identified. Their "id" field must be annotated with `@EntityId`. The property on the command that will be used to find the Entity that the message should be routed to, defaults to the name of the field that was annotated. For example, when annotating the field "myEntityId", the command must define a property with that same name. This means either a `getMyEntityId` or a `myEntityId()` method must be present. If the name of the field and the routing property differ, you may provide a value explicitly using `@EntityId(routingKey = "customRoutingProperty")`.

If no Entity can be found in the annotated Collection or Map, Axon throws an IllegalStateException; apparently, the aggregate is not capable of processing that command at that point in time.

> **Note**
>
> The field declaration for both the Collection or Map should contain proper generics to allow Axon to identify the type of Entity contained in the Collection or Map. If it is not possible to add the generics in the declaration (e.g. because you're using a custom implementation which already defines generic types), you must specify the type of entity used in the `entityType` property on the `@AggregateMember` annotation.

### External Command Handlers

In certain cases, it is not possible, or desired to route a command directly to an Aggregate instance. In such case, it is possible to register a Command Handler object.

A Command Handler object is a simple (regular) object, which has `@CommandHandler` annotated methods. Unlike in the case of an Aggregate, there is only a single instance of a Command Handler object, which handles all commands of the types it declares in its methods.

```java
public class MyAnnotatedHandler {

    @CommandHandler
    public void handleSomeCommand(SomeCommand command, @MetaDataValue("userId") String userId) {
        // whatever logic here
    }

    @CommandHandler(commandName = "myCustomCommand")
    public void handleCustomCommand(SomeCommand command) {
       // handling logic here
    }

}

// To register the annotated handlers to the command bus:
Configurer configurer = ...
configurer.registerCommandHandler(c -> new MyAnnotatedHandler());
```

### Returning results from Command Handlers
In some cases, the component dispatching a Command needs information about the processing results of a Command. A Command handler method can return a value from its method. That value will be provided to the sender as the result of the command.

One exception is the `@CommandHandler` on an Aggregate's constructor. In this case, instead of returning the return value of the method (which is the Aggregate itself), the value of the `@AggregateIdentifier` annotated field is returned instead

> **Note**
>
> While it's possible to return results from Commands, it should be used sparsely. The intent of the command should never be in getting a value, as that would be an indication the message should be designed as a Query Message instead. A typical example for a Command result is the identifier of a newly created entity.

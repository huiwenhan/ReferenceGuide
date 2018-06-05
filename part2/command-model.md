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

- UnitOfWork”类型的参数获取注入的当前工作单元。 这允许命令处理程序注册要在工作单元的特定阶段执行的操作，或获取对其注册的资源的访问权限。

- “Message”或“CommandMessage”类型的参数将获得完整的消息，同时包含有效内容和元数据。 如果方法需要多个元数据字段或包装消息的其他属性，这非常有用。

为了让Axon知道Aggregate类型的哪个实例应该处理Command消息，携带Command对象中的Aggregate Identifier的属性必须用`@ TargetAggregateIdentifier`标注。 注释可以放在字段或访问器方法（例如getter）上。

创建Aggregate实例的命令不需要标识目标聚合标识符，但建议在它们上注释集合标识符。

如果您更喜欢使用其他机制来路由命令，则可以通过提供自定义的“CommandTargetResolver”来覆盖该行为。 该类应该根据给定的命令返回聚合标识符和预期版本（如果有的话）。

> **Note**
>
> 当`@ CommandHandler`注释放置在一个Aggregate的构造函数上时，相应的命令将创建该聚合的一个新实例并将其添加到存储库。 这些命令不需要定位特定的聚合实例。 因此，这些命令不需要任何`@ TargetAggregateIdentifier`或`@ TargetAggregateVersion`注释，也不会为这些命令调用自定义的`CommandTargetResolver`。
>
>当一个命令创建一个聚合实例时，该命令的回调将在该命令成功执行时收到聚合标识符。

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

Axon配置API可用于配置Aggregate。 例如：

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

`@ CommandHandler`注释不限于聚合根。 将所有命令处理程序放在根中有时会导致聚合根上的大量方法，而其中许多方法只是将调用转发给其中一个基础实体。 如果是这种情况，您可以将`@ CommandHandler`注释放在其中一个基础实体的方法中。 对于Axon来查找这些带注释的方法，在聚合根中声明实体的字段必须标记为@ @ AggregateMember。 请注意，只有命令处理程序检查了注释字段的声明类型。 如果传入命令到达该实体时字段值为空，则会引发异常。

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
> 请注意，每个命令在聚合中必须只有一个处理程序。 这意味着你不能使用@CommandHandler注解多个实体（根或非），它们处理相同的命令类型。 如果您需要有条件地将命令路由到实体，则这些实体的父级应处理该命令，并根据所应用的条件转发该命令。
>
> 该字段的运行时类型不必完全是声明的类型。 但是，只检查`@ AggregateMember`注释字段的声明类型的`@ CommandHandler`方法。

也可以使用@ AggregateMember注解包含实体的集合和Map。 在后一种情况下，映射的值应包含实体，而键包含一个用作参考的值。

由于需要将命令路由到正确的实例，因此必须正确标识这些实例。 他们的“id”字段必须用`@ EntityId`注释。 将用于查找消息应该路由到的实体的命令属性默认为注释字段的名称。 例如，当注释字段“myEntityId”时，该命令必须定义具有相同名称的属性。 这意味着必须存在`getMyEntityId`或`myEntityId（）`方法。 如果字段的名称和路由属性不同，可以使用`@EntityId（routingKey =“customRoutingProperty”）`显式地提供一个值。

如果在带注释的集合或映射中找不到实体，则Axon会抛出IllegalStateException; 显然，汇总在该时间点无法处理该命令。

> **Note**
>
> 集合或映射的字段声明应包含适当的泛型，以允许Axon识别集合或映射中包含的实体的类型。 如果无法在声明中添加泛型（例如，因为您正在使用已定义泛型的自定义实现），则必须在`@ AggregateMember`注释的`entityType`属性中指定使用的实体类型。

### External Command Handlers

在某些情况下，不可能或不希望将命令直接路由到聚合实例。 在这种情况下，可以注册一个Command Handler对象。

一个Command Handler对象是一个简单的（常规）对象，它有`@ CommandHandler`注解的方法。 与Aggregate的情况不同，Command Handler对象只有一个实例，它处理它在其方法中声明的所有类型的命令。

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
在某些情况下，调度Command的组件需要有关Command的处理结果的信息。 命令处理程序方法可以从其方法返回一个值。 该值将作为命令的结果提供给发件人。

一个例外是Aggregate的构造函数中的@ CommandHandler。 在这种情况下，不是返回方法的返回值（它是Aggregate本身），而是返回“@ AggregateIdentifier”注释字段的值

> **Note**
>
> 虽然可以从命令返回结果，但应该使用稀疏。 该命令的意图不应该是获取值，因为这将表明该消息应该被设计为查询消息。 命令结果的典型示例是新创建的实体的标识符。

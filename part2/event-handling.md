Event Handling
===============

事件侦听器是对传入事件起作用的组件。 他们通常根据命令模型所做的决定来执行逻辑。 通常，这涉及更新视图模型或将更新转发到其他组件，例如第三方集成。 在某些情况下，事件处理程序会根据事件的（模式）自己抛出事件，或者甚至发送命令来触发进一步的更改。

Defining Event Handlers
-----------------------

在Axon中，一个对象可以声明一些Event Handler方法，用`@ EventHandler`注释它们。 该方法的声明参数定义了它将接收哪些事件。

Axon为以下参数类型提供开箱即用的支持：

* The first parameter is always the payload of the Event Message. In the case the Event Handlers doesn't need access to the payload of the message, you can specify the expected payload type on the `@EventHandler` annotation. When specified, the first parameter is resolved using the rules specified below. Do not configure the payload type on the annotation if you want the payload to be passed as a parameter.

* Parameters annotated with `@MetaDataValue` will resolve to the Meta Data value with the key as indicated on the annotation. If `required` is `false` (default), `null` is passed when the meta data value is not present. If `required` is `true`, the resolver will not match and prevent the method from being invoked when the meta data value is not present.

* Parameters of type `MetaData` will have the entire `MetaData` of an `EventMessage` injected.

* Parameters annotated with `@Timestamp` and of type `java.time.Instant` (or `java.time.temporal.Temporal`) will resolve to the timestamp of the `EventMessage`. This is the time at which the Event was generated.

* Parameters annotated with `@SequenceNumber` and of type `java.lang.Long` or `long` will resolve to the `sequenceNumber` of a `DomainEventMessage`. This provides the order in which the Event was generated (within the scope of the Aggregate it originated from).

* Parameters assignable to Message will have the entire `EventMessage` injected \(if the message is assignable to that parameter\). If the first parameter is of type message, it effectively matches an Event of any type, even if generic parameters would suggest otherwise. Due to type erasure, Axon cannot detect what parameter is expected. In such case, it is best to declare a parameter of the payload type, followed by a parameter of type Message.

* When using Spring and the Axon Configuration is activated (either by including the Axon Spring Boot Starter module, or by specifying `@EnableAxon` on your `@Configuration` file), any other parameters will resolve to autowired beans, if exactly one injectable candidate is available in the application context. This allows you to inject resources directly into `@EventHandler` annotated methods.

You can configure additional `ParameterResolver`s by implementing the `ParameterResolverFactory` interface and creating a file named `/META-INF/service/org.axonframework.common.annotation.ParameterResolverFactory` containing the fully qualified name of the implementing class. See [Advanced Customizations](../part4/advanced-customizations.md) for details.

在所有情况下，每个侦听器实例最多调用一个事件处理程序方法。 Axon将使用以下规则搜索最具体的调用方法：

1. On the actual instance level of the class hierarchy (as returned by `this.getClass()`), all annotated methods are evaluated

2. If one or more methods are found of which all parameters can be resolved to a value, the method with the most specific type is chosen and invoked

3. If no methods are found on this level of the class hierarchy, the super type is evaluated the same way

4. When the top level of the hierarchy is reached, and no suitable event handler is found, the event is ignored.

```java
// assume EventB extends EventA 
// and    EventC extends EventB
// and    a single instance of SubListener is registered

public class TopListener {

    @EventHandler
    public void handle(EventA event) {
    }

    @EventHandler
    public void handle(EventC event) {
    }
}

public class SubListener extends TopListener {

    @EventHandler
    public void handle(EventB event) {
    }
}
```

In the example above, the handler methods of `SubListener` will be invoked for all instances of `EventB` as well as `EventC` (as it extends `EventB`). In other words, the handler methods of `TopListener` will not receive any invocations for `EventC` at all. Since `EventA` is not assignable to `EventB` (it's its superclass), those will be processed by the handler method in `TopListener`.

Registering Event Handlers
-----------------------------
事件处理组件是通过一个`EventHandlingConfiguration`类来定义的，该类被注册为全局Axon`Configurer`的模块。 通常情况下，应用程序会定义一个`EventHandlingConfiguration`，但更大的模块化应用程序可能会选择为每个模块定义一个。

要使用`@ EventHandler`方法注册对象，请在`EventHandlingConfiguration`上使用`registerEventHandler`方法：

```java
// define an EventHandlingConfiguration
EventHandlingConfiguration ehConfiguration = new EventHandlingConfiguration()
    .registerEventHandler(conf -> new MyEventHandlerClass());

// the module needs to be registered with the Axon Configuration
Configurer axonConfigurer = DefaultConfigurer.defaultConfiguration()
    .registerModule(ehConfiguration);
```

有关使用Spring AutoConfiguration注册事件处理程序的详细信息,请看 [Event Handling Configuration](../part3/spring-boot-autoconfig.md#event-handling-configuration)

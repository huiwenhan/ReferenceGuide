Configuration API
=================

Axon在业务逻辑和基础架构配置方面保持严格的分离。 为了做到这一点，Axon将提供一些构建模块来处理基础设施问题，例如消息处理器周围的事务管理。 消息的实际有效负载和处理程序的内容在（尽可能多）Axon独立的Java类中实现。

为了简化这些基础设施组件的配置并定义它们与每个功能组件的关系，Axon提供了一个配置API。

Setting up a configuration
--------------------------

获取默认配置非常简单：

``` java
Configuration config = DefaultConfigurer.defaultConfiguration()
                                        .buildConfiguration();
```

此配置使用处理分派线程上的消息的实现提供分发消息的构建块。

显然，这种配置不会很有用。 您必须将您的Command Model对象和事件处理程序注册到此配置才有用。

To do so, use the `Configurer` instance returned by the `.defaultConfiguration()` method.

``` java
Configurer configurer = DefaultConfigurer.defaultConfiguration();
```

配置器提供了许多方法，允许您注册这些组件。 如何配置这些内容将在每个组件的相应章节中详细介绍。

组件注册的一般形式如下：

``` java
Configurer configurer = DefaultConfigurer.defaultConfiguration();
configurer.registerCommandHandler(c -> doCreateComponent());
```

请注意`registerCommandBus`调用中的lambda表达式。 该表达式的`c`参数是描述完整配置的配置对象。 如果您的组件需要其他组件正常运行，则可以使用此配置来检索它们。

例如，要注册需要序列化程序的命令处理程序，请执行以下操作：

``` java
configurer.registerCommandHandler(c -> new MyCommandHandler(c.serializer());
```

并非所有组件都有其明确的访问方法。 要从配置中检索组件，请使用：

``` java
configurer.registerCommandHandler(c -> new MyCommandHandler(c.getComponent(MyOtherComponent.class));
```

该组件必须使用`configurer.registerComponent（componentType，builderFunction）``在配置器中注册。 构建器函数将接收`Configuration`对象作为输入参数。

Setting up a configuration using Spring
---------------------------------------

使用Spring时，不需要明确使用`Configurer`。 相反，您可以简单地将`@ EnableAxon`放置在您的某个Spring`@ Configuration`类中。

Axon将使用Spring应用程序上下文来定位构建块的特定实现，并为那些不在那里的构件提供默认值。 因此，在Spring中，您不必使用`Configurer`来注册构建块，而必须在应用程序上下文中将它们作为@ Bean来使用。

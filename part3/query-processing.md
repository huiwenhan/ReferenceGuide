Query Dispatching
=================
自3.1版本以来，Axon Framework还为查询处理提供了组件。 尽管创建这样一个图层是非常直接的，但在这部分应用程序中使用Axon Framework有许多好处，例如重用拦截器和消息监视等功能。

接下来的部分概述了与使用Axon框架设置Query调度基础架构相关的任务。

Query Gateway
=============
查询网关是查询调度机制的便捷接口。 虽然您不需要使用网关来发送查询，但通常这是最简单的选择。 Axon提供了一个`QueryGateway`接口和`DefaultQueryGateway`实现。 查询网关提供了许多方法，允许您发送查询并同步等待一个或多个结果，具有超时或异步。 查询网关需要配置为可以访问Query Bus和一个（可能为空）QueryDispatchInterceptor列表。

Query Bus
=========

查询总线是将查询分派给查询处理程序的机制。 查询使用查询请求名称和查询响应类型的组合进行注册。 可以为相同的请求 - 响应组合注册多个处理程序，这可用于实现分散 - 聚集模式。 在调度查询时，客户端必须指出是否需要来自单个处理程序或所有处理程序的响应。

如果客户端请求单个处理程序的响应，并且没有找到处理程序，则会抛出“NoHandlerForQueryException”。 如果注册了多个处理程序，则由查询总线的实现决定实际调用哪个处理程序。

如果客户端请求所有处理程序的响应，则会返回结果流。 该流包含来自每个处理程序的成功处理查询的结果，以未指定的顺序。 如果查询没有处理程序，或者所有处理程序在处理请求时都抛出异常，则该流为空。

SimpleQueryBus
--------------

SimpleQueryBus是Axon 3.1中唯一的Query Bus实现。 它在调度它们的线程中直接处理查询。 `SimpleQueryBus`允许配置拦截器，参见[Query Interceptors]（＃query-interceptors）了解更多信息。

Query Interceptors
==================
One of the advantages of using a query bus is the ability to undertake action based on all incoming queries. Examples are logging or authentication, which you might want to do regardless of the type of query. This is done using Interceptors.

There are different types of interceptors: Dispatch Interceptors and Handler Interceptors. Dispatch Interceptors are invoked before a query is dispatched to a Query Handler. At that point, it may not even be sure that any handler exists for that query. Handler Interceptors are invoked just before a Query Handler is invoked.

Dispatch Interceptors
---------------------
Message Dispatch Interceptors are invoked when a query is dispatched on a Query Bus. They have the ability to alter the Query Message, by adding Meta Data, for example, or block the query by throwing an Exception. These interceptors are always invoked on the thread that dispatches the Query.
Message Dispatch Interceptors are invoked when a query is dispatched on a Query Bus. They have the ability to alter the Query Message, by adding Meta Data, for example, or block the query by throwing an Exception. These interceptors are always invoked on the thread that dispatches the Query.

### Structural validation

There is no point in processing a query if it does not contain all required information in the correct format. In fact, a query that lacks information should be blocked as early as possible. Therefore, an interceptor should check all incoming queries for the availability of such information. This is called structural validation.

Axon Framework has support for JSR 303 Bean Validation based validation. This allows you to annotate the fields on queries with annotations like `@NotEmpty` and `@Pattern`. You need to include a JSR 303 implementation (such as Hibernate-Validator) on your classpath. Then, configure a `BeanValidationInterceptor` on your Query Bus, and it will automatically find and configure your validator implementation. While it uses sensible defaults, you can fine-tune it to your specific needs.

> **Tip**
>
> You want to spend as few resources on an invalid queries as possible. Therefore, this interceptor is generally placed in the very front of the interceptor chain. In some cases, a Logging or Auditing interceptor might need to be placed in front, with the validating interceptor immediately following it.

The BeanValidationInterceptor also implements `MessageHandlerInterceptor`, allowing you to configure it as a Handler Interceptor as well.

Handler Interceptors
--------------------
Message Handler Interceptors can take action both before and after query processing. Interceptors can even block query processing altogether, for example for security reasons.

Interceptors must implement the `MessageHandlerInterceptor` interface. This interface declares one method, `handle`, that takes three parameters: the query message, the current `UnitOfWork` and an `InterceptorChain`. The `InterceptorChain` is used to continue the dispatching process.

Unlike Dispatch Interceptors, Handler Interceptors are invoked in the context of the Query Handler. That means they can attach correlation data based on the Message being handled to the Unit of Work, for example. This correlation data will then be attached to messages being created in the context of that Unit of Work.

Introduction
============

Axon is a lightweight framework that helps developers build scalable and extensible applications by addressing these concerns directly in the architecture. This reference guide explains what Axon is, how it can help you and how you can use it.

If you want to know more about Axon and its background, continue reading in [Axon Framework Background](#axon-framework-background). If you're eager to get started building your own application using Axon, go quickly to [Getting Started](#getting-started). If you're interested in helping out building the Axon Framework, [Contributing](#contributing-to-axon-framework) will contain the information you require. All help is welcome. Finally, this chapter covers some legal concerns in [License](#license-information).

Axon Framework Background
=========================

A brief history
---------------

随着时间的推移，对软件项目的需求迅速增加。 公司希望他们的（网络）应用程序与他们的业务一起发展。 这意味着不仅项目和代码库变得更加复杂，而且意味着不断添加，更改和（不幸不够）的功能被删除。 发现一个看起来很容易实现的功能可能要求开发团队拆分整个应用程序可能令人沮丧。 此外，今天的Web应用程序面向潜在数十亿人的受众，使可扩展性成为不争的要求。

虽然有很多应用程序和框架涉及可扩展性问题，例如GigaSpaces和Terracotta，但它们有一个基本缺陷。 这些堆栈尝试解决可伸缩性问题，同时让开发人员使用他们习惯的分层架构开发应用程序。 在某些情况下，他们甚至会阻止或严重限制使用真实域模型，从而迫使所有域逻辑进入服务。 尽管开始构建应用程序的速度会更快，但最终这种方法会导致复杂性增加并使开发速度放慢。

命令查询责任分离（CQRS）模式通过彻底改变应用程序架构的方式来解决这些问题。 逻辑不是将逻辑分成单独的层，而是基于它是在改变应用程序的状态还是在查询逻辑。 这意味着执行命令（可能改变应用程序状态的动作）由与查询应用程序状态的组件不同的组件执行。 这种分离的最重要原因是每个人都有不同的技术和非技术要求。 当执行命令时，查询组件（a）使用事件同步更新。 这种通过事件进行更新的机制，使得这种架构具有可扩展性，可伸缩性并最终能够更好地维护。

> **Note**
>
> A full explanation of CQRS is not within the scope of this document. If you would like to have more background information about CQRS, visit the Axon Framework website: [www.axonframework.org](http://www.axonframework.org/). It contains links to background information.

Since CQRS is fundamentally different than the layered-architecture which dominates today's software landscape, it is not uncommon for developers to walk into a few traps while trying to find their way around this architecture. That's why Axon Framework was conceived: to help developers implement CQRS applications while focusing on the business logic.

What is Axon?
-------------

通过支持开发人员应用命令查询责任分离（CQRS）体系结构模式，Axon Framework可帮助构建可扩展，可扩展和可维护的应用程序。 它通过提供最重要构建块的实现来实现，如聚合，存储库和事件总线（事件的调度机制）。 此外，Axon提供了注释支持，允许您构建聚集和事件侦听器，而无需将代码绑定到Axon特定的逻辑。 这使您可以专注于业务逻辑而不是管道，并帮助您使代码更容易独立测试。

Axon并不以任何方式试图隐藏开发人员的CQRS架构或其任何组件。 因此，根据团队规模，建议一个或多个开发人员全面了解每个团队的CQRS。 然而，Axon在确保将事件传递给正确的事件侦听器并按照正确的顺序同时处理它们方面提供了帮助。 这些多线程问题通常很难处理，导致难以跟踪的错误，有时会导致应用程序完全失败。 当你有紧迫的期限时，你可能甚至不想关心这些问题。 Axon的代码已经过彻底测试，可以防止这些类型的错误。

Axon框架由许多模块（罐子）组成，这些模块提供了构建可扩展基础架构的工具和组件。 Axon核心模块为不同组件提供基本的API，以及为单个JVM应用程序提供解决方案的简单实现。 其他模块通过提供专门的构建模块来解决可伸缩性或高性能问题。

When to use Axon?
-----------------

并非所有的应用程序都能从Axon中受益。 简单的CRUD（创建，读取，更新，删除）预计不会扩展的应用程序可能不会从CQRS或Axon中受益。 但是，有许多应用程序可以从Axon中受益。

可能受益于CQRS和Axon的应用程序是那些显示一个或多个以下特征的应用程序：

-  该应用程序可能会在很长一段时间内以新功能进行扩展。 例如，一家网上商店可能会开始跟踪订单进度的系统。 在后期阶段，这可以通过库存信息进行扩展，以确保在销售物品时更新库存。 甚至以后，会计可能需要记录销售的财务统计数据等。虽然很难预测未来软件项目的演变情况，但大多数类型的应用程序都清晰地表示出来。

-  该应用程序具有较高的读写比。 这意味着数据只会被写入几次，并且会被读取多次。 由于查询的数据源与用于命令验证的数据源不同，因此可以优化这些数据源以进行快速查询。 重复数据不再是一个问题，因为事件在数据更改时发布。

-   该应用程序以多种不同格式呈现数据。 现在许多应用程序在网页上显示信息时不会停下来。 例如，某些应用程序会每月发送一封电子邮件通知用户发生的可能与其相关的更改。 搜索引擎是另一个例子。 他们使用与您的应用程序相同的数据，但以针对快速搜索进行优化的方式。 报告工具可将信息汇总到可显示数据随时间变化的报告中。 这也是相同数据的不同格式。 使用Axon，每个数据源可以实时或按计划独立更新。

-  当应用程序与不同的受众有明确的分离组件时，它也可以从Axon中受益。 这种应用的一个例子是在线商店。 员工将更新网站上的产品信息和可用性，同时客户下订单并查询其订单状态。 借助Axon，这些组件可以部署在不同的机器上，并使用不同的策略进行扩展。 他们使用事件保持最新状态，Axon将派发给所有订阅的组件，而不管他们部署在哪台机器上。

-  与其他应用程序集成可能是繁琐的工作。 使用命令和事件严格定义应用程序的API可以更容易地与外部应用程序集成。 任何应用程序都可以发送命令或侦听由应用程序生成的事件。

Getting started
===============

This section will explain how you can obtain the binaries for Axon to get started. There are currently two ways: either download the binaries from our website or configure a repository for your build system (Maven, Gradle, etc).

Download Axon
-------------

You can download the Axon Framework from our downloads page: [axonframework.org/download](http://www.axonframework.org/download).

This page offers a number of downloads. Typically, you would want to use the latest stable release. However, if you're eager to get started using the latest and greatest features, you could consider using the snapshot releases instead. The downloads page contains a number of assemblies for you to download. Some of them only provide the Axon library itself, while others also provide the libraries that Axon depends on. There is also a "full" zip file, which contains Axon, its dependencies, the sources and the documentation, all in a single download.

If you really want to stay on the bleeding edge of development, you can clone the Git repository: <git://github.com/AxonFramework/AxonFramework.git>, or visit <https://github.com/AxonFramework/AxonFramework> to browse the sources online.

Configure Maven
---------------

If you use maven as your build tool, you need to configure the correct dependencies for your project. Add the following code in your dependencies section:

``` xml
<dependency>
    <groupId>org.axonframework</groupId>
    <artifactId>axon-core</artifactId>
    <version>${axon.version}</version>
</dependency>
```

Most of the features provided by the Axon Framework are optional and require additional dependencies. We have chosen not to add these dependencies by default, as they would potentially clutter your project with artifacts you don't need.

Infrastructure requirements
---------------------------

Axon Framework doesn't impose many requirements on the infrastructure. It has been built and tested against Java 8, making that more or less the only requirement.

Since Axon doesn't create any connections or threads by itself, it is safe to run on an Application Server. Axon abstracts all asynchronous behavior by using `Executor`s, meaning that you can easily pass a container managed Thread Pool, for example. If you don't use a full blown Application Server (e.g. Tomcat, Jetty or a stand-alone app), you can use the `Executors` class or the Spring Framework to create and configure Thread Pools.

When you're stuck
-----------------

While implementing your application, you might run into problems, wonder about why certain things are the way they are, or have some questions that need an answer. The Axon Users mailing list is there to help. Just send an email to [axonframework@googlegroups.com](mailto:axonframework@googlegroups.com). Other users as well as contributors to the Axon Framework are there to help with your issues.

If you find a bug, you can report them at [github.com/AxonFramework/AxonFramework/issues](https://github.com/AxonFramework/AxonFramework/issues). When reporting an issue, please make sure you clearly describe the problem. Explain what you did, what the result was and what you expected to happen instead. If possible, please provide a very simple Unit Test (JUnit) that shows the problem. That makes fixing it a lot simpler.

Contributing to Axon Framework
==============================

Development on the Axon Framework is never finished. There will always be more features that we like to include in our framework to continue making development of scalable and extensible applications easier. This means we are constantly looking for help in developing our framework.

There are a number of ways in which you can contribute to the Axon Framework:

-   You can report any bugs, feature requests or ideas for improvements on our issue page: [github.com/AxonFramework/AxonFramework/issues](https://github.com/AxonFramework/AxonFramework/issues). All ideas are welcome. Please be as exact as possible when reporting bugs. This will help us reproduce and thus solve the problem faster.

-   If you have created a component for your own application that you think might be useful to include in the framework, send us a patch or a zip containing the source code. We will evaluate it and try to fit it in the framework. Please make sure code is properly documented using javadoc. This helps us to understand what is going on.

-   If you know of any other way you think you can help us, do not hesitate to send a message to the [Axon Framework mailing list](mailto:axonframework@googlegroups.com).

Commercial Support
==================

Axon Framework is open source and freely available for anyone to use. However, if you have specific requirements, or just want to be assured of someone to be standby to help you in case of trouble, AxonIQ provides several commercial support services for Axon Framework. These services include Training, Consultancy and Operational Support and are provided by the people that know Axon more than anyone else.

For more information about AxonIQ and its services, visit [axoniq.io](http://axoniq.io) or [axoniq.io/services](http://axoniq.io/services).

License information
===================

The Axon Framework and its documentation are licensed under the Apache License, Version 2.0. You may obtain a copy of the License at <http://www.apache.org/licenses/LICENSE-2.0>.

Unless required by applicable law or agreed to in writing, software distributed under the License is distributed on an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. See the [License](http://www.apache.org/licenses/LICENSE-2.0) for the specific language governing permissions and limitations under the License.

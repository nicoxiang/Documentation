==================================
最佳实践和建议
==================================

你可以在 `StackOverflow <http://stackoverflow.com/questions/tagged/autofac>`_ 上使用 ``autofac`` 标签或在 `google discussion group <https://groups.google.com/forum/#forum/autofac>`_ 寻求Autofac使用上的指导, 但下面的小贴士也能很好地帮到你.

总是从嵌套的生命周期中解析依赖
=================================================

Autofac设计的初衷是替你 :doc:`追踪和释放资源 <../lifetime/disposal>` . 为了能达到这样的效果, 确保这个长期运行的应用被分割到各工作单元 (请求或事务) 并且服务根据工作单元级别的生命周期作用域被解析. ASP.NET中的 :doc:`per-request lifetime  <../faq/per-request-scope>` 就是这种技术的一个示例.

使用模块构建配置
=====================================

:doc:`Autofac 模块 <../configuration/modules>` 可以用来搭建起容器配置的结构, 并且允许注入发布时配置. 我们应该考虑使用模块作为一种更灵活的方法, 而不是单独地使用 :doc:`XML 配置 <../configuration/xml>` . 模块搭配XML配置是一个两全的方案.

在委托注册中使用 As<T>() 
=====================================

Autofac可以从你注册组件时的表达式中推断出实现类型:

.. sourcecode:: csharp

    builder.Register(c => new Component()).As<IComponent>();

...使类型 ``Component`` 成为了组件的 ``LimitType`` . 下面的其他类型转换机制是等同的但并没有提供正确的 ``LimitType``:

.. sourcecode:: csharp

    // Works, but avoid this
    builder.Register(c => (IComponent)new Component());

    // Works, but avoid this
    builder.Register<IComponent>(c => new Component());

使用构造器注入
=========================

众所周知地, 我们常使用构造方法注入必需的依赖, 而使用属性注入可选依赖. 不过还有一种可选方案, 就是使用 `"Null Object" <http://en.wikipedia.org/wiki/Null_Object_pattern>`_ 或 `"Special Case" <http://martinfowler.com/eaaCatalog/specialCase.html>`_ 模式, 来为可选服务提供默认的, 不做任何事的实现. 它能防止在组件实现中可能出现的特殊代码 (如 ``if (Logger != null) Logger.Log("message");``).

使用关系类型, 而不是服务定位器
============================================

给予组件访问容器的能力, 将它储存在公有静态属性中, 或者让一个全局的 "IoC" 类上的类似 ``Resolve()`` 的方法可用违反了使用依赖注入的意图. 这样的设计与 `Service Locator <http://martinfowler.com/articles/injection.html#UsingAServiceLocator>`_ 模式更为类似.

如果组件有依赖于容器 (或生命周期), 看一下它们是如何使用容器来取得服务的, 作为替代地, 把这些服务加入到组件 (依赖注入到的) 的构造函数参数中.

对于需要实例化其他组件或者与容器有交互的情况, 使用 :doc:`关系类型 <../resolve/relationships>` 这种更先进的方式.

以最普通到最特殊的顺序注册组件
===============================================

Autofac默认会覆盖组件的注册. 这意味着一个应用可以注册它所有的默认组件, 然后读取一个相关的配置文件来覆盖掉任何部署环境自定义的组件.

使用性能分析工具进行性能检测
======================================

在进行任何性能优化或对可能存在的内存泄漏进行一些猜想之前, **总是去运行一个性能分析工具** 如 `SlimTune <http://code.google.com/p/slimtune/>`_, `dotTrace <http://www.jetbrains.com/profiler/>`_, 或 `ANTS <http://www.red-gate.com/products/dotnet-development/ants-performance-profiler/>`_ 来看下到底时间花在哪里. 它可能并不在你所想的那些地方.

一次注册, 多次解析
===========================

如果可以避免的话, 尽量不要在工作单元内注册组件; 注册一个组件比解析一个的代价大多了. 使用嵌套的生命周期作用域和合适的 :doc:`实例作用域 <../lifetime/instance-scope>` 来保证各个工作单元实例是相互独立的.

用Lambda表达式注册常用组件
================================================

如果你需要压榨Autofac的性能, 你最好的做法是找出最常创建的组件, 并将它们以表达式注册而不是以类型, 如:

.. sourcecode:: csharp

    builder.RegisterType<Component>();

变成:

.. sourcecode:: csharp

    builder.Register(c => new Component());

这样可以在 ``Resolve()`` 调用时获得10倍的性能提升, 但这只对出现在多对象关系图中的组件有效. 更多lambda组件的信息见 :doc:`注册章节文档 <../register/index>` .

把容器当成不可变的
=================================

尽管 Autofac 提供了一个 ``Update()`` 方法用来更新一个已存在容器中的注册组件, 不过大部分情况下它也只是为了向后兼容 Autofac 2.x. 在所有可能的情况下, 你应该避免更新一个容器, 作为替代地, 你应该在容器创建之前就把组件注册好.

如果你在容器创建后修改容器, 你将面临一些风险, 特别是当你刚开始使用容器的时候. 以下的列表并不囊括所有可能存在的风险, 包括:

- :doc:`自启动组件 <../lifetime/startup>` 将会已经在运行了, 有可能会使用到你在update时复写的注册组件. 这些自启动组件是不会重新运行的.
- 已解析的服务有可能会对那些额外创建的依赖, 存在错误的引用.
- 可释放的组件有可能已经被解析了, 并将会驻留, 一直到拥有它们的生命周期被释放 - 即使这新的注册组件已表明了该可释放组件不会被使用了.
- 订阅了生命周期事件的组件注册, 在更新后也许会订阅错误的事件 - 事件在更新时不会重新初始化.

如果是在没有办法, 你很有可能会需要 ``Update()`` 更新一个容器, 但还是尽量避免这种情况.

**为了替代更新容器这种做法, 可以考虑在子生命周期中注册这些更改的组件.** :doc:`在生命周期章节中有这种做法的示例. <../lifetime/working-with-scopes>`

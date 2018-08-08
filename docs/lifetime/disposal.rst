========
释放
========
一个工作单元中包含的资源 - 数据库连接, 事务, 认证sessions, 文件句柄等. - 当工作完成时应该被释放. .NET 提供了 ``IDisposable`` 接口来帮助释放的概念更加地明确.

如果想要释放特定对象的, 一些 IoC 容器需要被显式地告知, 通过类似 ``ReleaseInstance()`` 这样的方法. 这样就使得保证使用正确的释放语法变得非常困难.

* 从实现非可释放代码转换到实现可释放组件意味着需要修改客户端代码.
* 本来客户端代码使用共享实例时也许已经忽略了对象释放, 当切换到非共享实例时基本上也不会去做释放/清理.

:doc:`Autofac使用生命周期作用域解决了这些问题 <index>` , 作为一种在工作单元中释放所有已创建组件的方法.

.. sourcecode:: csharp

    using (var scope = container.BeginLifetimeScope())
    {
      scope.Resolve<DisposableComponent>().DoSomething();

      // Components for scope disposed here, at the
      // end of the 'using' statement when the scope
      // itself is disposed.
    }

当工作单元开始时生命周期作用域被创建, 当工作单元完成时, 嵌套的容器可以释放其中所有的实例.

注册组件
======================

Autofac可以自动释放一些组件, 但你也可以手动指定释放机制.

组件注册为 ``InstancePerDependency()`` (默认) 或者 ``InstancePerLifetimeScope()`` 的一些变形 (如, ``InstancePerMatchingLifetimeScope()`` 或 ``InstancePerRequest()``).

如果你有一个单例组件 (注册为 ``SingleInstance()``) **他们将会存在于容器的整个生命周期内**. 由于容器的生命周期通常就是应用的生命周期, 这就意味着组件直到应用结束将不会被释放.

自动释放
------------------

为了充分利用自动的明确性释放, 你的组件必须实现 ``IDisposable``. 你可以按需注册你的组件然后在组件解析的生命周期的结尾, 组件的 ``Dispose()`` 方法将会被调用.

.. sourcecode:: csharp

    var builder = new ContainerBuilder();
    builder.RegisterType<SomeDisposableComponent>();
    var container = builder.Build();
    // Create nested lifetime scopes, resolve
    // the component, and dispose of the scopes.
    // Your component will be disposed with the scope.

特定释放
------------------

如果你的组件不实现 ``IDisposable`` 但仍然需要在生命周期作用域的结尾完成一些释放工作, 你可以使用 :doc:`释放生命周期事件 <events>`.

.. sourcecode:: csharp

    var builder = new ContainerBuilder();
    builder.RegisterType<SomeComponent>()
           .OnRelease(instance => instance.CleanUp());
    var container = builder.Build();
    // Create nested lifetime scopes, resolve
    // the component, and dispose of the scopes.
    // Your component's "CleanUp()" method will be
    // called when the scope is disposed.

注意 ``OnRelease()`` 重写 ``IDisposable.Dispose()`` 的处理逻辑. 如果你的组件同时实现 ``IDisposable`` *并且* 需要一些其他的释放方法, 你需要手动在 ``OnRelease()`` 中调用 ``Dispose()`` 或者你可以更新你的类让释放方法在 ``Dispose()`` 内部被调用.

禁止释放
------------------

组件默认被容器拥有并且会在适当的时候被它释放掉. 想要禁止这个行为, 注册组件为拥有外部所有权:

.. sourcecode:: csharp

    builder.RegisterType<SomeComponent>().ExternallyOwned();

容器不会调用以外部所有注册的对象的 ``Dispose()`` 方法. 以这种方式注册的组件何时释放取决于你.

另一种可以禁用释放的方法是使用 :doc:`隐式关系类型 <../resolve/relationships>` ``Owned<T>`` 和 :doc:`被拥有的实例 <../advanced/owned-instances>`. 这种情况下, 在你的消费代码中并不是传入一个依赖 ``T`` , 而是一个依赖 ``Owned<T>``. 你的消费代码需要负责释放.

.. sourcecode:: csharp

    public class Consumer
    {
      private Owned<DisposableComponent> _service;

      public Consumer(Owned<DisposableComponent> service)
      {
        _service = service;
      }

      public void DoWork()
      {
        // _service is used for some task
        _service.Value.DoSomething();

        // Here _service is no longer needed, so
        // it is released
        _service.Dispose();
      }
    }

你可以在 :doc:`被拥有的实例章节 <../advanced/owned-instances>` 阅读更多关于 ``Owned<T>`` 的内容.

从生命周期作用域解析组件
=======================================

生命周期作用域通过调用 ``BeginLifetimeScope()`` 创建. 最简单的是在 ``using`` 块中. 使用生命周期作用域来解析你的组件然后当工作单元完成后释放作用域.

.. sourcecode:: csharp

    using (var lifetime = container.BeginLifetimeScope())
    {
      var component = lifetime.Resolve<SomeComponent>();
      // component, and any of its disposable dependencies, will
      // be disposed of when the using block completes
    }

注意使用 :doc:`Autofac 集成库 <../integration/index>` , 基础的工作单元生命周期作用域将会被创建并且自动替你释放. 例如,  Autofac's :doc:`ASP.NET MVC 集成 <../integration/mvc>`, 一个生命周期作用域将会在web请求开始时被创建然后所有的组件将会从中解析. 在请求结束时, 作用域会被自动释放 - 你自己不需要额外创建作用域. 如果你使用 :doc:`这些集成库的其中之一 <../integration/index>`, 你应该清楚它为你自动创建了什么作用域.

你可以 :doc:`阅读更多关于创建生命周期作用域的内容 <working-with-scopes>`.

子作用域不会被自动释放
===========================================

如果生命周期作用域本身实现 ``IDisposable``, 你创建的生命周期作用域 **不会自动地释放.** 如果你创建了一个生命周期作用域, 你需要负责调用它的 ``Dispose()`` 来释放资源并且触发组件的自动释放. 通过 ``using`` 块可以轻松完成这一步, 但如果你没有使用 ``using`` 创建作用域, 当你不再使用时别忘了释放它.

区分 **你创建的** 生命周期作用域和 **集成库替你创建** 的生命周期作用域是非常重要的. 你不用担心管理集成的作用域 (如 ASP.NET 请求作用域) - 这些会自动完成. 然而, 如果你手动创建了你自己的作用域, 你需要负责它的释放.

已提供的实例
==================

如果你已经提供了 :doc:`一个实例组件注册 <../register/registration>` 给Autofac, Autofac将会获取实例的所有权并处理它的释放.

.. sourcecode:: csharp

    // If you do this, Autofac will dispose of the StringWriter
    // instance when the container is disposed.
    var output = new StringWriter();
    builder.RegisterInstance(output)
           .As<TextWriter>();

如果你想要自己来控制实例的释放, 你需要将实例注册为 ``ExternallyOwned()``.

.. sourcecode:: csharp

    // Using ExternallyOwned means you will be responsible for
    // disposing the StringWriter instead of Autofac.
    var output = new StringWriter();
    builder.RegisterInstance(output)
           .As<TextWriter>()
           .ExternallyOwned();

更复杂的层级结构
====================

最简单且最推荐的资源管理方案, 综上, 是两层: 一个 '根' 容器和一个为了各个工作单元而从中创建的生命周期作用域. 如果有可能要使用更复杂的容器和组件层级结构, 使用 :doc:`标记的生命周期作用域 <working-with-scopes>`.
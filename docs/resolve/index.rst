==================
解析服务
==================

在 :doc:`注册完组件并暴露相应的服务后 <../register/index>`, 你可以从创建的容器或其子 :doc:`生命周期 <../lifetime/index>` 中解析服务. 让我们使用 ``Resolve()`` 方法来实现:

.. sourcecode:: csharp

    var builder = new ContainerBuilder();
    builder.RegisterType<MyComponent>().As<IService>();
    var container = builder.Build();

    using(var scope = container.BeginLifetimeScope())
    {
      var service = scope.Resolve<IService>();
    }

我们应该注意到示例是从生命周期中解析服务而并非直接从容器中 - 当然, 你也应该这么做.

    **有时在我们的应用中也许可以从根容器中解析组件, 然而这么做有可能会导致内存泄漏.** 推荐你总是从生命周期中解析组件, 以确保服务实例被妥善地释放和垃圾回收. 在 :doc:`控制范围和生命周期章节 <../lifetime/index>` 阅读更多相关内容.

解析服务时, Autofac 自动链接起服务所需的整个依赖链上不同层级并解析所有的依赖来完整地构建服务. 如果你有处理不当的 :doc:`循环依赖 <../advanced/circular-dependencies>` 或缺少了必需的依赖, 你将得到一个 ``DependencyResolutionException``.

如果你不清楚一个服务是否被注册了, 你可以通过 ``ResolveOptional()`` 或 ``TryResolve()`` 尝试解析:

.. sourcecode:: csharp

    // If IService is registered, it will be resolved; if
    // it isn't registered, the return value will be null.
    var service = scope.ResolveOptional<IService>();

    // If IProvider is registered, the provider variable
    // will hold the value; otherwise you can take some
    // other action.
    IProvider provider = null;
    if(scope.TryResolve<IProvider>(out provider))
    {
      // Do something with the resolved provider value.
    }


``ResolveOptional()`` 和 ``TryResolve()`` 本质上都只是保证某个特定的服务 *已成功注册*. 如果该组件已注册, 解析成功. 如果解析本身失败 (例如, 某些必需的依赖未注册), **你依然会得到 DependencyResolutionException**. 如果你不清楚服务解析本身是否会成功并需要在解析成功或失败时进行不同操作, 将 ``Resolve()`` 包裹在 try/catch 块中.

解析服务的更多章节:

.. toctree::

    parameters.rst
    relationships.rst

你也许有兴趣查看 :doc:`更多章节 <../advanced/index>` 来学习 :doc:`命名服务和键服务 <../advanced/keyed-services>`, :doc:`使用组件元数据 <../advanced/metadata>`, 和其他与解析相关的章节.
===============
生命周期事件
===============

Autofac暴露了一些能在实例生命周期多个阶段拦截到的事件. 这些事件在组件注册时被订阅 (或者也可以通过附加到 ``IComponentRegistration`` 接口.

.. contents::
  :local:

激活时
============

``激活时`` 事件在组件被使用前触发. 你可以:

* 修改实例成另一个或者把实例包裹在代理对象中
* :doc:`进行属性注入或方法注入 <../register/prop-method-injection>`
* 进行其他的初始化工作

在一些情况下, 比如用 ``RegisterType<T>()``, 注册的实体类会被用于类型解析并被 ``ActivatingEventArgs`` 使用. 例如, 下面的实例将会抛一个类型转换的异常:

.. sourcecode:: csharp

    builder.RegisterType<TConcrete>() // FAILS: will throw at cast of TInterfaceSubclass
           .As<TInterface>()          // to type TConcrete
           .OnActivating(e => e.ReplaceInstance(new TInterfaceSubclass()));

简单的解决方案是通过两步完成注册:

.. sourcecode:: csharp

    builder.RegisterType<TConcrete>().AsSelf();
    builder.Register<TInterface>(c => c.Resolve<TConcrete>())
           .OnActivating(e => e.ReplaceInstance(new TInterfaceSubclass()));

激活后
===========

``激活后`` 事件在组件完全构造完成后触发. 你可以完成一些应用级别的需要基于组件已构建完成为前提的任务 - *这种情况较少*.

释放时
=========

``释放时`` 事件替换了 :doc:`原始的组件释放行为 <disposal>`. 组件需要实现 ``IDisposable`` 接口且不是被标记为 ``ExternallyOwned()`` , 它的原始的组件释放行为会调用 ``Dispose()`` 方法. 而未实现 ``IDisposable`` 接口或是被标记为外部拥有的组件,  它的原始的组件释放行为不会做任何事. ``OnRelease`` 用提供的实现方法替换这一释放行为.
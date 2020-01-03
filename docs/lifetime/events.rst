===============
生命周期事件
===============

Autofac暴露了一些能在实例生命周期多个阶段拦截到的事件. 这些事件在组件注册时被订阅 (或者也可以通过附加到 ``IComponentRegistration`` 接口.

.. contents::
  :local:

OnPreparing
===========

``OnPreparing`` 事件在一个新的组件实例被需要的时候被触发, 它在 ``OnActivating`` 之前被触发.

这个事件可以用于指定一组参数信息, Autofac在创建一个新的组件实例会考虑用到这些参数.

这个事件的主要作用是模拟或拦截services, 那些Autofac在正常的组件激活时作为参数传入的services, 通过用自定义的参数给 ``PreparingEventArgs`` 的  ``Parameters`` 属性赋值.

.. tip:: 

  在使用这个事件赋值参数时, 考虑下是否在注册时定义这些会更合适, 使用 :doc:`parameter registration <../register/parameters>`.

OnActivating
============

``OnActivating`` 事件在组件被使用前触发. 你可以:

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

OnActivated
===========

``OnActivated`` 事件在组件完全构造完成后触发. 你可以完成一些应用级别的需要基于组件已构建完成为前提的任务 - *这种情况较少*.

OnRelease
=========

``OnRelease`` 事件替换了 :doc:`原始的组件释放行为 <disposal>`. 组件需要实现 ``IDisposable`` 接口且不是被标记为 ``ExternallyOwned()`` , 它的原始的组件释放行为会调用 ``Dispose()`` 方法. 而未实现 ``IDisposable`` 接口或是被标记为外部拥有的组件,  它的原始的组件释放行为不会做任何事. ``OnRelease`` 用提供的实现方法替换这一释放行为.
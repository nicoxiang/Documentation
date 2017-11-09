==============
实例作用域
==============

实例作用域决定了对于相同的服务解析出的实例如何在请求之间被共享. 注意你最好是了解 :doc:`生命周期作用域的概念 <index>` 这样才能更好地理解下面的内容.

当一个请求出现并且需要某个服务, Autofac可以返回一个单一实例 (单一实例作用域), 也可以返回一个新的实例 (每个依赖的作用域), 或者返回一个在某种上下文中的单一实例, 例如, 一个线程或者一个HTTP请求 (每个生命周期作用域).

这种行为既可以发生于显式调用 ``Resolve()`` 返回的实例, 也可以发生于容器为了满足另一个组件的依赖而在内部创建的实例.

.. note::
  选择一个正确的生命周期作用域可以帮助你避免 :doc:`被囚禁依赖 <captive-dependencies>` 和其他一些因为组件保持时间过长或不足造成的陷阱. 这取决于开发者如何对他们的每个组件作出正确的选择.

.. contents::
  :local:

每个依赖一个实例
=======================

在其他容器中也被称为 'transient' 或者 'factory' . 使用每个依赖的作用域, **对于一个服务每次请求都会返回一个唯一的实例.**

如果没有指定特定的选项, **它将是默认的**.

.. sourcecode:: csharp

    var builder = new ContainerBuilder();

    // This...
    builder.RegisterType<Worker>();

    // ...is the same as this:
    builder.RegisterType<Worker>().InstancePerDependency();

当你解析一个每个依赖一个实例的组件时, 你每次获得一个新的实例.

.. sourcecode:: csharp

    using(var scope = container.BeginLifetimeScope())
    {
      for(var i = 0; i < 100; i++)
      {
        // Every one of the 100 Worker instances
        // resolved in this loop will be brand new.
        var w = scope.Resolve<Worker>();
        w.DoWork();
      }
    }

单一实例
===============

它也被称为 '单例.' 使用单一实例作用域, **在根容器和所有嵌套作用域内所有的请求都将会返回同一个实例**.

.. sourcecode:: csharp

    var builder = new ContainerBuilder();
    builder.RegisterType<Worker>().SingleInstance();

当你解析一个单一实例组件时, 无论你从哪里请求它, 你都讲获得相同的实例.

.. sourcecode:: csharp

    // It's generally not good to resolve things from the
    // container directly, but for singleton demo purposes
    // we do...
    var root = container.Resolve<Worker>();

    // We can resolve the worker from any level of nested
    // lifetime scope, any number of times.
    using(var scope1 = container.BeginLifetimeScope())
    {
      for(var i = 0; i < 100; i++)
      {
        var w1 = scope1.Resolve<Worker>();
        using(var scope2 = scope1.BeginLifetimeScope())
        {
          var w2 = scope2.Resolve<Worker>();

          // root, w1, and w2 are always literally the
          // same object instance. It doesn't matter
          // which lifetime scope it's resolved from
          // or how many times.
        }
      }
    }

每个生命周期作用域一个实例
===========================

这种作用域应用于嵌套生命周期. **每个生命周期作用域的组件在每个嵌套的生命周期作用域中最多只会有一个单一实例.**

它对于单个独立工作单元特定的对象非常有用, 该工作单元需要嵌套额外的逻辑工作单元. 每个内嵌的生命周期作用域将会得到一个已注册依赖的新的实例.

.. sourcecode:: csharp

    var builder = new ContainerBuilder();
    builder.RegisterType<Worker>().InstancePerLifetimeScope();

当你解析每个生命周期作用域的组件时, 每个内嵌的作用域之内你都会得到一个单一的实例 (例如, 每个工作单元).

.. sourcecode:: csharp

    using(var scope1 = container.BeginLifetimeScope())
    {
      for(var i = 0; i < 100; i++)
      {
        // Every time you resolve this from within this
        // scope you'll get the same instance.
        var w1 = scope1.Resolve<Worker>();
      }
    }

    using(var scope2 = container.BeginLifetimeScope())
    {
      for(var i = 0; i < 100; i++)
      {
        // Every time you resolve this from within this
        // scope you'll get the same instance, but this
        // instance is DIFFERENT than the one that was
        // used in the above scope. New scope = new instance.
        var w2 = scope2.Resolve<Worker>();
      }
    }

每个匹配的生命周期作用域一个实例
====================================

这和上面的 '每个生命周期作用域一个实例' 的概念类似, 但可以对实例的共享有更加精准的控制.

当你创建一个嵌套的生命周期作用域时, 你可以给作用域 "打标签" 或者 "命名" . **每个匹配生命周期作用域的组件在每个名称匹配的嵌套生命周期作用域中最多只会有一个单一实例.** 这就允许了你创建一系列 "有作用域的单例" , 其他嵌套的生命周期可以在不声明一个共享实例的情况下共享这种组件的实例.

它对于单个独立工作单元特定的对象非常有用, 例如, 一个 HTTP 请求, 它作为一个嵌套的生命周期在每个工作单元内都会被创建. 如果一个嵌套的生命周期每个 HTTP 请求创建一次, 那么任何每个生命周期作用域的组件在每个 HTTP 请求内都将只有一个实例. (下面还有更多详细解释.)

大多数的应用中, 只需一层的容器嵌套就足以表示工作单元的作用域. 如果需要更多的嵌套层级 (例如, 全局->请求->事务) 组件可以考虑使用标签来在层级关系中特定的层级共享.

.. sourcecode:: csharp

    var builder = new ContainerBuilder();
    builder.RegisterType<Worker>().InstancePerMatchingLifetimeScope("myrequest");

当你开始一个生命周期时, 提供的标签值和它就关联起来了. **如果你尝试从一个名称并不匹配的生命周期中解析一个每个匹配生命周期作用域的组件你会得到一个异常.**

.. sourcecode:: csharp

    // Create the lifetime scope using the tag.
    using(var scope1 = container.BeginLifetimeScope("myrequest"))
    {
      for(var i = 0; i < 100; i++)
      {
        var w1 = scope1.Resolve<Worker>();
        using(var scope2 = scope1.BeginLifetimeScope())
        {
          var w2 = scope2.Resolve<Worker>();

          // w1 and w2 are always the same object
          // instance because the component is per-matching-lifetime-scope,
          // so it's effectively a singleton within the
          // named scope.
        }
      }
    }

    // Create another lifetime scope using the tag.
    using(var scope3 = container.BeginLifetimeScope("myrequest"))
    {
      for(var i = 0; i < 100; i++)
      {
        // w3 will be DIFFERENT than the worker resolved in the
        // earlier tagged lifetime scope.
        var w3 = scope3.Resolve<Worker>();
        using(var scope4 = scope3.BeginLifetimeScope())
        {
          var w4 = scope4.Resolve<Worker>();

          // w3 and w4 are always the same object because
          // they're in the same tagged scope, but they are
          // NOT the same as the earlier workers (w1, w2).
        }
      }
    }

    // You can't resolve a per-matching-lifetime-scope component
    // if there's no matching scope.
    using(var noTagScope = container.BeginLifetimeScope())
    {
      // This throws an exception because this scope doesn't
      // have the expected tag and neither does any parent scope!
      var fail = noTagScope.Resolve<Worker>();
    }

每个请求一个实例
====================

一些应用本身就适合 "request" 类型的语法, 例如 ASP.NET :doc:`web forms <../integration/webforms>` 和 :doc:`MVC <../integration/mvc>` 应用. 在这些应用类型中, 拥有一组 "每个请求一个" 的单例非常有用."

**每个请求一个实例建立于每个匹配生命周期一个实例之上** , 通过另外提供了一个众所周知的作用域标签, 一个方便注册的方法, 和对一些普通应用类型的集成. 而在这些背后, 其实它本质上还是一个每个匹配生命周期一个实例.

这就意味着如果你试图解析一个注册为每个请求一个实例的组件但是并没有当前请求... 你将会得到一个异常.

:doc:`这边有一章问答详细的介绍了如何使用每个请求的生命周期. <../faq/per-request-scope>`

.. sourcecode:: csharp

    var builder = new ContainerBuilder();
    builder.RegisterType<Worker>().InstancePerRequest();

**ASP.NET Core 使用的是每个生命周期一个实例而不是每个请求一个实例.** 详情见 :doc:`ASP.NET Core 集成 <../integration/aspnetcore>`.

每次被拥有一个实例
==================

`Owned<T>` :doc:`隐式关系类型 <../resolve/relationships>` 创建了一个嵌套的生命周期作用域. 使用每次被拥有一个实例注册, 可以把该依赖的作用域绑定到拥有它的实例上.

.. sourcecode:: csharp

    var builder = new ContainerBuilder();
    builder.RegisterType<MessageHandler>();
    builder.RegisterType<ServiceForHandler>().InstancePerOwned<MessageHandler>();

示例中 ``ServiceForHandler`` 服务将会绑定上它拥有的 ``MessageHandler`` 示例的生命周期.

.. sourcecode:: csharp

    using(var scope = container.BeginLifetimeScope())
    {
      // The message handler itself as well as the
      // resolved dependent ServiceForHandler service
      // is in a tiny child lifetime scope under
      // "scope." Note that resolving an Owned<T>
      // means YOU are responsible for disposal.
      var h1 = scope.Resolve<Owned<MessageHandler>>();
      h1.Dispose();
    }

线程作用域
============

Autofac可以强制使绑定到一个线程的对象无法成为绑定到另一线程的组件的依赖. 如果没有合适的方法, 你可以使用生命周期作用域完成这一操作.

.. sourcecode:: csharp

    var builder = new ContainerBuilder();
    builder.RegisterType<MyThreadScopedComponent>()
           .InstancePerLifetimeScope();
    var container = builder.Build();

这样, 每个线程就有了它各自的生命周期作用域:

.. sourcecode:: csharp

    void ThreadStart()
    {
      using (var threadLifetime = container.BeginLifetimeScope())
      {
        var thisThreadsInstance = threadLifetime.Resolve<MyThreadScopedComponent>();
      }
    }

**IMPORTANT: 在这种多线程场景中, 你必须得注意父级作用域不能在派生出的线程下被释放了.** 否则你就会碰到一个很糟糕的情况, 如果你派生了一个子线程并在其中释放了父级作用域, 组件就不能解析了.

每个线程通过执行 ``ThreadStart()`` 将会得到它独立的 ``MyThreadScopedComponent`` 实例- 这个实例在生命周期作用域本质上是一个 "单例" . 因为被划分作用域的实例不会暴露给外在的作用域, 很容易就能保证线程组件是独立的.

你可以通过传入一个 ``ILifetimeScope`` 参数把父级的生命周期作用域注入到派生线程的代码中. Autofac会自动注入当前的生命周期作用域然后你就可以从其中创建嵌套的作用域了.

.. sourcecode:: csharp

    public class ThreadCreator
    {
      private ILifetimeScope _parentScope;

      public ThreadCreator(ILifetimeScope parentScope)
      {
        this._parentScope = parentScope;
      }

      public void ThreadStart()
      {
        using (var threadLifetime = this._parentScope.BeginLifetimeScope())
        {
          var thisThreadsInstance = threadLifetime.Resolve<MyThreadScopedComponent>();
        }
      }
    }

如果你想要指定的更严格一些, 可以使用每个匹配的生命周期作用域一个实例 (见上) 在内在生命周期中和线程作用域的组件连通 (它们仍然可以拥有外部容器注入的工厂/单例组件的依赖.) 这样做的结果如下:

.. image:: threadedcontainers.png

图表中的 'contexts' 指的是用 ``BeginLifetimeScope()`` 创建的容器.

===========================
隐式关系类型
===========================

为了支持 :doc:`组件和服务 <../glossary>` 之间的特殊关系, Autofac支持隐式地自动解析特定服务. 想要使用这种关系, 只需正常地注册你的组件, 但是在使用组件时改变构造方法的参数或者在 ``Resolve()`` 调用时改变被解析的类型, 这样就能引入特定的关系类型.

例如, 当Autofac正在注入一个 ``IEnumerable<ITask>`` 类型的构造方法参数, 它 **不会** 寻找提供 ``IEnumerable<ITask>`` 的组件. 而是寻找 ``ITask`` 的所有实现并且注入它们所有.

(不必担心 - 下面有很多示例展示了多种类别的使用方法和含义.)

注意: 如果想要覆盖这种默认的行为, *我们依然可以注册这些类型的显式实现*.

[本文基于 Nick Blumhardt 的博文 `The Relationship Zoo <http://nblumhardt.com/2010/01/the-relationship-zoo/>`_.]


支持的关系类型
============================

下表简述了Autofac中各种支持的关系类型并且展示了你可以用来使用它们的 .NET 类型. 每种关系类型在后面都有一个更详细的描述和用例.

=================================================== ==================================================== =======================================================
Relationship                                        Type                                                 Meaning
=================================================== ==================================================== =======================================================
*A* needs *B*                                       ``B``                                                Direct Dependency
*A* needs *B* at some point in the future           ``Lazy<B>``                                          Delayed Instantiation
*A* needs *B* until some point in the future        ``Owned<B>``                                         :doc:`Controlled Lifetime <../advanced/owned-instances>`
*A* needs to create instances of *B*                ``Func<B>``                                          Dynamic Instantiation
*A* provides parameters of types *X* and *Y* to *B* ``Func<X,Y,B>``                                      Parameterized Instantiation
*A* needs all the kinds of *B*                      ``IEnumerable<B>``, ``IList<B>``, ``ICollection<B>`` Enumeration
*A* needs to know *X* about *B*                     ``Meta<B>`` and ``Meta<B,X>``                        :doc:`Metadata Interrogation <../advanced/metadata>`
*A* needs to choose *B* based on *X*                ``IIndex<X,B>``                                      :doc:`Keyed Service <../advanced/keyed-services>` Lookup
=================================================== ==================================================== =======================================================

.. contents:: Relationship Type Details
  :local:
  :depth: 1


直接依赖 (B)
---------------------
*直接依赖* 关系是支持的最基本的关系 - 组件 ``A`` 需要服务 ``B``. 通过基础的构造方法和属性注入自动完成:

.. sourcecode:: csharp

    public class A
    {
      public A(B dependency) { ... }
    }

注册 ``A`` 和 ``B`` 组件, 然后解析:

.. sourcecode:: csharp

    var builder = new ContainerBuilder();
    builder.RegisterType<A>();
    builder.RegisterType<B>();
    var container = builder.Build();

    using(var scope = container.BeginLifetimeScope())
    {
      // B is automatically injected into A.
      var a = scope.Resolve<A>();
    }


延迟实例化 (Lazy<B>)
-------------------------------
*延迟依赖* 直到它第一次使用时才会被实例化. 通常用于当依赖并非频繁使用, 或者构造需要较大代价时. 想要使用延迟依赖, 在 ``A`` 的构造方法中使用 ``Lazy<B>`` :

.. sourcecode:: csharp

    public class A
    {
      Lazy<B> _b;

      public A(Lazy<B> b) { _b = b }

      public void M()
      {
          // The component implementing B is created the
          // first time M() is called
          _b.Value.DoSomething();
      }
    }

如果你有一个延迟依赖, 同时你也需要它的元数据, 可以使用 ``Lazy<B,M>`` 更不是更长的 ``Meta<Lazy<B>, M>``.


可控生命周期 (Owned<B>)
------------------------------
*被拥有依赖* 当它不再被需要时可以被它的所有者释放. 被拥有依赖通常对应了它所依赖组件执行的某些工作单元.

使用实现 ``IDisposable`` 的组件时, 关系类型非常有意思. :doc:`Autofac在生命周期作用域最后自动释放disposable的组件 <../lifetime/disposal>` , 但这也许会意味着一个组件会被持有过长时间; 或者你也许会想要自己来控制对象的释放. 这种情况下, 你可以使用 *被拥有依赖*.

.. sourcecode:: csharp

    public class A
    {
      Owned<B> _b;

      public A(Owned<B> b) { _b = b; }

      public void M()
      {
          // _b is used for some task
          _b.Value.DoSomething();

          // Here _b is no longer needed, so
          // it is released
          _b.Dispose();
      }
    }

在内部, Autofac创建一个小型的生命周期作用域, 在这个作用域内 ``B`` 服务被解析, 并且当你调用 ``Dispose()`` 时, 生命周期被释放. 这意味着 ``B`` 的释放将会 *同样释放它的依赖* 除非这些依赖是共享的 (例如, 单例).

这也意味着如果你有一个 ``InstancePerLifetimeScope()`` 注册并且把它作为 ``Owned<B>`` 解析, 你得到的实例和在同一生命周期解析出来的其他实例将会是不同的. 看下下面的示例:

.. sourcecode:: csharp

    var builder = new ContainerBuilder();
    builder.RegisterType<A>().InstancePerLifetimeScope();
    builder.RegisterType<B>().InstancePerLifetimeScope();
    var container = builder.Build();

    using(var scope = container.BeginLifetimeScope())
    {
      // Here we resolve a B that is InstancePerLifetimeScope();
      var b1 = scope.Resolve<B>();
      b1.DoSomething();

      // This will be the same as b1 from above.
      var b2 = scope.Resolve<B>();
      b2.DoSomething();

      // The B used in A will NOT be the same as the others.
      var a = scope.Resolve<A>();
      a.M();
    }

这样设计是因为你肯定不会想要在组件外部还要去做 ``B`` 的释放. 然而, 如果你不清楚这个的话它可能会带来一些疑惑.

如果你更想要随时控制 ``B`` 的释放, :doc:`以 ExternallyOwned() 注册B <../lifetime/disposal>`.


动态实例化 (Func<B>)
-------------------------------
使用 *自动生成工厂* 可以让你无需绑定组件到Autofac就能高效的调用 ``Resolve<T>()`` . 如果你需要创建不止一个所提供服务的实例, 或者如果你不确定是否你需要一个服务并且希望在运行时才去作出选择, 可以使用这种关系类型. This relationship is also useful in cases like :doc:`WCF integration <../integration/wcf>` where you need to create a new service proxy after faulting the channel.

使用这种关系类型, **生命周期对实例化的影响是无法改变的**. 如果你以 ``InstancePerDependency()`` 注册一个对象并且多次调用 ``Func<B>`` 方法, 你每次都会得到一个新的实例. 然而, 如果你以 ``SingleInstance()`` 注册一个对象并且多次调用 ``Func<B>`` 来解析对象, 你 *每次只会得到一个相同的对象*.

这种关系类型的示例如下:

.. sourcecode:: csharp

    public class A
    {
      Func<B> _b;

      public A(Func<B> b) { _b = b; }

      public void M()
      {
          var b = _b();
          b.DoSomething();
      }
    }


带参数实例化 (Func<X, Y, B>)
-------------------------------------------
你可以使用一个 *自动生成工厂* 来传参给解析方法. 这是区别于 :doc:`注册时传参 <../register/parameters>` 或 :doc:`手动解析时传参 <../resolve/parameters>` 的另一种替代方法:

.. sourcecode:: csharp

    public class A
    {
        Func<int, string, B> _b;

        public A(Func<int, string, B> b) { _b = b }

        public void M()
        {
            var b = _b(42, "http://hel.owr.ld");
            b.DoSomething();
        }
    }

在内部, Autofac会把Func的入参作为类型参数. 这就意味着 **自动生成工厂在入参列表不能有重复的类型.** 例如, 假设你有如下类型:

.. sourcecode:: csharp

    public class DuplicateTypes
    {
      public DuplicateTypes(int a, int b, string c)
      {
        // ...
      }
    }

你也许想要注册这个类型然后给它写了一个自动生成工厂. *你依然能解析Func, 但是你不能执行它.*

.. sourcecode:: csharp

    var func = scope.Resolve<Func<int, int, string, DuplicateTypes>>();

    // Throws a DependencyResolutionException:
    var obj = func(1, 2, "three");

在这种参数按类型匹配的松耦合的场景下, 你不必完全清楚指定对象构造方法的参数的顺序. 而如果你想要做重复参数的那种情况, 你需要自定义委托:

.. sourcecode:: csharp

    public delegate DuplicateTypes FactoryDelegate(int a, int b, string c);

使用 ``RegisterGeneratedFactory()`` 注册委托:

.. sourcecode:: csharp

    builder.RegisterType<DuplicateTypes>();
    builder.RegisterGeneratedFactory<FactoryDelegate>(new TypedService(typeof(DuplicateTypes)));

这样方法就能work了:

.. sourcecode:: csharp

    var func = scope.Resolve<FactoryDelegate>();
    var obj = func(1, 2, "three");

另一个选择是 :doc:`委托工厂, 你可以查看高级章节 <../advanced/delegate-factories>`.

如果你依然决定使用内置的自动生成工厂 (``Func<X, Y, B>``) 解析一个工厂, 并且保证每种类型入参只有一个, 它还是能正常运行的, 但是构造方法中相同的类型的参数都会是相同的值.

.. sourcecode:: csharp

    var func = container.Resolve<Func<int, string, DuplicateTypes>>();

    // This works and is the same as calling
    // new DuplicateTypes(1, 1, "three")
    var obj = func(1, "three");

你可以 :doc:`在高级章节 <../advanced/delegate-factories>` 阅读更多关于委托工厂的内容和 ``RegisterGeneratedFactory()`` 方法.

使用这种关系类型和使用委托工厂, **生命周期对实例化的影响是无法改变的** . 如果你以 ``InstancePerDependency()`` 注册一个对象并且多次调用 ``Func<X, Y, B>`` , 你每次都会得到一个新的实例. 然而, 如果你以 ``SingleInstance()`` 注册一个对象并且多次调用 ``Func<X, Y, B>`` 来解析对象, 你 *每次只会得到一个相同的对象, 无论你是否传入了不同的参数.* 只是传入不同的参数无法覆盖掉生命周期造成的影响.


可遍历型 (IEnumerable<B>, IList<B>, ICollection<B>)
------------------------------------------------------
*可遍历类型* 的依赖提供了相同服务 (接口) 的多个实现. 它在例如消息处理程序中非常有用, 当一个消息传入时, 多个注册成功的处理程序都会处理这个消息.

假设有个依赖接口定义如下:

.. sourcecode:: csharp

    public interface IMessageHandler
    {
      void HandleMessage(Message m);
    }

接下来, 你有一个依赖的消费者(使用依赖的地方), 在那里依赖都已经被注册了并且消费者需要所有已被注册的依赖:

.. sourcecode:: csharp

    public class MessageProcessor
    {
      private IEnumerable<IMessageHandler> _handlers;

      public MessageProcessor(IEnumerable<IMessageHandler> handlers)
      {
        this._handlers = handlers;
      }

      public void ProcessMessage(Message m)
      {
        foreach(var handler in this._handlers)
        {
          handler.HandleMessage(m);
        }
      }
    }

使用隐式可遍历关系类型可以轻松完成. 只要注册所有的依赖和消费者, 然后当你解析消费者时, *所有一系列匹配的依赖* 都会被作为可遍历型解析.

.. sourcecode:: csharp

    var builder = new ContainerBuilder();
    builder.RegisterType<FirstHandler>().As<IMessageHandler>();
    builder.RegisterType<SecondHandler>().As<IMessageHandler>();
    builder.RegisterType<ThirdHandler>().As<IMessageHandler>();
    builder.RegisterType<MessageProcessor>();
    var container = builder.Build();

    using(var scope = container.BeginLifetimeScope())
    {
      // When processor is resolved, it'll have all of the
      // registered handlers passed in to the constructor.
      var processor = scope.Resolve<MessageProcessor>();
      processor.ProcessMessage(m);
    }

**可遍历关系类型如果容器中没有一个匹配的已注册组件, 那么将会返回空集合.** 意思是, 上面示例如果你不注册任何 ``IMessageHandler`` 的实现, 将会抛错:

.. sourcecode:: csharp

    // This throws an exception - none are registered!
    scope.Resolve<IMessageHandler>();

*然而, 下面还是能work的:*

.. sourcecode:: csharp

    // This returns an empty list, NOT an exception:
    scope.Resolve<IEnumerable<IMessageHandler>>();

所以当你使用这种关系类型注入些东西时, 也许会觉得 "我懂了, 可能会得到一个null值" . 然而, 你只是会得到一个空列表.

Metadata Interrogation (Meta<B>, Meta<B, X>)
--------------------------------------------
The :doc:`Autofac metadata feature <../advanced/metadata>` lets you associate arbitrary data with services that you can use to make decisions when resolving. If you want to make those decisions in the consuming component, use the ``Meta<B>`` relationship, which will provide you with a string/object dictionary of all the object metadata:

.. sourcecode:: csharp

    public class A
    {
      Meta<B> _b;

      public A(Meta<B> b) { _b = b; }

      public void M()
      {
        if (_b.Metadata["SomeValue"] == "yes")
        {
          _b.Value.DoSomething();
        }
      }
    }

You can use :doc:`strongly-typed metadata <../advanced/metadata>` as well, by specifying the metadata type in the ``Meta<B, X>`` relationship:

.. sourcecode:: csharp

    public class A
    {
      Meta<B, BMetadata> _b;

      public A(Meta<B, BMetadata> b) { _b = b; }

      public void M()
      {
        if (_b.Metadata.SomeValue == "yes")
        {
          _b.Value.DoSomething();
        }
      }
    }

If you have a lazy dependency for which you also need metadata, you can use ``Lazy<B,M>`` instead of the longer ``Meta<Lazy<B>, M>``.

Keyed Service Lookup (IIndex<X, B>)
-----------------------------------
In the case where you have many of a particular item (like the ``IEnumerable<B>`` relationship) but you want to pick one based on :doc:`service key <../advanced/keyed-services>`, you can use the ``IIndex<X, B>`` relationship. First, register your services with keys:

.. sourcecode:: csharp

    var builder = new ContainerBuilder();
    builder.RegisterType<DerivedB>().Keyed<B>("first");
    builder.RegisterType<AnotherDerivedB>().Keyed<B>("second");
    builder.RegisterType<A>();
    var container = builder.Build();

Then consume the ``IIndex<X, B>`` to get a dictionary of keyed services:

.. sourcecode:: csharp

    public class A
    {
      IIndex<string, B> _b;

      public A(IIndex<string, B> b) { _b = b; }

      public void M()
      {
        var b = this._b["first"];
        b.DoSomething();
      }
    }


Composing Relationship Types
============================

Relationship types can be composed, so:

.. sourcecode:: csharp

    IEnumerable<Func<Owned<ITask>>>

Is interpreted correctly to mean:

 * All implementations, of
 * Factories, that return
 * :doc:`Lifetime-controlled<../advanced/owned-instances>`
 * ``ITask`` services

Relationship Types and Container Independence
=============================================
The custom relationship types in Autofac based on standard .NET types don't force you to bind your application more tightly to Autofac. They give you a programming model for container configuration that is consistent with the way you write other components (vs. having to know a lot of specific container extension points and APIs that also potentially centralize your configuration).

For example, you can still create a custom ``ITaskFactory`` in your core model, but provide an ``AutofacTaskFactory`` implementation based on ``Func<Owned<ITask>>`` if that is desirable.

Note that some relationships are based on types that are in Autofac (e.g., ``IIndex<X, B>``). Using those relationship types do tie you to at least having a reference to Autofac, even if you choose to use a different IoC container for the actual resolution of services.

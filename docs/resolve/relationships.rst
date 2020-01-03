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
*owned dependency* 当它不再被需要时可以被它的所有者释放. Owned dependencies通常和它所依赖组件执行的某些工作单元相对应.

使用实现 ``IDisposable`` 的组件时, 类之间的关系非常有趣. :doc:`Autofac在生命周期作用域最后自动释放disposable的组件 <../lifetime/disposal>` , 但这也许会意味着一个组件会被持有过长时间; 或者你也许会想要自己来控制对象的释放. 这种情况下, 你可以使用 *owned dependency*.

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
使用 *自动生成工厂* 可以让你无需绑定组件到Autofac就能高效的调用 ``Resolve<T>()`` . 如果你需要创建不止一个所提供服务的实例, 或者如果, 可以使用这种关系类型. This relationship is also useful in cases like :doc:`WCF integration <../integration/wcf>` where you need to create a new service proxy after faulting the channel.

使用这种关系类型, **生命周期对实例化的影响是无法改变的**. 如果你以 ``InstancePerDependency()`` 注册一个对象并且多次调用 ``Func<B>`` 方法, 你每次都会得到一个新的实例. 然而, 如果你以 ``SingleInstance()`` 注册一个对象并且多次调用 ``Func<B>`` 来解析对象, 你 *每次只会得到一个相同的对象*.

使用 *自动生成工厂* 可以让你在程序控制流中以编码的形式解析一个新的 `B`, 不需要直接依赖于Autofac库. 在下面这些情况下使用这种关系:

* 基于给定的服务你需要创建超过一个实例.
* 在设置service的时候你想要一些特殊的控制.
* 你不确定是否你需要一个服务并且希望在运行时才去作出选择.

This relationship is also useful in cases like :doc:`WCF integration <../integration/wcf>` where you need to create a new service proxy after faulting the channel.

``Func<B>`` 运作的时候和调用 ``Resolve<B>()`` 类似. 这意味着它不会局限于作用在对象的无参构造函数那样 - 它会绑定构造函数参数, 做属性注入, 并且会遵循 ``Resolve<B>()`` 一样的整个生命周期.

进一步地来说, 生命周期也同样有效. 如果你将对象注册为 ``InstancePerDependency()`` 并且调用 ``Func<B>`` 多次, 你将每次都得到一个新的实例; 如果你将对象注册为 ``SingleInstance()`` 并且调用 ``Func<B>`` 去解析对象多次, 你 *每次都会得到相同的实例*.

这种关系类型的示例如下:

.. sourcecode:: csharp

    public class B
    {
      public B() {}
      
      public void DoSomething() {}
    }

    public class A
    {
      Func<B> _newB;

      public A(Func<B> b) { _newB = b; }

      public void M()
      {
          var b = _newB();
          b.DoSomething();
      }
    }


带参数实例化 (Func<X, Y, B>)
-------------------------------------------
当对象的构造方法需要额外的参数时, 你同样可以使用一个 *自动生成工厂* 在为它提供参数. 因为 ``Func<B>`` 关系类似于 ``Resolve<B>()``, 那么同样地, ``Func<X, Y, B>`` 关系类似于调用 ``Resolve<B>(TypedParameter.From<X>(x), TypedParameter.From<Y>(y))`` - 既一种有参数的解析操作.
这是除了 :doc:`注册时传参 <../register/parameters>` 或 :doc:`手动解析时传参 <../resolve/parameters>` 之外的另一种替代方法:

.. sourcecode:: csharp

    public class B
    {
      public B(string someString, int id) {}
      
      public void DoSomething() {}
    }

    public class A
    {
        Func<int, string, B> _newB;

        public A(Func<int, string, B> b) { _newB = b }

        public void M()
        {
            var b = _newB(42, "http://hell.owor.ld");
            b.DoSomething();
        }
    }

注意因为我们是在解析实例而不是直接调用构造方法, 我们在声明参数的时候不一定要保持和构造方法定义参数的顺序一样, 我们也不需要提供构造方法中列出的 *所有* 参数. 
如果一些构造方法参数可以从生命周期中解析出来, 那么参数就可以从 ``Func<X, Y, B>`` 定义的签名中省略. 你只 *需要* 列出生命周期无法解析出的类型.

另一种做法是, 你可以使用这种方式覆盖掉已经从容器中解析出来的构造方法的参数, 转而使用一个已存在的实例.

示例:

.. sourcecode:: csharp

    //Suppose that P, Q & R are all registered with the Autofac Container.
    public class B
    {
      public B(int id, P peaDependency, Q queueDependency, R ourDependency) {}
      
      public void DoSomething() {}
    }

    public class A
    {
        Func<int, P, B> _newB;

        public A(Func<int, P, B> bFactory) { _newB = bFactory }

        public void M(P existingPea)
        {
            // The Q and R will be resolved by Autofac, but P will be existingPea instead.
            var b = _newB(42, existingPea);
            b.DoSomething();
        }
    }

在内部, Autofac仅仅基于类型去决定构造方法参数的值. 
结果就造成了 **自动生成工厂(auto-generated function factories)在入参列表中不能有重复的类型Internally**.

当使用这种关系的时候 **生命周期对实例化的影响是无法改变的**, 就像使用 :doc:`delegate factories <../advanced/delegate-factories>`.
如果你将对象注册为 ``InstancePerDependency()`` 并且调用 ``Func<X, Y, B>`` 多次, 你将每次都得到一个新的实例.
然而, 如果你将对象注册为 ``SingleInstance()`` 并且调用 ``Func<X, Y, B>`` 去解析对象多次, 你将 *每次都得到相同的实例, 无论你是否传入了不同的参数..* 只是传入不同的参数无法覆盖掉生命周期造成的影响.

如上所述, ``Func<X, Y, B>`` 把参数当做 ``TypedParameter`` 所以在入参列表中你不能有重复的类型. 例如, 假设你有这样的一个类:

.. sourcecode:: csharp

    public class DuplicateTypes
    {
      public DuplicateTypes(int a, int b, string c)
      {
        // ...
      }
    }

你也许希望注册这个类型, 并拥有一个它的自动生成工厂. *你依然能解析Func, 但是你不能执行它.*

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

如果你依然决意要使用内置的自动生成工厂 (``Func<X, Y, B>``) , 并且保证每种类型入参只有一个, 它还是能正常运行的, 但是构造方法中相同的类型的参数都会是相同的值.

.. sourcecode:: csharp

    var func = container.Resolve<Func<int, string, DuplicateTypes>>();

    // This works and is the same as calling
    // new DuplicateTypes(1, 1, "three")
    var obj = func(1, "three");

你可以 :doc:`在高级章节 <../advanced/delegate-factories>` 阅读更多关于委托工厂的内容和 ``RegisterGeneratedFactory()`` 方法.


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

元数据审查(Metadata Interrogation (Meta<B>, Meta<B, X>))
-------------------------------------------------------------
:doc:`Autofac元数据功能 <../advanced/metadata>` 允许你将任何数据和服务连接起来, 在解析时就可以用这些数据做出选择. 如果想在消费的组件中用这些数据做选择, 使用 ``Meta<B>`` 关联, 它提供给你一个所有元数据对象的string/object dictionary:

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

你也可以通过在 ``Meta<B, X>`` 关联中指定元数据类型来使用 :doc:`强类型元数据 <../advanced/metadata>` :

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

如果你有个延迟依赖也需要元数据, 可以使用 ``Lazy<B,M>`` 而不是更长的 ``Meta<Lazy<B>, M>``.

键控服务的查找(Keyed Service Lookup (IIndex<X, B>))
-----------------------------------------------------
有时, 对于某个特定的服务你有很多实现 (类似 ``IEnumerable<B>`` 关系) 但你想要根据某个 :doc:`服务键 <../advanced/keyed-services>` 挑选其一, 你可以使用 ``IIndex<X, B>`` 关系. 首先, 通过键注册你的服务:

.. sourcecode:: csharp

    var builder = new ContainerBuilder();
    builder.RegisterType<DerivedB>().Keyed<B>("first");
    builder.RegisterType<AnotherDerivedB>().Keyed<B>("second");
    builder.RegisterType<A>();
    var container = builder.Build();

然后消费 ``IIndex<X, B>`` 来得到一个键控服务的字典集合:

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


组合关系类型
============================

关系类型是可以被组合起来的, 因此:

.. sourcecode:: csharp

    IEnumerable<Func<Owned<ITask>>>

可以解释为:

 * All implementations, of
 * Factories, that return
 * :doc:`Lifetime-controlled<../advanced/owned-instances>`
 * ``ITask`` services

关系类型和容器独立性
=============================================
Autofac中基于标准.NET类型的自定义关系类型不会强迫你把应用和Autofac过于紧密地绑定在一起. 它们使你可以通过编程模式来进行容器配置, 就像你写其他普通组件一样 (相比之下, 你需要知道很多容器扩展点和API, 通过这些也就潜在地集中化了你的配置).

例如, 你仍然可以在核心的model中创建一个自定义的 ``ITaskFactory`` , 但是可以在需要时提供一个基于 ``Func<Owned<ITask>>`` 的 ``AutofacTaskFactory`` 实现.

注意有些关系类型基于Autofac中的类型 (如 ``IIndex<X, B>``). 使用这些关系类型的确会束缚你, 至少你得有一个对于Autofac的引用, 即使你在实际解析服务时选择一个不同的Ioc容器.

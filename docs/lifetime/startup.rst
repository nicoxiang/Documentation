=======================
启动时运行代码
=======================

Autofac允许容器创建完成时通知组件或者自动激活组件.

这里有三种自动激活机制可用:

* 可启动组件
* 自动激活组件
* 容器创建回调

所有三种情况, **都是在容器创建完成时, 组件就被激活了**.

<<<<<<< HEAD
**尽量少使用启动时运行代码.** 你有可能会因为过度使用它而陷入困境. 更多信息见 "Tips" 章节.

.. contents::
  :local:

=======
>>>>>>> 64f8901eccf98f2d356bd8e835f455cc1637b28c
可启动组件
====================

**可启动组件** 当容器刚刚创建完后就被激活, 并且会调用一个指定的方法来进行组件中某些初始化操作.

该组件的关键是实现 ``Autofac.IStartable`` 接口. 当容器创建完成后, 组件就会被激活然后 ``IStartable.Start()`` 方法就会被调用.

**对于每个组件的每一个实例, 这只会发生一次, 在容器第一次被创建的时候.** 手动解析可启动组件不会造成它们 ``Start()`` 方法被调用. 我们不推荐可启动组件实现其他服务, 或者以除 ``SingleInstance()`` 之外的形式被注册.

如果一个组件需要类似 ``Start()`` 的方法 *每次被激活时* 都被调用, 应该使用类似 ``OnActivated(激活后)`` 这样的 :doc:`生命周期事件 <events>` 替代.

想要创建可启动组件, 实现 ``Autofac.IStartable``:

.. sourcecode:: csharp

    public class StartupMessageWriter : IStartable
    {
       public void Start()
       {
          Console.WriteLine("App is starting up!");
       }
    }

然后注册你的组件并且 **确保将它指定** 为 ``IStartable`` 否则方法不会被调用:

.. sourcecode:: csharp

    var builder = new ContainerBuilder();
    builder
       .RegisterType<StartupMessageWriter>()
       .As<IStartable>()
       .SingleInstance();

容器被创建后, 该类型将会被激活并且 ``IStartable.Start()`` 方法会被调用. 在这个示例中, 会向控制台输出一条消息.

组件启动的顺序是无法被定义的, 不过, 自 Autofac 4.7.0 开始, 当一个实现 ``IStartable`` 接口的组件依赖于另一个实现 ``IStartable`` 的组件时, 后者的 ``Start()`` 将保证在依赖它的组件被激活前就已经被调用了:

.. sourcecode:: csharp

    static void Main(string[] args)
    {

        var builder = new ContainerBuilder();
        builder.RegisterType<Startable1>().AsSelf().As<IStartable>().SingleInstance();
        builder.RegisterType<Startable2>().As<IStartable>().SingleInstance();
        builder.Build();
    }

    class Startable1 : IStartable
    {
        public Startable1()
        {
            Console.WriteLine("Startable1 activated");
        }

        public void Start()
        {
            Console.WriteLine("Startable1 started");
        }
    }

    class Startable2 : IStartable
    {
        public Startable2(Startable1 startable1)
        {
            Console.WriteLine("Startable2 activated");
        }

        public void Start()
        {
            Console.WriteLine("Startable2 started");
        }
    }

Will output the following:

::

    Startable1 activated
    Startable1 started
    Startable2 activated
    Startable2 started

自动激活组件
=========================

**自动激活组件** 是一种当容器被创建时只需被激活一次的组件. 它是 "热启动" 型的, 容器上没有方法会被调用并且也不需要实现某个接口 - 即解析了一个组件的单例并且它不会持有实例的引用.

想要注册一个自动激活组件, 使用 ``AutoActivate()`` 注册扩展方法.

.. sourcecode:: csharp

    var builder = new ContainerBuilder();
    builder
       .RegisterType<TypeRequiringWarmStart>()
       .AsSelf()
       .AutoActivate();

注意: 如果当你注册一个 ``AutoActivate()`` 组件时 *忽略* 了 ``AsSelf()`` 或 ``As<T>()`` 调用 , 组件在容器创建后 *只* 会被注册为自动激活但不会被解析为 "自身" .

容器创建回调
=========================

你可以通过容器创建回调在容器创建时注册任何动作. 创建回调指的是一个 ``Action<IContainer>`` 并且会在容器 ``ContainerBuilder.Build`` 返回之前得到已创建完成的容器. 创建回调以他们注册的顺序执行:

.. sourcecode:: csharp

    var builder = new ContainerBuilder();
    builder
       .RegisterBuildCallback(c => c.Resolve<DbContext>());

    // The callback will run after the container is built
    // but before it's returned.
    var container = builder.Build();

你可以使用创建回调作为另一种容器创建时自动启动/预热对象的方法. 可以通过结合 :doc:`生命周期激活后事件 <events>` 和 ``单例`` 注册来完成. 

一个又长又僵硬的单元测试形式的示例如下:

.. sourcecode:: csharp

    public class TestClass
    {
      // Create a dependency chain like
      //    ==> 2 ==+
      // 4 =+       ==> 1
      //    ==> 3 ==+
      // 4 needs 2 and 3
      // 2 needs 1
      // 3 needs 1
      // Dependencies should start up in the order
      // 1, 2, 3, 4
      // or
      // 1, 3, 2, 4
      private class Dependency1
      {
        public Dependency1(ITestOutputHelper output)
        {
          output.WriteLine("Dependency1.ctor");
        }
      }

      private class Dependency2
      {
        private ITestOutputHelper output;

        public Dependency2(ITestOutputHelper output, Dependency1 dependency)
        {
          this.output = output;
          output.WriteLine("Dependency2.ctor");
        }

        public void Initialize()
        {
          this.output.WriteLine("Dependency2.Initialize");
        }
      }

      private class Dependency3
      {
        private ITestOutputHelper output;

        public Dependency3(ITestOutputHelper output, Dependency1 dependency)
        {
          this.output = output;
          output.WriteLine("Dependency3.ctor");
        }

        public void Initialize()
        {
          this.output.WriteLine("Dependency3.Initialize");
        }
      }

      private class Dependency4
      {
        private ITestOutputHelper output;

        public Dependency4(ITestOutputHelper output, Dependency2 dependency2, Dependency3 dependency3)
        {
          this.output = output;
          output.WriteLine("Dependency4.ctor");
        }

        public void Initialize()
        {
          this.output.WriteLine("Dependency4.Initialize");
        }
      }

      // Xunit passes this to the ctor of the test class
      // so we can capture console output.
      private ITestOutputHelper _output;

      public TestClass(ITestOutputHelper output)
      {
        this._output = output;
      }

      [Fact]
      public void OnActivatedDependencyChain()
      {
        var builder = new ContainerBuilder();
        builder.RegisterInstance(this._output).As<ITestOutputHelper>();
        builder.RegisterType<Dependency1>().SingleInstance();

        // The OnActivated replaces the need for IStartable. When an instance
        // is activated/created, it'll run the Initialize method as specified. Using
        // SingleInstance means that only happens once.
        builder.RegisterType<Dependency2>().SingleInstance().OnActivated(args => args.Instance.Initialize());
        builder.RegisterType<Dependency3>().SingleInstance().OnActivated(args => args.Instance.Initialize());
        builder.RegisterType<Dependency4>().SingleInstance().OnActivated(args => args.Instance.Initialize());

        // Notice these aren't in dependency order.
        builder.RegisterBuildCallback(c => c.Resolve<Dependency4>());
        builder.RegisterBuildCallback(c => c.Resolve<Dependency2>());
        builder.RegisterBuildCallback(c => c.Resolve<Dependency1>());
        builder.RegisterBuildCallback(c => c.Resolve<Dependency3>());

        // This will run the build callbacks.
        var container = builder.Build();

        // These effectively do NOTHING. OnActivated won't be called again
        // because they're SingleInstance.
        container.Resolve<Dependency1>();
        container.Resolve<Dependency2>();
        container.Resolve<Dependency3>();
        container.Resolve<Dependency4>();
      }
    }

这个单元测试示例将输出如下:

::

    Dependency1.ctor
    Dependency2.ctor
    Dependency3.ctor
    Dependency4.ctor
    Dependency2.Initialize
    Dependency3.Initialize
    Dependency4.Initialize

你会从输出中注意到回调和 ``OnActivated`` 方法是以依赖顺序执行的. 如果你想让激活 *和* 启动都是以依赖顺序执行 (不只是激活/解析), 这是一个解决方案.
<<<<<<< HEAD

<<<<<<< HEAD
Note if you don't use ``SingleInstance`` then ``OnActivated`` will be called for *every new instance of the dependency*. Since "warm start" objects are usually singletons and are expensive to create, this is generally what you want anyway.

Tips
====

**Order**: In general, startup logic happens in the order ``IStartable.Start()``, ``AutoActivate``, build callbacks. That said, it is *not guaranteed*. For example, as noted in the ``IStartable`` docs above, things will happen in dependency order rather than registration order. Further, Autofac reserves the right to change this order (e.g., refactor the calls to ``IStartable.Start()`` and ``AutoActivate`` into build callbacks). If you need to control the specific order in which initialization logic runs, it's better to write your own initialization logic where you can control the order.

**Avoid creating lifetime scopes during IStartable.Start or AutoActivate**: If your startup logic includes the creation of a lifetime scope from which components will be resolved, this scope won't have all the startables executed yet. By creating the scope, you're forcing a race condition. This sort of logic would be better to execute in custom logic after the container is built rather than as part of an ``IStartable``.

**Avoid overusing startup logic**: The ability to run startup logic on container build may feel like it's also a good fit for orchestrating general application startup logic. Given the ordering and other challenges you may run into, it is recommended you keep *application startup* logic separate from *dependency startup* logic.

**Consider OnActivated and SingleInstance for lazy initialization**: Instead of using build callbacks or startup logic, consider using :doc:`the lifetime event OnActivated <events>` with a ``SingleInstance`` registration so the initialization can happen on an object but not be tied to the order of container build.
=======
注意如果你不调用 ``SingleInstance`` 那么 ``OnActivated`` 方法将会在 *每个依赖的新实例* 创建时被调用. 由于 "热启动" 对象通常是单例且创建需要消耗较大资源, 所以还是以单例注册吧.
>>>>>>> 100% of lifetime startup
=======

注意如果你不调用 ``SingleInstance`` 那么 ``OnActivated`` 方法将会在 *每个依赖的新实例* 创建时被调用. 由于 "热启动" 对象通常是单例且创建需要消耗较大资源, 所以还是以单例注册吧.
>>>>>>> 64f8901eccf98f2d356bd8e835f455cc1637b28c

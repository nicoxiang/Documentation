=====================
注册概念
=====================

我们通过创建 ``ContainerBuilder`` 来注册 :doc:`组件 <../glossary>` 并且告诉容器哪些 :doc:`组件 <../glossary>` 暴露了哪些 :doc:`服务 <../glossary>`.

**组件** 可以通过 **反射** (注册指定的.NET类或开放结构的泛型)创建; 通过提供现成的 **实例** (你已创建的一个对象实例)创建; 或者通过 lambda **表达式** (一个执行实例化对象的匿名方法)来创建. ``ContainerBuilder`` 包含一组 ``Register()`` 方法来帮你实现以上操作.

每个组件暴露一个或多个 **服务** ,他们使用 ``ContainerBuilder`` 上的 ``As()`` 方法连接起来.

.. sourcecode:: csharp

    // Create the builder with which components/services are registered.
    var builder = new ContainerBuilder();

    // Register types that expose interfaces...
    builder.RegisterType<ConsoleLogger>().As<ILogger>();

    // Register instances of objects you create...
    var output = new StringWriter();
    builder.RegisterInstance(output).As<TextWriter>();

    // Register expressions that execute to create objects...
    builder.Register(c => new ConfigReader("mysection")).As<IConfigReader>();

    // Build the container to finalize registrations
    // and prepare for object resolution.
    var container = builder.Build();

    // Now you can resolve services using Autofac. For example,
    // this line will execute the lambda expression registered
    // to the IConfigReader service.
    using(var scope = container.BeginLifetimeScope())
    {
      var reader = scope.Resolve<IConfigReader>();
    }

.. _register-registration-reflection-components:

反射组件
=====================

通过类型注册
----------------

通过反射生成的组件通常是由类型注册的:

.. sourcecode:: csharp

    var builder = new ContainerBuilder();
    builder.RegisterType<ConsoleLogger>();
    builder.RegisterType(typeof(ConfigReader));

当使用基于反射的组件时, **Autofac 自动为你的类从容器中寻找匹配拥有最多参数的构造方法**.

例如，你有一个拥有三个构造函数的类:

.. sourcecode:: csharp

    public class MyComponent
    {
        public MyComponent() { /* ... */ }
        public MyComponent(ILogger logger) { /* ... */ }
        public MyComponent(ILogger logger, IConfigReader reader) { /* ... */ }
    }

现在，在你的容器中注册组件和服务:

.. sourcecode:: csharp

    var builder = new ContainerBuilder();
    builder.RegisterType<MyComponent>();
    builder.RegisterType<ConsoleLogger>().As<ILogger>();
    var container = builder.Build();

    using(var scope = container.BeginLifetimeScope())
    {
      var component = scope.Resolve<MyComponent>();
    }

当你解析组件时, Autofac发现 ``ILogger`` 已被注册, 但你并没有注册 ``IConfigReader`` . 在这种情况下, 第二个构造方法会被选中因为它是能在容器中能找到最多参数的那个.

**基于反射的组件有个重要的需要注意的地方:** 任何通过 ``RegisterType`` 注册的组件必须是个具体的类型. 虽然组件可以暴露抽象类和接口作为 :doc:`服务 <../glossary>`, 但你不能注册一个抽象类/接口组件. 你这样想就明白了: 在幕后, Autofac其实是创建了一个你注册对象的实例. 你无法 "new up" 一个抽象类或一个接口. 你得有个具体的实现, 对吧?

指定构造函数
------------------------

你可以使用 ``UsingConstructor`` 方法和构造方法中一系列代表参数类型的类型来 **手动指定一个构造函数** , 通过这种方式使用和覆盖注册组件自动选择的构造函数:

.. sourcecode:: csharp

    builder.RegisterType<MyComponent>()
           .UsingConstructor(typeof(ILogger), typeof(IConfigReader));

要注意的是, 在解析时你仍然需要提供必要的参数，否则在你尝试解析对象时将出现错误. 你可以 :doc:`在注册时传参 <parameters>` 或 :doc:`在解析时传参 <../resolve/parameters>`.

实例组件
===================

有时候, 你也许会希望提前生成一个对象的实例并将它加入容器以供注册组件时使用. 你可以通过使用 ``RegisterInstance`` 方法:

.. sourcecode:: csharp

    var output = new StringWriter();
    builder.RegisterInstance(output).As<TextWriter>();

当你这样做时你需要考虑一些事情, Autofac :doc:`自动处理已注册组件的释放 <../lifetime/disposal>` ,你也许想要自己来控制生命周期而不是让Autofac来帮你调用 ``Dispose`` 释放你的对象. 在这种情况下, 你需要通过 ``ExternallyOwned`` 方法来注册实例:

.. sourcecode:: csharp

    var output = new StringWriter();
    builder.RegisterInstance(output)
           .As<TextWriter>()
           .ExternallyOwned();

当将Autofac集成到一个现有的应用程序(已存在一个单例实例且需要在容器中被组件使用)时, 注册已提供的实例组件同样非常方便. 而不是直接把这些组件绑定到单例实例上, 它可以在容器中注册为一个实例:

.. sourcecode:: csharp

    builder.RegisterInstance(MySingleton.Instance).ExternallyOwned();

这样能确保静态单例最终能被淘汰, 而被容器管理取而代之.

通过某一实例暴露的默认服务是该实例的实体类. 详见下面的"服务 vs. 组件".

.. _register-registration-lambda-expression-components:

Lambda表达式组件
============================

反射在组件创建时是个很好的选择. 但是, 当组件创建不再是简单的调用构造方法时, 事情将变得混乱起来.

Autofac接收一个委托或者lambda表达式, 用作组件创建者:

.. sourcecode:: csharp

  builder.Register(c => new A(c.Resolve<B>()));

表达式提供的参数 ``c``  是 *组件上下文* (一个 ``IComponentContext`` 对象) , 组件在其中被创建. 你可以使用这个参数来从容器中解析出其他值来帮助创建你的组件. **使用这个参数而不是一个闭包来访问容器非常重要** 这样可以保证 :doc:`对象精确的释放 <../lifetime/disposal>` 并且可以很好的支持嵌套容器.

使用该上下文参数可以满足其他依赖的成功引入 - 例如, ``A`` 需要一个构造方法参数 ``B`` ,而 ``B`` 可能还有其他的依赖关系.

表达式创建的组件提供的默认服务是表达式的推断返回类型.

下面的一些示例使用反射组件很难满足需求但是用lambda表达式可以很好地解决问题.

复杂参数
------------------
构造方法参数不会总是简单的常量. 不要困惑于如何使用XML配置的语法构造某种类型的值, 试试下面的代码:

.. sourcecode:: csharp

    builder.Register(c => new UserSession(DateTime.Now.AddMinutes(25)));

(当然, session过期时间你应该会在配置文件里获取 - 不过没关系, 你已经抓住重点了对吧 ;))

参数注入
------------------
Autofac提供了 :doc:`一流的方法可用来完成参数注入 <prop-method-injection>`, 你可以使用表达式和属性初始化来填充参数:

.. sourcecode:: csharp

    builder.Register(c => new A(){ MyB = c.ResolveOptional<B>() });

``ResolveOptional`` 方法会尝试解析, 但如果服务没有注册, 不会抛出错误. (如果服务成功注册但是无法成功解析, 你依然会得到一个错误.) 这是 :doc:`解析服务 <../resolve/index>` 的一种方式.

**在大多数情况下不推荐使用参数注入.** 使用其他的例如 `空对象模式 <http://en.wikipedia.org/wiki/Null_Object_pattern>`_, 重载构造方法或参数默认值这些方式, 用构造方法注入可以创建出可选依赖的更清爽, "不可变" 组件.

通过参数值选择具体的实现
-------------------------------------------------

创建分离组件的一大好处是具体的类型可以是多种多样的. 指定具体的类型通常可以在运行时完成, 而不仅仅是配置时期:

.. sourcecode:: csharp

    builder.Register<CreditCard>(
      (c, p) =>
        {
          var accountId = p.Named<string>("accountId");
          if (accountId.StartsWith("9"))
          {
            return new GoldCard(accountId);
          }
          else
          {
            return new StandardCard(accountId);
          }
        });

示例中, ``CreditCard`` 通过两种类实现, ``GoldCard`` 和 ``StandardCard`` - 哪个类会被实例化取决于运行时期间提供的account ID.

示例中 :doc:`创建方法的参数 <../resolve/parameters>` 通过第二个可选的参数 ``p`` 传入.

解析时可以这样:

.. sourcecode:: csharp

    var card = container.Resolve<CreditCard>(new NamedParameter("accountId", "12345"));

如果声明一个创建 ``CreditCard`` 实例的委托和 :doc:`一个委托工厂 <../advanced/delegate-factories>` , 语法可以变得更加干净, 类型安全.

开放泛型组件
=======================

Autofac支持开放泛型. 使用 ``RegisterGeneric()`` 方法:

.. sourcecode:: csharp

    builder.RegisterGeneric(typeof(NHibernateRepository<>))
           .As(typeof(IRepository<>))
           .InstancePerLifetimeScope();

当容器请求一个匹配的服务类型时, Autofac将会找到对应的封闭类型的具体实现:

.. sourcecode:: csharp

    // Autofac will return an NHibernateRepository<Task>
    var tasks = container.Resolve<IRepository<Task>>();

注册具体的服务类型 (e.g. ``IRepository<Person>``) 会覆盖开放类型的版本.

服务 vs. 组件
=======================

注册 :doc:`组件 <../glossary>` 时, 我们得告诉Autofac, 组件暴露了哪些 :doc:`服务 <../glossary>` . 默认地, 类型注册时大部分情况下暴露它们自身:

.. sourcecode:: csharp

    // This exposes the service "CallLogger"
    builder.RegisterType<CallLogger>();

组件能够被它暴露的服务 :doc:`解析 <../resolve/index>` . 示例中:

.. sourcecode:: csharp

    // This will work because the component
    // exposes the type by default:
    scope.Resolve<CallLogger>();

    // This will NOT work because we didn't
    // tell the registration to also expose
    // the ILogger interface on CallLogger:
    scope.Resolve<ILogger>();

你可以让一个组件暴露任意数量的服务:

.. sourcecode:: csharp

    builder.RegisterType<CallLogger>()
           .As<ILogger>()
           .As<ICallInterceptor>();

暴露服务后, 你就可以解析基于该服务的组件了. 但请注意, 一旦你将组件暴露为一个特定的服务, 默认的服务 (组件类型) 将被覆盖:

.. sourcecode:: csharp

    // These will both work because we exposed
    // the appropriate services in the registration:
    scope.Resolve<ILogger>();
    scope.Resolve<ICallInterceptor>();

    // This WON'T WORK anymore because we specified
    // service overrides on the component:
    scope.Resolve<CallLogger>();

如果你既想组件暴露一系列特定的服务, 又想让它暴露默认的服务, 可以使用 ``AsSelf`` 方法:

.. sourcecode:: csharp

    builder.RegisterType<CallLogger>()
           .AsSelf()
           .As<ILogger>()
           .As<ICallInterceptor>();

这样所有的解析就都能成功了:

.. sourcecode:: csharp

    // These will all work because we exposed
    // the appropriate services in the registration:
    scope.Resolve<ILogger>();
    scope.Resolve<ICallInterceptor>();
    scope.Resolve<CallLogger>();

默认注册
=====================
如果不止一个组件暴露了相同的服务, **Autofac将使用最后注册的组件作为服务的提供方**:

.. sourcecode:: csharp

    builder.RegisterType<ConsoleLogger>().As<ILogger>();
    builder.RegisterType<FileLogger>().As<ILogger>();

上例中, ``FileLogger`` 将会作为 ``ILogger`` 默认的服务提供方因为它是最后被注册的.

想要覆盖这种行为, 使用 ``PreserveExistingDefaults()`` 方法修改:

.. sourcecode:: csharp

    builder.RegisterType<ConsoleLogger>().As<ILogger>();
    builder.RegisterType<FileLogger>().As<ILogger>().PreserveExistingDefaults();

上例中, ``ConsoleLogger`` 将会作为 ``ILogger`` 默认的服务提供方因为最后注册的 ``FileLogger`` 使用了 ``PreserveExistingDefaults()``.

有条件的注册
========================

.. note:: 有条件的注册自Autofac **4.4.0** 引入

大多数情况下, 像上面那样覆盖默认的注册其实已经足够让我们在运行时成功地解析正确的组件了. 我们可以使用 ``PreserveExistingDefaults()`` 保证组件以正确的顺序被注册; 对于复杂的条件和行为我们也可以利用 lambda表达式/委托 注册处理的很不错了.

但依然有些场景应该是你不想碰到的:

- 你不想在程序中有些功能在正常运作的情况下某个组件还会出现. 例如, 如果你解析了 ``IEnumerable<T>`` 的服务(一堆服务), 所有实现了这些服务的已注册组件都将被返回, 不管你是否使用了 ``PreserveExistingDefaults()``. 大多数情况下这样也行, 但在某些极端情况下你不希望如此.
- 你只想要在其他一些组件 *未被* 注册的情况下才注册组件; 或者只想在其他一些组件 *已被* 注册的情况下. 你不会从容器中解析出你不想要的东西, 并且你也不用修改已经创建的容器. 能够基于其他的注册情况来进行有条件的组件注册非常好用.

这边有两种好用的注册扩展方法:

- ``OnlyIf()`` - 提供一个表达式, 使用一个 ``IComponentRegistryBuilder`` 来决定注册是否发生.
- ``IfNotRegistered()`` - 有其他服务已被注册的情况下阻止注册发生的快捷方法.

这些方法在 ``ContainerBuilder.Build()`` 时执行并且以实际组件注册的顺序执行. 下面是一些展示它们如何工作的示例:

.. sourcecode:: csharp

    var builder = new ContainerBuilder();

    // Only ServiceA will be registered.
    // Note the IfNotRegistered takes the SERVICE TYPE to
    // check for (the As<T>), NOT the COMPONENT TYPE
    // (the RegisterType<T>).
    builder.RegisterType<ServiceA>()
           .As<IService>();
    builder.RegisterType<ServiceB>()
           .As<IService>()
           .IfNotRegistered(typeof(IService));

    // HandlerA WILL be registered - it's running
    // BEFORE HandlerB has a chance to be registered
    // so the IfNotRegistered check won't find it.
    //
    // HandlerC will NOT be registered because it
    // runs AFTER HandlerB. Note it can check for
    // the type "HandlerB" because HandlerB registered
    // AsSelf() not just As<IHandler>(). Again,
    // IfNotRegistered can only check for "As"
    // types.
    builder.RegisterType<HandlerA>()
           .AsSelf()
           .As<IHandler>()
           .IfNotRegistered(typeof(HandlerB));
    builder.RegisterType<HandlerB>()
           .AsSelf()
           .As<IHandler>();
    builder.RegisterType<HandlerC>()
           .AsSelf()
           .As<IHandler>()
           .IfNotRegistered(typeof(HandlerB));

    // Manager will be registered because both an IService
    // and HandlerB are registered. The OnlyIf predicate
    // can allow a lot more flexibility.
    builder.RegisterType<Manager>()
           .As<IManager>()
           .OnlyIf(reg =>
             reg.IsRegistered(new TypedService(typeof(IService))) &&
             reg.IsRegistered(new TypedService(typeof(HandlerB))));

    // This is when the conditionals actually run. Again,
    // they run in the order the registrations were added
    // to the ContainerBuilder.
    var container = builder.Build();

注册的配置
==============================
你可以 :doc:`使用 XML 或or 编程式配置 ("模块") <../configuration/index>` 来提供注册的群组或者在运行时改变注册. 对于一些动态的注册的生成或者有条件的注册逻辑, 你可以 :doc:`使用 Autofac 模块 <../configuration/modules>` .

动态提供的注册
==================================
:doc:`Autofac模块 <../configuration/modules>` 是引入动态注册逻辑或简单切面功能的最简单的方法. 例如, 你可以使用一个模块 :doc:`动态地在被解析的服务上附加一个log4net logger实例 <../examples/log4net>`.

如果想要完成更加动态的操作, 例如添加对新的 :doc:`隐式关系类型 <../resolve/relationships>` 的支持, 你可以 :doc:`在高级概念章节查看注册源模块 <../advanced/registration-sources>`.

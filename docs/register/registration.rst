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

通过某一实例暴露的默认服务是该实例的具体类. 详见下面的"服务 vs. 组件".

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

Open Generic Components
=======================

Autofac supports open generic types. Use the ``RegisterGeneric()`` builder method:

.. sourcecode:: csharp

    builder.RegisterGeneric(typeof(NHibernateRepository<>))
           .As(typeof(IRepository<>))
           .InstancePerLifetimeScope();

When a matching service type is requested from the container, Autofac will map this to an equivalent closed version of the implementation type:

.. sourcecode:: csharp

    // Autofac will return an NHibernateRepository<Task>
    var tasks = container.Resolve<IRepository<Task>>();

Registration of a specialized service type (e.g. ``IRepository<Person>``) will override the open generic version.

Services vs. Components
=======================

When you register :doc:`components <../glossary>`, you have to tell Autofac which :doc:`services <../glossary>` that component exposes. By default, most registrations will just expose themselves as the type registered:

.. sourcecode:: csharp

    // This exposes the service "CallLogger"
    builder.RegisterType<CallLogger>();

Components can only be :doc:`resolved <../resolve/index>` by the services they expose. In this simple example it means:

.. sourcecode:: csharp

    // This will work because the component
    // exposes the type by default:
    scope.Resolve<CallLogger>();

    // This will NOT work because we didn't
    // tell the registration to also expose
    // the ILogger interface on CallLogger:
    scope.Resolve<ILogger>();

You can expose a component with any number of services you like:

.. sourcecode:: csharp

    builder.RegisterType<CallLogger>()
           .As<ILogger>()
           .As<ICallInterceptor>();

Once you expose a service, you can resolve the component based on that service. Note, however, that once you expose a component as a specific service, the default service (the component type) is overridden:

.. sourcecode:: csharp

    // These will both work because we exposed
    // the appropriate services in the registration:
    scope.Resolve<ILogger>();
    scope.Resolve<ICallInterceptor>();

    // This WON'T WORK anymore because we specified
    // service overrides on the component:
    scope.Resolve<CallLogger>();

If you want to expose a component as a set of services as well as using the default service, use the ``AsSelf`` method:

.. sourcecode:: csharp

    builder.RegisterType<CallLogger>()
           .AsSelf()
           .As<ILogger>()
           .As<ICallInterceptor>();

Now all of these will work:

.. sourcecode:: csharp

    // These will all work because we exposed
    // the appropriate services in the registration:
    scope.Resolve<ILogger>();
    scope.Resolve<ICallInterceptor>();
    scope.Resolve<CallLogger>();

Default Registrations
=====================
If more than one component exposes the same service, **Autofac will use the last registered component as the default provider of that service**:

.. sourcecode:: csharp

    builder.Register<ConsoleLogger>().As<ILogger>();
    builder.Register<FileLogger>().As<ILogger>();

In this scenario, ``FileLogger`` will be the default for ``ILogger`` because it was the last one registered.

To override this behavior, use the ``PreserveExistingDefaults()`` modifier:

.. sourcecode:: csharp

    builder.Register<ConsoleLogger>().As<ILogger>();
    builder.Register<FileLogger>().As<ILogger>().PreserveExistingDefaults();

In this scenario, ``ConsoleLogger`` will be the default for ``ILogger`` because the later registration for ``FileLogger`` used ``PreserveExistingDefaults()``.

Conditional Registration
========================

.. note:: Conditional registration was introduced in Autofac **4.4.0**

In most cases, overriding registrations as noted in the section above, "Default Registrations," is enough to get the right component resolved at runtime. Ensuring things get registered in the right order; using ``PreserveExistingDefaults()``; and taking advantage of lambda/delegate registrations for more complex conditions and behavior can get you pretty far.

There can be a few scenarios where this may not be the way you want to go:

- You don't want the component present in the system if something else is handling the functionality. For example, if you resolve an ``IEnumerable<T>`` of a service, all of the registered components implementing that service will be returned, whether or not you've used ``PreserveExistingDefaults()``. Usually this is fine, but there are some edge cases where you may not want that.
- You only want to register the component if some other component *isn't* registered; or only if some other component *is* registered. You can't resolve things out of a container that you're building, and you shouldn't update a container that's already built. Being able to conditionally register a component based on other registrations can be helpful.

There are two registration extensions that can help in these cases:

- ``OnlyIf()`` - Provide a lambda that uses an ``IComponentRegistry`` to determine if a registration should happen.
- ``IfNotRegistered()`` - Shortcut to stop a registration from happening if some other service is already registered.

These extensions run at the time of ``ContainerBuilder.Build()`` and will execute in the order of the actual component registrations. Here are some examples showing how they work:

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

Configuration of Registrations
==============================
You can :doc:`use XML or programmatic configuration ("modules") <../configuration/index>` to provide groups of registrations together or change registrations at runtime. You can also use :doc:`use Autofac modules <../configuration/modules>` for some dynamic registration generation or conditional registration logic.

Dynamically-Provided Registrations
==================================
:doc:`Autofac modules <../configuration/modules>` are the simplest way to introduce dynamic registration logic or simple cross-cutting features. For example, you can use a module to :doc:`dynamically attach a log4net logger instance to a service being resolved <../examples/log4net>`.

If you find that you need even more dynamic behavior, such as adding support for a new :doc:`implicit relationship type <../resolve/relationships>`, you might want to :doc:`check out the registration sources section in the advanced concepts area <../advanced/registration-sources>`.

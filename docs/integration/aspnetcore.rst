============
ASP.NET Core
============

ASP.NET Core (previously ASP.NET 5) 改变了以前依赖注入框架集成进ASP.NET的方法. 以前, 每个功能 - MVC, Web API, 等. - 都有它自己的 "依赖解析器(dependency resolver)" 机制并且只是'钩子'钩住的方式有些轻微的区别. ASP.NET Core 通过 `Microsoft.Extensions.DependencyInjection <https://github.com/aspnet/DependencyInjection>`_ 引入了 `conforming container <http://blog.ploeh.dk/2014/05/19/conforming-container/>`_ 机制, 包含了请求生命周期作用域, 服务注册等等的统一概念.

**本章节解释了ASP.NET Core集成.** 如果你正在使用传统的ASP.NET, :doc:`请见传统ASP.NET集成章节 <aspnet>`.

如果你只是使用.NET Core而没有用ASP.NET Core, :doc:`这里有一个更简单的示例 <netcore>` 展示了它的集成.

.. contents::
  :local:

入门 (With ConfigureContainer)
=====================================

ASP.NET Core 1.1 引入了强类型容器配置的能力. 它提供了 ``ConfigureContainer`` 方法, 你可以在方法内注册Autofac的东西, 这样可以和 ``ServiceCollection`` 注册东西分开.

* Nuget引入 ``Autofac.Extensions.DependencyInjection`` 包.
* 在你的 ``Program.Main`` 方法内, 配置 ``WebHostBuilder`` 的地方, 调用 ``AddAutofac`` 把Autofac挂到startup管道中.
* 在 ``Startup`` 类的 ``ConfigureServices`` 方法中先用其他库提供的扩展方法注册东西到 ``IServiceCollection`` .
* 在 ``Startup`` 类的 ``ConfigureContainer`` 方法中直接注册东西到Autofac ``ContainerBuilder``.

``IServiceProvider`` 会自动替你创建, 因此你无需做任何事只要 *注册东西* 即可.

.. sourcecode:: csharp

    public class Program
    {
      public static void Main(string[] args)
      {
        // The ConfigureServices call here allows for
        // ConfigureContainer to be supported in Startup with
        // a strongly-typed ContainerBuilder.
        var host = new WebHostBuilder()
            .UseKestrel()
            .ConfigureServices(services => services.AddAutofac())
            .UseContentRoot(Directory.GetCurrentDirectory())
            .UseIISIntegration()
            .UseStartup<Startup>()
            .Build();

        host.Run();
      }
    }

    public class Startup
    {
      public Startup(IHostingEnvironment env)
      {
        var builder = new ConfigurationBuilder()
            .SetBasePath(env.ContentRootPath)
            .AddJsonFile("appsettings.json", optional: true, reloadOnChange: true)
            .AddJsonFile($"appsettings.{env.EnvironmentName}.json", optional: true)
            .AddEnvironmentVariables();
        this.Configuration = builder.Build();
      }

      public IConfigurationRoot Configuration { get; private set; }

      // ConfigureServices is where you register dependencies. This gets
      // called by the runtime before the ConfigureContainer method, below.
      public void ConfigureServices(IServiceCollection services)
      {
        // Add services to the collection. Don't build or return
        // any IServiceProvider or the ConfigureContainer method
        // won't get called.
        services.AddMvc();
      }

      // ConfigureContainer is where you can register things directly
      // with Autofac. This runs after ConfigureServices so the things
      // here will override registrations made in ConfigureServices.
      // Don't build the container; that gets done for you. If you
      // need a reference to the container, you need to use the
      // "Without ConfigureContainer" mechanism shown later.
      public void ConfigureContainer(ContainerBuilder builder)
      {
          builder.RegisterModule(new AutofacModule());
      }

      // Configure is where you add middleware. This is called after
      // ConfigureContainer. You can use IApplicationBuilder.ApplicationServices
      // here if you need to resolve things from the container.
      public void Configure(
        IApplicationBuilder app,
        ILoggerFactory loggerFactory)
      {
          loggerFactory.AddConsole(this.Configuration.GetSection("Logging"));
          loggerFactory.AddDebug();
          app.UseMvc();
      }
    }

入门 (Without ConfigureContainer)
========================================

如果你在创建你的容器时需要更多的灵活性或者你需要存储所创建容器的引用 (如, 这样你就可以在应用停止时自己释放容器), 你需要跳过 ``ConfigureContainer`` 并且在 ``ConfigureServices`` 中注册所有东西. 这也是你在ASP.NET Core 1.0中需要采取的方式.

* Nuget引入 ``Autofac.Extensions.DependencyInjection`` 包.
* 在你的 ``Startup`` 类的 ``ConfigureServices`` 方法中...

  - 通过 ``Populate`` 把注册的服务从 ``IServiceCollection`` 填充到 ``ContainerBuilder`` .
  - 直接注册服务到 ``ContainerBuilder`` .
  - 创建容器.
  - 使用容器创建 ``AutofacServiceProvider`` 并返回.

* 在你的 ``Startup`` 类的 ``Configure`` 方法中, 你可以选择性地在应用停止时注册 ``IApplicationLifetime.ApplicationStopped`` 事件释放容器.

.. sourcecode:: csharp

    public class Startup
    {
      public Startup(IHostingEnvironment env)
      {
        var builder = new ConfigurationBuilder()
            .SetBasePath(env.ContentRootPath)
            .AddJsonFile("appsettings.json", optional: true, reloadOnChange: true)
            .AddJsonFile($"appsettings.{env.EnvironmentName}.json", optional: true)
            .AddEnvironmentVariables();
        this.Configuration = builder.Build();
      }

      public IContainer ApplicationContainer { get; private set; }

      public IConfigurationRoot Configuration { get; private set; }

      // ConfigureServices is where you register dependencies. This gets
      // called by the runtime before the Configure method, below.
      public IServiceProvider ConfigureServices(IServiceCollection services)
      {
        // Add services to the collection.
        services.AddMvc();

        // Create the container builder.
        var builder = new ContainerBuilder();

        // Register dependencies, populate the services from
        // the collection, and build the container. If you want
        // to dispose of the container at the end of the app,
        // be sure to keep a reference to it as a property or field.
        //
        // Note that Populate is basically a foreach to add things
        // into Autofac that are in the collection. If you register
        // things in Autofac BEFORE Populate then the stuff in the
        // ServiceCollection can override those things; if you register
        // AFTER Populate those registrations can override things
        // in the ServiceCollection. Mix and match as needed.
        builder.Populate(services);
        builder.RegisterType<MyType>().As<IMyType>();
        this.ApplicationContainer = builder.Build();

        // Create the IServiceProvider based on the container.
        return new AutofacServiceProvider(this.ApplicationContainer);
      }

      // Configure is where you add middleware. This is called after
      // ConfigureServices. You can use IApplicationBuilder.ApplicationServices
      // here if you need to resolve things from the container.
      public void Configure(
        IApplicationBuilder app,
        ILoggerFactory loggerFactory,
        IApplicationLifetime appLifetime)
      {
          loggerFactory.AddConsole(this.Configuration.GetSection("Logging"));
          loggerFactory.AddDebug();

          app.UseMvc();

          // If you want to dispose of resources that have been resolved in the
          // application container, register for the "ApplicationStopped" event.
          // You can only do this if you have a direct reference to the container,
          // so it won't work with the above ConfigureContainer mechanism.
          appLifetime.ApplicationStopped.Register(() => this.ApplicationContainer.Dispose());
      }
    }

配置方法命名约定
=======================================

``Configure``, ``ConfigureServices``, 和 ``ConfigureContainer`` 方法都支持基于你应用中 ``IHostingEnvironment.EnvironmentName`` 参数的环境特定命名约定. 默认地, 名称为 ``Configure``, ``ConfigureServices``, 和 ``ConfigureContainer``. 如果你想要环境特定设置, 你可以把环境名称放在 ``Configure`` 部分后面, 类似 ``ConfigureDevelopment``, ``ConfigureDevelopmentServices``, 和 ``ConfigureDevelopmentContainer``. 如果方法并不以匹配的环境名称显示, 它会回到默认方法.

这意味着你不必使用 :doc:`Autofac配置 <../configuration/index>` 在生产环境和开发环境之间切换; 你可以在 ``Startup`` 中以编程形式设置.

.. sourcecode:: csharp

    public class Startup
    {
      public Startup(IHostingEnvironment env)
      {
        // Do Startup-ish things like read configuration.
      }

      // This is the default if you don't have an environment specific method.
      public void ConfigureServices(IServiceCollection services)
      {
        // Add things to the service collection.
      }

      // This only gets called if your environment is Development. The
      // default ConfigureServices won't be automatically called if this
      // one is called.
      public void ConfigureDevelopmentServices(IServiceCollection services)
      {
        // Add things to the service collection that are only for the
        // development environment.
      }

      // This is the default if you don't have an environment specific method.
      public void ConfigureContainer(ContainerBuilder builder)
      {
        // Add things to the Autofac ContainerBuilder.
      }

      // This only gets called if your environment is Production. The
      // default ConfigureContainer won't be automatically called if this
      // one is called.
      public void ConfigureProductionContainer(ContainerBuilder builder)
      {
        // Add things to the ContainerBuilder that are only for the
        // production environment.
      }

      // This is the default if you don't have an environment specific method.
      public void Configure(IApplicationBuilder app, ILoggerFactory loggerFactory)
      {
        // Set up the application.
      }

      // This only gets called if your environment is Staging. The
      // default Configure won't be automatically called if this one is called.
      public void ConfigureStaging(IApplicationBuilder app, ILoggerFactory loggerFactory)
      {
        // Set up the application for staging.
      }
    }

`ASP.NET Core中的StartupLoader类 <https://github.com/aspnet/Hosting/blob/rel/1.1.0/src/Microsoft.AspNetCore.Hosting/Internal/StartupLoader.cs>`_ 是在应用启动时定位调用方法的. 如果你想更深层次地了解它是如何运作的, 可以看下该类.

依赖注入钩子
==========================

不像 :doc:`传统ASP.NET集成 <aspnet>`, ASP.NET Core的设计秉承依赖注入的理念. 这意味着如果你想知道, `如何注入服务到MVC views <https://docs.asp.net/en/latest/mvc/views/dependency-injection.html>`_ 它现在是ASP.NET Core控制(记录)的  - 除了像上面那样设置你的服务提供者(service provider)你还需要一些Autofac特定的操作.

这里有一些特别关注DI集成的ASP.NET Core文档链接:

* `ASP.NET Core dependency injection fundamentals <https://docs.asp.net/en/latest/fundamentals/dependency-injection.html>`_
* `Controller injection <https://docs.asp.net/en/latest/mvc/controllers/dependency-injection.html>`_
* `The Subtle Perils of Controller Dependency Injection in ASP.NET Core MVC <http://www.strathweb.com/2016/03/the-subtle-perils-of-controller-dependency-injection-in-asp-net-core-mvc/>`_
* `Filter injection <https://docs.asp.net/en/latest/mvc/controllers/filters.html#configuring-filters>`_
* `View injection <https://docs.asp.net/en/latest/mvc/views/dependency-injection.html>`_
* `Authorization requirement handlers injection <https://docs.asp.net/en/latest/security/authorization/dependencyinjection.html>`_
* `Middleware options injection <https://docs.asp.net/en/latest/migration/http-modules.html#loading-middleware-options-through-direct-injection>`_
* `Middleware 'Invoke' method injection <https://docs.asp.net/en/latest/fundamentals/middleware.html>`_
* `Wiring up EF 6 with ASP.NET Core <https://docs.asp.net/en/latest/data/entity-framework-6.html#setup-connection-strings-and-dependency-injection>`_

与传统ASP.NET的区别
================================

如果你使用Autofac其他的 :doc:`ASP.NET集成 <aspnet>` 你应该对它们和迁移至ASP.NET Core的关键区别感兴趣.

* **使用InstancePerLifetimeScope(每个生命周期作用域一个实例)而不是InstancePerRequest(每个请求一个实例).** 以前的ASP.NET集成你可以注册依赖为 ``InstancePerRequest`` , 能保证每次HTTP请求只有唯一的依赖实例被创建. 这是有用的因为Autofac负责 :doc:`建立每个请求生命周期作用域 <../faq/per-request-scope>`. 随着 ``Microsoft.Extensions.DependencyInjection`` 的引入, 每个请求和其他子生命周期作用域的创建现在是框架提供的 `conforming container <http://blog.ploeh.dk/2014/05/19/conforming-container/>`_ 的一部分, 因此所有的子生命周期作用域是被同等对待的 - 现在已经不再有特别的 "请求级别作用" . 不再是注册你的依赖为 ``InstancePerRequest``, 而使用 ``InstancePerLifetimeScope`` , 你也可以得到相同的行为. 注意如果你在web请求中创建 *你自己的生命周期作用域* , 你将会在这些子作用域中得到新的实例.
* **不再需要依赖解析器(DependencyResolver).** 其他ASP.NET集成机制在许多地方需要创建基于Autofac的自定义依赖解析器. 使用 ``Microsoft.Extensions.DependencyInjection`` 和 ``Startup.ConfigureServices`` 方法, 你现在只要返回 ``IServiceProvider`` , "神奇的事就发生了." 在控制器, 类等内部. 如果你需要手动定位服务, 拿 ``IServiceProvider`` 即可.
* **没有特殊的中间件.** 以前的 :doc:`OWIN集成 <owin>` 需要特殊的Autofac中间件的注册, 用来管理请求生命周期作用域. ``Microsoft.Extensions.DependencyInjection`` 现在做了这些繁重的工作, 因此现在不需要注册额外的中间件了.
* **不再需要手动注册控制器.** 你以前需要用Autofac手动注册所有的控制器这样DI才会work. ASP.NET Core框架现在自动传入所有控制器给服务解析因此你不必手动注册.
* **没有通过依赖注入触发中间件的扩展方法.** :doc:`OWIN集成 <owin>` 有类似 ``UseAutofacMiddleware()`` 的扩展方法来允许依赖注入进入中间件内. 这些现在都将自动发生, 通过组合 `自动注入构造方法参数和动态解析中间件Invoke方法的参数 <http://docs.asp.net/en/latest/fundamentals/middleware.html>`_. ASP.NET Core框架负责了所有的这些事.
* **MVC 和 Web API 现在是一个东西了.** 以前根据你是使用 MVC 还是 Web API ,有不同的方法hook进DI. 这两件东西在ASP.NET Core中被整合了, 因此只需构建一处依赖解析器, 只需维护一份配置.
* **控制器不再从容器中解析; 只有控制器构造方法.** 这意味着控制器生命周期, 属性注入, 和其他的事不再归Autofac管理 - 它们归ASP.NET Core管理. 你可以使用 ``AddControllersAsServices()`` 改变 - 见下面的讨论.

控制器作为服务
=======================

默认地, ASP.NET Core 会从容器中解析控制器 *参数* 但不会从中解析 *控制器* . 这不是个问题但它意味着:

* *控制器* 的生命周期归框架管理, 而非请求生命周期.
* *控制器构造方法参数* 归请求生命周期管理.
* 在控制器注册时做的特别的连结 (如属性注入) 将不会生效.

你可以通过在用service collection注册MVC时指定 ``AddControllersAsServices()`` 来改变. 这么做可以在调用 ``builder.Populate(services)`` 时自动注册控制器类型到 ``IServiceCollection`` .

.. sourcecode:: csharp

    public class Startup
    {
      // Omitting extra stuff so you can see the important part...
      public IServiceProvider ConfigureServices(IServiceCollection services)
      {
        // Add controllers as services so they'll be resolved.
        services.AddMvc().AddControllersAsServices();

        var builder = new ContainerBuilder();

        // When you do service population, it will include your controller
        // types automatically.
        builder.Populate(services);

        // If you want to set up a controller for, say, property injection
        // you can override the controller registration after populating services.
        builder.RegisterType<MyController>().PropertiesAutowired();

        this.ApplicationContainer = builder.Build();
        return new AutofacServiceProvider(this.ApplicationContainer);
      }
    }

这里有一篇更详尽的文章 `Filip Woj's blog <http://www.strathweb.com/2016/03/the-subtle-perils-of-controller-dependency-injection-in-asp-net-core-mvc/>`_. 注意其中的一位评论者 `发现RC2中把控制器作为服务时发生了一些改变 <http://www.strathweb.com/2016/03/the-subtle-perils-of-controller-dependency-injection-in-asp-net-core-mvc/#comment-2702995712>`_.

Multitenant Support
===================

Due to the way ASP.NET Core is eager about generating the request lifetime scope it causes multitenant support to not quite work out of the box. Sometimes the ``IHttpContextAccessor``, commonly used in tenant identification, also isn't set up in time. The `Autofac.AspNetCore.Multitenant <https://github.com/autofac/Autofac.AspNetCore.Multitenant>`_ package was added to fix that.

To enable multitenant support:

* Add a reference to the ``Autofac.AspNetCore.Multitenant`` NuGet package.
* In your ``Program.Main`` when building the web host...

  * Include a call to the ``UseAutofacMultitenantRequestServices`` extension and let Autofac know how to locate your multitenant container.
  * **Do not use** the ``ConfigureContainer`` support listed above. You can't do that because it won't give you a chance to create your multitenant container.

* Change your ``Startup.ConfigureServices`` method to return ``IServiceProvider``, create your multitenant container, and return an ``AutofacServiceProvider`` using that container.

Here's an example of what you do in ``Program.Main``:

.. sourcecode:: csharp

    public class Program
    {
      public static void Main(string[] args)
      {
        var host = new WebHostBuilder()
          .UseKestrel()
          .UseContentRoot(Directory.GetCurrentDirectory())

          // This enables the request lifetime scope to be properly spawned from
          // the container rather than be a child of the default tenant scope.
          // The ApplicationContainer static property is where the multitenant container
          // will be stored once it's built.
          .UseAutofacMultitenantRequestServices(() => Startup.ApplicationContainer)
          .UseIISIntegration()
          .UseStartup<Startup>()
          .Build();

        host.Run();
      }
    }

...and here's what ``Startup`` looks like:

.. sourcecode:: csharp

    public class Startup
    {
      // Omitting extra stuff so you can see the important part...
      public IServiceProvider ConfigureServices(IServiceCollection services)
      {
        services.AddMvc();
        var builder = new ContainerBuilder();
        builder.Populate(services);

        var container = builder.Build();
        var strategy = new MyTenantIdentificationStrategy();
        var mtc = new MultitenantContainer(strategy, container);
        Startup.ApplicationContainer = mtc;
        return new AutofacServiceProvider(mtc);
      }

      // This is what the middleware will use to create your request lifetime scope.
      public static MultitenantContainer ApplicationContainer { get; set; }
    }


Using a Child Scope as a Root
=============================

In a complex application you may want to keep services registered using ``Populate()`` in a child lifetime scope. For example, an application that does some self-hosting of ASP.NET Core components may want to keep the MVC registrations and such isolated from the main container. The ``Populate()`` method offers an overload to allow you to specify a tagged child lifetime scope that should serve as the "container" for items.

.. note::

   If you use this, you will not be able to use the ASP.NET Core support for ``IServiceProviderFactory{TContainerBuilder}`` (the ``ConfigureContainer`` support). This is because ``IServiceProviderFactory{TContainerBuilder}`` assumes it's working at the root level.

:doc:`The .NET Core integration documentation shows an example of using a child lifetime scope as a root. <netcore>`

Example
=======

There is an example project showing ASP.NET Core integration `in the Autofac examples repository <https://github.com/autofac/Examples/tree/master/src/AspNetCoreExample>`_.

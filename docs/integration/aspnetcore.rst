============
ASP.NET Core
============

ASP.NET Core (previously ASP.NET 5) 改变了以前依赖注入框架集成进ASP.NET的方法. 以前, 每个功能 - MVC, Web API, 等. - 都有它自己的 "依赖解析器(dependency resolver)" 机制并且只是'钩子'钩住的方式有些轻微的区别. ASP.NET Core 通过 `Microsoft.Extensions.DependencyInjection <https://github.com/aspnet/DependencyInjection>`_ 引入了 `conforming container <http://blog.ploeh.dk/2014/05/19/conforming-container/>`_ 机制, 包含了请求生命周期作用域, 服务注册等等的统一概念.

在 ASP.NET Core 3.0, 引入了 "generic app hosting" 机制, 它可以应用在非 ASP.NET Core 应用中.

**本章节解释了ASP.NET Core集成和 generic .NET Core hosting 集成.** 如果你正在使用传统的ASP.NET, :doc:`请见传统ASP.NET集成章节 <aspnet>`.

如果你只是使用.NET Core而没有用ASP.NET Core, :doc:`这里有一个更简单的示例 <netcore>` 展示了它的集成方式.

.. contents::
  :local:

入门
=======

* Nuget引入 ``Autofac.Extensions.DependencyInjection`` 包.
* 在你的 ``Program.Main`` 方法内, 将Autofac附加给托管机制.(见下例)
* 在 ``Startup`` 类的 ``ConfigureServices`` 方法中用其他库提供的扩展方法注册东西到 ``IServiceCollection`` .
* 在 ``Startup`` 类的 ``ConfigureContainer`` 方法中直接注册东西到Autofac ``ContainerBuilder``.

``IServiceProvider`` 会自动替你创建, 因此你无需做任何事只要 *注册东西* 即可.

ASP.NET Core 1.1 - 2.2
----------------------

下面的示例演示了 **ASP.NET Core 1.1 - 2.2** 使用方法, 你可以调用 ``WebHostBuilder`` 的 ``services.AddAutofac()``. **这不适用于ASP.NET Core 3+** 或 .NET Core 3+ generic hosting 支持 - ASP.NET Core 3 需要你直接指定一个service provider factory而不是把它加入进service collection.

.. sourcecode:: csharp

    public class Program
    {
      public static void Main(string[] args)
      {
        // ASP.NET Core 1.1 - 2.2:
        // The ConfigureServices call here allows for
        // ConfigureContainer to be supported in Startup with
        // a strongly-typed ContainerBuilder.
        // AddAutofac() is a convenience method for
        // services.AddSingleton<IServiceProviderFactory<ContainerBuilder>>(new AutofacServiceProviderFactory())
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

下面的示例演示了 **ASP.NET Core 1.1 - 2.2** 使用方法, 你可以从 ``ConfigureServices(IServiceCollection services)`` 返回一个 ``IServiceProvider``. **这不适用于ASP.NET Core 3+** 或 .NET Core 3+ generic hosting 支持 - ASP.NET Core 3 已经废弃了从 ``ConfigureServices`` 返回service provider的能力.

.. sourcecode:: csharp

    public class Startup
    {
      public Startup(IHostingEnvironment env)
      {
        // In ASP.NET Core 3.0 env will be an IWebHostEnvironment , not IHostingEnvironment.
        var builder = new ConfigurationBuilder()
            .SetBasePath(env.ContentRootPath)
            .AddJsonFile("appsettings.json", optional: true, reloadOnChange: true)
            .AddJsonFile($"appsettings.{env.EnvironmentName}.json", optional: true)
            .AddEnvironmentVariables();
        this.Configuration = builder.Build();
      }

      public IConfigurationRoot Configuration { get; private set; }

      public ILifetimeScope AutofacContainer { get; private set; }

      // ConfigureServices is where you register dependencies and return an `IServiceProvider` implemented by `AutofacServiceProvider`.
      // This is the old, not recommended way, and is NOT SUPPORTED in ASP.NET Core 3.0+.
      public IServiceProvider ConfigureServices(IServiceCollection services)
      {
        // Add services to the collection
        services.AddOptions();

        // Create a container-builder and register dependencies
        var builder = new ContainerBuilder();

        // Populate the service-descriptors added to `IServiceCollection`
        // BEFORE you add things to Autofac so that the Autofac
        // registrations can override stuff in the `IServiceCollection`
        // as needed
        builder.Populate(services);

        // Register your own things directly with Autofac
        builder.RegisterModule(new MyApplicationModule());

        AutofacContainer = builder.Build();

        // this will be used as the service-provider for the application!
        return new AutofacServiceProvider(AutofacContainer);
      }

      // Configure is where you add middleware.
      // You can use IApplicationBuilder.ApplicationServices
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

ASP.NET Core 3.0+ and Generic Hosting
-------------------------------------

**在ASP.NET Core 3.0托管方式发生了变化** 并且需要不同的集成方式. 你不能在从 ``ConfigureServices`` 中返回 ``IServiceProvider``, 也不能再将你的service provider factory加入到service collection.

下面是ASP.NET Core 3+ 和 .NET Core 3+ generic hosting support的集成方式:

.. sourcecode:: csharp

    public class Program
    {
      public static void Main(string[] args)
      {
        // ASP.NET Core 3.0+:
        // The UseServiceProviderFactory call attaches the
        // Autofac provider to the generic hosting mechanism.
        var host = Host.CreateDefaultBuilder(args)
            .UseServiceProviderFactory(new AutofacServiceProviderFactory())
            .ConfigureWebHostDefaults(webHostBuilder => {
              webHostBuilder
                .UseContentRoot(Directory.GetCurrentDirectory())
                .UseIISIntegration()
                .UseStartup<Startup>();
            })
            .Build();

        host.Run();
      }
    }

Startup类
-------------

在你的Startup类中 (各版本ASP.NET Core基本一致) 你可以使用 ``ConfigureContainer`` 访问 Autofac container builder 并且直接使用Autofac注册东西.

.. sourcecode:: csharp

    public class Startup
    {
      public Startup(IHostingEnvironment env)
      {
        // In ASP.NET Core 3.0 `env` will be an IWebHostEnvironment, not IHostingEnvironment.
        var builder = new ConfigurationBuilder()
            .SetBasePath(env.ContentRootPath)
            .AddJsonFile("appsettings.json", optional: true, reloadOnChange: true)
            .AddJsonFile($"appsettings.{env.EnvironmentName}.json", optional: true)
            .AddEnvironmentVariables();
        this.Configuration = builder.Build();
      }

      public IConfigurationRoot Configuration { get; private set; }

      public ILifetimeScope AutofacContainer { get; private set; }

      // ConfigureServices is where you register dependencies. This gets
      // called by the runtime before the ConfigureContainer method, below.
      public void ConfigureServices(IServiceCollection services)
      {
        // Add services to the collection. Don't build or return
        // any IServiceProvider or the ConfigureContainer method
        // won't get called.
        services.AddOptions();
      }

      // ConfigureContainer is where you can register things directly
      // with Autofac. This runs after ConfigureServices so the things
      // here will override registrations made in ConfigureServices.
      // Don't build the container; that gets done for you by the factory.
      public void ConfigureContainer(ContainerBuilder builder)
      {
        // Register your own things directly with Autofac, like:
        builder.RegisterModule(new MyApplicationModule());
      }

      // Configure is where you add middleware. This is called after
      // ConfigureContainer. You can use IApplicationBuilder.ApplicationServices
      // here if you need to resolve things from the container.
      public void Configure(
        IApplicationBuilder app,
        ILoggerFactory loggerFactory)
      {
        // If, for some reason, you need a reference to the built container, you
        // can use the convenience extension method GetAutofacRoot.
        this.AutofacContainer = app.ApplicationServices.GetAutofacRoot();

        loggerFactory.AddConsole(this.Configuration.GetSection("Logging"));
        loggerFactory.AddDebug();
        app.UseMvc();
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

这是ASP.NET Core应用托管的一个功能 - 它并不是Autofac的行为. `ASP.NET Core中的StartupLoader类 <https://github.com/aspnet/Hosting/blob/rel/1.1.0/src/Microsoft.AspNetCore.Hosting/Internal/StartupLoader.cs>`_ 是在应用启动时定位调用方法的. 如果你想更深层次地了解它是如何运作的, 可以看下该类.

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

* **使用InstancePerLifetimeScope(每个生命周期作用域一个实例)而不是InstancePerRequest(每个请求一个实例).** 以前的ASP.NET集成你可以注册依赖为 ``InstancePerRequest`` , 能保证每次HTTP请求只有唯一的依赖实例被创建. 这是有用的因为Autofac负责 :doc:`建立每个请求生命周期作用域 <../faq/per-request-scope>`. 随着 ``Microsoft.Extensions.DependencyInjection`` 的引入, 每个请求和其他子生命周期作用域的创建现在是框架提供的 `conforming container <http://blog.ploeh.dk/2014/05/19/conforming-container/>`_ 的一部分, 因此所有的子生命周期作用域是被同等对待的 - 现在已经不再有特别的 "请求级别作用" . 现在不再注册你的依赖为 ``InstancePerRequest``, 而使用 ``InstancePerLifetimeScope`` , 你也可以得到相同的行为. 注意如果你在web请求中创建 *你自己的生命周期作用域* , 你将会在这些子作用域中得到新的实例.
* **不再需要依赖解析器(DependencyResolver).** 其他ASP.NET集成机制在许多地方需要创建基于Autofac的自定义依赖解析器. 使用 ``Microsoft.Extensions.DependencyInjection`` 和 ``Startup.ConfigureServices`` 方法, 你现在只要返回 ``IServiceProvider`` , "神奇的事就发生了." 在控制器, 类等内部. 如果你需要手动定位服务, 拿 ``IServiceProvider`` 即可.
* **没有特殊的中间件.** 以前的 :doc:`OWIN集成 <owin>` 需要特殊的Autofac中间件的注册, 用来管理请求生命周期作用域. ``Microsoft.Extensions.DependencyInjection`` 现在做了这些繁重的工作, 因此现在不需要注册额外的中间件了.
* **不再需要手动注册控制器.** 你以前需要用Autofac手动注册所有的控制器这样DI才会work. ASP.NET Core框架现在在服务解析过程中将自动传入所有控制器, 因此你不必手动注册.
* **没有通过依赖注入触发中间件的扩展方法.** :doc:`OWIN集成 <owin>` 有类似 ``UseAutofacMiddleware()`` 的扩展方法来允许依赖注入进入中间件内. 现在这些通过结合 `自动注入构造方法参数和动态解析中间件Invoke方法的参数 <https://docs.asp.net/en/latest/fundamentals/middleware.html>`_ , 都能自动完成, . ASP.NET Core框架负责了所有的这些事.
* **MVC 和 Web API 现在是一个东西了.** 以前根据你是使用 MVC 还是 Web API ,有不同的方法hook进DI. 这两件东西在ASP.NET Core中被整合了, 因此只需构建一处依赖解析器, 只需维护一份配置.
* **控制器不再从容器中解析; 只有控制器构造方法.** 这意味着控制器生命周期, 属性注入, 和其他的事不再归Autofac管理 - 它们归ASP.NET Core管理. 你可以使用 ``AddControllersAsServices()`` 改变 - 见下面的讨论.

控制器作为服务
=======================

默认地, ASP.NET Core 会从容器中解析控制器 *参数* 但不会从中解析 *控制器* . 这并不是个问题但它意味着:

* *控制器* 的生命周期归框架管理, 而非请求生命周期.
* *控制器构造方法参数* 归请求生命周期管理.
* 在控制器注册时做的特别的连结 (如属性注入) 将不会生效.

通过在用service collection注册MVC时指定 ``AddControllersAsServices()`` , 你可以改变这个行为. 这么做可以在service provider factory调用 ``builder.Populate(services)`` 时自动注册控制器类型到 ``IServiceCollection``.

.. sourcecode:: csharp

    public class Startup
    {
      // Omitting extra stuff so you can see the important part...
      public void ConfigureServices(IServiceCollection services)
      {
        // Add controllers as services so they'll be resolved.
        services.AddMvc().AddControllersAsServices();
      }

      public void ConfigureContainer(ContainerBuilder builder)
      {
        // If you want to set up a controller for, say, property injection
        // you can override the controller registration after populating services.
        builder.RegisterType<MyController>().PropertiesAutowired();
      }
    }

这里有一篇更详尽的文章 `Filip Woj's blog <http://www.strathweb.com/2016/03/the-subtle-perils-of-controller-dependency-injection-in-asp-net-core-mvc/>`_. 注意其中的一位评论者 `发现RC2中把控制器作为服务时发生了一些改变 <http://www.strathweb.com/2016/03/the-subtle-perils-of-controller-dependency-injection-in-asp-net-core-mvc/#comment-2702995712>`_.

多租户支持
===================

由于ASP.NET Core想要早早地生成请求生命周期作用域, 这会导致多租户支持无法达到开箱即用的效果. 有时用于识别租户身份的 ``IHttpContextAccessor`` , 也无法被及时地构建. `Autofac.AspNetCore.Multitenant <https://github.com/autofac/Autofac.AspNetCore.Multitenant>`_ 包就是用于解决这个问题的.

为了启用多租户支持:

* 添加 ``Autofac.AspNetCore.Multitenant`` NuGet包引用.
* 在 ``Program.Main`` 中构建web host时调用 ``UseServiceProviderFactory`` 和 ``AutofacMultitenantServiceProviderFactory``. 提供一个配置租户的回调.
* 在 ``Startup.ConfigureServices`` 和 ``Startup.ConfigureContainer`` 中注册进入 **根容器(root container)** 的东西(那些非租户特有的).
* 在回调中 (如``Startup.ConfigureMultitenantContainer``) 构建你的多租户容器.

下面是 ``Program.Main`` 的一个示例:

.. sourcecode:: csharp

    public class Program
    {
      public static async Task Main(string[] args)
      {
        var host = Host
          .CreateDefaultBuilder(args)
          .UseServiceProviderFactory(new AutofacMultitenantServiceProviderFactory(Startup.ConfigureMultitenantContainer))
          .ConfigureWebHostDefaults(webHostBuilder => webHostBuilder.UseStartup<Startup>())
          .Build();

        await host.RunAsync();
      }
    }

... ``Startup`` 类似这样:

.. sourcecode:: csharp

    public class Startup
    {
      // Omitting extra stuff so you can see the important part...
      public void ConfigureServices(IServiceCollection services)
      {
        // This will all go in the ROOT CONTAINER and is NOT TENANT SPECIFIC.
        services.AddMvc();

        // This adds the required middleware to the ROOT CONTAINER and is required for multitenancy to work.
        services.AddAutofacMultitenantRequestServices();
      }

      public void ConfigureContainer(ContainerBuilder builder)
      {
        // This will all go in the ROOT CONTAINER and is NOT TENANT SPECIFIC.
        builder.RegisterType<Dependency>().As<IDependency>();
      }

      public static MultitenantContainer ConfigureMultitenantContainer(IContainer container)
      {
        // This is the MULTITENANT PART. Set up your tenant-specific stuff here.
        var strategy = new MyTenantIdentificationStrategy();
        var mtc = new MultitenantContainer(strategy, container);
        mtc.ConfigureTenant("a", cb => cb.RegisterType<TenantDependency>().As<IDependency>());
        return mtc;
      }
    }


Using a Child Scope as a Root
=============================

In a complex application you may want to keep services partitioned such that the root container is shared across different parts of the app, but a child lifetime scope is used for the hosted portion (e.g., the ASP.NET Core piece).

In standard ASP.NET Core integration and generic hosted application support there's an ``AutofacChildLifetimeScopeServiceProviderFactory`` you can use instead of the standard ``AutofacServiceProviderFactory``. This allows you to provide configuration actions that will be attached to a specific named lifetime scope rather than a built container.

.. sourcecode:: csharp

    public class Program
    {
      public static async Task Main(string[] args)
      {
        // create the root-container and register global dependencies
        var containerBuilder = new ContainerBuilder();

        builder.RegisterType<SomeGlobalDependency>()
          .As<ISomeGlobalDependency>()
          .InstancePerLifetimeScope();

        var container = containerBuilder.Build();

        // The UseServiceProviderFactory call attaches the
        // Autofac provider to the generic hosting mechanism.
          var hostOne = Host
            .CreateDefaultBuilder(args)
            .UseServiceProviderFactory(new AutofacChildLifetimeScopeServiceProviderFactory(container.BeginLifetimeScope("root-one")))
            .ConfigureWebHostDefaults(webHostBuilder => {
              webHostBuilder
                .UseContentRoot(AppContext.BaseDirectory)
                // Each host listens to a different URL, they have the same root container to share SingleInstance
                // things, but they each have  their own logical root lifetime scope. Registering things
                // as `InstancePerMatchingLifetimeScope("root-one")` (the name of the scope given above)
                // will result in a singleton that's ONLY used by this first host.
                .UseUrls("http://localhost:5000")
                .UseStartup<StartupOne>();
            })
            .Build();

        // The UseServiceProviderFactory call attaches the
        // Autofac provider to the generic hosting mechanism.
          var hostTwo = Host
            .CreateDefaultBuilder(args)
            .UseServiceProviderFactory(new AutofacChildLifetimeScopeServiceProviderFactory(container.BeginLifetimeScope("root-two")))
            .ConfigureWebHostDefaults(webHostBuilder => {
              webHostBuilder
                .UseContentRoot(AppContext.BaseDirectory)
                // As with the first host, the second host will share the root container but have its own
                // root lifetime scope `root-two`. Things registered `InstancePerMatchingLifetimeScope("root-two")`
                // will be singletons ONLY used by this second host.
                .UseUrls("http://localhost:5001")
                .UseStartup<StartupTwo>();
            })
            .Build();

        await Task.WhenAll(hostOne.RunAsync(), hostTwo.RunAsync())
      }
    }

This will change how your ``Startup`` class works - you won't use a ``ContainerBuilder`` directly in ``ConfigureContainer``, now it's an ``AutofacChildLifetimeScopeConfigurationAdapter``:

.. sourcecode:: csharp

    public class StartupOne
    {
      // IHostingEnvironment when running applications below ASP.NET Core 3.0
      public Startup(IWebHostEnvironment env)
      {
        // Fill this in if needed...
      }

      public void ConfigureServices(IServiceCollection services)
      {
        // The usual ConfigureServices registrations on the service collection...
      }

      // Here's the change for child lifetime scope usage! Register your "root"
      // child lifetime scope things with the adapter.
      public void ConfigureContainer(AutofacChildLifetimeScopeConfigurationAdapter config)
      {
          config.Add(builder => builder.RegisterModule(new AutofacHostOneModule()));
      }

      public void Configure(
        IApplicationBuilder app,
        ILoggerFactory loggerFactory)
      {
          // The usual app configuration stuff...
      }
    }

    public class StartupTwo
    {
      // IHostingEnvironment when running applications below ASP.NET Core 3.0
      public Startup(IWebHostEnvironment env)
      {
        // Fill this in if needed...
      }

      public void ConfigureServices(IServiceCollection services)
      {
        // The usual ConfigureServices registrations on the service collection...
      }

      // Here's the change for child lifetime scope usage! Register your "root"
      // child lifetime scope things with the adapter.
      public void ConfigureContainer(AutofacChildLifetimeScopeConfigurationAdapter config)
      {
          config.Add(builder => builder.RegisterModule(new AutofacHostTwoModule()));
      }

      public void Configure(
        IApplicationBuilder app,
        ILoggerFactory loggerFactory)
      {
          // The usual app configuration stuff...
      }
    }


If you're not using the service provider factory, the ``Populate()`` method offers an overload to allow you to specify a tagged child lifetime scope that should serve as the "container" for items.

:doc:`The .NET Core integration documentation also shows an example of using a child lifetime scope as a root. <netcore>`

Using a child lifetime scope as the root is not compatible with multitenant support. You must choose one or the other, not both.

示例
=======

`Autofac示例代码仓库 <https://github.com/autofac/Examples/tree/master/src/AspNetCoreExample>`_ 里有一个展示了ASP.NET Core集成的示例项目.

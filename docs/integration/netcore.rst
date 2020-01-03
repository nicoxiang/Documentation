============
.NET Core
============

.NET Core 通过 `Microsoft.Extensions.DependencyInjection <https://github.com/aspnet/DependencyInjection>`_ 引入了 `conforming container <http://blog.ploeh.dk/2014/05/19/conforming-container/>`_ . ``Autofac.Extensions.DependencyInjection`` 包实现了它的抽象以此来通过Autofac提供依赖注入.

集成 :doc:`ASP.NET Core <aspnetcore>` 和它非常相似因为整个框架统一了抽象和依赖注入. :doc:`ASP.NET Core 集成章节<aspnetcore>` 在ASP.NET Core (和 generically hosted application) 使用的相关主题内容方面有更详细的信息.

.. contents::
  :local:

入门
===========

通过 ``Microsoft.Extensions.DependencyInjection`` 包在你的.NET Core应用中使用Autofac:

* 从NuGet引入 ``Autofac.Extensions.DependencyInjection`` 包.
* 在应用startup处 (如, 在 ``Program`` 或  ``Startup`` 类)...

  - 使用框架扩展方法注册服务进 ``IServiceCollection`` .
  - 将这些服务填充到Autofac中.
  - 添加Autofac注册并且重写.
  - 创建容器.
  - 使用容器创建 ``AutofacServiceProvider`` .

.. sourcecode:: csharp

    public class Program
    {
      public static void Main(string[] args)
      {
        // The Microsoft.Extensions.DependencyInjection.ServiceCollection
        // has extension methods provided by other .NET Core libraries to
        // regsiter services with DI.
        var serviceCollection = new ServiceCollection();

        // The Microsoft.Extensions.Logging package provides this one-liner
        // to add logging services.
        serviceCollection.AddLogging();

        var containerBuilder = new ContainerBuilder();

        // Once you've registered everything in the ServiceCollection, call
        // Populate to bring those registrations into Autofac. This is
        // just like a foreach over the list of things in the collection
        // to add them to Autofac.
        containerBuilder.Populate(serviceCollection);

        // Make your Autofac registrations. Order is important!
        // If you make them BEFORE you call Populate, then the
        // registrations in the ServiceCollection will override Autofac
        // registrations; if you make them AFTER Populate, the Autofac
        // registrations will override. You can make registrations
        // before or after Populate, however you choose.
        containerBuilder.RegisterType<MessageHandler>().As<IHandler>();

        // Creating a new AutofacServiceProvider makes the container
        // available to your app using the Microsoft IServiceProvider
        // interface so you can use those abstractions rather than
        // binding directly to Autofac.
        var container = containerBuilder.Build();
        var serviceProvider = new AutofacServiceProvider(container);
      }
    }

**你并不是必须要使用Microsoft.Extensions.DependencyInjection.** 如果你在写一个不需要它的.NET Core应用或者你不在使用其他库提供的依赖注入扩展, 你可以直接Autofac. 也许你只需要调用 ``Populate()`` 而不需要 ``AutofacServiceProvider``. 使用对你的应用有用的模块.

把子作用域作为根作用域
=============================

在一个复杂的应用中你也许会想要在子生命周期作用域中用 ``Populate()`` 注册服务. 例如, 一个有ASP.NET Core组件自托管的应用也许会想要使MVC注册等等和主容器独立起来. ``Populate()`` 方法提供了一个重载可以允许你指定一个带标签的子生命周期作用域, 让它成为MVC注册等等这些的 "容器".

.. note::

   如果这么做, 你将会无法使用ASP.NET Core ``IServiceProviderFactory{TContainerBuilder}`` 的支持 ( ``ConfigureContainer`` 支持). 因为 ``IServiceProviderFactory{TContainerBuilder}`` 假设它是工作在根级别的.

.. sourcecode:: csharp

    public class Program
    {
      private const string RootLifetimeTag = "MyIsolatedRoot";

      public static void Main(string[] args)
      {
        var serviceCollection = new ServiceCollection();
        serviceCollection.AddLogging();

        var containerBuilder = new ContainerBuilder();
        containerBuilder.RegisterType<MessageHandler>().As<IHandler>();
        var container = containerBuilder.Build();

        using(var scope = container.BeginLifetimeScope(RootLifetimeTag, b =>
        {
          b.Populate(serviceCollection, RootLifetimeTag);
        }))
        {
          // This service provider will have access to global singletons
          // and registrations but the "singletons" for things registered
          // in the service collection will be "rooted" under this
          // child scope, unavailable to the rest of the application.
          //
          // Obviously it's not super helpful being in this using block,
          // so likely you'll create the scope at app startup, keep it
          // around during the app lifetime, and dispose of it manually
          // yourself during app shutdown.
          var serviceProvider = new AutofacServiceProvider(scope);
        }
      }
    }

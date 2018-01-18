====
OWIN
====

`OWIN (Open Web Interface for .NET) <http://owin.org/>`_ 是一种更简单的构建web应用的模式, 它不需要将应用和web服务器捆绑在一起. 为了做到这一点, 我们引入了"中间件"的概念, 用来创建请求传输的管道.

由于OWIN处理应用管道(探测请求出入等)的方式有所不同, 它集成Autofac的方式和集成更"基础"的ASP.NET应用的方式也有所不同. `你可以在此阅读OWIN和它如何工作的内容. <http://www.asp.net/aspnet/overview/owin-and-katana/an-overview-of-project-katana>`_

**有个需要记住的关键点是OWIN中间件的注册顺序非常重要.** 中间件会以注册的顺序被处理, 就像一个链子一样, 因此你需要首先注册基础的东西(例如Autofac中间件).

入门
===========

在你的OWIN管道中使用Autofac, 你需要:

* 引入 ``Autofac.Owin`` NuGet包.
* 创建你的Autofac容器.
* 在OWIN中注册Autofac中间件并将容器传递给它.

.. sourcecode:: csharp

    public class Startup
    {
      public void Configuration(IAppBuilder app)
      {
        var builder = new ContainerBuilder();
        // Register dependencies, then...
        var container = builder.Build();

        // Register the Autofac middleware FIRST. This also adds
        // Autofac-injected middleware registered with the container.
        app.UseAutofacMiddleware(container);

        // ...then register your other middleware not registered
        // with Autofac.
      }
    }

查看其他独立的 :doc:`ASP.NET集成库 <aspnet>` 页面, 了解不同应用类型和它们如何处理OWIN支持的详情.

中间件的依赖注入
==================================

通常当你在应用中注册OWIN中间件时, 你会使用到中间件附带的扩展方法. 例如 :doc:`Web API <webapi>` 有 ``app.UseWebApi(config);`` 扩展方法. 这种中间件的注册方式都是定义为静态的, 将无法注入依赖.

对于自定义中间件, 你可以允许Autofa通过注册中间件到容器来注入依赖进中间件, 而不是用静态扩展方法注册中间件.

.. sourcecode:: csharp

    var builder = new ContainerBuilder();
    builder.RegisterType<MyCustomMiddleware>();
    //...
    var container = builder.Build();

    // This will add the Autofac middleware as well as the middleware
    // registered in the container.
    app.UseAutofacMiddleware(container);

当你调用 ``app.UseAutofacMiddleware(container);`` Autofac中间件本身将会被加入到管道中, 任何注册在容器中的 ``Microsoft.Owin.OwinMiddleware`` 类也会在它之后被加入到管道中.

这种方式注册的中间件, 在每次有请求经过OWIN管道时, 将会从请求生命周期作用域被解析.

控制中间件顺序
============================

对于简单的场景来说, ``app.UseAutofacMiddleware(container);`` 将会同时完成添加Autofac生命周期到OWIN请求作用域, 和添加以Autofac注册的中间件到管道中两件事.

如果你想要对允许了依赖注入的(DI-enabled)中间件有更精确的控制, 你可以使用 ``UseAutofacLifetimeScopeInjector`` 和 ``UseMiddlewareFromContainer`` 扩展方法.

.. sourcecode:: csharp

    var builder = new ContainerBuilder();
    builder.RegisterType<MyCustomMiddleware>();
    //...
    var container = builder.Build();

    // This adds ONLY the Autofac lifetime scope to the pipeline.
    app.UseAutofacLifetimeScopeInjector(container);

    // Now you can add middleware from the container into the pipeline
    // wherever you like. For example, this adds custom DI-enabled middleware
    // AFTER the Web API middleware/handling.
    app.UseWebApi(config);
    app.UseMiddlewareFromContainer<MyCustomMiddleware>();

示例
=======

`Autofac示例代码仓库 <https://github.com/autofac/Examples/tree/master/src/WebApiExample.OwinSelfHost>`_ 里有一个展示了Web API结合OWIN自托管的示例项目.
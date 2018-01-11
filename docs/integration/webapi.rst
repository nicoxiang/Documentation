=======
Web API
=======

Web API 2 集成需要 `Autofac.WebApi2 NuGet 包 <https://www.nuget.org/packages/Autofac.WebApi2>`_.

Web API 集成需要 `Autofac.WebApi NuGet 包 <https://www.nuget.org/packages/Autofac.WebApi/>`_.

Web API 集成提供了controllers, model binders和 action filters的依赖注入. 它同样添加了 :doc:`每个请求生命周期支持 <../faq/per-request-scope>`.

**本章节解释了 ASP.NET classic Web API 集成.** 如果你使用ASP.NET Core, :doc:`见ASP.NET Core集成章节 <aspnetcore>`.

.. contents::
  :local:

入门
===========
为了把Autofac集成进Web API你需要引用Web API integration NuGet package, 注册控制器, 设置依赖解析器(Dependency Resolver). 你可以选择性地启用其他功能.

.. sourcecode:: csharp

    protected void Application_Start()
    {
      var builder = new ContainerBuilder();

      // Get your HttpConfiguration.
      var config = GlobalConfiguration.Configuration;

      // Register your Web API controllers.
      builder.RegisterApiControllers(Assembly.GetExecutingAssembly());

      // OPTIONAL: Register the Autofac filter provider.
      builder.RegisterWebApiFilterProvider(config);

      // OPTIONAL: Register the Autofac model binder provider.
      builder.RegisterWebApiModelBinderProvider();

      // Set the dependency resolver to be Autofac.
      var container = builder.Build();
      config.DependencyResolver = new AutofacWebApiDependencyResolver(container);
    }

下面的内容进一步详细解释了其他功能和如何使用它们.

获取HttpConfiguration
=========================

在Web API中, 构建应用需要设置 ``HttpConfiguration`` 对象的值, 更新它的属性. 你获取配置(configuration)的地方会根据你如何托管应用而不同. 在文档中, 我们将提到"你的 ``HttpConfiguration``", 你需要作出选择如何获取它.

基础 IIS 托管, ``HttpConfiguration`` 就是 ``GlobalConfiguration.Configuration``.

.. sourcecode:: csharp

    var builder = new ContainerBuilder();
    var config = GlobalConfiguration.Configuration;
    builder.RegisterApiControllers(Assembly.GetExecutingAssembly());
    var container = builder.Build();
    config.DependencyResolver = new AutofacWebApiDependencyResolver(container);

自托管, ``HttpConfiguration`` 就是你的 ``HttpSelfHostConfiguration`` 实例.

.. sourcecode:: csharp

    var builder = new ContainerBuilder();
    var config = new HttpSelfHostConfiguration("http://localhost:8080");
    builder.RegisterApiControllers(Assembly.GetExecutingAssembly());
    var container = builder.Build();
    config.DependencyResolver = new AutofacWebApiDependencyResolver(container);

而对于OWIN集成, ``HttpConfiguration`` 是在你应用startup类创建的, 并会传到Web API中间件.

.. sourcecode:: csharp

    var builder = new ContainerBuilder();
    var config = new HttpConfiguration();
    builder.RegisterApiControllers(Assembly.GetExecutingAssembly());
    var container = builder.Build();
    config.DependencyResolver = new AutofacWebApiDependencyResolver(container);

注册控制器
====================

在应用startup的地方, 当你创建Autofac容器时, 你应该注册你的MVC控制器和它们的依赖. 这通常发生在OWIN startup类或在 ``Global.asax`` 的 ``Application_Start`` 方法中.

默认地实现 ``IHttpController`` 且名称以 ``Controller`` 为后缀的类将会被注册.

.. sourcecode:: csharp

    var builder = new ContainerBuilder();

    // You can register controllers all at once using assembly scanning...
    builder.RegisterApiControllers(Assembly.GetExecutingAssembly());

    // ...or you can register individual controlllers manually.
    builder.RegisterType<ValuesController>().InstancePerRequest();

如果你的控制器并不遵循常规命名规则, 你应该选择使用 ``RegisterApiControllers`` 方法的重载提供一个自定义的后缀.

.. sourcecode:: csharp

    // You can also use assembly scanning to register controllers with a custom suffix.
    builder.RegisterApiControllers("MyCustomSuffix", Assembly.GetExecutingAssembly());

设置依赖解析器(Dependency Resolver)
===================================

创建完你的容器后, 把它传入到一个新建的 ``AutofacWebApiDependencyResolver`` 类的实例中. 把这个新的解析器(resolver)附加到你的 ``HttpConfiguration.DependencyResolver`` 来让Web API知道它应该使用 ``AutofacWebApiDependencyResolver`` 来定位服务. 这是Autofac对于 ``IDependencyResolver`` 接口的实现.

.. sourcecode:: csharp

    var container = builder.Build();
    config.DependencyResolver = new AutofacWebApiDependencyResolver(container);

通过依赖注入提供过滤器
========================================
因为特性(attributes)通过反射API创建, 你不能自己调用构造方法. 这就使得你在使用特性除了属性注入没有了其他选择. Autofac Web API集成提供了一种机制, 允许你创建实现过滤器接口 (``IAutofacActionFilter``, ``IAutofacAuthorizationFilter`` 和 ``IAutofacExceptionFilter``)  的类, 然后就可以通过使用容器构造器(container builder)的注册语法将它们和需要的控制器或action方法连接起来.

注册过滤器提供者(Filter Provider)
---------------------------------

你需要实现注册过滤器提供者因为它做了基于注册的方式连接过滤器的工作. 可以通过调用容器构造器的 ``RegisterWebApiFilterProvider`` 方法和提供一个 ``HttpConfiguration`` 实例完成.

.. sourcecode:: csharp

    var builder = new ContainerBuilder();
    builder.RegisterWebApiFilterProvider(config);

实现过滤器接口
------------------------------

你的类需要继承自集成中定义的适当的过滤器接口, 而不是原生Web API框架中的过滤器特性(filter attributes). 下面的过滤器是一个action filter并实现了 ``IAutofacActionFilter`` 而不是 ``System.Web.Http.Filters.IActionFilter``.

.. sourcecode:: csharp

    public class LoggingActionFilter : IAutofacActionFilter
    {
      readonly ILogger _logger;

      public LoggingActionFilter(ILogger logger)
      {
        _logger = logger;
      }

      public Task OnActionExecutingAsync(HttpActionContext actionContext, CancellationToken cancellationToken)
      {
        _logger.Write(actionContext.ActionDescriptor.ActionName);
        return Task.FromResult(0);
      }

      public Task OnActionExecutedAsync(HttpActionExecutedContext actionExecutedContext, CancellationToken cancellationToken)
      {
        _logger.Write(actionExecutedContext.ActionContext.ActionDescriptor.ActionName);
        return Task.FromResult(0);
      }
    }

注意示例中没有真正的异步代码运行所以它返回 ``Task.FromResult(0)``, 这是一种返回 "empty task" 常用的方法. 如果你的filter确实需要异步代码, 你可以返回一个真正的 ``Task`` 对象或像其他异步方法一样使用 ``async``/``await`` 代码.

注册过滤器
-------------------

对于要执行的过滤器, 你要用容器注册它, 并告知容器应该作用于哪个控制器, 也可选作用于哪个action. 通过使用下面的 ``ContainerBuilder`` 扩展方法完成:

- ``AsWebApiActionFilterFor<TController>()``
- ``AsWebApiActionFilterOverrideFor<TController>()``
- ``AsWebApiAuthorizationFilterFor<TController>()``
- ``AsWebApiAuthorizationOverrideFilterFor<TController>()``
- ``AsWebApiAuthenticationFilterFor<TController>()``
- ``AsWebApiAuthenticationOverrideFilterFor<TController>()``
- ``AsWebApiExceptionFilterFor<TController>()``
- ``AsWebApiExceptionOverrideFilterFor<TController>()``

这些方法需要一个泛型的类型参数用作传入控制器的类型, 和一个可选的lambda表达式用来表示filter应该作用于控制器上的某个指定的方法. 如果你不提供lambda表达式filter将会应用与控制器上的所有方法, 和放置一个控制器级别的filter特性其实是一样的.

你可以应用任意多的filters. 注册一个类型的filter不会移除或替换掉之前的已注册filters.

下面的示例中filter被应用到 ``ValuesController`` 的 ``Get`` 方法上.

.. sourcecode:: csharp

    var builder = new ContainerBuilder();
     
    builder.Register(c => new LoggingActionFilter(c.Resolve<ILogger>()))
        .AsWebApiActionFilterFor<ValuesController>(c => c.Get(default(int)))
        .InstancePerRequest();

当应用filter到一个action方法上时需要一个用 ``default`` 关键字和参数数据类型结合成的参数, 作为lambda表达式中的一个占位符. 例如, 上例中的 ``Get`` action方法需要一个 ``int`` 参数, 并用 ``default(int)`` 作为lambda表达式中的强类型占位符.

也可以在泛型类参数中提供一个基类控制器, 来让filter作用域所有的继承的控制器. 另外, 你也可以让你的action方法的lambda表达式对应于基类控制器上的一个方法, 这样它就会应用于所有继承控制器的该方法上.

过滤器重载
----------------
注册filters时, 有基础的注册方法如 ``AsWebApiActionFilterFor<TController>()`` 和重载注册方法如 ``AsWebApiActionFilterOverrideFor<TController>()``. 重载方法的关键是提供一种方式来保证某个filter先执行. 你可以有任意多的重载filter - 它们并不是 *替换* filters, 而只是 *先* 运行.

Filters将会以此顺序执行:

- Controller-scoped overrides
- Action-scoped overrides
- Controller scoped filters
- Action scoped filters

为什么我们使用非标准的过滤器接口
-----------------------------------------

如果你想知道为什么我们引入了特殊的接口, 看一下Web API ``IActionFilter`` 接口中的签名就很显而易见了.

.. sourcecode:: csharp

    public interface IActionFilter : IFilter
    {
      Task<HttpResponseMessage> ExecuteActionFilterAsync(HttpActionContext actionContext, CancellationToken cancellationToken, Func<Task<HttpResponseMessage>> continuation);
    }

比较下你需要实现的Autofac接口.

.. sourcecode:: csharp

    public interface IAutofacActionFilter
    {
      Task OnActionExecutedAsync(HttpActionExecutedContext actionExecutedContext, CancellationToken cancellationToken);

      Task OnActionExecutingAsync(HttpActionContext actionContext, CancellationToken cancellationToken);
    }

问题就出在 ``OnActionExecutingAsync`` 和 ``OnActionExecutedAsync`` 方法其实是定义在 ``ActionFilterAttribute`` 上的而不是 ``IActionFilter`` 接口上. Web API大量使用 ``System.Threading.Tasks`` 命名空间意味着特性中用适当的错误处理将返回的task串联起来需要大量的代码 ( ``ActionFilterAttribute`` 包含了将近100行这样的代码). 这绝对不是你想自己处理的事.

Autofac引入了新的接口, 允许你集中注意实现filter的代码而不是应付所有的细节问题. 在内部它创建了真正的Web API特性的自定义实例, 从容器中解析filter的具体实现并在适当的时候执行.

另一个对内部特性进行封装的原因是为了filters支持 ``InstancePerRequest`` 生命周期作用域. 见下面详情.

标准Web API过滤器特性都是单例
-------------------------------------------------

也许你会注意到如果你使用标准Web API filters, 那么你将无法使用 ``InstancePerRequest`` 依赖.

有别于 :doc:`MVC <mvc>`, Web API中的filter provider不允许你指定某个filter实例不应该被缓存. 意味着 **Web API中所有的过滤器特性实际上都是单例, 存在于应用的整个生命周期中.**

如果你想要在filter中获取per-request依赖, 你会发现只有使用Autofac filter接口才有用. 使用标准的Web API filters, 依赖只会被注入一次 - filter第一次解析的时候 - 以后再也不会.

**如果你无法使用Autofac接口并且你需要在你的filters里使用per-request或instance-per-dependency服务, 你必须用服务定位(service location).** 幸运的是, Web API可以很方便地获得当前请求的作用域 - 它和 ``HttpRequestMessage`` 一起提供.

下面是一个filter使用服务定位的示例, 用Web API的 ``IDependencyScope`` 获得per-request依赖:

.. sourcecode:: csharp

    public interface ServiceCallActionFilterAttribute : ActionFilterAttribute
    {
      public override void OnActionExecuting(HttpActionContext actionContext)
      {
        // Get the request lifetime scope so you can resolve services.
        var requestScope = actionContext.Request.GetDependencyScope();

        // Resolve the service you want to use.
        var service = requestScope.GetService(typeof(IMyService)) as IMyService;

        // Do the rest of the work in the filter.
        service.DoWork();
      }
    }


实例过滤器不会被注入
-----------------------------------

设置filters的时候, 你也许会像下面这样手动添加filters到集合:

.. sourcecode:: csharp

    config.Filters.Add(new MyActionFilter());

**Autofac将不会注入以这种方式注册的filters中的属性.** 这就和当你使用 ``RegisterInstance`` 把预先构建的对象实例放进Autofac是一样的 - Autofac并不会注入或修改预先构建的示例. 这同样适用于预先构建好并加入到filter集合中的filter实例. 当使用filters特性的时候(上面提到的), 你可以通过服务定位而不是属性注入来解决.

通过依赖注入提供Model Binders
==============================================

Autofac Web API 集成提供了通过依赖注入解析model binders的功能, 并用流式接口将binders和类型联系起来.

注册Binder Provider
----------------------------

你需要注册Autofac model binder provide, 这样它就能在需要时解析任何已注册的 ``IModelBinder`` 实现. 通过调用容器构造器的 ``RegisterWebApiModelBinderProvider`` 方法实现.

.. sourcecode:: csharp

    var builder = new ContainerBuilder();
    builder.RegisterWebApiModelBinderProvider();

注册 Model Binders
----------------------

只要你实现 ``System.Web.Http.ModelBinding.IModelBinder`` 来处理绑定的事, 将它注册到Autofac并让Autofac知道哪些类型应该使用该binder.

.. sourcecode:: csharp

    builder
      .RegisterType<AutomobileBinder>()
      .AsModelBinderForTypes(typeof(CarModel), typeof(TruckModel));

使用ModelBinderAttribute标记参数
-----------------------------------------

即使你已经注册了你的model binder, 你还需要将你的参数用 ``[ModelBinder]`` 特性标记, 这样Web API才能知道使用model binder而不是media type formatter来绑定你的model. 你不必再指定model binder类型, 但你需要用该特性来标记参数. `这在Web API的官方文档中也有提到. <https://docs.microsoft.com/en-us/aspnet/web-api/overview/formats-and-model-binding/parameter-binding-in-aspnet-web-api>`_

.. sourcecode:: csharp

    public HttpResponseMessage Post([ModelBinder] CarModel car) { ... }

Per-Controller-Type Services
============================

Web API has an interesting feature that allows you to configure the set of Web API services (those such as ``IActionValueBinder``) that should be used per-controller-type by adding an attribute that implements the ``IControllerConfiguration`` interface to your controller.

Through the ``Services`` property on the ``HttpControllerSettings`` parameter passed to the ``IControllerConfiguration.Initialize`` method you can override the global set of services. This attribute-based approach seems to encourage you to directly instantiate service instances and then override the ones registered globally. Autofac allows these per-controller-type services to be configured through the container instead of being buried away in an attribute without dependency injection support.

Add the Controller Configuration Attribute
------------------------------------------

There is no escaping adding an attribute to the controller that the configuration should be applied to because that is the extension point defined by Web API. The Autofac integration includes an ``AutofacControllerConfigurationAttribute`` that you can apply to your Web API controllers to indicate that they require per-controller-type configuration.

The point to remember here is that **the actual configuration of what services should be applied will be done when you build your container** and there is no need to implement any of that in an actual attribute. In this case, the attribute can be considered as purely a marker that indicates that the container will define the configuration information and provide the service instances.

.. sourcecode:: csharp

    [AutofacControllerConfiguration]
    public class ValuesController : ApiController
    {
      // Implementation...
    }

Supported Services
------------------

The supported services can be divided into single-style or multiple-style services. For example, you can only have one ``IHttpActionInvoker`` but you can have multiple ``ModelBinderProvider`` services.

You can use dependency injection for the following single-style services:

- ``IHttpActionInvoker``
- ``HttpActionSelector``
- ``ActionValueBinder``
- ``IBodyModelValidator``
- ``IContentNegotiator``
- ``IHttpControllerActivator``
- ``ModelMetadataProvider``

The following multiple style services are supported:

- ``ModelBinderProvider``
- ``ModelValidatorProvider``
- ``ValueProviderFactory``
- ``MediaTypeFormatter``

In the list of the multiple-style services above the ``MediaTypeFormatter`` is actually the odd one out. Technically, it isn't actually a service and is added to the ``MediaTypeFormatterCollection`` on the ``HttpControllerSettings`` instance and not the ``ControllerServices`` container. We figured that there was no reason to exclude ``MediaTypeFormatter`` instances from dependency injection support and made sure that they could be resolved from the container per-controller type, too.

Service Registration
--------------------

Here is an example of registering a custom ``IHttpActionSelector`` implementation as ``InstancePerApiControllerType()`` for the ``ValuesController``. When applied to a controller type all derived controllers will also receive the same configuration; the ``AutofacControllerConfigurationAttribute`` is inherited by derived controller types and the same behavior applies to the registrations in the container. When you register a single-style service it will always replace the default service configured at the global level.

.. sourcecode:: csharp

    builder.Register(c => new CustomActionSelector())
           .As<IHttpActionSelector>()
           .InstancePerApiControllerType(typeof(ValuesController));

Clearing Existing Services
--------------------------

By default, multiple-style services are appended to the existing set of services configured at the global level. When registering multiple-style services with the container you can choose to clear the existing set of services so that only the ones you have registered as ``InstancePerApiControllerType()`` will be used. This is done by setting the ``clearExistingServices`` parameter to ``true`` on the ``InstancePerApiControllerType()`` method. Existing services of that type will be removed if any of the registrations for the multiple-style service indicate that they want that to happen.

.. sourcecode:: csharp

    builder.Register(c => new CustomModelBinderProvider())
           .As<ModelBinderProvider>()
           .InstancePerApiControllerType(
              typeof(ValuesController),
              clearExistingServices: true);

Per-Controller-Type Service Limitations
---------------------------------------

If you are using per-controller-type services, it is not possible to take dependencies on other services that are registered as ``InstancePerRequest()``. The problem is that Web API is caching these services and is not requesting them from the container each time a controller of that type is created. It is most likely not possible for Web API to easily add that support that without introducing the notion of a key (for the controller type) into the DI integration, which would mean that all containers would need to support keyed services.

Batching
========

If you choose to use the `Web API batching functionality <https://blogs.msdn.microsoft.com/webdev/2013/11/01/introducing-batch-support-in-web-api-and-web-api-odata/>`_, be aware that the initial multipart request to the batch endpoint is where Web API creates the request lifetime scope. The child requests that are part of the batch all take place in-memory and will share that same request lifetime scope - you won't get a different scope for each child request in the batch.

This is due to the way the batch handling is designed within Web API and copies properties from the parent request to the child request. One of the properties that is intentionally copied by the ASP.NET Web API framework from parent to children is the request lifetime scope. There is no workaround for this and is outside the control of Autofac.

OWIN Integration
================

If you are using Web API :doc:`as part of an OWIN application <owin>`, you need to:

* Do all the stuff for standard Web API integration - register controllers, set the dependency resolver, etc.
* Set up your app with the :doc:`base Autofac OWIN integration <owin>`.
* Add a reference to the `Autofac.WebApi2.Owin <http://www.nuget.org/packages/Autofac.WebApi2.Owin/>`_ NuGet package.
* In your application startup class, register the Autofac Web API middleware after registering the base Autofac middleware.

.. sourcecode:: csharp

    public class Startup
    {
      public void Configuration(IAppBuilder app)
      {
        var builder = new ContainerBuilder();

        // STANDARD WEB API SETUP:

        // Get your HttpConfiguration. In OWIN, you'll create one
        // rather than using GlobalConfiguration.
        var config = new HttpConfiguration();

        // Register your Web API controllers.
        builder.RegisterApiControllers(Assembly.GetExecutingAssembly());

        // Run other optional steps, like registering filters,
        // per-controller-type services, etc., then set the dependency resolver
        // to be Autofac.
        var container = builder.Build();
        config.DependencyResolver = new AutofacWebApiDependencyResolver(container);

        // OWIN WEB API SETUP:

        // Register the Autofac middleware FIRST, then the Autofac Web API middleware,
        // and finally the standard Web API middleware.
        app.UseAutofacMiddleware(container);
        app.UseAutofacWebApi(config);
        app.UseWebApi(config);
      }
    }

A common error in OWIN integration is use of the ``GlobalConfiguration.Configuration``. **In OWIN you create the configuration from scratch.** You should not reference ``GlobalConfiguration.Configuration`` anywhere when using the OWIN integration.

Unit Testing
============

When unit testing an ASP.NET Web API app that uses Autofac where you have ``InstancePerRequest`` components registered, you'll get an exception when you try to resolve those components because there's no HTTP request lifetime during a unit test.

The :doc:`per-request lifetime scope <../faq/per-request-scope>` topic outlines strategies for testing and troubleshooting per-request-scope components.

Example
=======

There is an example project showing Web API in conjunction with OWIN self hosting `in the Autofac examples repository <https://github.com/autofac/Examples/tree/master/src/WebApiExample.OwinSelfHost>`_.

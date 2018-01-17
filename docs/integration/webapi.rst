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

Web API有一个有趣的功能, 它允许你通过在controller上放置一个实现 ``IControllerConfiguration`` 接口的特性, 这样就能配置一系列的Web API服务(如 ``IActionValueBinder``)为per-controller-type的.

通过传递到 ``IControllerConfiguration.Initialize`` 方法中的 ``HttpControllerSettings`` 参数上的 ``Services`` 属性, 你可以重写全局的服务集合. 这种基于特性的方法意在鼓励你直接初始化服务实例并且重写全局注册的服务. Autofac允许通过容器来配置per-controller-type services而不会因为使用特性的缘故无法得到依赖注入的支持.

添加Controller Configuration特性
------------------------------------------

添加一个特性到应用配置的控制器上还是免不了的, 因为这是Web API定义的扩展点. Autofac集成包含 ``AutofacControllerConfigurationAttribute`` , 你可以把它应用到你的Web API控制器上来表明它们需要per-controller-type配置.

这边需要记住的一点是, **配置到底哪些服务应该被应用的这件事, 将会在你创建容器的时候完成** , 我们并不需要在某个具体的特性中去完成这些事. 在这种情况下, 该特性其实可以被单纯的认为是一个标记, 用来表明容器将会定义配置信息并会提供服务实例.

.. sourcecode:: csharp

    [AutofacControllerConfiguration]
    public class ValuesController : ApiController
    {
      // Implementation...
    }

支持的服务(Supported Services)
---------------------------------

支持的服务可以分为单一型或多重型. 例如, 你只可以有一个 ``IHttpActionInvoker`` 但你可以有用多个 ``ModelBinderProvider`` 服务.

依赖注入支持下列单一型服务:

- ``IHttpActionInvoker``
- ``HttpActionSelector``
- ``ActionValueBinder``
- ``IBodyModelValidator``
- ``IContentNegotiator``
- ``IHttpControllerActivator``
- ``ModelMetadataProvider``

支持下列多重型服务:

- ``ModelBinderProvider``
- ``ModelValidatorProvider``
- ``ValueProviderFactory``
- ``MediaTypeFormatter``

在上面的多重型服务列表中, ``MediaTypeFormatter`` 实际上可以说是单独的. 从技术角度上说, 它并不是一个真正的服务, 它只是被添加到 ``HttpControllerSettings`` 实例上的 ``MediaTypeFormatterCollection`` 中而不是 ``ControllerServices`` 容器中. 我们觉得没理由从依赖注入支持的服务中中排除掉 ``MediaTypeFormatter`` 实例, 并且确保了它们也可以从容器per-controller type被解析出来.

服务注册
--------------------

这里有一个 ``ValuesController`` 注册自定义 ``IHttpActionSelector`` 实现为 ``InstancePerApiControllerType()`` 的例子. 应用到一个控制器的时候所有继承的控制器也会获得相同的配置; ``AutofacControllerConfigurationAttribute`` 被派生的控制器继承, 在容器注册中也会被应用相同的行为. 当你注册一个单一型服务时它总是会替换掉在全局层面配置的默认服务.

.. sourcecode:: csharp

    builder.Register(c => new CustomActionSelector())
           .As<IHttpActionSelector>()
           .InstancePerApiControllerType(typeof(ValuesController));

清除现存的服务
--------------------------

默认地, 多重型服务会被附加到在全局层面配置的现存服务集合上. 当你在容器上注册多重型服务时你可以选择清除现存的服务集合, 这样的话只有你注册为 ``InstancePerApiControllerType()`` 的服务才会被使用. 可以设置 ``InstancePerApiControllerType()`` 的 ``clearExistingServices`` 参数为 ``true`` 来完成. 任何多重型服务只要表明它们希望该类的现存服务被清除, 那么服务就会被清除.

.. sourcecode:: csharp

    builder.Register(c => new CustomModelBinderProvider())
           .As<ModelBinderProvider>()
           .InstancePerApiControllerType(
              typeof(ValuesController),
              clearExistingServices: true);

Per-Controller-Type Service 局限性
---------------------------------------

如果你在使用per-controller-type services, 不可以引用其他注册为 ``InstancePerRequest()`` 的服务. 问题在于Web API会缓存这些服务, 并且不会在每次该控制器类被创建时请求它们. Web API不太容易添加这样的支持, 除非引入key(for the controller type)的概念到DI集成中, 意味着所有的容器需要支持带键值的服务(keyed service).

批处理
========

如果你选择使用 `Web API 批处理功能 <https://blogs.msdn.microsoft.com/webdev/2013/11/01/introducing-batch-support-in-web-api-and-web-api-odata/>`_, 要知道初始的multipart请求到达batch endpoint时Web API创建了请求生命周期. 批处理的子请求都发生在内存中并且会共享相同的请求生命周期 - 在一个批处理中对于每个子请求你不会得到不同的生命周期作用域.

这是因为批处理(batch)的处理方式在Web API内部就设计好了, 会拷贝父请求的属性到子请求中. 这些属性中有一个就被ASP.NET Web API框架有意地从父请求拷贝到子请求, 它就是请求生命周期作用域. 这没办法解决, 它超出了Autofac的控制范围.

OWIN 集成
================

如果你正在使用Web API :doc:`作为OWIN应用的一部分 <owin>`, 你需要:

* 完成所有基础Web API集成的工作 - 注册控制器, 设置依赖解析器等.
* 用 :doc:`基础的Autofac OWIN集成 <owin>` 创建你的应用.
* 添加 `Autofac.WebApi2.Owin <http://www.nuget.org/packages/Autofac.WebApi2.Owin/>`_ 引用NuGet package.
* 应用startup类中, 在注册基础Autofac中间件后注册Autofac Web API中间件.

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

一个常见出现的错误是使用 ``GlobalConfiguration.Configuration``. **在OWIN中你会从头开始创建配置.** 在使用OWIN集成的时候你不应该在任何地方引用 ``GlobalConfiguration.Configuration`` .

单元测试
============

当单元测试一个使用Autofac注册了 ``InstancePerRequest`` 组件的ASP.NET Web API应用时, 当你尝试解析这些组件时你会得到一个异常因为在单元测试中并没有HTTP请求生命周期.

:doc:`per-request lifetime scope <../faq/per-request-scope>` 章节概述了测试和检查per-request-scope组件的对策.

示例
=======

`Autofac示例代码仓库 <https://github.com/autofac/Examples/tree/master/src/WebApiExample.OwinSelfHost>`_ 里有一个展示了Web API结合OWIN自托管的示例项目.

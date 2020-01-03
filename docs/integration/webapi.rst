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
因为特性(attributes)通过反射API创建, 你不能自己调用构造方法. 
这就使得你在使用特性时除了属性注入没有了其他选择. 
Autofac Web API集成提供了一种机制, 允许你创建实现过滤器接口 (``IAutofacActionFilter``, ``IAutofacContinuationActionFilter``, ``IAutofacAuthorizationFilter`` and ``IAutofacExceptionFilter``) 的类, 然后就可以通过使用容器构造器(container builder)的注册语法将它们和需要的控制器或action方法连接起来.

注册过滤器提供者(Filter Provider)
---------------------------------

你需要实现注册过滤器提供者因为它做了基于注册的方式连接过滤器的工作. 可以通过调用容器构造器的 ``RegisterWebApiFilterProvider`` 方法和提供一个 ``HttpConfiguration`` 实例完成.

.. sourcecode:: csharp

    var builder = new ContainerBuilder();
    builder.RegisterWebApiFilterProvider(config);

Standard Action Filter Interface
********************************

``IAutofacActionFilter`` 接口允许你定义一个过滤器, 可以在action的执行前后触发, 类似于继承 ``ActionFilterAttribute``.

下面的过滤器是一个action过滤器, 它实现了 ``IAutofacActionFilter`` 而不是 ``System.Web.Http.Filters.IActionFilter``.

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

Continuation Action Filter Interface
*************************************

除了上面示例中普通的 ``IAutofacActionFilter``, 还有一种 ``IAutofacContinuationActionFilter``. 这个接口和Action Filter功能类似, 
但它并没有 ``OnActionExecutingAsync`` 和 ``OnActionExecutedAsync`` 方法, 它遵循 continuation
style, 只有一个 ``ExecuteActionFilterAsync`` 方法, 方法接受一个回调, 回调用于执行调用链中的下一个过滤器.

如果你想要将整个请求包裹在一个 ``using`` 块中, 你也许就会想用 ``IAutofacContinuationActionFilter`` 替代 ``IAutofacActionFilter``,
例如你想要给请求分配一个 ``TransactionScope``, 像下面这样:

.. sourcecode:: csharp

    public class TransactionScopeFilter : IAutofacContinuationActionFilter
    {
        public async Task<HttpResponseMessage> ExecuteActionFilterAsync(
            HttpActionContext actionContext,
            CancellationToken cancellationToken,
            Func<Task<HttpResponseMessage>> next)
        {
            using (new TransactionScope(TransactionScopeAsyncFlowOption.Enabled))
            {
                return await next();
            }
        }
    }

.. note:: 

  普通的 ``IAutofacActionFilter`` 运行在continuation filter内部, 所以异步上下文在
  ``OnActionExecutingAsync``, action方法本身, 和过滤器的 ``OnActionExecutedAsync`` 之间都会被保留. 

注册过滤器
-------------------

对于要执行的过滤器, 你要用容器注册它, 并告知容器应该作用于哪个控制器, 也可选作用于哪个action. 通过使用下面的 ``ContainerBuilder`` 扩展方法完成:

- ActionFilter
- ActionFilterOverride
- AuthenticationFilter
- AuthenticationFilterOverride
- AuthorizationFilter
- AuthorizationFilterOverride
- ExceptionFilter
- ExceptionFilterOverride

对于每种过滤器类型, 都有几个注册方法:

``AsWebApi{FilterType}ForAllControllers``
  注册此过滤器让它在所有控制器的所有action方法上都生效, 类似注册一个全局Web API过滤器.

``AsWebApi{FilterType}For<TController>()``
  给特定的控制器注册过滤器, 类似于在控制器级别放置一个基于特性的过滤器.

  指定一个控制器的基类可以让过滤器应用于所有继承自它的类.

  如果你正在特定的action上应用一个过滤器特性, 该方法接受一个可选的lambda表达式参数, 用来指明过滤器应该应用在控制器的哪个特定的方法上.

  下面的示例中一个Action过滤器被应用在 ``ValuesController`` 的 ``Get`` action 方法上.

  .. sourcecode:: csharp

      var builder = new ContainerBuilder();
       
      builder.Register(c => new LoggingActionFilter(c.Resolve<ILogger>()))
          .AsWebApiActionFilterFor<ValuesController>(c => c.Get(default(int)))
          .InstancePerRequest();

  当应用filter到一个action方法上时需要一个用 ``default`` 关键字和参数数据类型结合成的参数, 作为lambda表达式中的一个占位符. 例如, 上例中的 ``Get`` action方法需要一个 ``int`` 参数, 并用 ``default(int)`` 作为lambda表达式中的强类型占位符.

``AsWebApi{FilterType}Where()``
  ``*Where`` 方法允许你指定一个predicate, 通过它可以在附加过滤器到控制器和/或actions的时候有更多自定义的选择. 

  下面的示例中一个Exception过滤器被应用在所有POST方法上:

  .. sourcecode:: csharp

      var builder = new ContainerBuilder();
       
      builder.Register(c => new LoggingExceptionFilter(c.Resolve<ILogger>()))
          .AsWebApiExceptionFilterWhere(action => action.SupportedHttpMethods.Contains(HttpMethod.Post))
          .InstancePerRequest();

  另外还有一个版本的predicate接受一个 ``ILifetimeScope`` 参数, 你可以在你的predicate内部消费服务:

  .. sourcecode:: csharp

      var builder = new ContainerBuilder();
       
      builder.Register(c => new LoggingExceptionFilter(c.Resolve<ILogger>()))
          .AsWebApiExceptionFilterWhere((scope, action) => scope.Resolve<IFilterConfig>().ShouldFilter(action))
          .InstancePerRequest();

  .. note:: 

     Filter predicates are invoked once for each action/filter combination; they are not invoked on every request.

你可以应用任意数量的过滤器. 为一个类型注册一个过滤器不会移除或过滤掉之前注册的过滤器.

你可以把你的过滤器注册方法串联起来, 把一个过滤器附加到多个控制器上, 如:

.. sourcecode:: csharp

  builder.Register(c => new LoggingActionFilter(c.Resolve<ILogger>()))
      .AsWebApiActionFilterFor<LoginController>()
      .AsWebApiActionFilterFor<ValuesController>(c => c.Get(default(int)))
      .AsWebApiActionFilterFor<ValuesController>(c => c.Post(default(string)))
      .InstancePerRequest();

过滤器重载
----------------
注册filters时, 有基础的注册方法如 ``AsWebApiActionFilterFor<TController>()`` 和重载注册方法如 ``AsWebApiActionFilterOverrideFor<TController>()``. 

重载方法的关键是提供一种方式来保证某个filter先执行. 你可以有任意多的重载filter - 它们并不是 *替换* filters, 而只是 *先* 运行.

Filters将会以此顺序执行:

- Controller-scoped overrides
- Action-scoped overrides
- Controller scoped filters
- Action scoped filters

在Autofac Action过滤器中给Response赋值
------------------------------------------------

和基础的Web API过滤器一样,  你可以在一个action过滤器的 ``OnActionExecutingAsync`` 方法中设置 ``HttpResponseMessage``.

.. sourcecode:: csharp

  class RequestRejectionFilter : IAutofacActionFilter
  {
    public Task OnActionExecutingAsync(HttpActionContext actionContext, CancellationToken cancellationToken)
    {
      // Request is not valid for some reason.
      actionContext.Response = actionContext.Request.CreateErrorResponse(HttpStatusCode.BadRequest, "Request not valid");
      return Task.FromResult(0);
    }

    public void Task OnActionExecutedAsync(HttpActionExecutedContext actionExecutedContext, CancellationToken cancellationToken)
    {
    }
  }

为了与基础的Web API行为相匹配, 如果你设置 ``Response`` 属性, 那么后续的action过滤器将不会被触发. 
然而, 任何已经触发的action过滤器的 ``OnActionExecutedAsync`` 方法还是会被调用, 相关response也会被填充.

标准Web API过滤器特性都是单例
-------------------------------------------------

也许你会注意到如果你使用标准Web API filters, 那么你将无法使用 ``InstancePerRequest`` 依赖.

有别于 :doc:`MVC <mvc>`, Web API中的filter provider不允许你指定某个filter实例不应该被缓存. 意味着 **Web API中所有的过滤器特性实际上都是单例, 存在于应用的整个生命周期中.**

如果你想要在filter中获取per-request依赖, 你会发现只有使用Autofac filter接口才有用. 使用标准的Web API filters, 依赖只会被注入一次 - filter第一次解析的时候 - 以后再也不会.

**如果你无法使用Autofac接口并且你需要在你的filters里使用per-request或instance-per-dependency服务, 你必须用服务定位(service location).** 幸运的是, Web API可以很方便地获得当前请求的作用域 - 它和 ``HttpRequestMessage`` 一起提供.

下面是一个filter使用服务定位的示例, 用Web API的 ``IDependencyScope`` 获得per-request依赖:

.. sourcecode:: csharp

    public class ServiceCallActionFilterAttribute : ActionFilterAttribute
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
* 添加 `Autofac.WebApi2.Owin <https://www.nuget.org/packages/Autofac.WebApi2.Owin/>`_ 引用NuGet package.
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

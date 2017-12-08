===
MVC
===

Autofac一直保持更新以支持最新版本的ASP.NET MVC, 因此文档也是保持最新. 总的来说, 集成在跨版本之间依然是相当一致的.

MVC集成需要 `Autofac.Mvc5 NuGet package <http://www.nuget.org/packages/Autofac.Mvc5/>`_.

MVC集成提供了controllers, model binders, action filters, 和views的依赖注入. 它同样添加 :doc:`每个请求生命周期支持 <../faq/per-request-scope>`.

**本章节解释了ASP.NET classic MVC集成.** 如果你使用ASP.NET Core, :doc:`见ASP.NET Core集成章节 <aspnetcore>`.

.. contents::
  :local:

入门
===========
为了把Autofac集成进MVC你需要引用 MVC integration NuGet package, 注册控制器, 设置依赖解析器(Dependency Resolver). 你可以选择性地启用其他功能.

.. sourcecode:: csharp

    protected void Application_Start()
    {
      var builder = new ContainerBuilder();

      // Register your MVC controllers. (MvcApplication is the name of
      // the class in Global.asax.)
      builder.RegisterControllers(typeof(MvcApplication).Assembly);

      // OPTIONAL: Register model binders that require DI.
      builder.RegisterModelBinders(typeof(MvcApplication).Assembly);
      builder.RegisterModelBinderProvider();

      // OPTIONAL: Register web abstractions like HttpContextBase.
      builder.RegisterModule<AutofacWebTypesModule>();

      // OPTIONAL: Enable property injection in view pages.
      builder.RegisterSource(new ViewRegistrationSource());

      // OPTIONAL: Enable property injection into action filters.
      builder.RegisterFilterProvider();

      // OPTIONAL: Enable action method parameter injection (RARE).
      builder.InjectActionInvoker();

      // Set the dependency resolver to be Autofac.
      var container = builder.Build();
      DependencyResolver.SetResolver(new AutofacDependencyResolver(container));
    }

下面的内容进一步详细解释了其他功能和如何使用它们.

注册控制器
====================

在应用startup的地方, 当你创建Autofac容器时, 你应该注册你的MVC控制器和它们的依赖. 这通常发生在OWIN startup类或在 ``Global.asax`` 的 ``Application_Start`` 方法中.

.. sourcecode:: csharp

    var builder = new ContainerBuilder();

    // You can register controllers all at once using assembly scanning...
    builder.RegisterControllers(typeof(MvcApplication).Assembly);

    // ...or you can register individual controlllers manually.
    builder.RegisterType<HomeController>().InstancePerRequest();

注意ASP.NET MVC通过控制器具体类型来判断请求哪个控制器, 因此把它们以 ``As<IController>()`` 注册是不正确的. 同时, 如果你人为注册控制器并且选择特定的生命周期, 你必须以 ``InstancePerDependency()`` 或 ``InstancePerRequest()`` 注册 - **如果你尝试对多个请求复用同一个控制器实例, ASP.NET MVC会报错**.

设置依赖解析器(Dependency Resolver)
========================================

创建完你的容器后, 把它传入到一个新建的 ``AutofacDependencyResolver`` 类的实例中. 使用静态的 ``DependencyResolver.SetResolver`` 方法来让ASP.NET MVC知道它应该用 ``AutofacDependencyResolver`` 来定位服务. 这是Autofac对于 ``IDependencyResolver`` 接口的实现.

.. sourcecode:: csharp

    var container = builder.Build();
    DependencyResolver.SetResolver(new AutofacDependencyResolver(container));

注册Model Binders
======================

你可以选择性地允许对model binders的依赖注入. 和控制器类似, model binders (实现 ``IModelBinder`` 的类) 也可以在应用startup的地方被注册进容器. 你可以用 ``RegisterModelBinders()`` 方法来做. 千万记住要用 ``RegisterModelBinderProvider()`` 扩展方法注册 ``AutofacModelBinderProvider``. 这是Autofac对于 ``IModelBinderProvider`` 接口的实现.

.. sourcecode:: csharp

    builder.RegisterModelBinders(Assembly.GetExecutingAssembly());
    builder.RegisterModelBinderProvider();

因为 ``RegisterModelBinders()`` 扩展方法使用程序集扫描来添加你需要的model binders, 来指定model binders (``IModelBinder`` 的实现) 注册了哪些类.

而这些通过使用 ``Autofac.Integration.Mvc.ModelBinderTypeAttribute`` 完成, 如:

.. sourcecode:: csharp

    [ModelBinderType(typeof(string))]
    public class StringBinder : IModelBinder
    {
      public override object BindModel(ControllerContext controllerContext, ModelBindingContext bindingContext)
      {
        // Implementation here
      }
    }

如果它是为了多个类型而被注册的, 那么可以给类添加多个 ``ModelBinderTypeAttribute`` 实例.

注册Web抽象对象
=========================

MVC集成包括一个Autofac模块, 该模块会为web抽象类添加 :doc:`HTTP请求生命周期作用域的 <../faq/per-request-scope>` 注册. 它将允许你把web抽象对象作为一个依赖放进你的类中并且在运行时注入正确的值.

包括以下抽象类:

* ``HttpContextBase``
* ``HttpRequestBase``
* ``HttpResponseBase``
* ``HttpServerUtilityBase``
* ``HttpSessionStateBase``
* ``HttpApplicationStateBase``
* ``HttpBrowserCapabilitiesBase``
* ``HttpFileCollectionBase``
* ``RequestContext``
* ``HttpCachePolicyBase``
* ``VirtualPathProvider``
* ``UrlHelper``

想要使用这些抽象对象, 使用基础的 ``RegisterModule()`` 方法添加 ``AutofacWebTypesModule`` 进容器.

.. sourcecode:: csharp

    builder.RegisterModule<AutofacWebTypesModule>();

启用视图页(View Pages)的属性注入
========================================

你可以通过在创建应用容器前添加 ``ViewRegistrationSource`` 到你的 ``ContainerBuilder`` 来启用你的MVC视图 :doc:`属性注入 <../register/prop-method-injection>`.

.. sourcecode:: csharp

    builder.RegisterSource(new ViewRegistrationSource());

你的视图页必须继承自某个MVC支持的创建视图的类. 使用Razor视图引擎时它是 ``WebViewPage`` 类.

.. sourcecode:: csharp

    public abstract class CustomViewPage : WebViewPage
    {
      public IDependency Dependency { get; set; }
    }

使用web forms视图引擎时 ``ViewPage``, ``ViewMasterPage`` 和 ``ViewUserControl`` 类都是支持的.

.. sourcecode:: csharp

    public abstract class CustomViewPage : ViewPage
    {
      public IDependency Dependency { get; set; }
    }

确保你真实视图页继承自你的自定义基础类. 对于Razor视图引擎可以通过 ``.cshtml`` 文件内部的 ``@inherits`` 指令完成::

    @inherits Example.Views.Shared.CustomViewPage

使用web forms视图引擎时你可以在 ``.aspx`` 文件的 ``@ Page`` 指令上设置 ``Inherits`` 属性.

.. sourcecode:: aspx-cs

    <%@ Page Language="C#" MasterPageFile="~/Views/Shared/Site.Master" Inherits="Example.Views.Shared.CustomViewPage"%>

**因为ASP.NET MVC内部的问题, 对于Razor布局页(layout pages) 依赖注入不可用.** Razor视图可以, 但是布局页不可以. `详情见 issue #349. <https://github.com/autofac/Autofac/issues/349#issuecomment-33025529>`_

启用Action Filters的属性注入
============================================

想要启用Action Filters的属性注入, 在创建容器前调用 ``ContainerBuilder`` 的 ``RegisterFilterProvider()`` 方法并把它传给 ``AutofacDependencyResolver``.

.. sourcecode:: csharp

    builder.RegisterFilterProvider();

这允许你给filter attributes添加属性, 容器中所有匹配的已注册的依赖将被注入进这些属性中.

例如, 下面的action filter将会从容器中拿到 ``ILogger`` 实例 (假设你注册了 ``ILogger``. 注意 **特性(attribute)本身不需要注册进容器中**.

.. sourcecode:: csharp

    public class CustomActionFilter : ActionFilterAttribute
    {
      public ILogger Logger { get; set; }

      public override void OnActionExecuting(ActionExecutingContext filterContext)
      {
        Logger.Log("OnActionExecuting");
      }
    }

对于其他filter attribute类型例如authorization attributes同样处理.

.. sourcecode:: csharp

    public class CustomAuthorizeAttribute : AuthorizeAttribute
    {
      public ILogger Logger { get; set; }

      protected override bool AuthorizeCore(HttpContextBase httpContext)
      {
        Logger.Log("AuthorizeCore");
        return true;
      }
    }

把特性应用到actions上, 整个工作就完成了.

.. sourcecode:: csharp

    [CustomActionFilter]
    [CustomAuthorizeAttribute]
    public ActionResult Index()
    {
    }

启用Action参数的注入
=====================================

尽管不普遍, 还是有些人想要在action方法调用时让Autofac给参数填充值. **我们推荐你使用控制器的构造方法注入而不是action方法注入** 但如果你想要的话你还是可以启用action参数的注入:

.. sourcecode:: csharp

    // The Autofac ExtensibleActionInvoker attempts to resolve parameters
    // from the request lifetime scope IF the model binder can't bind
    // to the parameter.
    builder.RegisterType<ExtensibleActionInvoker>().As<IActionInvoker>();
    builder.InjectActionInvoker();

注意在使用 ``InjectActionInvoker()`` 机制时你也可以使用自定义的invoker.

.. sourcecode:: csharp

    builder.RegisterType<MyCustomActionInvoker>().As<IActionInvoker>();
    builder.InjectActionInvoker();

OWIN集成
================

如果你使用MVC :doc:`作为OWIN应用的一部分时 <owin>`, 你需要:

* 完成基础MVC集成要做的所有事 - 注册控制器, 设置依赖解析器等.
* 用 :doc:`基础的Autofac OWIN集成 <owin>` 创建你的应用.
* 添加 `Autofac.Mvc5.Owin <http://www.nuget.org/packages/Autofac.Mvc5.Owin/>`_ 引用NuGet package.
* 应用startup类中, 在注册基础Autofac中间件后注册Autofac MVC中间件.

.. sourcecode:: csharp

    public class Startup
    {
      public void Configuration(IAppBuilder app)
      {
        var builder = new ContainerBuilder();

        // STANDARD MVC SETUP:

        // Register your MVC controllers.
        builder.RegisterControllers(typeof(MvcApplication).Assembly);

        // Run other optional steps, like registering model binders,
        // web abstractions, etc., then set the dependency resolver
        // to be Autofac.
        var container = builder.Build();
        DependencyResolver.SetResolver(new AutofacDependencyResolver(container));

        // OWIN MVC SETUP:

        // Register the Autofac middleware FIRST, then the Autofac MVC middleware.
        app.UseAutofacMiddleware(container);
        app.UseAutofacMvc();
      }
    }

**次要问题: MVC并不是100%运行在OWIN管道中.** 它仍然需要 ``HttpContext.Current`` 和其他一些非OWIN的东西. 在应用startup的地方, 当MVC注册路由时, 它实例化了一个 ``IControllerFactory`` 随后创建了两个请求生命周期作用域. 它只发生在应用startup的路由注册时期, 而并非每个请求处理时, 但这仍然需要被知晓. 这是一个两个管道错乱的构件. `我们尝试解决 <https://github.com/autofac/Autofac.Mvc/issues/5>`_ 但并没有找到一个清楚合理的方法.

Using "Plugin" Assemblies
=========================

If you have controllers in a "plugin assembly" that isn't referenced by the main application `you'll need to register your controller plugin assembly with the ASP.NET BuildManager <http://www.paraesthesia.com/archive/2013/01/21/putting-controllers-in-plugin-assemblies-for-asp-net-mvc.aspx>`_.

You can do this through configuration or programmatically.

**If you choose configuration**, you need to add your plugin assembly to the ``/configuration/system.web/compilation/assemblies`` list. If your plugin assembly isn't in the ``bin`` folder, you also need to update the ``/configuration/runtime/assemblyBinding/probing`` path.

.. sourcecode:: xml

    <?xml version="1.0" encoding="utf-8"?>
    <configuration>
      <runtime>
        <assemblyBinding xmlns="urn:schemas-microsoft-com:asm.v1">
          <!--
              If you put your plugin in a folder that isn't bin, add it to the probing path
          -->
          <probing privatePath="bin;bin\plugins" />
        </assemblyBinding>
      </runtime>
      <system.web>
        <compilation>
          <assemblies>
            <add assembly="The.Name.Of.Your.Plugin.Assembly.Here" />
          </assemblies>
        </compilation>
      </system.web>
    </configuration>

**If you choose programmatic registration**, you need to do it during pre-application-start before the ASP.NET ``BuildManager`` kicks in.

Create an initializer class to do the assembly scanning/loading and registration with the ``BuildManager``:

.. sourcecode:: csharp

    using System.IO;
    using System.Reflection;
    using System.Web.Compilation;

    namespace MyNamespace
    {
      public static class Initializer
      {
        public static void Initialize()
        {
          var pluginFolder = new DirectoryInfo(HostingEnvironment.MapPath("~/plugins"));
          var pluginAssemblies = pluginFolder.GetFiles("*.dll", SearchOption.AllDirectories);
          foreach (var pluginAssemblyFile in pluginAssemblyFiles)
          {
            var asm = Assembly.LoadFrom(pluginAssemblyFile.FullName);
            BuildManager.AddReferencedAssembly(asm);
          }
        }
      }
    }

Then be sure to register your pre-application-start code with an assembly attribute:

.. sourcecode:: csharp

    [assembly: PreApplicationStartMethod(typeof(Initializer), "Initialize")]

Using the Current Autofac DependencyResolver
============================================

Once you set the MVC ``DependencyResolver`` to an ``AutofacDependencyResolver``, you can use ``AutofacDependencyResolver.Current`` as a shortcut to getting the current dependency resolver and casting it to an ``AutofacDependencyResolver``.

Unfortunately, there are some gotchas around the use of ``AutofacDependencyResolver.Current`` that can result in things not working quite right. Usually these issues arise by using a product like `Glimpse <http://getglimpse.com/>`_ or `Castle DynamicProxy <http://www.castleproject.org/projects/dynamicproxy/>`_ that "wrap" or "decorate" the dependency resolver to add functionality. If the current dependency resolver is decorated or otherwise wrapped/proxied, you can't cast it to ``AutofacDependencyResolver`` and there's no single way to "unwrap it" or get to the actual resolver.

Prior to version 3.3.3 of the Autofac MVC integration, we tracked the current dependency resolver by dynamically adding it to the request lifetime scope. This got us around issues where we couldn't unwrap the ``AutofacDependencyResolver`` from a proxy... but it meant that ``AutofacDependencyResolver.Current`` would only work during a request lifetime - you couldn't use it in background tasks or at application startup.

Starting with version 3.3.3, the logic for locating ``AutofacDependencyResolver.Current`` changed to first attempt to cast the current dependency resolver; then to specifically look for signs it was wrapped using `Castle DynamicProxy <http://www.castleproject.org/projects/dynamicproxy/>`_ and unwrap it via reflection. Failing that... we can't find the current ``AutofacDependencyResolver`` so we throw an ``InvalidOperationException`` with a message like:

    The dependency resolver is of type 'Some.Other.DependencyResolver' but was expected to be of type 'Autofac.Integration.Mvc.AutofacDependencyResolver'. It also does not appear to be wrapped using DynamicProxy from the Castle Project. This issue could be the result of a change in the DynamicProxy implementation or the use of a different proxy library to wrap the dependency resolver.

The typical place where this is seen is when using the action filter provider via ``ContainerBuilder.RegisterFilterProvider()``. The filter provider needs to access the Autofac dependency resolver and uses ``AutofacDependencyResolver.Current`` to do it.

If you see this, it means you're decorating the resolver in a way that can't be unwrapped and functions that rely on ``AutofacDependencyResolver.Current`` will fail. The current solution is to not decorate the dependency resolver.

Glimpse Integration
===================

Integration of an MVC application with Glimpse when using Autofac is pretty much the same as with any other integration. **However, if you use action method parameter injection** (e.g., with ``builder.InjectActionInvoker()``) then Glimpse execution inspection will fail.

You can work around this by adding the following to your Glimpse configuration:

.. sourcecode:: xml

    <glimpse defaultRuntimePolicy="On" endpointBaseUri="~/Glimpse.axd">
      <inspectors>
        <ignoredTypes>
          <add type="Glimpse.Mvc.Inspector.ExecutionInspector, Glimpse.Mvc"/>
        </ignoredTypes>
      </inspectors>
      <tabs>
        <ignoredTypes>
          <add type="Glimpse.Mvc.Tab.Execution, Glimpse.Mvc"/>
        </ignoredTypes>
      </tabs>
  </glimpse>

Again, you **only need to do this if you're using the action parameter injection**. This is one of the many reasons it's recommended to use controller constructor injection instead of action method parameter injection.

For more info on why this is (including links to the associated info from Glimpse), `check out this issue <https://github.com/autofac/Autofac.Mvc/issues/7>`_.

Unit Testing
============

When unit testing an ASP.NET MVC app that uses Autofac where you have ``InstancePerRequest`` components registered, you'll get an exception when you try to resolve those components because there's no HTTP request lifetime during a unit test.

The :doc:`per-request lifetime scope <../faq/per-request-scope>` topic outlines strategies for testing and troubleshooting per-request-scope components.

Example
=======

There is an example project showing ASP.NET MVC integration `in the Autofac examples repository <https://github.com/autofac/Examples/tree/master/src/MvcExample>`_.
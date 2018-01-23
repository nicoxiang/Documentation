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

使用 "插件型" 程序集
=========================

如果你的控制器在并不是主应用直接引用的 "插件型程序集" 中, `你需要用ASP.NET BuildManager注册你的控制器插件程序集 <http://www.paraesthesia.com/archive/2013/01/21/putting-controllers-in-plugin-assemblies-for-asp-net-mvc.aspx>`_.

你可以通过配置或以编程形式完成.

**如果你选择通过配置**, 你需要添加你的插件程序集到 ``/configuration/system.web/compilation/assemblies`` 列表. 如果你的插件程序集不在 ``bin`` 文件夹中, 你还需要修改 ``/configuration/runtime/assemblyBinding/probing`` 路劲.

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

**如果你选择通过编程形式**, 你需要在pre-application-start阶段, ASP.NET ``BuildManager`` 开始生效前去做.

创建一个initializer类, 用 ``BuildManager`` 来做程序集扫描/加载和注册的工作:

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

确保用一个程序集特性来注册你的pre-application-start代码:

.. sourcecode:: csharp

    [assembly: PreApplicationStartMethod(typeof(Initializer), "Initialize")]

使用Current Autofac DependencyResolver
============================================

只要你设置了MVC的 ``DependencyResolver`` 为 ``AutofacDependencyResolver``, 你就可以使用 ``AutofacDependencyResolver.Current`` 作为获取当前依赖解析器的快捷方法并且把它转换为 ``AutofacDependencyResolver``.

不幸的是, 在使用 ``AutofacDependencyResolver.Current`` 会存在一些问题导致无法正常运行. 通常问题由使用 `Glimpse <http://getglimpse.com/>`_ 或 `Castle DynamicProxy <http://www.castleproject.org/projects/dynamicproxy/>`_ 这类会 "包装(wrap)" 或 "修饰(decorate)" 依赖解析器来添加功能的产品导致. 如果当前依赖解析器被修饰或包装/被代理(proxied), 你不可以将它转换成 ``AutofacDependencyResolver`` 也无法 "解开包装" 或获得真实的解析器.

Autofac MVC 集成 3.3.3 版本之前, 我们通过把当前依赖解析器动态地添加到请求生命周期作用域来跟踪它. 这就让我们遇到了问题因为我们不能从代理中解开 ``AutofacDependencyResolver`` 的包装... 但它也意味着 ``AutofacDependencyResolver.Current`` 只能在请求生命周期内有效 - 你无法在后台任务或应用启动时使用它.

从 3.3.3 版本开始, 定位 ``AutofacDependencyResolver.Current`` 的逻辑修改为首先尝试转换当前依赖解析器; 如果转换失败会去特意地查找解析器被 `Castle DynamicProxy <http://www.castleproject.org/projects/dynamicproxy/>`_ 包装了的一个标识并且通过反射解开包装. 这个时候如果我们还是... 我们无法找到当前 ``AutofacDependencyResolver`` 因此我们会抛出 ``InvalidOperationException`` 异常信息如下:

    The dependency resolver is of type 'Some.Other.DependencyResolver' but was expected to be of type 'Autofac.Integration.Mvc.AutofacDependencyResolver'. It also does not appear to be wrapped using DynamicProxy from the Castle Project. This issue could be the result of a change in the DynamicProxy implementation or the use of a different proxy library to wrap the dependency resolver.

这通常会出现在通过 ``ContainerBuilder.RegisterFilterProvider()`` 使用action filter provider时. Filter provider 会用到Autofac依赖解析器并使用 ``AutofacDependencyResolver.Current`` .

如果你看到该错误, 这意味着你正在以一种无法解开的方式修饰解析器, 那些依赖 ``AutofacDependencyResolver.Current`` 的方法将会失败. 目前的解决方案是不要修饰解析器.

Glimpse 集成
===================

使用Autofac时, MVC应用和Glimpse的集成和其它的集成大体上一致. **然而, 如果你在使用action方法参数注入** (如, 使用 ``builder.InjectActionInvoker()``) 那么Glimpse运行时检查将会失败.

可以通过添加如下Glimpse配置解决:

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

再次提醒, 你 **只需要在使用action方法参数注入时这么做**. 这也是我们推荐使用控制器构造方法参数注入而不是action方法参数注入的众多原因之一.

更多关于为什么这样的原因 (包括Glimpse相关信息的链接), `请查看此 issue <https://github.com/autofac/Autofac.Mvc/issues/7>`_.

单元测试
============

当单元测试一个使用Autofac注册了 ``InstancePerRequest`` 组件的ASP.NET MVC应用时, 当你尝试解析这些组件时你会得到一个异常因为在单元测试中并没有HTTP请求生命周期.

:doc:`per-request lifetime scope <../faq/per-request-scope>` 章节概述了测试和检查per-request-scope组件的对策.

示例
=======

`Autofac示例代码仓库 <https://github.com/autofac/Examples/tree/master/src/MvcExample>`_ 里有一个展示了ASP.NET MVC集成的示例项目.
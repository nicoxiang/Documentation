=========
Web Forms
=========

ASP.NET web forms集成需要 `Autofac.Web NuGet package <https://www.nuget.org/packages/Autofac.Web/>`_.

Web forms集成为后台代码(code-behind)类提供了依赖注入. 它同样添加 :doc:`每个请求生命周期支持 <../faq/per-request-scope>`.

**本章节解释了ASP.NET classic web forms集成.** 如果你使用ASP.NET Core, :doc:`见ASP.NET Core集成章节 <aspnetcore>`.

.. contents::
  :local:

入门
===========
为了把Autofac集成进web forms你需要引入web forms集成NuGet包, 添加模块到 ``web.config``, 并让你应用的 ``Global`` 类实现 ``IContainerProviderAccessor`` 接口.

添加模块到 ``web.config``:

.. sourcecode:: xml

    <configuration>
      <system.web>
        <httpModules>
          <!-- This section is used for IIS6 -->
          <add
            name="ContainerDisposal"
            type="Autofac.Integration.Web.ContainerDisposalModule, Autofac.Integration.Web"/>
          <add
            name="PropertyInjection"
            type="Autofac.Integration.Web.Forms.PropertyInjectionModule, Autofac.Integration.Web"/>
        </httpModules>
      </system.web>
      <system.webServer>
        <!-- This section is used for IIS7 -->
        <modules>
          <add
            name="ContainerDisposal"
            type="Autofac.Integration.Web.ContainerDisposalModule, Autofac.Integration.Web"
            preCondition="managedHandler"/>
          <add
            name="PropertyInjection"
            type="Autofac.Integration.Web.Forms.PropertyInjectionModule, Autofac.Integration.Web"
            preCondition="managedHandler"/>
        </modules>
      </system.webServer>
    </configuration>

实现 ``IContainerProviderAccessor``:

.. sourcecode:: csharp

    public class Global : HttpApplication, IContainerProviderAccessor
    {
      // Provider that holds the application container.
      static IContainerProvider _containerProvider;

      // Instance property that will be used by Autofac HttpModules
      // to resolve and inject dependencies.
      public IContainerProvider ContainerProvider
      {
        get { return _containerProvider; }
      }

      protected void Application_Start(object sender, EventArgs e)
      {
        // Build up your application container and register your dependencies.
        var builder = new ContainerBuilder();
        builder.RegisterType<SomeDependency>();
        // ... continue registering dependencies...

        // Once you're done registering things, set the container
        // provider up with your registrations.
        _containerProvider = new ContainerProvider(builder.Build());
      }
    }

下面的内容进一步详细解释了这些功能做了什么和如何使用它们.

添加模块到Web.config
=========================

Autofac管理组件生命周期和添加依赖注入到ASP.NET管道的方法是实现 `IHttpModule <https://msdn.microsoft.com/en-us/library/system.web.ihttpmodule.aspx>`_ . 你需要在 ``web.config`` 中进行配置.

下面的配置片段展示了配置的模块.

.. sourcecode:: xml

    <configuration>
      <system.web>
        <httpModules>
          <!-- This section is used for IIS6 -->
          <add
            name="ContainerDisposal"
            type="Autofac.Integration.Web.ContainerDisposalModule, Autofac.Integration.Web"/>
          <add
            name="PropertyInjection"
            type="Autofac.Integration.Web.Forms.PropertyInjectionModule, Autofac.Integration.Web"/>
        </httpModules>
      </system.web>
      <system.webServer>
        <!-- This section is used for IIS7 -->
        <modules>
          <add
            name="ContainerDisposal"
            type="Autofac.Integration.Web.ContainerDisposalModule, Autofac.Integration.Web"
            preCondition="managedHandler"/>
          <add
            name="PropertyInjection"
            type="Autofac.Integration.Web.Forms.PropertyInjectionModule, Autofac.Integration.Web"
            preCondition="managedHandler"/>
        </modules>
      </system.webServer>
    </configuration>

注意有两块不同模块出现 - IIS6和IIS7各一块 - **建议同时采用两者**. ASP.NET开发者服务器(ASP.NET Developer Server)会使用IIS6配置, 即使你的目标发布环境是IIS7. 如果你使用IIS Express, 他会使用IIS7配置.

你看到的模块做了以下有趣的事:

- **ContainerDisposalModule** 只要请求完成, 就会让Autofac释放掉所有在处理请求过程中创建的组件.
- **PropertyInjectionModule** 在页面生命周期开始前注入依赖. Autofac提供了另一个可替换的模块 ``UnsetPropertyInjectionModule`` , 它只会设置web forms/controls上值为null的属性. (两者只能用其一.)

Global.asax实现IContainerProviderAccessor
===================================================

依赖注入模块需要 ``HttpApplication`` 实例支持 ``IContainerProviderAccessor``. 下面是一个完整的global类:

.. sourcecode:: csharp

    public class Global : HttpApplication, IContainerProviderAccessor
    {
      // Provider that holds the application container.
      static IContainerProvider _containerProvider;

      // Instance property that will be used by Autofac HttpModules
      // to resolve and inject dependencies.
      public IContainerProvider ContainerProvider
      {
        get { return _containerProvider; }
      }

      protected void Application_Start(object sender, EventArgs e)
      {
        // Build up your application container and register your dependencies.
        var builder = new ContainerBuilder();
        builder.RegisterType<SomeDependency>();
        // ... continue registering dependencies...

        // Once you're done registering things, set the container
        // provider up with your registrations.
        _containerProvider = new ContainerProvider(builder.Build());
      }
    }

``Autofac.Integration.Web.IContainerProvider`` 暴露了两个有用的属性: ``ApplicationContainer`` 和 ``RequestLifetime``.

- ``ApplicationContainer`` 是在应用启动时创建的根容器.
- ``RequestLifetime`` 是一个建立在应用容器上的组件 :doc:`生命周期作用域 <../lifetime/index>` , 它将会在当前web请求结束时释放. 无论是手动依赖解析还是服务查找, 它都会被用到. 它包含的组件(单例除外)对于当前请求是特有的(即是 :doc:`每个请求生命周期依赖 <../faq/per-request-scope>` 解析的地方).

小贴士
===============

构造页面及控件以支持DI
------------------------------------------

为了把依赖注入进web forms pages (``System.Web.UI.Page`` 实例) 或用户控件 (``System.Web.UI.UserControl`` 实例) **你必须把这些依赖暴露为公有的并允许设置**. 这样才能让 ``PropertyInjectionModule`` 为你填充这些属性.

确保在应用启动时注册你需要的依赖.

.. sourcecode:: csharp

    var builder = new ContainerBuilder();
    builder.RegisterType<Component>().As<IService>().InstancePerRequest();
    // ... continue registering dependencies and then build the
    // container provider...
    _containerProvider = new ContainerProvider(builder.Build());

然后在你页面的后台代码中, 为你需要的依赖创建公有的get/set属性:

.. sourcecode:: csharp

    // MyPage.aspx.cs
    public partial class MyPage : Page
    {
      // This property will be set for you by the PropertyInjectionModule.
      public IService MyService { get; set; }

      protected void Page_Load(object sender, EventArgs e)
      {
        // Now you can use the property that was set for you.
        label1.Text = this.MyService.GetMessage();
      }
    }

公有属性注入的处理对于用户控件同样也是有用的 - 只要在应用启动时注册依赖并为依赖提供公有的get/set属性.

务必要注意 **只有当用户控件是在页面请求生命周期的PreLoad阶段被创建并加入到页面控件集合的情况下, 属性才会被自动注入**. 动态创建的控件, 无论是在代码中创建还是通过类似Repeater这样的模板创建, 此时都无法拿到依赖并且必需手动注入属性.

手动依赖注入
-------------------------

在有些情况下, 例如程序创建用户控件或其它对象时, 你需要手动注入到该对象的属性上. 为此, 你需要:

- 获取当前应用实例.
- 转化成 ``Autofac.Integration.Web.IContainerProviderAccessor``.
- 从应用实例中获取容器提供者(container provider).
- 从 ``IContainerProvider`` 获取 ``RequestLifetime`` 并且使用 ``InjectProperties()`` 方法注入到该对象的属性上.

代码应该是这样的:

.. sourcecode:: csharp

    var cpa = (IContainerProviderAccessor)HttpContext.Current.ApplicationInstance;
    var cp = cpa.ContainerProvider;
    cp.RequestLifetime.InjectProperties(objectToSet);

注意你需要 ``Autofac`` 和 ``Autofac.Integration.Web`` 命名空间来让属性注入有效因为 ``InjectProperties()`` 是 ``Autofac`` 命名空间下的扩展方法.

通过特性显式注入
---------------------------------

当添加依赖注入到一个已经存在的应用中时, 有时候要区分有些web forms页面需要注入它们的依赖, 而有些不需要. ``Autofac.Integration.Web`` 中的 ``InjectPropertiesAttribute``, 搭配 ``AttributedInjectionModule`` 能够达到这一目的.

**如果你选择使用AttributedInjectionModule, 那么除非你用特殊的特性标记, 否则依赖将不会自动注入到公有属性上.**

首先, 从你的 ``web.config`` 文件中移除 ``PropertyInjectionModule`` 并且使用 ``AttributedInjectionModule`` 替换:

.. sourcecode:: xml

    <configuration>
      <system.web>
        <httpModules>
          <!-- This section is used for IIS6 -->
          <add
            name="ContainerDisposal"
            type="Autofac.Integration.Web.ContainerDisposalModule, Autofac.Integration.Web"/>
          <add
            name="AttributedInjection"
            type="Autofac.Integration.Web.Forms.AttributedInjectionModule, Autofac.Integration.Web"/>
        </httpModules>
      </system.web>
      <system.webServer>
        <!-- This section is used for IIS7 -->
        <modules>
          <add
            name="ContainerDisposal"
            type="Autofac.Integration.Web.ContainerDisposalModule, Autofac.Integration.Web"
            preCondition="managedHandler"/>
          <add
            name="AttributedInjection"
            type="Autofac.Integration.Web.Forms.AttributedInjectionModule, Autofac.Integration.Web"
            preCondition="managedHandler"/>
        </modules>
      </system.webServer>
    </configuration>

完成后, 页面和控件将默认不会注入依赖. 它们必须被标记为 ``Autofac.Integration.Web.Forms.InjectPropertiesAttribute`` 或 ``Autofac.Integration.Web.Forms.InjectUnsetPropertiesAttribute``. 区别在于:

- ``InjectPropertiesAttribute`` 只要相关的组件用Autofac注册了, 它总会设置页面或控件上的公有属性.
- ``InjectUnsetPropertiesAttribute`` 如果相关的组件用Autofac注册了且页面或控件上的公有属性为null, 它才会去设值.

.. sourcecode:: csharp

    [InjectProperties]
    public partial class MyPage : Page
    {
      // This property will be set for you by the AttributedInjectionModule.
      public IService MyService { get; set; }

      // ...use the property later as needed.
    }

通过基础页(Base Page)类完成依赖注入
----------------------------------------

如果你不想使用模块来自动注入依赖 (如, 上面提到的 ``AttributedInjectionModule`` 或 ``PropertyInjectionModule`` ), 你通过创建一个基础页这种更为手动的方式来集成Autofac, 在页面请求的 ``PreInit`` 阶段完成手动注入依赖.

这种方法允许你让那些需要注入依赖的页面继承自一个基础页类. 如果你只有一小部分页面需要依赖注入并且你不想要管道中存在 ``AttributedInjectionModule`` , 这种方法非常棒. (你依然需要 ``ContainerDisposalModule``.) 如果你不只是只有一小部分页面, 考虑通过特性来显式注入依赖非常有利.

.. sourcecode:: csharp

    protected void Page_PreInit(object sender, EventArgs e)
    {
      var cpa = (IContainerProviderAccessor)HttpContext.Current.ApplicationInstance;
      var cp = cpa.ContainerProvider;
      cp.RequestLifetime.InjectProperties(this);
    }

自定义依赖注入模块
-----------------------------------

如果提供的 *Property*, *Unset Property* 和 *Attributed* 依赖注入的模式都不合适, 创建一个自定义注入行为也很容易. 只要继承 ``Autofac.Integration.Web.DependencyInjectionModule`` 并在 ``Web.config`` 中使用结果.

这是要实现的一个抽象成员:

.. sourcecode:: csharp

    protected abstract IInjectionBehaviour GetInjectionBehaviourForHandlerType(Type handlerType);

返回的 ``IInjectionBehaviour`` 可以是预定义的 ``NoInjection``, ``PropertyInjection`` 或 ``UnsetPropertyInjection`` 属性中一个; 或者是 ``IInjectionBehaviour`` 接口的一个自定义实现.

示例
=======

`Autofac示例代码仓库 <https://github.com/autofac/Examples/tree/master/src/WebFormsExample>`_ 里有一个展示了ASP.NET web forms集成的示例项目.
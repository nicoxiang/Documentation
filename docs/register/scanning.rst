=================
程序集扫描
=================

Autofac可以在程序集中通过约定找到和注册组件. 你可以扫描和注册单独的类型, 也可以专门扫描 :doc:`Autofac模块 <../configuration/modules>`.

扫描类型
==================

Autofac也可以被认为是约定驱动注册或扫描的, 它可以根据用户指定的规则注册一组类型:

.. sourcecode:: csharp

    var dataAccess = Assembly.GetExecutingAssembly();

    builder.RegisterAssemblyTypes(dataAccess)
           .Where(t => t.Name.EndsWith("Repository"))
           .AsImplementedInterfaces();

每次 ``RegisterAssemblyTypes()`` 方法调用将应用一组规则 - 如果有多组不同的组件注册时, 我们有必要多次调用 ``RegisterAssemblyTypes()`` .

过滤类型
---------------

``RegisterAssemblyTypes()`` 接收包含一个或多个程序集的数组作为参数. 默认地, **程序中所有具体的类都将被注册.** 包括内部类(internal)和嵌套的私有类. 你可以使用LINQ表达式过滤注册的类型集合.

4.8.0 版本中 ``PublicOnly()`` 扩展方法被引入了, 这使得数据的封装变得更容易了. 如果你只想要你的公有方法被注册, 使用 ``PublicOnly()``:

.. sourcecode:: csharp

    builder.RegisterAssemblyTypes(asm)
           .PublicOnly();

要过滤注册的类型, 使用 ``Where()`` 表达式:

.. sourcecode:: csharp

    builder.RegisterAssemblyTypes(asm)
           .Where(t => t.Name.EndsWith("Repository"));

要从扫描类型中排除类型, 使用 ``Except()`` 表达式:

.. sourcecode:: csharp

    builder.RegisterAssemblyTypes(asm)
           .Except<MyUnwantedType>();

``Except()`` 表达式同样允许你自定义被排除类型的注册规则:

.. sourcecode:: csharp

    builder.RegisterAssemblyTypes(asm)
           .Except<MyCustomisedType>(ct =>
              ct.As<ISpecial>().SingleInstance());

也可以使用多个过滤条件, 他们将会以AND逻辑连接.

.. sourcecode:: csharp

    builder.RegisterAssemblyTypes(asm)
           .PublicOnly()
           .Where(t => t.Name.EndsWith("Repository"))
           .Except<MyUnwantedRepository>();


指定服务
-------------------

``RegisterAssemblyTypes()`` 的注册语法是 :doc:`单个类型注册语法 <index>` 的超集, 因此类似 ``As()`` 方法同样适用于程序集:

.. sourcecode:: csharp

    builder.RegisterAssemblyTypes(asm)
           .Where(t => t.Name.EndsWith("Repository"))
           .As<IRepository>();

``As()`` 和 ``Named()`` 方法还有接收lambda表达式的重载, 表达式决定了对于一个类型而言, 它提供了哪些服务:

.. sourcecode:: csharp

    builder.RegisterAssemblyTypes(asm)
           .As(t => t.GetInterfaces()[0]);

和普通组件注册一样, 多次调用 ``As()`` 方法, 暴露的服务叠加.

为了能够更容易地建立常用的约定, Autofac添加了一些额外的注册方法:

+-------------------------------+---------------------------------------+--------------------------------------------------------+
| Method                        | Description                           | Example                                                |
+===============================+=======================================+========================================================+
| ``AsImplementedInterfaces()`` | Register the type as providing        | ::                                                     |
|                               | all of its public interfaces as       |                                                        |
|                               | services (excluding ``IDisposable``). |      builder.RegisterAssemblyTypes(asm)                |
|                               |                                       |             .Where(t => t.Name.EndsWith("Repository")) |
|                               |                                       |             .AsImplementedInterfaces();                |
+-------------------------------+---------------------------------------+--------------------------------------------------------+
| ``AsClosedTypesOf(open)``     | Register types that are assignable to | ::                                                     |
|                               | a closed instance of the open         |                                                        |
|                               | generic type.                         |      builder.RegisterAssemblyTypes(asm)                |
|                               |                                       |             .AsClosedTypesOf(typeof(IRepository<>));   |
+-------------------------------+---------------------------------------+--------------------------------------------------------+
| ``AsSelf()``                  | The default: register types as        | ::                                                     |
|                               | themselves - useful when also         |                                                        |
|                               | overriding the default with another   |      builder.RegisterAssemblyTypes(asm)                |
|                               | service specification.                |             .AsImplementedInterfaces()                 |
|                               |                                       |             .AsSelf();                                 |
+-------------------------------+---------------------------------------+--------------------------------------------------------+

扫描模块
====================

我们通过 ``RegisterAssemblyModules()`` 方法进行模块扫描, 正如它名字的表达的意思那样. 它通过提供的程序集扫描 :doc:`Autofac模块 <../configuration/modules>`, 创建模块的实例, 然后使用当前的container builder来注册它们.

例如, 假设下面两个普通的模块类在同一个程序集中, 并且每个模块注册一个组件:

.. sourcecode:: csharp

    public class AModule : Module
    {
      protected override void Load(ContainerBuilder builder)
      {
        builder.Register(c => new AComponent()).As<AComponent>();
      }
    }

    public class BModule : Module
    {
      protected override void Load(ContainerBuilder builder)
      {
        builder.Register(c => new BComponent()).As<BComponent>();
      }
    }

``RegisterAssemblyModules()`` 的重载 *不接受类型参数* , 它将会注册所提供程序集列表中的所有实现 ``IModule`` 的类. 在下面的示例中 **所有的模块** 都将被注册:

.. sourcecode:: csharp

    var assembly = typeof(AComponent).Assembly;
    var builder = new ContainerBuilder();

    // Registers both modules
    builder.RegisterAssemblyModules(assembly);

使用 *泛型类型参数* 的 ``RegisterAssemblyModules()`` 的重载允许你指定一个所有模块都必须从它派生的基类. 在下面的示例中 **只有一个模块** 被注册了因为扫描被限制了:

.. sourcecode:: csharp

    var assembly = typeof(AComponent).Assembly;
    var builder = new ContainerBuilder();

    // Registers AModule but not BModule
    builder.RegisterAssemblyModules<AModule>(assembly);

使用 *一个Type对象参数* 的 ``RegisterAssemblyModules()`` 和使用泛型类型参数的重载作用差不多但它允许你指定一个也许会在运行时才被决定的type. 在下面的示例中 **只有一个模块** 被注册了因为扫描被限制了:

.. sourcecode:: csharp

    var assembly = typeof(AComponent).Assembly;
    var builder = new ContainerBuilder();

    // Registers AModule but not BModule
    builder.RegisterAssemblyModules(typeof(AModule), assembly);

IIS 托管的 Web 应用
===========================

当在IIS托管的应用中使用程序集扫描时, 你可能会因为程序集的位置遇到一个小小的问题. (:doc:`问答章节中的一个问题 <../faq/iis-restart>`)

应用第一次启动时IIS托管应用里面所有的程序集都被加载进 ``AppDomain`` , 但是 **当AppDomain被IIS回收时, 程序集只会按需加载.**

为了避免这个问题, 使用位于 `System.Web.Compilation.BuildManager <https://msdn.microsoft.com/en-us/library/system.web.compilation.buildmanager.aspx>`_ 的 `GetReferencedAssemblies() <https://msdn.microsoft.com/en-us/library/system.web.compilation.buildmanager.getreferencedassemblies.aspx>`_ 方法来获取相关程序集的列表:

.. sourcecode:: csharp

    var assemblies = BuildManager.GetReferencedAssemblies().Cast<Assembly>();

它会立刻强制相关的程序集加载进 ``AppDomain`` 使其可以被用于模块扫描.

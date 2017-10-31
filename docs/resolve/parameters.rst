=============================
解析时传参
=============================

:doc:`解析服务 <../index>` 时, 你也许会发现需要传参. (如果你在注册时已经知道了参数值, :doc:`你也可以在注册时提供 <../register/parameters>`.)

``Resolve()`` 方法使用一个可变长度的参数列表接受 :doc:`等同于注册时传递的参数类型 <../register/parameters>` . 另外, :doc:`委托工厂 <../advanced/delegate-factories>` 和 ``Func<T>`` :doc:`隐式关系类型 <../resolve/relationships>` 同样提供了解析时传参的途径.

可用参数类型
=========================

Autofac提供了多种不同的参数匹配机制:

* ``NamedParameter`` - 通过名称匹配目标参数
* ``TypedParameter`` - 通过类型匹配目标参数 (需要匹配具体类型)
* ``ResolvedParameter`` - 灵活的参数匹配

``NamedParameter`` 和 ``TypedParameter`` 只能提供常量.

``ResolvedParameter`` 可作为一种提供'从容器中动态获取到的值'的方法, 例如, 通过名称解析服务.

反射组件的参数
=====================================

当你解析一个基于反射的组件时, 该类型的构造方法也许会需要一个你在运行时才会知道的参数值, 这些参数值你并不能在注册时就知道. 你可以使用 ``Resolve()`` 方法中的一个参数来提供该值.

假设你有一个需要传入 configuration section name 的 configuration reader :

.. sourcecode:: csharp

    public class ConfigReader : IConfigReader
    {
      public ConfigReader(string configSectionName)
      {
        // Store config section name
      }

      // ...read configuration based on the section name.
    }

你可以在调用 ``Resolve()`` 时传参:

.. sourcecode:: csharp

    var reader = scope.Resolve<ConfigReader>(new NamedParameter("configSectionName", "sectionName"));

:doc:`跟注册时传参一样 <../register/parameters>`, 假设 ``ConfigReader`` 组件 :doc:`使用反射注册 <../register/registration>`, 示例中的 ``NamedParameter`` 将会对应于相关名称的构造方法参数.

如果你有不止一个参数, 在 ``Resolve()`` 方法中一并传入即可:

.. sourcecode:: csharp

    var service = scope.Resolve<AnotherService>(
                    new NamedParameter("id", "service-identifier"),
                    new TypedParameter(typeof(Guid), Guid.NewGuid()),
                    new ResolvedParameter(
                      (pi, ctx) => pi.ParameterType == typeof(ILog) && pi.Name == "logger",
                      (pi, ctx) => LogManager.GetLogger("service")));

Lambda表达式组件的参数
============================================

lambda表达式组件注册时, 你需要在lambda表达式中添加参数处理因此在调用 ``Resolve()`` 方法时将值传入, 这样你就能体验它的好处了.

组件注册表达式中, 改变一下用以注册的委托签名, 之后我们就能充分使用入参了. 因为你不只是传入一个 ``IComponentContext`` 参数, 还要同时传入 ``IComponentContext`` 和 ``IEnumerable<Parameter>``:

.. sourcecode:: csharp

    // Use TWO parameters to the registration delegate:
    // c = The current IComponentContext to dynamically resolve dependencies
    // p = An IEnumerable<Parameter> with the incoming parameter set
    builder.Register((c, p) =>
                     new ConfigReader(p.Named<string>("configSectionName")))
           .As<IConfigReader>();

这样当你解析 ``IConfigReader`` 时, 你的lambda表达式将会使用你传入的参数值:

.. sourcecode:: csharp

    var reader = scope.Resolve<IConfigReader>(new NamedParameter("configSectionName", "sectionName"));

不显式调用Resolve传参
=====================================================

Autofac支持两种功能, 它们允许你自动生成支持强类型参数列表的服务 "工厂" , 这些参数列表会在解析时用到. 这是一种创建需要参数组件实例的更清晰的方法.

- :doc:`委托工厂 <../advanced/delegate-factories>` 允许你定义工厂委托方法.
- The ``Func<T>`` :doc:`隐式关系类型 <../resolve/relationships>` 可以提供自动生成工厂的方法.
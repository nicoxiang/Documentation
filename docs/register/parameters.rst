==============================
注册时传参
==============================

:doc:`注册组件 <registration>` 时你可以提供一组参数, 可以在基于该组件的 :doc:`服务解析 <../resolve/index>` 时使用. (如果你想要在解析时提供参数, :doc:`当然也是可以的 <../resolve/parameters>`.)

可使用的参数类型
=========================

Autofac提供了多种不同的参数匹配机制:

* ``NamedParameter`` - 通过名字匹配目标参数
* ``TypedParameter`` - 通过类型匹配目标参数 (需要匹配具体的类型)
* ``ResolvedParameter`` - 复杂参数的匹配

``NamedParameter`` 和 ``TypedParameter`` 只支持常量值.

``ResolvedParameter`` 可以用于提供不同的值来从容器中动态获取对象, 例如, 通过名字解析服务.

反射组件的参数
=====================================

当你注册一个基于反射的组件时, 类型的构造方法也许会需要一个无法从容器中解析出来的参数. 你可以在注册时提供该值.

假设你有个 configuration reader 需要传入一个 configuration section name :

.. sourcecode:: csharp

    public class ConfigReader : IConfigReader
    {
      public ConfigReader(string configSectionName)
      {
        // Store config section name
      }

      // ...read configuration based on the section name.
    }

你可以使用lambda表达式组件:

.. sourcecode:: csharp

    builder.Register(c => new ConfigReader("sectionName")).As<IConfigReader>();

或者在反射组件注册时传参:

.. sourcecode:: csharp

    // Using a NAMED parameter:
    builder.RegisterType<ConfigReader>()
           .As<IConfigReader>()
           .WithParameter("configSectionName", "sectionName");

    // Using a TYPED parameter:
    builder.RegisterType<ConfigReader>()
           .As<IConfigReader>()
           .WithParameter(new TypedParameter(typeof(string), "sectionName"));

    // Using a RESOLVED parameter:
    builder.RegisterType<ConfigReader>()
           .As<IConfigReader>()
           .WithParameter(
             new ResolvedParameter(
               (pi, ctx) => pi.ParameterType == typeof(string) && pi.Name == "configSectionName",
               (pi, ctx) => "sectionName"));

Lambda表达式组件的参数
============================================

使用lambda表达式组件注册, 你可以不用 *在注册时* 传入参数值, 而是可以 *在服务解析时* 传入具体的参数值. (:doc:`阅读使用参数解析章节. <../resolve/parameters>`)

在组件注册表达式中, 你可以用入参来改变你用于注册的委托签名. 并且不要只接受一个 ``IComponentContext`` 参数, 接受一个 ``IComponentContext`` 和一个 ``IEnumerable<Parameter>``:

.. sourcecode:: csharp

    // Use TWO parameters to the registration delegate:
    // c = The current IComponentContext to dynamically resolve dependencies
    // p = An IEnumerable<Parameter> with the incoming parameter set
    builder.Register((c, p) =>
                     new ConfigReader(p.Named<string>("configSectionName")))
           .As<IConfigReader>();

当 :doc:`使用参数解析时 <../resolve/parameters>`, 你的lambda表达式会传入这些参数的值:

.. sourcecode:: csharp

    var reader = scope.Resolve<IConfigReader>(new NamedParameter("configSectionName", "sectionName"));
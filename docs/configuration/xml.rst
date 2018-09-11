==========================
JSON/XML 配置
==========================

大多数的IoC容器在提供可编程配置接口的同时, 也会支持基于JSON/XML文件配置, Autofac也不例外.

Autofac鼓励通过 ``ContainerBuilder`` 类进行编程配置. 使用可编程接口是容器设计的核心. 当遇到实体类无法在编译时被选中或配置的情况, 推荐使用JSON 或 XML.

在深入挖掘JSON/XML配置章节前, 请确保已阅读 :doc:`Modules<modules>` - 该章节解释了如何处理比JSON/XML组件注册所能容许的更复杂的场景. JSON/XML配置并不是编程配置的功能替代方案, 复杂的场景也许会需要JSON/XML和模块(modules)配置的组合.

.. contents::
  :local:
  :depth: 2

使用 Microsoft Configuration 配置(4.0+)
===============================================

.. note::

   Microsoft Configuration 适用于Autofac.Configuration 4.0+ 版本. 之前版本的configuration包无法适用.

随着 `Microsoft.Extensions.Configuration <https://www.nuget.org/packages/Microsoft.Extensions.Configuration>`_, 和 Autofac.Configuration 4.0.0 的发布, Autofac利用了更复杂的配置模型, 而在此之前它受限于应用的配置文件. 如果你以前使用基于 ``app.config`` 或 ``web.config`` 的配置文件, 你将需要迁移你的配置到最新的格式, 并更改你进行应用容器配置的方式.

入门
-----------
基础的应用配置有以下几步:

1. 创建能够被 ``Microsoft.Extensions.Configuration`` 读取的JSON或XML配置文件.

   * JSON配置使用 ``Microsoft.Extensions.Configuration.Json``
   * XML配置使用 ``Microsoft.Extensions.Configuration.Xml``

2. 使用 ``Microsoft.Extensions.Configuration.ConfigurationBuilder`` 构建配置.
3. 创建一个新的 ``Autofac.Configuration.ConfigurationModule`` , 然后把构建好的 ``Microsoft.Extensions.Configuration.IConfiguration`` 传递进去.
4. 用你的容器注册 ``Autofac.Configuration.ConfigurationModule``.

一个含有简单注册的配置文件类似如下:

.. sourcecode:: json

    {
      "defaultAssembly": "Autofac.Example.Calculator",
      "components": [{
        "type": "Autofac.Example.Calculator.Addition.Add, Autofac.Example.Calculator.Addition",
        "services": [{
          "type": "Autofac.Example.Calculator.Api.IOperation"
        }],
        "injectProperties": true
      }, {
        "type": "Autofac.Example.Calculator.Division.Divide, Autofac.Example.Calculator.Division",
        "services": [{
          "type": "Autofac.Example.Calculator.Api.IOperation"
        }],
        "parameters": {
          "places": 4
        }
      }]
    }

JSON更清晰易读, 但如果你更喜欢XML, 相同的配置类似如下:

.. sourcecode:: xml

    <?xml version="1.0" encoding="utf-8" ?>
    <autofac defaultAssembly="Autofac.Example.Calculator">
        <components name="0">
            <type>Autofac.Example.Calculator.Addition.Add, Autofac.Example.Calculator.Addition</type>
            <services name="0" type="Autofac.Example.Calculator.Api.IOperation" />
            <injectProperties>true</injectProperties>
        </components>
        <components name="1">
            <type>Autofac.Example.Calculator.Division.Divide, Autofac.Example.Calculator.Division</type>
            <services name="0" type="Autofac.Example.Calculator.Api.IOperation" />
            <injectProperties>true</injectProperties>
            <parameters>
                <places>4</places>
            </parameters>
        </components>
    </autofac>

*注意XML中组件和服务有顺序的 "命名(naming)" - 这是由于 Microsoft.Extensions.Configuration 处理有序集合(数组) 的方式所致.*

构建你的配置并像下面这样用 Autofac ``ContainerBuilder`` 进行注册:

.. sourcecode:: csharp

    // Add the configuration to the ConfigurationBuilder.
    var config = new ConfigurationBuilder();
    // config.AddJsonFile comes from Microsoft.Extensions.Configuration.Json
    // config.AddXmlFile comes from Microsoft.Extensions.Configuration.Xml
    config.AddJsonFile("autofac.json");

    // Register the ConfigurationModule with Autofac.
    var module = new ConfigurationModule(config.Build());
    var builder = new ContainerBuilder();
    builder.RegisterModule(module);

默认程序集
-----------------
你可以在配置中指定一个 "默认程序集" 选项, 来帮助你以更短的方式写类型. 如果你在一个类型或接口引用中没有指定一个限定程序集的类型名称, 它将会假定在默认程序集中.


.. sourcecode:: json

    {
      "defaultAssembly": "Autofac.Example.Calculator"
    }

组件
----------
组件是你注册最多的东西. 从生命周期到参数, 每个组件你都可以指定很多东西.

组件会被加入到配置的顶级的 ``components`` 元素中. 里面是你想要注册的组件数组.

下面的示例展示了一个拥有 *所有选项* 的组件, 只是为了达到说明语法的目的. 实际在任何组件的注册中你都不会用上里面所有的选项.

.. sourcecode:: json

    {
      "components": [{
        "type": "Autofac.Example.Calculator.Addition.Add, Autofac.Example.Calculator.Addition",
        "services": [{
          "type": "Autofac.Example.Calculator.Api.IOperation"
        }, {
          "type": "Autofac.Example.Calculator.Api.IAddOperation",
          "key": "add"
        }],
        "autoActivate": true,
        "injectProperties": true,
        "instanceScope": "per-dependency",
        "metadata": [{
          "key": "answer",
          "value": 42,
          "type": "System.Int32, mscorlib"
        }],
        "ownership": "external",
        "parameters": {
          "places": 4
        },
        "properties": {
          "DictionaryProp": {
            "key": "value"
          },
          "ListProp": [1, 2, 3, 4, 5]
        }
      }]
    }

====================== ======================================================================================================================================================= ===========================================================================
Element Name           Description                                                                                                                                             Valid Values
====================== ======================================================================================================================================================= ===========================================================================
``type``               The only required thing. The concrete class of the component (assembly-qualified if in an assembly other than the default).                             Any .NET type name that can be created through reflection.
``services``           An array of :doc:`services exposed by the component<../register/registration>`. Each service must have a ``type`` and may optionally specify a ``key``. Any .NET type name that can be created through reflection.
``autoActivate``       A Boolean indicating if the component should :doc:`auto-activate<../lifetime/startup>`.                                                                 ``true``, ``false``
``injectProperties``   A Boolean indicating whether :doc:`property (setter) injection<../register/prop-method-injection>` for the component should be enabled.                 ``true``, ``false``
``instanceScope``      :doc:`Instance scope<../lifetime/instance-scope>` for the component.                                                                                    ``singleinstance``, ``perlifetimescope``, ``perdependency``, ``perrequest``
``metadata``           An array of :doc:`metadata values <../advanced/metadata>` to associate with the component. Each item specifies the ``name``, ``type``, and ``value``.   Any :doc:`metadata values <../advanced/metadata>`.
``ownership``          Allows you to control :doc:`whether the lifetime scope disposes the component or your code does<../lifetime/disposal>`.                                 ``lifetimescope``, ``external``
``parameters``         A name/value dictionary where the name of each element is the name of a constructor parameter and the value is the value to inject.                     Any parameter in the constructor of the component type.
``properties``         A name/value dictionary where the name of each element is the name of a property and the value is the value to inject.                                  Any settable property on the component type.
====================== ======================================================================================================================================================= ===========================================================================

注意 ``parameters`` 和 ``properties`` 都支持字典和枚举类型. 你可以在上面示例中看到如何以JSON格式指定这些选项.

模块
-------

在Autofac使用 :doc:`modules<modules>` 时, 你可以使用配置注册这些模块和组件.

模块会被加入到配置的顶级的 ``modules`` 元素中. 里面是你想要注册的模块数组.

下面的示例展示了一个拥有 *所有选项* 的模块, 只是为了达到说明语法的目的. 实际在任何模块的注册中你都不会用上里面所有的选项.

.. sourcecode:: json

    {
      "modules": [{
        "type": "Autofac.Example.Calculator.OperationModule, Autofac.Example.Calculator",
        "parameters": {
          "places": 4
        },
        "properties": {
          "DictionaryProp": {
            "key": "value"
          },
          "ListProp": [1, 2, 3, 4, 5]
        }
      }]
    }

====================== ======================================================================================================================================================= ===============================================================================================
Element Name           Description                                                                                                                                             Valid Values
====================== ======================================================================================================================================================= ===============================================================================================
``type``               The only required thing. The concrete class of the module (assembly-qualified if in an assembly other than the default).                                Any .NET type name that derives from ``Autofac.Module`` that can be created through reflection.
``parameters``         A name/value dictionary where the name of each element is the name of a constructor parameter and the value is the value to inject.                     Any parameter in the constructor of the module type.
``properties``         A name/value dictionary where the name of each element is the name of a property and the value is the value to inject.                                  Any settable property on the module type.
====================== ======================================================================================================================================================= ===============================================================================================

注意 ``parameters`` 和 ``properties`` 都支持字典和枚举类型. 你可以在上面示例中看到如何以JSON格式指定这些选项.

允许 *使用不同的参数/属性集合注册相同的模块多次* , 如果你这样选择的话.

类型名称
----------
不管在什么情况下, 如果你看到一个类型名称 (组件类, 服务类, 模块类) , 它应该都指的是 `标准的, 限定程序集的类型名称 <https://msdn.microsoft.com/en-us/library/yfsftwz6(v=vs.110).aspx>`_ , 你可以正常地传入进 ``Type.GetType(string typename)``. 如果该类型在 ``默认程序集`` 中, 你可以去掉程序集名称, 但如果加上也是没有关系的.

限定程序集的类型名称包括带命名空间的完整类型, 逗号, 以及程序集的名称, 如 ``Autofac.Example.Calculator.OperationModule, Autofac.Example.Calculator``. 这个示例中, ``Autofac.Example.Calculator.OperationModule`` 是类型, 它在 ``Autofac.Example.Calculator`` 程序集中.

泛型稍有点复杂. 配置不支持泛型, 因此你也必须指定每个泛型参数的完整限定名.

例如, 假设你在 ``ConfigWithGenericsDemo`` 程序集中有一个 repository ``IRepository<T>``. 同时假设我们有一个类 ``StringRepository`` 实现 ``IRepository<string>``. 为了在配置中注册它, 大致如下:

.. sourcecode:: json

    {
      "components": [{
        "type": "ConfigWithGenericsDemo.StringRepository, ConfigWithGenericsDemo",
        "services": [{
          "type": "ConfigWithGenericsDemo.IRepository`1[[System.String, mscorlib]], ConfigWithGenericsDemo"
        }]
      }]
    }

如果你很难搞清楚你的类型名称是什么, 你也可以用代码做如下的事:


.. sourcecode:: csharp

    // Write the type name to the Debug output window and
    // copy/paste it out of there into your config.
    System.Diagnostics.Debug.WriteLine(typeof(IRepository<string>).AssemblyQualifiedName);

与传统配置的区别
-------------------------------------
从基于 (4.0 版本前) ``app.config`` 的格式迁移至新的格式时, 我们需要知道一些关键的改变:

- **没有ConfigurationSettingsReader了.** ``Microsoft.Extensions.Configuration`` 已经完全替换了旧的XML格式配置. 传统的配置文档不再适用于 4.0+ 系列的配置包.
- **多配置文件的处理不一样了.** 传统的配置有一个 ``files`` 元素, 它可以一次自动拉取多个文件作为配置. 现在可以使用 ``Microsoft.Extensions.Configuration.ConfigurationBuilder`` 来完成.
- **支持自动激活组件.** 你现在可以指定 :doc:`组件应自动激活 <../lifetime/startup>`, 在之前的配置方式中是不行的.
- **XML使用子元素而不是特性(attributes).** 这可以帮助XML和JSON使用 ``Microsoft.Extensions.Configuration`` 时解析时得相同, 这样就可以正确地组合XML和JSON配置源了.
- **使用XML需要以数字命名组件和服务.** ``Microsoft.Extensions.Configuration`` 需要每个配置项拥有一个名字和值. 它支持有序集合(数组)的方式是显示地给集合中未命名的元素一个数字 ("0", "1", 等等). 你可以在上面的入门中找到示例. 如果你不使用JSON, 你需要注意当心 ``Microsoft.Extensions.Configuration`` 所需要的, 否则将无法得到你想要的结果.
- **支持每个请求一个作用域(Per-request lifetime scope).** 之前你无法配置元素有 :doc:`每个请求一个作用域(per-request lifetime scope) <../lifetime/instance-scope>`. 现在你可以了.
- **名称/值中的破折号不存在了.** XML元素的名称以前包含类似 ``inject-properties`` - 为了使其以JSON配置的格式work, 现在这些是驼峰大小写, 如 ``injectProperties``.
- **服务在子元素中指定.** 传统的配置允许服务被定义在组件的顶部. 新系统中需要所有服务在 ``services`` 集合中.

额外的建议
---------------
新的 ``Microsoft.Extensions.Configuration`` 机制增加了很多灵活性. 你可以利用:

- **支持环境变量.** 你可以使用 ``Microsoft.Extensions.Configuration.EnvironmentVariables`` 来允许配置随着环境的改变而改变. 根据环境改变Autofac的注册, 可以更方便地在不动到代码的前提下去调试, 打补丁, 或者修复些东西.
- **简便的配置合并.**  ``ConfigurationBuilder`` 允许你从多个源创建配置并将它们合并为一个. 如果你有很多配置, 可以考虑扫描你的那些配置文件, 动态地构建配置而不是hardcoding路径.
- **自定义配置源.** 你可以自己实现 ``Microsoft.Extensions.Configuration.ConfigurationProvider`` 来支持不仅仅文件配置. 如果你想要集中化配置, 可以考虑数据库或是支持REST API的配置源.

使用应用配置(传统4.0版本前)
===========================================================

.. note::

   下面讨论的传统的应用配置适用于 3.x 和更早的 Autofac.Configuration 版本. 4.0+版本的configuration包无法适用.

在 `Microsoft.Extensions.Configuration <https://www.nuget.org/packages/Microsoft.Extensions.Configuration>`_ 和使用更新后配置模型之前, Autofac和.NET应用配置文件 (``app.config`` / ``web.config``) 绑定在一起. 3.x 系列的 Autofac.Configuration 包, 是这样进行配置的.

建立
-----

使用传统的配置机制, 你需要在靠近你配置文件顶部的地方定义一个section handler::

    <?xml version="1.0" encoding="utf-8" ?>
    <configuration>
        <configSections>
            <section name="autofac" type="Autofac.Configuration.SectionHandler, Autofac.Configuration"/>
        </configSections>

然后, 提供一个区段(section)来描述你的组件::

    <autofac defaultAssembly="Autofac.Example.Calculator.Api">
        <components>
            <component
                type="Autofac.Example.Calculator.Addition.Add, Autofac.Example.Calculator.Addition"
                service="Autofac.Example.Calculator.Api.IOperation" />

            <component
                type="Autofac.Example.Calculator.Division.Divide, Autofac.Example.Calculator.Division"
                service="Autofac.Example.Calculator.Api.IOperation" >
                <parameters>
                    <parameter name="places" value="4" />
                </parameters>
            </component>

``默认程序集`` 特性是可选的, 它允许使用限定命名空间类型名而不是完全限定的类型名. 这样可以不这么杂乱并减少字符, 特别是如果你每个程序集使用一个配置文件 (见如下额外配置文件部分.)

组件
----------
组件是你注册最多的东西. 从生命周期到参数, 每个组件你都可以指定很多东西.

组件特性
~~~~~~~~~~~~~~~~~~~~

下表中可以作为 ``组件`` 元素的特性 (默认值与编程式API默认值相同):

====================== =============================================================================================================================== =================================================================
Attribute Name         Description                                                                                                                     Valid Values
====================== =============================================================================================================================== =================================================================
``type``               The only required attribute. The concrete class of the component (assembly-qualified if in an assembly other than the default.) Any .NET type name that can be created through reflection.
``service``            A service exposed by the component. For more than one service, use the nested ``services`` element.                             As for ``type``.
``instance-scope``     Instance scope - see :doc:`Instance Scope<../lifetime/instance-scope>`.                                                         ``per-dependency``, ``single-instance`` or ``per-lifetime-scope``
``instance-ownership`` Container's ownership over the instances - see the ``InstanceOwnership`` enumeration.                                           ``lifetime-scope`` or ``external``
``name``               A string name for the component.                                                                                                Any non-empty string value.
``inject-properties``  Enable property (setter) injection for the component.                                                                           ``yes``, ``no``.
====================== =============================================================================================================================== =================================================================

组件子元素
~~~~~~~~~~~~~~~~~~~~~~~~

============== =======================================================================================================================================================
Element        Description
============== =======================================================================================================================================================
``services``   A list of ``service`` elements, whose element content contains the names of types exposed as services by the component (see the ``service`` attribute.)
``parameters`` A list of explicit constructor parameters to set on the instances (see example above.)
``properties`` A list of explicit property values to set (syntax as for ``parameters``.)
``metadata``   A list of ``item`` nodes with ``name``, ``value`` and ``type`` attributes.
============== =======================================================================================================================================================

在XML配置语法中, 有一些编程式API拥有的功能消失了 - 例如泛型注册. 在这种情况下推荐使用模块(modules).

模块
-------

使用组件来配置容器是非常细粒度的并且很快就会变得冗长. Autofac支持把组件打包进 :doc:`模块<./modules>` , 在提供灵活配置的时候来封装实现.

模块通过类型注册::

    <modules>
        <module type="MyModule" />

你可以用上面注册组件相同的方式, 添加嵌套的 ``parameters`` 和 ``properties`` 到模块注册中.

额外的配置文件
-----------------------

你可以加入额外的配置文件如下::

    <files>
        <file name="Controllers.config" section="controllers" />

配置容器
-------------------------

首先, 你必须 **在你的项目中引入 Autofac.Configuration.dll**.

为了配置容器, 你可以用XML配置区段(configuration section)的名称来初始化 ``ConfigurationSettingsReader``:

.. sourcecode:: csharp

    var builder = new ContainerBuilder();
    builder.RegisterModule(new ConfigurationSettingsReader("mycomponents"));
    // Register other components and call Build() to create the container.

container settings reader将会重写掉已注册的默认组件; 你可以让你的程序以合理的默认值运行, 然后重写那些只在特定部署情况必要的组件注册.

多文件或区段(sections)
--------------------------

你可以在同一个容器中使用多个 settings readers , 来读取不同的区段甚至不同的配置文件, 如果文件名传给 ``ConfigurationSettingsReader`` 构造方法.

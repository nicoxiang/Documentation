=======
模块
=======

介绍
============

IoC 使用 :doc:`组件 <../glossary>` 作为应用的基础构件. 为了达到 :doc:`部署时配置 <xml>` 的目的, 提供访问组件的构造方法参数和属性的能力是一种非常常用的做法.

这种做法是值得怀疑的, 原因如下:

 * **构造方法是可以改变的**: 改变组件的构造方法参数和属性将会破坏已部署的 ``App.config`` 文件 - 这些问题将会在开发进程的后期体现出来.
 * **JSON/XML 会变得不易维护**: 大量组件的配置文件会变得不便维护.
 * **"代码" 开始出现在配置中**: 暴露组件的构造方法参数和属性违背了应用内部 '封装' 的理念 - 这些细节的东西不应存在于配置文件中.

这时就要用到模块.

**模块能简化配置和发布, 而说白了, 它就是一个能用于绑定一系列相关组件的类.** 模块暴露了一组深思熟虑的, 有限制的配置参数, 这些参数可以独立于实现模块的组件而单独变化.

模块中的组件仍然使用在组件/服务级别的依赖来从其他模块访问组件.

**模块本身不通过依赖注入.** 它们是用于配置容器的, 事实上不会像其他组件那样被注册和解析. 例如, 如果你的模块接收一个构造方法参数, 你需要自己传入它. 它不能来自容器中.

模块的优势
=====================

降低配置复杂度
----------------------------------

用 IoC 配置应用时, 通常要给一些参数赋值, 而这些参数在不同的组件之间传递. 模块把相关的配置项分组到一个地方以降低为了赋值寻找正确组件的压力.

模块的实现决定了在内部模块的配置参数如何映射组件的构造方法参数和属性.

配置参数是显式的
-------------------------------------

直接通过组件配置的应用在升级时会需要考虑很多方面. 因为我们有可能通过一个每个网站都不同的配置文件来给任何类的任何属性设值, 所以重构将会不再安全.

创建模块限制了用户可配置的配置参数, 使得对于维护的程序员来说这些参数变得是显式的.

你也可以避免在一个好的程序元素和一个好的配置参数之间的权衡.

对内部应用架构的抽象
------------------------------------------------------

通过组件配置应用意味着配置会根据这些东西不同, 例如, 使用 ``enum`` vs. 创建策略类(strategy classes). 使用模块隐藏了应用架构的这些细节, 保证了配置文件简洁.

更加的类型安全
------------------

当组成应用的类会根据部署环境变得多样的时候, 类型安全也会有所下降. 而通过XML配置文件注册大量的组件, 反而也会加剧这个问题.

模块以编程的方式构建, 因此里面所有的组件注册逻辑都能在编译时期被检查到.

动态注册
---------------------

在模块中配置的组件是动态的: 模块的行为基于运行时环境而不一样. 而如果用纯粹的基于组件的配置, 即使并非不可能, 这也并非易事.

高级拓展
-------------------

模块不仅仅可以用于简单的类型注册 - 你也可以附加到组件解析事件, 扩展参数如何解析, 或者执行些其他的拓展. :doc:`log4net 集成模块示例 <../examples/log4net>` 展示了这样的一个模块.

示例
=======

Autofac中, 模块实现 ``Autofac.Core.IModule`` 接口. 通常继承于 ``Autofac.Module`` 抽象类.

这个模块提供了 ``IVehicle`` 服务:

.. sourcecode:: csharp

    public class CarTransportModule : Module
    {
      public bool ObeySpeedLimit { get; set; }

      protected override void Load(ContainerBuilder builder)
      {
        builder.Register(c => new Car(c.Resolve<IDriver>())).As<IVehicle>();

        if (ObeySpeedLimit)
          builder.Register(c => new SaneDriver()).As<IDriver>();
        else
          builder.Register(c => new CrazyDriver()).As<IDriver>();
      }
    }

封装配置
--------------------------

我们的 ``CarTransportModule`` 提供了 ``ObeySpeedLimit`` 配置参数, 而没有暴露它的实现其实是在理智的(sane)和疯狂的(carzy)司机之间选择的. 使用模块的客户端可以通过这样做来表明它的意图:

.. sourcecode:: csharp

    builder.RegisterModule(new CarTransportModule() {
        ObeySpeedLimit = true
    });

或以 ``Microsoft.Extensions.Configuration`` :doc:`配置格式 <xml>`:

.. sourcecode:: json

    {
      "modules": [{
        "type": "MyNamespace.CarTransportModule, MyAssembly",
        "properties": {
          "ObeySpeedLimit": true
        }
      }]
    }

这非常有用因为模块的实现可以变化同时无需连锁变动. 毕竟, 这就是封装的思想.

灵活的重写
-----------------------

虽然 ``CarTransportModule`` 的客户端主要关心 ``IVehicle`` 服务, 但模块也用容器注册 ``IDriver`` 依赖. 这确保了配置仍然能在部署时期被重写因为组成模块的组件是被独立注册的.

使用Autofac时在以编程方式配置 *后* 添加XML配置是一种 '最佳做法' , 如:

.. sourcecode:: csharp

    builder.RegisterModule(new CarTransportModule());
    builder.RegisterModule(new ConfigurationSettingsReader());

这样的话, 可以在 :doc:`配置文件 <xml>` 中完成 '紧急情况' 下的重写:

.. sourcecode:: json

    {
      "components": [{
        "type": "MyNamespace.LearnerDriver, MyAssembly",
        "services": [{
          "type": "MyNamespace.IDriver, MyAssembly"
        }]
      }]
    }

因此, 模块增加了封装性但不阻止你调整内部结构, 如果必须的话.

适应部署环境
======================================

模块可以是动态的 - 这就意味着, 它们可以根据执行环境进行自我配制.

当模块Load的时候, 它可以做一些类似检查环境这样很棒的事:

.. sourcecode:: csharp

    protected override void Load(ContainerBuilder builder)
    {
      if (Environment.OSVersion.Platform == PlatformID.Unix)
        RegisterUnixPathFormatter(builder);
      else
        RegisterWindowsPathFormatter(builder);
    }

模块常用场景
============================

 * 配置相关的服务以提供一个子系统, 如, NHibernate数据访问
 * 打包的可选应用功能 '插件'
 * 提供集成进系统的预构建的包, 如, 一个账户系统
 * 把常用的一组类似的服务注册在一起, 如, 一组 file format converters
 * 新建或自定义容器配置的机制, e.g. JSON/XML配置是用模块实现的; 使用特性配置也可以通过这种方式添加
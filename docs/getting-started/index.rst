===============
入门
===============

将Autofac整合到你的应用的基本模式如下:

- 按照 *控制反转* (IoC) 的思想构建你的应用.
- 添加Autofac引用.
- 在应用的 startup 处...
- 创建 `ContainerBuilder`.
- 注册组件.
- 创建容器,将其保存以备后续使用.
- 应用执行阶段...
- 从容器中创建一个生命周期.
- 在此生命周期作用域内解析组件实例.

本入门通过一个简单的控制台应用来介绍上述步骤. 一旦你学会了这些基础，你可以查看wiki的剩余部分来了解更多高级用法和 :doc:`集成 WCF, ASP.NET, 和其他应用类型 <../integration/index>`.

构建应用
===========================

控制反转背后的核心思想是, 我们不再将类绑定在应用里,让类自己去 "new up" 他们的依赖, 而是反过来在类的构造方法中将依赖传递进去. 如果你想进一步深入的话, `Martin Fowler 有一篇解释 dependency injection/inversion of control 非常好的文章 <http://martinfowler.com/articles/injection.html>`_ .

在我们的示例应用中, 我们将定义一个类来输出当前的日期. 然而,我们不希望和 ``Console`` 绑定因为我们想在控制台不可用的情况依然可以测试和使用这个类.

同样，我们会将输出日期的方法定义成抽象的(接口),这样如果我们后续想要切换版本输出 *明天* 的日期,我们很快就能搞定.

尝试如下代码:

.. sourcecode:: csharp

    using System;

    namespace DemoApp
    {
      // This interface helps decouple the concept of
      // "writing output" from the Console class. We
      // don't really "care" how the Write operation
      // happens, just that we can write.
      public interface IOutput
      {
        void Write(string content);
      }

      // This implementation of the IOutput interface
      // is actually how we write to the Console. Technically
      // we could also implement IOutput to write to Debug
      // or Trace... or anywhere else.
      public class ConsoleOutput : IOutput
      {
        public void Write(string content)
        {
          Console.WriteLine(content);
        }
      }

      // This interface decouples the notion of writing
      // a date from the actual mechanism that performs
      // the writing. Like with IOutput, the process
      // is abstracted behind an interface.
      public interface IDateWriter
      {
        void WriteDate();
      }

      // This TodayWriter is where it all comes together.
      // Notice it takes a constructor parameter of type
      // IOutput - that lets the writer write to anywhere
      // based on the implementation. Further, it implements
      // WriteDate such that today's date is written out;
      // you could have one that writes in a different format
      // or a different date.
      public class TodayWriter : IDateWriter
      {
        private IOutput _output;
        public TodayWriter(IOutput output)
        {
          this._output = output;
        }

        public void WriteDate()
        {
          this._output.Write(DateTime.Today.ToShortDateString());
        }
      }
    }

现在我们已经有了一组结构合理的依赖,是时候让Autofac混合进来了!

添加 Autofac 引用
======================

第一步是把Autofac的引用添加进项目. 在这次示例中，我们只使用Autofac的核心部分. :doc:`其他应用类型可能需要添加额外的Autofac集成类库. <../integration/index>`.

最简单的方法是通过 NuGet. "Autofac" 包涵盖了你需要的所有核心功能.

.. image:: gsnuget.png

应用启动
===================

在应用启动的地方, 你需要添加一个 `ContainerBuilder` 并且通过它注册你的 :doc:`组件 <../glossary>` . *组件* 可以是一个表达式, .NET 类型, 或者其他暴露一个或多个 *服务* 的一段代码, 同时它也可以引入其他的 *依赖* .

简而言之, 考虑有下面这样实现一个接口的.NET类型:

.. sourcecode:: csharp

    public class SomeType : IService
    {
    }

你可以通过两种方法访问该类型:

- 通过类型本身, ``SomeType``
- 通过接口, ``IService``

这个示例中, *组件* 指的是 ``SomeType`` 而它暴露的 *服务* 指的是 ``SomeType`` 和 ``IService``.

在Autofac中, 你需要通过 ``ContainerBuilder`` 注册, 如下:

.. sourcecode:: csharp

    // Create your builder.
    var builder = new ContainerBuilder();

    // Usually you're only interested in exposing the type
    // via its interface:
    builder.RegisterType<SomeType>().As<IService>();

    // However, if you want BOTH services (not as common)
    // you can say so:
    builder.RegisterType<SomeType>().AsSelf().As<IService>();

对于我们的示例应用, 我们需要注册所有的组件 (类) 并且暴露他们的服务 (接口) , 这样对象就能很好地连接起来.

同时我们还要保存这个容器，这样就可以在后续解析类型.

.. sourcecode:: csharp

    using System;
    using Autofac;

    namespace DemoApp
    {
      public class Program
      {
        private static IContainer Container { get; set; }

        static void Main(string[] args)
        {
          var builder = new ContainerBuilder();
          builder.RegisterType<ConsoleOutput>().As<IOutput>();
          builder.RegisterType<TodayWriter>().As<IDateWriter>();
          Container = builder.Build();

          // The WriteDate method is where we'll make use
          // of our dependency injection. We'll define that
          // in a bit.
          WriteDate();
        }
      }
    }

现在我们已经拥有了一个注册了所有 *组件* 的 *容器* , 并且他们暴露了合适的 *服务* . 开始使用它们吧.

应用执行
=====================

在应用程序执行阶段, 你需要充分利用这些刚注册的组件. 你可以从一个 *生命周期* 中 *解析* 它们.

容器本身是也是一个生命周期, 从技术角度来说, 你可以直接从Container解析组件. 然而, **我们并不推荐直接这么做**.

解析组件时, 根据 :doc:`定义的实例作用域 <../lifetime/instance-scope>`, 创建一个对象的新实例. (解析一个组件大致相当于调用"new"实例化一个类. 虽然这个概念看上去有点过于简单化了, 但是从类比的角度来说也是合适的). 一些组件需要被释放 (实现``IDisposable``接口) - :doc:`Autofac会为你在生命周期释放时处理组件的释放 <../lifetime/disposal>`.

然而, 容器在应用的生命周期内一直存在. 如果你直接从该容器中解析了太多东西, 应用结束时将会有一堆东西等着被释放. 这是非常不合适的 (很有可能造成"内存泄漏").

因此, 我们可以从容器中创建一个 *子生命周期* 并从中解析. 当你完成了解析组件, 释放掉子生命周期, 其他所有也就随之被一并清理干净了.

(当使用 :doc:`Autofac 集成类库 <../integration/index>` 时, 大部分情况下子生命周期创建已经完成了, 因此无需考虑.)

对于我们的示例应用程序, 我们在生命周期内实现"WriteDate"方法并在结束调用后释放它.

.. sourcecode:: csharp

    namespace DemoApp
    {
      public class Program
      {
        private static IContainer Container { get; set; }

        static void Main(string[] args)
        {
          // ...the stuff you saw earlier...
        }

        public static void WriteDate()
        {
          // Create the scope, resolve your IDateWriter,
          // use it, then dispose of the scope.
          using (var scope = Container.BeginLifetimeScope())
          {
            var writer = scope.Resolve<IDateWriter>();
            writer.WriteDate();
          }
        }
      }
    }

现在当运行程序时...

- "WriteDate"方法向Autofac请求一个 ``IDateWriter``.
- Autofac发现 ``IDateWriter`` 对应 ``TodayWriter`` 因此开始创建 ``TodayWriter``.
- Autofac发现 ``TodayWriter`` 在它构造方法中需要一个 ``IOutput``.
- Autofac发现 ``IOutput`` 对应 ``ConsoleOutput`` 因此开始创建新的 ``ConsoleOutput`` 实例.
- Autofac使用新的 ``ConsoleOutput`` 实例完成 ``TodayWriter`` 的创建.
- Autofac返回完整构建的 ``TodayWriter`` 给"WriteDate"使用.

之后，如果你希望你的应用输出一个不同的日期, 你可以实现另外一个 ``IDateWriter`` 然后在应用启动时改变一下注册过程. 你不需要修改任何其他的类. 耶, 这就是控制反转!

**注意: 通常来说, 服务定位模式大多情况应被看作是一种反模式** `(阅读文章) <http://blog.ploeh.dk/2010/02/03/ServiceLocatorIsAnAntiPattern.aspx>`_. 也就是说, 在代码中四处人为地创建生命周期而少量地使用容器并不是最佳的方式. 使用 :doc:`Autofac 集成类库 <../integration/index>` 时你通常不必做在示例应用中的这些事. 这些东西都会在应用的中心,"顶层"的位置得到解决, 人为的处理是极少存在的. 当然, 如何构建你的应用取决于你自身.

更进一步
=============

这个例子告诉你怎么使用Autofac，但依然有很多你可以做的.

- 查看 :doc:`集成类库 <../integration/index>` 列表, 看看如何将Autofac集成进你的应用.
- 学习 :doc:`注册组件的方法 <../register/index>` 来提高灵活性.
- 学习 :doc:`Autofac配置选项 <../configuration/index>` 使你更好地管理的组件的注册.

需要帮助?
==========

- 你可以 `在StackOverflow上提问 <http://stackoverflow.com/questions/tagged/autofac>`_.
- 你可以 `参与 Autofac Google Group <https://groups.google.com/forum/#forum/autofac>`_.
- 这里有一篇基础 `Autofac 教程 <http://www.codeproject.com/KB/architecture/di-with-autofac.aspx>`_ on CodeProject.
- 如果你想深入, 我们有 :doc:`高级调试tips <../advanced/debugging>`.

源代码Build
====================

支持Visual Studio的项目源代码 `托管在GitHub <https://github.com/autofac/Autofac>`_. Build说明和贡献源码细节可以查看 :doc:`贡献者向导 <../contributors>`.

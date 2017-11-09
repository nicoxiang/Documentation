====================
被囚禁依赖
====================

当一个想要存在 *短* 时间的组件被另一个存在 *长* 时间的组件持有过长时间时, "被囚禁依赖" 就发生了. `这篇 Mark Seemann 的博客 <http://blog.ploeh.dk/2014/06/02/captive-dependency/>`_ 很好地阐述了这个概念.

**Autofac不一定会阻止你创建被囚禁依赖.** 你有时候会发现你会因为囚禁的发生而得到一个解析异常, 但不会总是这样. 阻止被囚禁依赖是开发者的责任.

一般的准则
============

避免被囚禁依赖一般的准则:

**消费组件的生命周期应该小于或等于服务被消费的生命周期.**

基本地来说, 不能允许传入一个每个请求一个实例类型的依赖到一个单例中因为它将被持有太长时间了.

简单示例
==============

假设你有一个 web 应用, 使用一些传入请求中的信息来决定应该连接到哪个正确的数据库. 你应该会有以下组件:

- 一个传入当前请求和一个数据库连接工厂的 *仓库(repository)* .
- 类似 ``HttpContext`` 的 *current request* ,可以被用于决定业务逻辑.
- *数据库连接工厂* , 接受一系列参数并返回正确的数据库连接.

在这个示例中, 考虑下每个组件的 :doc:`生命周期作用域 <instance-scope>` . 明显是当前的请求上下文 - 你会选 *每个请求一个实例*. 那其他的呢?

对于 *仓库* 来说, 假设你选择 "单例" . 一个单例只创建一次并会缓存在应用的整个生存期内. 如果你选择 "单例" , 请求上下文会被传入并被在应用的整个生存期内一直被持有着 - 即使当前请求已经结束, 旧的请求依然会被持有. 仓库是长期的, 但一直持有着一个更小生存期的组件. **这就是被囚禁依赖.**

然而, 假设你让仓库成为 "每个请求一个实例" - 那么现在它将和当前请求存在一样长的时间并且不会超过. 和它需要的请求上下文一样, 因此现在不是被囚禁的. 仓库和请求上下文将会在同一时间 (请求结尾) 被释放, 万事大吉.

再进一步地, 假设你让仓库成为 "每个依赖一个实例" , 你每次将会得到一个新的实例. 这样也是OK的因为它想要存在比当前请求 *更短* 的时间. 它不会持有请求过长时间, 所以也不是被囚禁的.

数据库连接工厂的思考过程类似, 但会有些许不同. 因为也许工厂实例化比较耗资源或者需要维护一些内在的状态来保持正常运作. 你应该不会希望它是 "每个请求一个实例" 或 "每个依赖一个实例." 实际上你应该需要它是一个单例.

**短生存期的依赖来持有更长生存期的依赖是没问题的.** 如果你的仓库是 "每个请求一个实例" 或 "每个依赖一个实例" , 这样依然可以. 数据库连接工厂是有意存在更长时间的.

代码示例
============

这里有个单元测试展示了强制创建一个被囚禁依赖是什么样的. 示例中, "rule manager" 用于处理一系列被用在应用中的 "rules" .

.. sourcecode:: csharp

        public class RuleManager
        {
          public RuleManager(IEnumerable<IRule> rules)
          {
              this.Rules = rules;
          }

          public IEnumerable<IRule> Rules { get; private set; }
        }

        public interface IRule { }

        public class SingletonRule : IRule { }

        public class InstancePerDependencyRule : IRule { }


        [Fact]
        public void CaptiveDependency()
        {
            var builder = new ContainerBuilder();

            // The rule manager is a single-instance component. It
            // will only ever be instantiated one time and the cached
            // instance will be used thereafter. It will be always be resolved
            // from the root lifetime scope (the container) because
            // it needs to be shared.
            builder.RegisterType<RuleManager>()
                   .SingleInstance();

            // This rule is registered instance-per-dependency. A new
            // instance will be created every time it's requested.
            builder.RegisterType<InstancePerDependencyRule>()
                   .As<IRule>();

            // This rule is registered as a singleton. Like the rule manager
            // it will only ever be resolved one time and will be resolved
            // from the root lifetime scope.
            builder.RegisterType<SingletonRule>()
                   .As<IRule>()
                   .SingleInstance();

            using (var container = builder.Build())
            using (var scope = container.BeginLifetimeScope("request"))
            {
              // The manager will be a singleton. It will contain
              // a reference to the singleton SingletonRule, which is
              // fine. However, it will also hold onto an InstancePerDependencyRule
              // which may not be OK. The InstancePerDependencyRule that it
              // holds will live for the lifetime of the container inside the
              // RuleManager and will last until the container is disposed.
              var manager = scope.Resolve<RuleManager>();
            }
        }

注意上面的示例并没有直接地展示, 但是如果你想要在调用 ``container.BeginLifetimeScope()`` 时动态地为这些rules添加注册, 这些动态的注册 *将不会被包含* 在被解析的 ``RuleManager`` 中. ``RuleManager``, 作为一个单例, 是从根容器中被解析的, 这时动态添加的注册还不存在.

下面的另一个示例展示了, 当创建一个错误绑定到一个子生命周期作用域的被囚禁依赖时, 你将得到一个异常.

.. sourcecode:: csharp

        public class RuleManager
        {
          public RuleManager(IEnumerable<IRule> rules)
          {
              this.Rules = rules;
          }

          public IEnumerable<IRule> Rules { get; private set; }
        }

        public interface IRule { }

        public class SingletonRule : IRule
        {
          public SingletonRule(InstancePerRequestDependency dep) { }
        }

        public class InstancePerRequestDependency { }


        [Fact]
        public void CaptiveDependency()
        {
            var builder = new ContainerBuilder();

            // Again, the rule manager is a single-instance component,
            // resolved from the root lifetime and cached thereafter.
            builder.RegisterType<RuleManager>()
                   .SingleInstance();

            // This rule is registered as a singleton. Like the rule manager
            // it will only ever be resolved one time and will be resolved
            // from the root lifetime scope.
            builder.RegisterType<SingletonRule>()
                   .As<IRule>()
                   .SingleInstance();

            // This rule is registered on a per-request basis. It only exists
            // during the request.
            builder.RegisterType<InstancePerRequestDependency>()
                   .As<IRule>()
                   .InstancePerMatchingLifetimeScope("request");

            using (var container = builder.Build())
            using (var scope = container.BeginLifetimeScope("request"))
            {
              // PROBLEM: When the SingletonRule is resolved as part of the dependency
              // chain for the rule manager, the InstancePerRequestDependency in
              // the rule constructor will fail to be resolved because the rule
              // is coming from the root lifetime scope but the InstancePerRequestDependency
              // doesn't exist there.
              Assert.Throws<DependencyResolutionException>(() => scope.Resolve<RuleManager>());
            }
        }


例外
=====================

应用的开发者有责任决定是否被囚禁依赖是否是可以接受的, 开发者也许会决定例如, 让单例去接受传入一个 "每个依赖一个实例" 服务, 是能够接受的.

例如, 也许你有一个缓存类, 用于创建后有意地只缓存消费组件的这段生命周期内的东西. 如果消费者是单例的, 缓存能被用于在应用整个生命周期内存储数据; 如果消费者是 "每个请求一个实例" 那么它只在单个web请求内存储数据. 在这样的示例中, 你也许会 *有意地* 让一个长生存期的组件传入依赖到一个更短生存期的组件中.

这是可以接受的, 只要应用的开发者理解用这样的生命周期创建对象的后果. 也就是说, 如果你打算这么做, 要是有意的而不是无意的.
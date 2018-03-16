==========================
log4net集成模块
==========================

尽管没有特定的程序集支持log4net, 但你可以使用一个很小的自定义模块来轻松注入 ``log4net.ILog`` .

该模块也是一个使用 :doc:`Autofac模块 <../configuration/modules>` 很好的示例, 它不仅仅只是简单的配置 - 它们对于做一些高级的扩展很有帮助.

下面就是这么个示例模块, 它配置了Autofac让它在组件被激活时, 将上面的 ``ILog`` 参数进行注入. 这个示例模块同时处理了构造方法注入和属性注入.

.. sourcecode:: csharp

    public class LoggingModule : Autofac.Module
    {
      private static void InjectLoggerProperties(object instance)
      {
        var instanceType = instance.GetType();

        // Get all the injectable properties to set.
        // If you wanted to ensure the properties were only UNSET properties,
        // here's where you'd do it.
        var properties = instanceType
          .GetProperties(BindingFlags.Public | BindingFlags.Instance)
          .Where(p => p.PropertyType == typeof(ILog) && p.CanWrite && p.GetIndexParameters().Length == 0);

        // Set the properties located.
        foreach (var propToSet in properties)
        {
          propToSet.SetValue(instance, LogManager.GetLogger(instanceType), null);
        }
      }

      private static void OnComponentPreparing(object sender, PreparingEventArgs e)
      {
        e.Parameters = e.Parameters.Union(
          new[]
          {
            new ResolvedParameter( 
                (p, i) => p.ParameterType == typeof(ILog), 
                (p, i) => LogManager.GetLogger(p.Member.DeclaringType)
            ),
          });
      }

      protected override void AttachToComponentRegistration(IComponentRegistry componentRegistry, IComponentRegistration registration)
      {
        // Handle constructor parameters.
        registration.Preparing += OnComponentPreparing;

        // Handle properties.
        registration.Activated += (sender, e) => InjectLoggerProperties(e.Instance);
      }
    }

**性能提醒**: 在写这篇文章的时候, 我们发现调用 ``LogManager.GetLogger(type)`` 会有一些轻微的性能下降因为内部的log manager锁住了loggers集合, 使它没法直接拿到合适的logger. 可以对该模块进行一些改性, 添加对logger实例的缓存, 这样你就能重复使用它们而不会在调用 ``LogManager`` 时有锁冲突了.

感谢Rich Tebb/Bailey Ling的原始想法和所做的贡献, 发布于 `Autofac newsgroup <https://groups.google.com/forum/#!msg/autofac/Qb-dVPMbna0/s-jLeWeST3AJ>`_.

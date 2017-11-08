============================
开始使用生命周期作用域
============================

创建一个新的生命周期作用域
=============================

你可以通过在一个已存在的生命周期上调用 ``BeginLifetimeScope()`` 方法来创建另一个生命周期作用域, 以根容器作为起始. **生命周期作用域是可释放的并且追踪组件的释放, 因此确保你总是调用了 "Dispose()"" 或者把它们包裹在 "using" 语句内.**

.. sourcecode:: csharp

  using(var scope = container.BeginLifetimeScope())
  {
    // Resolve services from a scope that is a child
    // of the root container.
    var service = scope.Resolve<IService>();

    // You can also create nested scopes...
    using(var unitOfWorkScope = scope.BeginLifetimeScope())
    {
      var anotherService = unitOfWorkScope.Resolve<IOther>();
    }
  }

给生命周期作用域打标签
========================

有时候你想要在工作单元之间共享服务但是你并不想这些服务像单例那样全局共享. web应用中的 "per-request" 生命周期就是一个普遍的例子. (:doc:`你可以在 "实例作用域" 章节阅读更多关于每个请求作用域的内容. <instance-scope>`) 在这种情况下, 你会想要给你的生命周期打上标签并且以 ``InstancePerMatchingLifetimeScope()`` 注册服务.

例如, 假设你有一个发送电子邮件的组件. 你系统中的一个业务逻辑层也许会需要发送不止一封邮件, 因此你可以在各个独立的业务逻辑块共享组件. 然而, 你并不想要发邮件组件是一个全局单例. 你可以像下面这样做:

.. sourcecode:: csharp

  // Register your transaction-level shared component
  // as InstancePerMatchingLifetimeScope and give it
  // a "known tag" that you'll use when starting new
  // transactions.
  var builder = new ContainerBuilder();
  builder.RegisterType<EmailSender>()
         .As<IEmailSender>()
         .InstancePerMatchingLifetimeScope("transaction");

  // Both the order processor and the receipt manager
  // need to send email notifications.
  builder.RegisterType<OrderProcessor>()
         .As<IOrderProcessor>();
  builder.RegisterType<ReceiptManager>()
         .As<IReceiptManager>();

  var container = builder.Build();


  // Create transaction scopes with a tag.
  using(var transactionScope = container.BeginLifetimeScope("transaction"))
  {
    using(var orderScope = transactionScope.BeginLifetimeScope())
    {
      // This would resolve an IEmailSender to use, but the
      // IEmailSender would "live" in the parent transaction
      // scope and be shared across any children of the
      // transaction scope because of that tag.
      var op = orderScope.Resolve<IOrderProcessor>();
      op.ProcessOrder();
    }

    using(var receiptScope = transactionScope.BeginLifetimeScope())
    {
      // This would also resolve an IEmailSender to use, but it
      // would find the existing IEmailSender in the parent
      // scope and use that. It'd be the same instance used
      // by the order processor.
      var rm = receiptScope.Resolve<IReceiptManager>();
      rm.SendReceipt();
    }
  }

同样地, :doc:`你可以在 "实例作用域" 章节阅读更多关于带标签的作用域和每个请求作用域的内容. <instance-scope>`

向生命周期作用域内添加注册
========================================

Autofac允许你在创建生命周期作用域的同时随手添加. 如果你做一些 "焊接式的" 有限制的注册重写或者如果你只是在作用域内需要一些不想全局注册的额外的东西, 它将非常有用. 你可以通过给 ``BeginLifetimeScope()`` 传递一个引用 ``ContainerBuilder`` 的lambda表达式并在其中添加注册来完成.

.. sourcecode:: csharp

  using(var scope = container.BeginLifetimeScope(
    builder =>
    {
      builder.RegisterType<Override>().As<IService>();
      builder.RegisterModule<MyModule>();
    }))
  {
    // The additional registrations will be available
    // only in this lifetime scope.
  }

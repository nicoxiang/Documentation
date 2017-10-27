=============================
属性和方法注入
=============================

尽管构造方法参数注入是一种传值给组件的首选的方法, 但你同样可以使用属性或方法注入来传值.

**属性注入** 使用可写属性而不是构造方法参数来完成注入. **方法注入** 通过调用方法来设置依赖.

属性注入
==================

如果组件是一个 :ref:`lambda表达式组件 <register-registration-lambda-expression-components>`, 使用对象构造器:

.. sourcecode:: csharp

    builder.Register(c => new A { B = c.Resolve<B>() });

为了支持 :doc:`循环依赖 <../advanced/circular-dependencies>`, 可以使用 :doc:`激活后事件处理程序(activated event handler) <../lifetime/events>`:

.. sourcecode:: csharp

    builder.Register(c => new A()).OnActivated(e => e.Instance.B = e.Context.Resolve<B>());

如果组件是一个 :ref:`反射组件 <register-registration-reflection-components>`, 使用 ``PropertiesAutowired()`` 修饰语来注入属性:

.. sourcecode:: csharp

    builder.RegisterType<A>().PropertiesAutowired();

如果你需要绑定一个特定的属性和它的值, 使用 ``WithProperty()`` 修饰语:

.. sourcecode:: csharp

    builder.RegisterType<A>().WithProperty("PropertyName", propertyValue);

方法注入
================

想要调用一个方法来设置组件上的某个值, 最简单的方法是使用 :ref:`lambda表达式组件 <register-registration-lambda-expression-components>` 然后在activator中进行正确的方法调用:

.. sourcecode:: csharp

    builder.Register(c => {
      var result = new MyObjectType();
      var dep = c.Resolve<TheDependency>();
      result.SetTheDependency(dep);
      return result;
    });

如果你没法使用注册lambda表达式, 你可以添加一个 :doc:`激活时事件处理程序(activating event handler) <../lifetime/events>`:

.. sourcecode:: csharp

    builder
      .Register<MyObjectType>()
      .OnActivating(e => {
        var dep = e.Context.Resolve<TheDependency>();
        e.Instance.SetTheDependency(dep);
      });
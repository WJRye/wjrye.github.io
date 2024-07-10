---
layout: post
title: Java 动态代理 Proxy
categories: [Java]
description: 在 Java  中的动态代理，实际上指的就是反射包 java.lang.reflect 下的类 Proxy。
keywords: Java, Proxy
mermaid: false
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false
---

## ## 动态代理

### 类Proxy

在 Java  中的动态代理，实际上指的就是反射包`java.lang.reflect`下的 类 `Proxy`。

> Proxy 提供用于创建动态代理类和实例的静态方法，它还是由这些方法创建的所有动态代理类的超类。
> 动态代理类（以下简称为代理类）是一个实现在创建类时在运行时指定的接口列表的类，该类具有下面描述的行为。 代理接口 是代理类实现的一个接口。 代理实例 是代理类的一个实例。 每个代理实例都有一个关联的调用处理程序 对象，它可以实现接口 InvocationHandler。通过其中一个代理接口的代理实例上的方法调用将被指派到实例的调用处理程序的 Invoke 方法，并传递代理实例、识别调用方法的 java.lang.reflect.Method 对象以及包含参数的 Object 类型的数组。调用处理程序以适当的方式处理编码的方法调用，并且它返回的结果将作为代理实例上方法调用的结果返回。

使用动态代理，主要是使用类 `Proxy` 的一个静态方法 `newProxyInstance`：

```
    public static Object newProxyInstance(ClassLoader loader,
                                          Class<?>[] interfaces,
                                          InvocationHandler h)
        throws IllegalArgumentException {

        /...
   }
```

该方法返回一个指定接口的代理类实例，该接口可以将方法调用指派到指定的调用处理程序（InvocationHandler）。

该方法有三个参数：

- loader：定义代理类的类加载器
- interfaces：代理类要实现的接口列表
- h：指派方法调用的调用处理程序

关于参数 h，它是一个`InvocationHandler`接口，表示代理实例的调用处理程序 实现的接口。每个代理实例都具有一个关联的调用处理程序。对代理实例调用方法时，将对方法调用进行编码并将其指派到它的调用处理程序的 `invoke` 方法。

方法返回值表示：一个带有代理类的指定调用处理程序的代理实例，它由指定的类加载器定义，并实现指定的接口。

### 例子分析

定义一个接口 UserService：

```
//服务用户的接口
public interface UserService {

    /**
     * 获取用户列表
     * 
     * @return 用户列表
     */
    List<User> getUsers();
}
```

定义一个数据模型类 User：

```
public class User {

    public int age;
    public String name;

    public User(int age, String name) {
        this.age = age;
        this.name = name;
    }

}
```

测试`newProxyInstance`方法：

```
public class TestProxy {

    public static void main(String[] args) {
        Class<?> clazz = UserService.class;
        Object object = Proxy.newProxyInstance(clazz.getClassLoader(), new Class[] { clazz }, new InvocationHandler() {

            @Override
            public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                return null;
            }

        });
        System.out.println(object.getClass().getName());
    }
}
输出结果：
com.sun.proxy.$Proxy0
```

从方法返回值，感觉比较奇怪。返回值是一个`com.sun.proxy.$Proxy0`类的实例对象。要想知道`$Proxy0`类是怎么来的，先看下`newProxyInstance`的方法实体：

```
   //代理类构造函数的参数类型
   private static final Class<?>[] constructorParams =
        { InvocationHandler.class };

   public static Object newProxyInstance(ClassLoader loader,
                                          Class<?>[] interfaces,
                                          InvocationHandler h)
        throws IllegalArgumentException
    {
         //判空检查
        Objects.requireNonNull(h);
        //克隆传递进来的接口类
        final Class<?>[] intfs = interfaces.clone();


        //...省略

        /*
         * 1.查找或生成代理类
         */
        Class<?> cl = getProxyClass0(loader, intfs);

        /*
         * Invoke its constructor with the designated invocation handler.
         */
        try {
            //...省略

             //2.通过反射生成构造器对象
            final Constructor<?> cons = cl.getConstructor(constructorParams);
            final InvocationHandler ih = h;
            if (!Modifier.isPublic(cl.getModifiers())) {
                // BEGIN Android-changed: Excluded AccessController.doPrivileged call.
                /*
                AccessController.doPrivileged(new PrivilegedAction<Void>() {
                    public Void run() {
                        cons.setAccessible(true);
                        return null;
                    }
                });
                */

                cons.setAccessible(true);
                // END Android-removed: Excluded AccessController.doPrivileged call.
            }

            //3.通过反射创建出对象，参数是传递进来的InvocationHandler对象h
            return cons.newInstance(new Object[]{h});
        } catch (IllegalAccessException|InstantiationException e) {
            throw new InternalError(e.toString(), e);
        } catch (InvocationTargetException e) {
            Throwable t = e.getCause();
            if (t instanceof RuntimeException) {
                throw (RuntimeException) t;
            } else {
                throw new InternalError(t.toString(), t);
            }
        } catch (NoSuchMethodException e) {
            throw new InternalError(e.toString(), e);
        }
    }
```

上面的步骤大概可以概括为：

1. 生成代理类Proxy0 (指的就是cl)，Proxy0有一个构造函数，构造函数的参数是 InvocationHandler 类型。
2. 通过反射，生成类Proxy0的一个构造器对象。
3. 再通过反射，生成类Proxy0的一个实例对象。

关于生成代理类Proxy0，再看下方法`getProxyClass0(loader, intfs)` :

```
    /**
     * Generate a proxy class.  Must call the checkProxyAccess method
     * to perform permission checks before calling this.
     */
    private static Class<?> getProxyClass0(ClassLoader loader,
                                           Class<?>... interfaces) {
        if (interfaces.length > 65535) {
            throw new IllegalArgumentException("interface limit exceeded");
        }

        // If the proxy class defined by the given loader implementing
        // the given interfaces exists, this will simply return the cached copy;
        // otherwise, it will create the proxy class via the ProxyClassFactory
        return proxyClassCache.get(loader, interfaces);
    }
```

如果`proxyClassCache`对象缓存的有代理类，就直接返回，否则就通过类`ProxyClassFactory`生成一个代理类。

类`ProxyClassFactory`是 类 `Proxy` 中的一个静态内部类。它主要负责代理类Proxy0的生成。这里不讲代理类Proxy0的生成细节了，但可以看下代理类Proxy0到底是一个怎样的类。

类Proxy0：

```
package com.sun.proxy;

import java.util.List;

import com.wangjiang.example.User;
import com.wangjiang.example.UserService;

public final class $Proxy0 extends Proxy implements UserService {
    private static Method m1;
    private static Method m3;
    private static Method m0;
    private static Method m2;

    public $Proxy0(InvocationHandler paramInvocationHandler) {
        super(paramInvocationHandler);
    }

    public final boolean equals(Object paramObject) {
        try {
            return ((Boolean) this.h.invoke(this, m1, new Object[] { paramObject })).booleanValue();
        } catch (Error | RuntimeException localError) {
            throw localError;
        } catch (Throwable localThrowable) {
            throw new UndeclaredThrowableException(localThrowable);
        }
    }

    public final List<User> getUsers() {
        try {
            return this.h.invoke(this, m3,(Object[])null);
        } catch (Error | RuntimeException localError) {
            throw localError;
        } catch (Throwable localThrowable) {
            throw new UndeclaredThrowableException(localThrowable);
        }
    }

    public final int hashCode() {
        try {
            return ((Integer) this.h.invoke(this, m0, null)).intValue();
        } catch (Error | RuntimeException localError) {
            throw localError;
        } catch (Throwable localThrowable) {
            throw new UndeclaredThrowableException(localThrowable);
        }
    }

    public final String toString() {
        try {
            return (String) this.h.invoke(this, m2, null);
        } catch (Error | RuntimeException localError) {
            throw localError;
        } catch (Throwable localThrowable) {
            throw new UndeclaredThrowableException(localThrowable);
        }
    }

    static
  {
    try
    {
      m1 = Class.forName("java.lang.Object").getMethod("equals", new Class[] { Class.forName("java.lang.Object") });
      m3 = Class.forName("com.wangjiang.example.UserService").getMethod("getUsers");
      m0 = Class.forName("java.lang.Object").getMethod("hashCode", new Class[0]);
      m2 = Class.forName("java.lang.Object").getMethod("toString", new Class[0]);
    }
    catch (NoSuchMethodException localNoSuchMethodException)
    {
      throw new NoSuchMethodError(localNoSuchMethodException.getMessage());
    }
    catch (ClassNotFoundException localClassNotFoundException)
    {
      throw new NoClassDefFoundError(localClassNotFoundException.getMessage());
    }
  }
```

可以看到类 Proxy0 继承了类 Proxy，实现了接口 UserService。它的构造方法参数是一个`InvocationHandler` 对象，这个对象就是 用类Proxy 中的`newProxyInstance`方法传递的InvocationHandler 对象给它赋值的。在执行UserService接口方法的时候，会通过这个InvocationHandler 对象回调它的`invoke`方法，通过`invoke`方法会将接口方法对象和接口方法参数传递过去，这样相当于有方法拦截的作用。

再来看具体例子：

```
public class TestProxy {

    public static void main(String[] args) {
        System.getProperties().put("sun.misc.ProxyGenerator.saveGeneratedFiles", "true");

        Class<?> clazz = UserService.class;
        UserService userService = (UserService) Proxy.newProxyInstance(clazz.getClassLoader(), new Class[] { clazz },
                new InvocationHandler() {

                    @Override
                    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                        System.out.println("invoke-->method.getName() = " + method.getName());

                        List<User> users = new ArrayList<User>();
                        users.add(new User(22, "wangjiang"));
                        return users;
                    }

                });
        List<User> users = userService.getUsers();
        for (User user : users)
            System.out.println("name: " + user.name + " ;age: " + user.age);

    }

}
输出结果：
invoke-->method.getName() = getUsers
name: wangjiang ;age: 22
```

看完这个例子，我相信可以明白动态代理的运行原理了。

---

## 总结

通过 类`Proxy` 的 `newProxyInstance()` 方法，可以对 任何接口进行动态代理，也就是在不需要实现具体接口的情况下，交给类`Proxy` 来处理接口。

在**Retrofit**框架中，在创建服务接口的时候，就是利用了动态代理：

```
 public <T> T create(final Class<T> service) {
    Utils.validateServiceInterface(service);
    if (validateEagerly) {
      eagerlyValidateMethods(service);
    }
    return (T) Proxy.newProxyInstance(service.getClassLoader(), new Class<?>[] { service },
        new InvocationHandler() {
          private final Platform platform = Platform.get();
          private final Object[] emptyArgs = new Object[0];

          @Override public @Nullable Object invoke(Object proxy, Method method,
              @Nullable Object[] args) throws Throwable {
            // If the method is a method from Object then defer to normal invocation.
            if (method.getDeclaringClass() == Object.class) {
              return method.invoke(this, args);
            }
            if (platform.isDefaultMethod(method)) {
              return platform.invokeDefaultMethod(method, service, proxy, args);
            }
            return loadServiceMethod(method).invoke(args != null ? args : emptyArgs);
          }
        });
  }
```

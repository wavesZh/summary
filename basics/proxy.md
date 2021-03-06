    # proxy


## 代理

保持接口不变的情况下，改变接口定义的方法的行为。比如aop。

其实可以通过继承的方式覆写方法和组合实现代理的效果。
[inheritance vs aggregation](https://stackoverflow.com/questions/269496/inheritance-vs-aggregation)

1. 静态代理

hard code使用代理模式实现：

```java
public interface HelloWorld {
    void sayHelloWorld();
}
public class HelloWorldImpl implements HelloWorld {
    void sayHelloWorld() {
        // say hello world
    }
}
public class HelloWorldProxy implements HelloWorld {

    private HelloWorld real ;

    public HelloWorldProxy(HelloWorld real) {
        this.real = real;
    }

    void sayHelloWorld() {
        // add other code
        real.sayHelloWorld();
        // add other code 
    }
}
```

突然感觉静态代理模式和装饰者模式十分相似。其主要区别在于目的： 装饰者模式侧重动态增强功能，代理模式侧重控制对象。

2. 动态代理

[jdk动态代理为什么传接口，内部实现接口，而不直接传类，直接继承类？](https://www.zhihu.com/question/264948554)

静态代理使用硬编码，在运行前就实现了代理类，那么动态代理即是在运行时生成代理类。

* jdk代理

通过组合实现代理。`只能为接口创建代理实例`。 

通过下面code，将动态生成的proxy class文件输出到项目root的`com/sun/proxy`
`System.setProperty("sun.misc.ProxyGenerator.saveGeneratedFiles", "true")`


~~~java
package com.sun.proxy;

import com.waves.demo.proxy.HelloWorld;
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;
import java.lang.reflect.UndeclaredThrowableException;

public final class $Proxy0 extends Proxy implements HelloWorld {
    // 被代理类的接口方法实现
    private static Method m1;
    private static Method m2;
    private static Method m4;
    private static Method m0;

    // InvocationHandler对于被代理方法进行了扩展。
    public $Proxy0(InvocationHandler var1) throws  {
        super(var1);
    }

    public final boolean equals(Object var1) throws  {
        try {
            return (Boolean)super.h.invoke(this, m1, new Object[]{var1});
        } catch (RuntimeException | Error var3) {
            throw var3;
        } catch (Throwable var4) {
            throw new UndeclaredThrowableException(var4);
        }
    }

    public final String toString() throws  {
        try {
            return (String)super.h.invoke(this, m2, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    public final void sayHelloWorld() throws  {
        try {
            super.h.invoke(this, m4, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }


    public final int hashCode() throws  {
        try {
            return (Integer)super.h.invoke(this, m0, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    static {
        try {
            m1 = Class.forName("java.lang.Object").getMethod("equals", Class.forName("java.lang.Object"));
            m2 = Class.forName("java.lang.Object").getMethod("toString");
            m4 = Class.forName("com.waves.mongodbdemo.proxy.HelloWorld").getMethod("sayHelloWorld");
            m0 = Class.forName("java.lang.Object").getMethod("hashCode");
        } catch (NoSuchMethodException var2) {
            throw new NoSuchMethodError(var2.getMessage());
        } catch (ClassNotFoundException var3) {
            throw new NoClassDefFoundError(var3.getMessage());
        }
    }
}

~~~

由上可见被代理的方法都是`public`,是接口`HelloWorld`的方法和`equals`、`toString`方法。


* cglib代理

通过继承实现代理。那么`不能被继承的方法就不能被代理`，如final,private。



## spring aop

### 配置项

* `proxyTargetClass`: 是否cglib优先。
* `optimize`: 是否对生成代理策略进行优化(看源码感觉没什么用)。true :  进行优化，如果有接口就代理接口(使用JDK动态代理)，没有接口代理类（CGLIB代理）
* `opaque`: 是否可以cast成`Advised`，`Advised`可以查询和修改配置信息呀。
`exposeProxy`: 暴露proxy。比较常用。当调用内部方法如父类方法，想要触发aop，则需要暴露proxy。
`frozen`: 是否冻结`Advised`配置信息。

~~~java
public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {
    if (config.isOptimize() || config.isProxyTargetClass() || hasNoUserSuppliedProxyInterfaces(config)) {
        Class<?> targetClass = config.getTargetClass();
        if (targetClass == null) {
            throw new AopConfigException("TargetSource cannot determine target class: " +
                    "Either an interface or a target is required for proxy creation.");
        }
        if (targetClass.isInterface() || Proxy.isProxyClass(targetClass)) {
            return new JdkDynamicAopProxy(config);
        }
        return new ObjenesisCglibAopProxy(config);
    }
    else {
        return new JdkDynamicAopProxy(config);
    }
}
~~~



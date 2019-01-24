# 类加载器

类加载器主要作用是通过类的全限定名获取取二进制字节流。

类加载过程：加载，验证，准备，解析，初始化。类加载器主要作用在加载阶段。


## jdk原生类加载器

### 1. 启动类加载器

别名：Bootstrap ClassLoader

加载路径： <JAVA_HOME>\lib or -Xbootclasspathk

### 2. 扩展类加载器

别名：Extension ClassLoader

加载路径：<JAVA_HOME>\lib\ext or java.ext.dir

### 3. 应用程序类加载器

别名： Application ClassLoader, 系统类加载器

加载路径：用户路径 user.dir


## 自定义类加载器


[ClassLoader.getSystemClassLoader](https://docs.oracle.com/javase/6/docs/api/java/lang/ClassLoader.html#getSystemClassLoader%28%29)
>If the system property "java.system.class.loader" is defined when this method is first invoked then the value of that property is taken to be the name of a class that will be returned as the system class loader. The class is loaded using the default system class loader and must define a public constructor that takes a single parameter of type ClassLoader which is used as the delegation parent.

1. 如果使用java.system.class.loader定义系统类加载器，自定义类加载器必须带有一个类型为classLoader参数的构造方法。

2. 自定义类加载器是由原生的系统类加载器加载的。


## 双亲委派模型

双亲：启动类加载器 + 父类加载器。父子关系一般是组合的形式。

每个类加载器都有有双亲，优先将类加载请求委派给父类加载器加载，递归，因此所有的请求都会到达启动类加载器。只有当父类加载器无法完成该类的加载，子类加载器才会尝试加载该类。

该模型保证了一个类只会被同一个类加载加载。父类加载器拿不到子类加载器加载的类的。

## 破坏双亲委派

1. [spi](spi.md)

rt.jar中有很多开放的api接口供外部实现，如java.sql.Driver，由启动类加载器加载，但是启动类加载器不知道spi的逻辑的，故无法找到该api服务提供者。




~~~java
package java.sql;
public interface Driver {
	.... ...
}
~~~

~~~java
package java.sql;
public class DriverManager {
	static {
        loadInitialDrivers();
        println("JDBC DriverManager initialized");
    }

    private static void loadInitialDrivers() {
		... ...
		ServiceLoader<Driver> loadedDrivers = ServiceLoader.load(Driver.class);
	    Iterator<Driver> driversIterator = loadedDrivers.iterator();
	    ... ... 
	    while(driversIterator.hasNext()) {
	        driversIterator.next();
	    }
	}
}

public class ServiceLoader {


	private class LazyIterator
        implements Iterator<S> {

 		private boolean hasNextService() {
 			... ...
 			String fullName = PREFIX + service.getName();
 			configs = loader.getResources(fullName);
 			... ...
 		}

 		private S nextService() {
 			... ...
			c = Class.forName(cn, false, loader);
			// 初始化，
			S p = service.cast(c.newInstance());
 			... ...

 		}

    }
}

package com.mysql.jdbc;
public class Driver extends NonRegisteringDriver implements java.sql.Driver {
    static {
        DriverManager.registerDriver(new Driver());
    }
}
~~~

`DriverMananger`被`BootStrap ClassLoader`加载时，先执行`static`块。`ServiceLoader`主要作用是加载api服务提供者。启动类加载器将请求委派给线程上下文类加载器(默认是应用程序类加载器)。
`hasNextService`:，显式调用从`META-INF/services/`获取到相应的class资源URL。 
`nextService`:根据全限定名获取class并初始化，com.mysql.jdbc.Driver 的静态方法块执行，注册驱动。


为什么jdbc破坏了这个原则呢？  
`DriverManager`由`BootStrap ClassLoader`加载，但是却拿到了由子类加载器`Application ClassLoader`加载的`com.mysql.jdbc.Driver`。类A引用了多个其他类B，B的类加载器应该是A的类加载器或者其父类加载器，不应该是其子类加载器。


2. 热部署

## link

[不遵循双亲委派的类加载器 -- 线程上下文类加载器](http://techlog.cn/article/list/10183177)
[SPI的ClassLoader问题](https://www.jianshu.com/p/bfa4b171096a)
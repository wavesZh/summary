# THINK IN CODE

## link

[各种 Java Thread State 第一分析法则](https://www.cnblogs.com/zhengyun_ustc/archive/2013/03/18/tda.html)


## 1. 将文本中的数据导入redis

java: Files.lines(Paths) -> 数据加工 -> fork-join -> redis pipeline


## 2. 单例模式-静态内部类的原理

~~~java
 public static class SingleTonHolder {
        public static SingleTon INSTANCE = new SingleTon();
        // static {
        //     System.out.println("holder");
        // }

    }

    private SingleTon() {
        // System.out.println("init SingleTon");
    }

    // static {
    //     System.out.println("in");
    // }

    public static SingleTon getInstance() {
        return SingleTonHolder.INSTANCE;
    }
~~~

选择静态内部类实现单例的优势： 
1. 懒加载
2. 线程安全

<<深入了解java虚拟机>>：
> 一个类何时初始化即类加载？
>1. 遇到new、getstatic、putstatic或invokestatic、这4条字节码指令时，如果类没有被初始化，则需要先触发其初始化。  
（1）new：使用new关键字实例化对象的时候。   
（2）getstatic或putstatic：读取或设置一个类的静态字段（不包括被final关键字修饰的变量，被final修饰、已在编译期把结果放入常量池的静态字段字段除外）。  
（3）invokestatic：调用一个类的静态方法。  
>2. 使用java.lang.reflect包的方法对类进行进行反射调用时。
>3. 子类初始化时。（如果一个类的在初始化时父类还没有初始化，则先初始化父类）
>4. 虚拟机启动时，用户需要指定一个要执行的主类（包含main()方法的那个类），虚拟机会先初始化这个主类。
>5. 当使用JDK1.7的动态语言支持时，如果一个java.lang.invoke.MethodHandler实例最后的解析结果REF_getStatic、REF_putStatic、REF_invokeStatic的方法句柄时。



再调用`getInstance()`静态方式的时候，使得`SingleTon`初始化，读取了一个类的静态字段`INSTANCE`，使得`SingleTonHolder`初始化，进而创建`INSTANCE`。这样实现了懒加载。至于线程安全的实现，需要先了解下`<clinit>()`方法。

<<深入了解java虚拟机>>：
>虚拟机会保证一个类的<clinit>()方法在多线程环境中被正确地加锁、同步，如果多个线程同时去初始化一个类，那么只会有一个线程去执行这个类的<clinit>()方法，其他线程都需要阻塞等待，直到活动线程执行<clinit>()方法完毕。如果在一个类的<clinit>()方法中有耗时很长的操作，就可能造成多个进程阻塞(需要注意的是，其他线程虽然会被阻塞，但如果执行<clinit>()方法后，其他线程唤醒之后不会再次进入<clinit>()方法。同一个加载器下，一个类型只会初始化一次。)，在实际应用中，这种阻塞往往是很隐蔽的。

另外，`<clinit>()`方法只会在jvm第一次加载class文件时调用，包括静态变量初始化语句和静态块的执行。同一个加载器下，一个类型只会初始化一次。故`INSTANCE`（静态变量）的创建是线程安全的。

劣势：

无法传参初始化。

ps: DCL（double check lock）单例

```java
public class SingleTon{
  private volatile static SingleTon INSTANCE = null;
  private SingleTon(){}
  public static SingleTon getInstance(){if(INSTANCE == null){
   synchronized(SingleTon.class){
     if(INSTANCE == null){ 
        INSTANCE = new SingleTon();
       } 
     } 
        return INSTANCE; 
    } 
  }
}
```

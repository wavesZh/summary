# Problem : is not eligible for getting processed by all BeanPostProcessors

在 Spring 服务中发现了 “xxx is not eligible for getting processed by all BeanPostProcessors” 相关日志，其表示，xxx Bean 将不会被所有的 BeanPostProcessor 处理。这点需要注意，如果 xxx bean 原本需要被 AOP 增强，则会造成 xxx 无法被增强。

造成这种情况是因为 BeanPostProcessor 在 Spring 中也是属于 Bean, 在 BeanPostProcessor 创建时，如果其依赖于 xxx 等普通 Bean，则会提前创建 xxx，导致 xxx 无法被该 BeanPostProcessor 处理。

```java
public static void registerBeanPostProcessors(
        ConfigurableListableBeanFactory beanFactory, AbstractApplicationContext applicationContext) {

    String[] postProcessorNames = beanFactory.getBeanNamesForType(BeanPostProcessor.class, true, false);

    // Register BeanPostProcessorChecker that logs an info message when
    // a bean is created during BeanPostProcessor instantiation, i.e. when
    // a bean is not eligible for getting processed by all BeanPostProcessors.

    // 已经存在的 BeanPostProcessor Bean（Spring自带的） + BeanPostProcessorChecker（自己） + 需要创建的 BeanPostProcessor
    int beanProcessorTargetCount = beanFactory.getBeanPostProcessorCount() + 1 + postProcessorNames.length;
    // 用于检查是否 Bean 是否能被所有 BeanPostProcessor 处理
    beanFactory.addBeanPostProcessor(new BeanPostProcessorChecker(beanFactory, beanProcessorTargetCount));

    // 1. 先创建 PriorityOrdered postProcessor 并 beanFactory.addBeanPostProcessor(postProcessor)
    // 2. 然后创建 Ordered postProcessor 
    // 3. 最后创建 Non Ordered postProcessor
    ... 

    // 
    beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(applicationContext));

}        
```

由上可知：如果 需要创建 BeanPostProcessor 为 PriorityOrdered 且其依赖于 xxx Bean， 则会导致 xxx 无法被部分PriorityOrdered/全部Ordered 以及 Non Ordered postProcessor 处理。
@ConfigurationProperties + apollo + @ApolloConfigChangeListener

如官方所属，如果使用 @ConfigurationProperties 注入配置且需要实现热更新，需要额外代码实现：

```java
@Component
public class PropertiesRefresher implements ApplicationContextAware {
    private static final Logger logger = LoggerFactory.getLogger(PropertiesRefresher.class);
    public static final String CLASS_NAME = "org.springframework.cloud.context.environment.EnvironmentChangeEvent";

    private ApplicationContext applicationContext;


    @ApolloConfigChangeListener(value = {"application","application.yml","kafka.yml"})
    public void onChange(ConfigChangeEvent changeEvent) {
        refreshProperties(changeEvent);
    }

    private void refreshProperties(ConfigChangeEvent changeEvent) {
        /**
         * rebind configuration beans,
         * @see org.springframework.cloud.context.properties.ConfigurationPropertiesRebinder#onApplicationEvent
         */
        Object event = null;
        try {
            Class<?> targetClass = Class.forName(CLASS_NAME);
            Constructor<?> constructor = targetClass.getConstructor(Set.class);
            event = constructor.newInstance(changeEvent.changedKeys());
        } catch (Exception e) {
            logger.error("apollo properties refresh can not create instance for  class: {}", CLASS_NAME);
        }

        if (event != null) {
            this.applicationContext.publishEvent(event);
            logger.info("properties refreshed!");
        }
    }

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        this.applicationContext = applicationContext;
    }
}
```

以上只是我认为为比较简便的方式，另外还有通过 RefreshScope 实现，不过需要知道响应的配置类的 bean 名称等。


集成过程中，这种方式失效，通过跟其他正常服务对比，`ConfigurationPropertiesRebinder#onApplicationEvent(EnvironmentChangeEvent event)` 方法逻辑：

重新进行属性绑定， beans 中不含业务定义的 @ConfigurationProperties 修饰的 bean。

```java
public void rebind() {
    this.errors.clear();
    for (String name : this.beans.getBeanNames()) {
        rebind(name);
    }
}
```

查看 beans 的源来，在 ` ` 这个自动装配类中自动注入的。其中自动状态的还有 `ConfigurationPropertiesBeans`，恰好是 beans 的类型。

```java
public class ConfigurationPropertiesBeans implements BeanPostProcessor, ApplicationContextAware
```
其是一个 `BeanPostProcessor`， 在 `postProcessBeforeInitialization` 方法将 @ConfigurationProperties 修饰的 bean 保存。快真相了。、、

那为什么没有保存了。debug 发现，`ConfigurationPropertiesRebinderAutoConfiguration` 自动装配没有生效，然后联想到项目中的 `spring.autoconfigure.exclude` 配置，其当初是为了去除不必要的 bean，提高服务启动速度。在其中过来包含着 `org.springframework.cloud.autoconfigure.ConfigurationPropertiesRebinderAutoConfiguration`。


在 debug 期间发生了奇怪的事情，刷新了两次 refresh。 应该是跟 sc 的启动原理相关。


开箱即用=自动装配的理解：

在之前，我认为自动装配主要体现在bean自动实例化。其实不然，应该是在bean的配置自动匹配。要得到一个完整一个bean，除了实例化，还要注入相关的配置。自动装配主要体现在配置的自动化。





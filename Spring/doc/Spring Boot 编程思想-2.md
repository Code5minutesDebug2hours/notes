# 摘要

第十三天，写了自定义 starter，介绍了 @Conditional\*Class、@Conditional\*Bean

第十四天，第 400 页，各种条件注解源码讲解。

第十五天，第 422 页，SpringApplication 初始化阶段，以及部分 run 阶段的配置信息。

第十六天，第 438 页，Spring Boot 运行阶段，部分 Spring 事件 以及 Spring Boot 事件。

第十七天，第 457 页，Spring  AbstractApplicationContext 简单事件发布。

第十八天，第 469 页，Spring 事件以及 Listener。

第十九天。

# 第十三天

## 自定义 Spring Boot Starter

> A full Spring Boot Starter for a library may contain the following components:
>
> + The autoconfigure module that contains the auto-configuration code.
> + The starter moudule that provides a dependency to auto-configure moudule as well as the library and any additional dependencies that are typically useful. In a nutshell, adding the starter should provide everything needed to start using that library.

官方建议将自动装配代码放在 autoconfigure 模块中，starter 模块以来该模块，并且附加其他需要的依赖。参考 swagger-ui。

不要用 server、management、spring 等作为配置key 的前缀。这些 namespace 是外部化配置 @ConfigurationProperties 前缀属性 prefix，同时，sping-boot-configuration-processor能够帮助 @ConfigurationProperties Bean 生成 IDE 辅助元信息。

@Configuration 类是自动装配的底层实现，并且搭配 @Conditional 注解，使其能够合理地在不同环境中运作。

## @Conditional*Class

@ConditionalOnClass 类存在

@ConditionalOnMissingClass 类不存在

一般是成对存在的，为了防止某个 Class 不存在导致自动装配失败。

## @Conditional*Bean

@ConditionalOnBean

@ConditionalOnMissingBean

也是成对存在的。

| 属性方法     | 属性类型       | 语义说明                 | 使用场景                               | 起始版本 |
| ------------ | -------------- | ------------------------ | -------------------------------------- | -------- |
| value()      | Class[]        | Bean 类型集合            | 类型安全的属性设置                     | 1.0      |
| type()       | String[]       | Bean 类名集合            | 当类型不存在时的属性设置               | 1.3      |
| annotation() | Class[]        | Bean 声明注解类型集合    | 当 Bean 标注了某种注解类型时           | 1.0      |
| name()       | String[]       | Bean 名称集合            | 指定具体 Bean 名称集合                 | 1.0      |
| search()     | SearchStrategy | 层次性应用上下文搜索策略 | 三种应用上下文搜索策略：当前、父及所有 | 1.0      |

# 第十四天

## @Conditional*Property

属性来源 Spring Environment。Java 系统属性和环境变量时典型的 Spring Environment 属性配置来源（PropertySource）。在 Spring Boot 场景中，application.properties 也是其中来源之一。整合了三个地方的属性。

|     属性方法     |                           使用说明                           | 默认值 | 多值属性 | 起始版本 |
| :--------------: | :----------------------------------------------------------: | :----: | :------: | :------: |
|     prefix()     |                       配置属性名称前缀                       |   “”   |    否    |   1.1    |
|     value()      |                  Name() 的别名，参考 name()                  | 空数组 |    是    |   1.1    |
|      name()      | 如果 prefix() 不为空，则完整配置属性名称为 prefix()+name)，否则为 name()  的内容 | 空数组 |    是    |   1.2    |
|  havingValue()   |           表示期望的配置属性值，并且禁止使用 false           |   “”   |    否    |   1.2    |
| matchIfMissing() |              用于判断当前属性值不存在时是否匹配              | false  |    否    |   1.2    |

解决 Spring Boot Environment 某些变量缺失问题

1. 增加属性设置——`new SpringApplicationBuilder(CurrentClazz.class).properties("formatter.enable=true").run(args);`// “=”前后不能有空格。
2. 调整 @ConditionalOnProperty#matchIfMissing 属性——`matchIfMissing = true` 表示当属性配置不存在时，同样视作匹配。

三个 Environment 来源，会覆盖，优先级依次向下，也就是说，1 会覆盖 2，2 会覆盖 3，3 优先级最低。

1. 环境变量
2. 启动时输入参数
3. properties、yml 等配置

## Resource 条件注解

@ConditionalOnResource

默认用 `classpath://` 做协议前缀，其他协议 Spring framework 没有默认实现。

## Web 应用条件注解

@ConditionalOnWebApplication

@ConditionalOnNotWebApplication

## Spring 表达式条件注解

@ConditionalOnExpression("#{spring.aop.auto:true}")

和 

@ConditionalOnExpression("spring.aop.auto:true")

是一样的，对应的 Condition 类有个 wrapIfNecessary 方法加上这个表达式。

# 第十五天

## 理解 SpringApplication 初始化阶段

构造方法总览

```java
@SuppressWarnings({ "unchecked", "rawtypes" })
public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
   this.resourceLoader = resourceLoader;
   Assert.notNull(primarySources, "PrimarySources must not be null");
   this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
   // <1>
   this.webApplicationType = WebApplicationType.deduceFromClasspath();
   // <2>
   setInitializers((Collection) getSpringFactoriesInstances(ApplicationContextInitializer.class));
   // <3>
   setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
   // <4>
   this.mainApplicationClass = deduceMainApplicationClass();
}
```

### 1. 推断 Web 应用类型

`webApplicationType = deduceFromClasspath.deduceFromClasspath();` 自动加载哪种类型的 webApplication

```java
static WebApplicationType deduceFromClasspath() {
   if (ClassUtils.isPresent(WEBFLUX_INDICATOR_CLASS, null) && !ClassUtils.isPresent(WEBMVC_INDICATOR_CLASS, null)
         && !ClassUtils.isPresent(JERSEY_INDICATOR_CLASS, null)) {
      return WebApplicationType.REACTIVE;
   }
   for (String className : SERVLET_INDICATOR_CLASSES) {
      if (!ClassUtils.isPresent(className, null)) {
         return WebApplicationType.NONE;
      }
   }
   return WebApplicationType.SERVLET;
}
```

总结出：

1. 当 DispatcherHandler 存在时，并且 DispatcherServlet 不存在时，这时为 Reactive 应用，就是仅依赖 WebFlux 时。
2. 当 Servlet 和 ConfigurableWebApplicationContext 均不存在时，当前应用为非 Web 应用，即 WebApplicationType.NONE，因为这些是 Spring Web MVC 必需的依赖。
3. 当 Spring WebFlux 和 Spring Web MVC 同时存在时，还是 Servlet 应用。

## 2. 加载 Spring 应用上下文初始器

ApplicationContextInitializer

 `setInitializers((Collection) getSpringFactoriesInstances(ApplicationContextInitializer.class));`

ApplicationContextInitializer 的实现必须拥有无参构造器。

分两步

1. **SpringApplication#getSpringFactoriesInstances**

```java
private <T> Collection<T> getSpringFactoriesInstances(Class<T> type) {
   return getSpringFactoriesInstances(type, new Class<?>[] {});
}

private <T> Collection<T> getSpringFactoriesInstances(Class<T> type, Class<?>[] parameterTypes, Object... args) {
   ClassLoader classLoader = getClassLoader();
   // Use names and ensure unique to protect against duplicates
   Set<String> names = new LinkedHashSet<>(SpringFactoriesLoader.loadFactoryNames(type, classLoader));
   List<T> instances = createSpringFactoriesInstances(type, parameterTypes, classLoader, args, names);
   AnnotationAwareOrderComparator.sort(instances);
   return instances;
}

private <T> List<T> createSpringFactoriesInstances(Class<T> type, Class<?>[] parameterTypes,
                                                   ClassLoader classLoader, Object[] args, Set<String> names) {
    List<T> instances = new ArrayList<>(names.size());
    for (String name : names) {
        try {
            Class<?> instanceClass = ClassUtils.forName(name, classLoader);
            Assert.isAssignable(type, instanceClass);
            Constructor<?> constructor = instanceClass.getDeclaredConstructor(parameterTypes);
            T instance = (T) BeanUtils.instantiateClass(constructor, args);
            instances.add(instance);
        }
        catch (Throwable ex) {
            throw new IllegalArgumentException("Cannot instantiate " + type + " : " + name, ex);
        }
    }
    return instances;
}
```

2. **SpringApplication#setInitializers**

```java
public void setInitializers(Collection<? extends ApplicationContextInitializer<?>> initializers) {
   this.initializers = new ArrayList<>();
   this.initializers.addAll(initializers);
}
```

### 3. 加载 Spring 应用时间监听器（ApplicationListener）

   `setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));`

和上面类似，都是覆盖性更新。

```java
public void setListeners(Collection<? extends ApplicationListener<?>> listeners) {
   this.listeners = new ArrayList<>();
   this.listeners.addAll(listeners);
}
```

### 4. 推断应用引导类

**SpringApplication#deduceMainApplicationClass**

  `this.mainApplicationClass = deduceMainApplicationClass();`

```java
private Class<?> deduceMainApplicationClass() {
   try {
      StackTraceElement[] stackTrace = new RuntimeException().getStackTrace();
      for (StackTraceElement stackTraceElement : stackTrace) {
         if ("main".equals(stackTraceElement.getMethodName())) {
            return Class.forName(stackTraceElement.getClassName());
         }
      }
   }
   catch (ClassNotFoundException ex) {
      // Swallow and continue
   }
   return null;
}
```

## SpringApplication 配置阶段

1. 调整 SpringApplication 设置；
2. 增加 SpringApplication 配置源；
3. 调整 Spring Boot 外部化配置（Externalized Configuration）。

### 调整 SpringApplication 设置

可以调用 setBannerMode(Banner.Mode.OFF)，关闭其打印。

### 增加 SpringApplication 配置源

没什么好说的。

# 第十六天

## SpringApplication 运行阶段

总览

+ SpringApplication 准备阶段
+ ApplicationContext 启动阶段
+ ApplicationContext 启动后阶段

**SpringApplication#run**

```java
public ConfigurableApplicationContext run(String... args) {
   StopWatch stopWatch = new StopWatch();
   stopWatch.start();
   ConfigurableApplicationContext context = null;
   Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList<>();
   configureHeadlessProperty();
   // <1>
   SpringApplicationRunListeners listeners = getRunListeners(args);
   listeners.starting();
   try {
      ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
      ConfigurableEnvironment environment = prepareEnvironment(listeners, applicationArguments);
      configureIgnoreBeanInfo(environment);
      Banner printedBanner = printBanner(environment);
      context = createApplicationContext();
      exceptionReporters = getSpringFactoriesInstances(SpringBootExceptionReporter.class,
            new Class[] { ConfigurableApplicationContext.class }, context);
      prepareContext(context, environment, listeners, applicationArguments, printedBanner);
      refreshContext(context);
      afterRefresh(context, applicationArguments);
      stopWatch.stop();
      if (this.logStartupInfo) {
         new StartupInfoLogger(this.mainApplicationClass).logStarted(getApplicationLog(), stopWatch);
      }
      listeners.started(context);
      callRunners(context, applicationArguments);
   }
   catch (Throwable ex) {
      handleRunFailure(context, ex, exceptionReporters, listeners);
      throw new IllegalStateException(ex);
   }

   try {
      listeners.running(context);
   }
   catch (Throwable ex) {
      handleRunFailure(context, ex, exceptionReporters, null);
      throw new IllegalStateException(ex);
   }
   return context;
}
```



## 理解 SpringApplicationRunListeners 及 SpringApplicationRunListener （两个不一样，少个 s ，不同类）

**SpringApplication#getRunListeners**

```java
private SpringApplicationRunListeners getRunListeners(String[] args) {
   Class<?>[] types = new Class<?>[] { SpringApplication.class, String[].class };
   return new SpringApplicationRunListeners(logger,
         getSpringFactoriesInstances(SpringApplicationRunListener.class, types, this, args));
}
```

**SpringApplicationRunListeners** 组合模式，还是要理解 SpringApplicationRunListener

```java
class SpringApplicationRunListeners {

   private final Log log;

   private final List<SpringApplicationRunListener> listeners;

   SpringApplicationRunListeners(Log log, Collection<? extends SpringApplicationRunListener> listeners) {
      this.log = log;
      this.listeners = new ArrayList<>(listeners);
   }

   public void starting() {
      for (SpringApplicationRunListener listener : this.listeners) {
         listener.starting();
      }
   }
}
```

**SpringApplicationRunListener**

可以理解为 Spring Boot 应用的运行时监听器，以下是它的方法以及对应的说明。

|                     监听方法                     |                         运行阶段说明                         | Spring Boot 起始版本 |          Spring Boot 事件           |
| :----------------------------------------------: | :----------------------------------------------------------: | :------------------: | :---------------------------------: |
|                    starting()                    |                      Spring 应用刚启动                       |         1.0          |      ApplicationStartingEvent       |
|   environmentPrepared(ConfigurableEnvironment)   |        ConfigurableEnvironment 准备妥当，允许将其调整        |         1.0          | ApplicationEnvironmentPreparedEvent |
| contextPrepared(ConfigurableApplicationContext)  |    ConfigurableApplicationContext 准备妥当，允许将其调整     |         1.0          |                                     |
|  contextLoaded(ConfigurableApplicationContext)   |      ConfigurableApplicationContext 已装载，但仍未启动       |         1.0          |      ApplicationPreparedEvent       |
|     started(ConfigurableAPplicationContext)      | ConfigurableApplicationContext 已启动，此时 Spring Bean 已经初始化完成 |         2.0          |       ApplicationStartedEvent       |
|     running(ConfigurableAPplicationContext)      |                     Spring 应用正在运行                      |         2.0          |        ApplicationReadyEvent        |
| failed(ConfigurableAPplicationContext,Throwable) |                     Spring 应用运行失败                      |         2.0          |       ApplicationFailedEvent        |

**EventPublishingRunListener**

```java
public class EventPublishingRunListener implements SpringApplicationRunListener, Ordered {

   private final SpringApplication application;

   private final String[] args;

   private final SimpleApplicationEventMulticaster initialMulticaster;

   public EventPublishingRunListener(SpringApplication application, String[] args) {
      this.application = application;
      this.args = args;
      this.initialMulticaster = new SimpleApplicationEventMulticaster();
      for (ApplicationListener<?> listener : application.getListeners()) {
         this.initialMulticaster.addApplicationListener(listener);
      }
   }

   @Override
   public int getOrder() {
      return 0;
   }

   @Override
   public void starting() {
      this.initialMulticaster.multicastEvent(new ApplicationStartingEvent(this.application, this.args));
   }

   @Override
   public void environmentPrepared(ConfigurableEnvironment environment) {
      this.initialMulticaster
            .multicastEvent(new ApplicationEnvironmentPreparedEvent(this.application, this.args, environment));
   }

   @Override
   public void contextPrepared(ConfigurableApplicationContext context) {
      this.initialMulticaster
            .multicastEvent(new ApplicationContextInitializedEvent(this.application, this.args, context));
   }
    
    // ....... 省略一大段代码
}
```



Spring Boot 事件和 Spring 事件还是有些差异的。

Spring 事件是由 Spring 应用上下文 ApplicationContext 对象触发的。然而 Spring  Boot 的事件发布者则是 SpringApplication#initialMulticaster 属性（SimpleApplicationEventMulticaster 类型），并且 

SimpleApplicationEventMulticaster 也来自 Spring Framework。下面讨论两者联系。



## Spring Event & Spring Boot Event

### 理解 Spring Event

极客时间那个课讲过了。

**事件对象应该遵守“默认”规则，继承 EventObject。**

**同时事件监听者必须是 EventListener 实例，不过 EventListener 仅为标记接口，类似 Serializable。**

泛型监听，极客时间的课程也讲了。

3.0 之后还引入了 SmartApplicationListener

# 第十七天

### Spring 事件发布

```java
public interface ApplicationEventMulticaster {

   /**
    * Add a listener to be notified of all events.
    * @param listener the listener to add
    */
   void addApplicationListener(ApplicationListener<?> listener);

   /**
    * Add a listener bean to be notified of all events.
    * @param listenerBeanName the name of the listener bean to add
    */
   void addApplicationListenerBean(String listenerBeanName);

   /**
    * Remove a listener from the notification list.
    * @param listener the listener to remove
    */
   void removeApplicationListener(ApplicationListener<?> listener);

   /**
    * Remove a listener bean from the notification list.
    * @param listenerBeanName the name of the listener bean to add
    */
   void removeApplicationListenerBean(String listenerBeanName);

   /**
    * Remove all listeners registered with this multicaster.
    * <p>After a remove call, the multicaster will perform no action
    * on event notification until new listeners are being registered.
    */
   void removeAllListeners();

   /**
    * Multicast the given application event to appropriate listeners.
    * <p>Consider using {@link #multicastEvent(ApplicationEvent, ResolvableType)}
    * if possible as it provides a better support for generics-based events.
    * @param event the event to multicast
    */
   void multicastEvent(ApplicationEvent event);

   /**
    * Multicast the given application event to appropriate listeners.
    * <p>If the {@code eventType} is {@code null}, a default type is built
    * based on the {@code event} instance.
    * @param event the event to multicast
    * @param eventType the type of event (can be null)
    * @since 4.2
    */
   void multicastEvent(ApplicationEvent event, @Nullable ResolvableType eventType);

}
```

1. ApplicationEventMulticaster 注册 ApplicationListener

SimpleApplicationEventMulticaster 与 ApplicationListener 的关系图（非UML）。

![SimpleApplicationEventMulticaster.png](https://i.loli.net/2020/12/25/PFNJjqEclYT4kDn.png)

Spring Boot 时间监听器均经过排序。

2. ApplicationEventMulticaster 广播事件。

两个与广播事件相关的方法。

**ApplicationEventMulticaster#multicastEvent(ApplicationEvent)**

**ApplicationEventMulticaster#multicastEvent(ApplicationEvent,ResolvableType)**

`SimpleApplicationEventMulticaster` 实现了上面两个方法，并且也是 Spring Framework 唯一实现。

**SimpleApplicationEventMulticaster#multicastEvent**

```java
@Override
public void multicastEvent(ApplicationEvent event) {
   multicastEvent(event, resolveDefaultEventType(event));
}

@Override
public void multicastEvent(final ApplicationEvent event, @Nullable ResolvableType eventType) {
   ResolvableType type = (eventType != null ? eventType : resolveDefaultEventType(event));
   Executor executor = getTaskExecutor();
   for (ApplicationListener<?> listener : getApplicationListeners(event, type)) {
      if (executor != null) {
         executor.execute(() -> invokeListener(listener, event));
      }
      else {
         invokeListener(listener, event);
      }
   }
}
```

> 其中 ResolvableType 是 4.2 开始引入的。ResolvableType 是为简化 Java 反射 API 而提供的组件，能够轻松地获得泛型类型等。

**SimpleApplicationEventMulticaster#invokerListener** 上面允许一步处理监听事件，不过无论是 Spring Framework 还是 Spring Boot 均未使用该方法来提升为异步执行，并且由于 EventPublishingRunListener 的封装，使得 Spring Boot 事件监听器无法异步执行。

```java
/**
 * Invoke the given listener with the given event.
 * @param listener the ApplicationListener to invoke
 * @param event the current event to propagate
 * @since 4.1
 */
protected void invokeListener(ApplicationListener<?> listener, ApplicationEvent event) {
   ErrorHandler errorHandler = getErrorHandler();
   if (errorHandler != null) {
      try {
         doInvokeListener(listener, event);
      }
      catch (Throwable err) {
         errorHandler.handleError(err);
      }
   }
   else {
      doInvokeListener(listener, event);
   }
}

@SuppressWarnings({"rawtypes", "unchecked"})
private void doInvokeListener(ApplicationListener listener, ApplicationEvent event) {
   try {
      listener.onApplicationEvent(event);
   }
   catch (ClassCastException ex) {
      String msg = ex.getMessage();
      if (msg == null || matchesClassCastMessage(msg, event.getClass())) {
         // Possibly a lambda-defined listener which we could not resolve the generic event type for
         // -> let's suppress the exception and just log a debug message.
         Log logger = LogFactory.getLog(getClass());
         if (logger.isTraceEnabled()) {
            logger.trace("Non-matching event type for listener: " + listener, ex);
         }
      }
      else {
         throw ex;
      }
   }
}

private boolean matchesClassCastMessage(String classCastMessage, Class<?> eventClass) {
   // On Java 8, the message starts with the class name: "java.lang.String cannot be cast..."
   if (classCastMessage.startsWith(eventClass.getName())) {
      return true;
   }
   // On Java 11, the message starts with "class ..." a.k.a. Class.toString()
   if (classCastMessage.startsWith(eventClass.toString())) {
      return true;
   }
   // On Java 9, the message used to contain the module name: "java.base/java.lang.String cannot be cast..."
   int moduleSeparatorIndex = classCastMessage.indexOf('/');
   if (moduleSeparatorIndex != -1 && classCastMessage.startsWith(eventClass.getName(), moduleSeparatorIndex + 1)) {
      return true;
   }
   // Assuming an unrelated class cast failure...
   return false;
}
```

3. ApplicationEventMulticaster 与 ApplicationContext 之间的关系

可以使用 ApplicationEventPublisher 发布 ApplicationEvent。

```java
public interface ApplicationEventPublisher {
    
   default void publishEvent(ApplicationEvent event) {
      publishEvent((Object) event);
   }
    
   void publishEvent(Object event);

}
```

**AbstractApplicationContext#prepareBeanFactory** refresh 方法

```java
/**
 * Configure the factory's standard context characteristics,
 * such as the context's ClassLoader and post-processors.
 * @param beanFactory the BeanFactory to configure
 */
protected void prepareBeanFactory(ConfigurableListableBeanFactory beanFactory) {
   // Tell the internal bean factory to use the context's class loader etc.
	// 省略........

   // BeanFactory interface not registered as resolvable type in a plain factory.
   // MessageSource registered (and found for autowiring) as a bean.
   beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
   beanFactory.registerResolvableDependency(ResourceLoader.class, this);
   // 1. 自己就是 ApplicationEventPublisher
   beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
   beanFactory.registerResolvableDependency(ApplicationContext.class, this);
   // 2. 
   // Register early post-processor for detecting inner beans as ApplicationListeners.
   beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(this));
	// 省略........
}
```

**AbstractApplicationContext#publishEvent**

```java
/**
 * Publish the given event to all listeners.
 * @param event the event to publish (may be an {@link ApplicationEvent}
 * or a payload object to be turned into a {@link PayloadApplicationEvent})
 * @param eventType the resolved event type, if known
 * @since 4.2
 */
protected void publishEvent(Object event, @Nullable ResolvableType eventType) {
   Assert.notNull(event, "Event must not be null");

   // Decorate event as an ApplicationEvent if necessary
   ApplicationEvent applicationEvent;
   if (event instanceof ApplicationEvent) {
      applicationEvent = (ApplicationEvent) event;
   }
   else {
      applicationEvent = new PayloadApplicationEvent<>(this, event);
      if (eventType == null) {
         eventType = ((PayloadApplicationEvent<?>) applicationEvent).getResolvableType();
      }
   }

   // Multicast right now if possible - or lazily once the multicaster is initialized
   if (this.earlyApplicationEvents != null) {
      this.earlyApplicationEvents.add(applicationEvent);
   }
   else {
      getApplicationEventMulticaster().multicastEvent(applicationEvent, eventType);
   }

   // Publish event via parent context as well...
   if (this.parent != null) {
      if (this.parent instanceof AbstractApplicationContext) {
         ((AbstractApplicationContext) this.parent).publishEvent(event, eventType);
      }
      else {
         this.parent.publishEvent(event);
      }
   }
}
```

开发只管 ApplicationEvent 类型以及对应的 ApplicationListener 的实现即可。

## Spring 内建事件

+ ContextRefreshedEvent: ；
+ ContextStartedEvent：Spring 应用上下文启动事件；
+ ContextStoppedEvent：Spring 应用上下文停止事件；
+ ContextClosedEvent：Spring 应用上下文关闭事件。

![image.png](https://i.loli.net/2020/12/28/z9ToG1SvHuZObMK.png)

### Spring 应用上下文就绪事件——ContextRefreshedEvent

当 ConfigurableApplicationContext#refresh 方法执行到finishRefresh 方法时，Spring 应用上下文发布 ContextRefreshedEvent：

refresh 方法会调用这个。

```java
protected void finishRefresh() {

   // Publish the final event.
   publishEvent(new ContextRefreshedEvent(this));

}
```

> 通常 `ApplicationListener<ContextRefreshedEvent>`实现类舰艇该事件，用于获取需要的 Bean，防止出现 Bean 提早初始化带来的潜在风险。
>
> 通常 BeanPostProceesor 也能用于获取指定的 Bean 对象，BeanFactory、ApplicationListener\<ContextRefreshedEvent\>选择后者是更安全的实践。

# 第十八天

### Spring 应用上下文启停事件——ContextStartedEvent 和 ContextStoppedEvent

**AbstractApplicationContext**

```java
public abstract class AbstractApplicationContext extends DefaultResourceLoader
		implements ConfigurableApplicationContext {
    
//---------------------------------------------------------------------
// Implementation of Lifecycle interface
//---------------------------------------------------------------------

    @Override
    public void start() {
       getLifecycleProcessor().start();
       publishEvent(new ContextStartedEvent(this));
    }

    @Override
    public void stop() {
       getLifecycleProcessor().stop();
       publishEvent(new ContextStoppedEvent(this));
    }
    
}
```

在绝大多数场景下，以上两个方法不会被调用。

**RestartEndpoint**

```java
@Endpoint(
    id = "restart",
    enableByDefault = false
)
public class RestartEndpoint implements ApplicationListener<ApplicationPreparedEvent> {}

@Endpoint(id = "pause")
public class PauseEndpoint {
    
    public PauseEndpoint() {}
    
    @WriteOperation
    public Boolean pause() {
        if (RestartEndpoint.this.isRunning()) {
            RestartEndpoint.this.doPause();
            return true;
        } else {
            return false;
        }
    }
    
}
public synchronized void doPause() {
    if (this.context != null) {
        this.context.stop();
    }

}
@Endpoint(id = "resume")
@ConfigurationProperties("management.endpoint.resume")
public class ResumeEndpoint {
    
    public ResumeEndpoint() {
    }

    @WriteOperation
    public Boolean resume() {
        if (!RestartEndpoint.this.isRunning()) {
            RestartEndpoint.this.doResume();
            return true;
        } else {
            return false;
        }
    }
    
}
```

### Spring 应用上下文关闭事件——ContextClosedEvent

 

```java
/**
 * Close this application context, destroying all beans in its bean factory.
 * <p>Delegates to {@code doClose()} for the actual closing procedure.
 * Also removes a JVM shutdown hook, if registered, as it's not needed anymore.
 * @see #doClose()
 * @see #registerShutdownHook()
 */
@Override
public void close() {
   synchronized (this.startupShutdownMonitor) {
      doClose();
      // If we registered a JVM shutdown hook, we don't need it anymore now:
      // We've already explicitly closed the context.
      if (this.shutdownHook != null) {
         try {
            Runtime.getRuntime().removeShutdownHook(this.shutdownHook);
         }
         catch (IllegalStateException ex) {
            // ignore - VM is already shutting down
         }
      }
   }
}

/**
 * Actually performs context closing: publishes a ContextClosedEvent and
 * destroys the singletons in the bean factory of this application context.
 * <p>Called by both {@code close()} and a JVM shutdown hook, if any.
 * @see org.springframework.context.event.ContextClosedEvent
 * @see #destroyBeans()
 * @see #close()
 * @see #registerShutdownHook()
 */
protected void doClose() {
    // Check whether an actual close attempt is necessary...
    if (this.active.get() && this.closed.compareAndSet(false, true)) {
        if (logger.isDebugEnabled()) {
            logger.debug("Closing " + this);
        }

        LiveBeansView.unregisterApplicationContext(this);

        try {
            // Publish shutdown event.
            publishEvent(new ContextClosedEvent(this));
        }
        catch (Throwable ex) {
            logger.warn("Exception thrown from ApplicationListener handling ContextClosedEvent", ex);
        }

        // Stop all Lifecycle beans, to avoid delays during individual destruction.
        if (this.lifecycleProcessor != null) {
            try {
                this.lifecycleProcessor.onClose();
            }
            catch (Throwable ex) {
                logger.warn("Exception thrown from LifecycleProcessor on context close", ex);
            }
        }

        // Destroy all cached singletons in the context's BeanFactory.
        destroyBeans();

        // Close the state of this context itself.
        closeBeanFactory();

        // Let subclasses do some final clean-up if they wish...
        onClose();

        // Reset local application listeners to pre-refresh state.
        if (this.earlyApplicationListeners != null) {
            this.applicationListeners.clear();
            this.applicationListeners.addAll(this.earlyApplicationListeners);
        }

        // Switch to inactive.
        this.active.set(false);
    }
}
```

在 Runtime 中移除 ShutdownHook 线程。

### 4. Spring 应用上下文事件——ApplicationContextEvent

自定义的

```java
package me.young1lin.spring.boot.thinking.event;

import javax.annotation.PostConstruct;

import org.springframework.beans.BeansException;
import org.springframework.context.ApplicationContext;
import org.springframework.context.ApplicationContextAware;
import org.springframework.context.ApplicationEvent;

/**
 * @author <a href="mailto:young1lin0108@gmail.com">young1lin</a>
 * @since 2020/12/28 上午8:04
 * @version 1.0
 */
public class CustomizeEvent extends ApplicationEvent implements ApplicationContextAware {

   private final String address;

   private final String test;

   private ApplicationContext applicationContext;

   /**
    * Create a new {@code ApplicationEvent}.
    * @param source the object on which the event initially occurred or with
    * which the event is associated (never {@code null})
    * @param address address
    * @param test test
    */
   public CustomizeEvent(Object source, String address, String test) {
      super(source);
      this.address = address;
      this.test = test;
   }


   @Override
   public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
      this.applicationContext = applicationContext;
   }

   @PostConstruct
   public void test(){
      applicationContext.publishEvent(new CustomizeEvent("","",""));
   }

}
```

## 4. Spring 事件监听

1. ApplicationListener 监听 Spring 内建事件

2. ApplicationListener 监听自定义 Spring 泛型事件

3. ApplicationListener 监听实现原理

**AbstractApplicationContext#doClose**

```java
// Destroy all cached singletons in the context's BeanFactory.
destroyBeans();

protected void destroyBeans() {
    getBeanFactory().destroySingletons();
}


```

## SpringApplication 的初始化
1、获取监听，SpringApplicationRunListeners listeners = getRunListeners(args); 最终是从spring.factories获取的
spring.factories配置文件中监听的配置如下
# Run Listeners
org.springframework.boot.SpringApplicationRunListener=\
org.springframework.boot.context.event.EventPublishingRunListener

```


 public ConfigurableApplicationContext run(String... args) {
        Startup startup = SpringApplication.Startup.create();
        if (this.properties.isRegisterShutdownHook()) {
            shutdownHook.enableShutdownHookAddition();
        }

        DefaultBootstrapContext bootstrapContext = this.createBootstrapContext();
        ConfigurableApplicationContext context = null;
        this.configureHeadlessProperty();
  	 // 1. 获取Spring的监听器类，这里是从 spring.factories 中去获取，默认的是以 org.springframework.boot.SpringApplicationRunListener 为key,获取到的监听器类型为 EventPublishingRunListener。
        SpringApplicationRunListeners listeners = this.getRunListeners(args);
	// 1.1 监听器发送启动事件
        listeners.starting(bootstrapContext, this.mainApplicationClass);

        try {
		// 封装参数
            ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
		// 2. 构造容器环境。将容器的一些配置内容加载到 environment  中
            ConfigurableEnvironment environment = this.prepareEnvironment(listeners, bootstrapContext, applicationArguments);
  		// 打印信息对象
            Banner printedBanner = this.printBanner(environment);
		// 3. 创建上下文对象
            context = this.createApplicationContext();
            context.setApplicationStartup(this.applicationStartup);
	    // 4. 准备刷新上下文
            this.prepareContext(bootstrapContext, context, environment, listeners, applicationArguments, printedBanner);
		// 5. 刷新上下文
            this.refreshContext(context);
		// 结束刷新，留待扩展功能，并未实现什么
            this.afterRefresh(context, applicationArguments);
		//启动
            startup.started();
            if (this.properties.isLogStartupInfo()) {
                (new StartupInfoLogger(this.mainApplicationClass, environment)).logStarted(this.getApplicationLog(), startup);
            }

            listeners.started(context, startup.timeTakenToStarted());
	// 调用 ApplicationRunner 和 CommandLineRunner 对应的方法
            this.callRunners(context, applicationArguments);
        } catch (Throwable ex) {
            throw this.handleRunFailure(context, ex, listeners);
        }

        try {
            if (context.isRunning()) {
                listeners.ready(context, startup.ready());
            }

            return context;
        } catch (Throwable ex) {
            throw this.handleRunFailure(context, ex, (SpringApplicationRunListeners)null);
        }
    }

```
2、环境变量的构造
ConfigurableEnvironment environment = this.prepareEnvironment(listeners, bootstrapContext, applicationArguments);

```
	private ConfigurableEnvironment prepareEnvironment(SpringApplicationRunListeners listeners,
			ApplicationArguments applicationArguments) {
		// Create and configure the environment
		// 获取或者创建 environment。这里获取类型是 StandardServletEnvironment 
		ConfigurableEnvironment environment = getOrCreateEnvironment();
		// 将入参配置到环境配置中
		configureEnvironment(environment, applicationArguments.getSourceArgs());
		ConfigurationPropertySources.attach(environment);
		// 发布环境准备事件。
		listeners.environmentPrepared(environment);
		bindToSpringApplication(environment);
		if (!this.isCustomEnvironment) {
			environment = new EnvironmentConverter(getClassLoader()).convertEnvironmentIfNecessary(environment,
					deduceEnvironmentClass());
		}
		ConfigurationPropertySources.attach(environment);
		return environment;
	}
```
<img width="1118" height="728" alt="image" src="https://github.com/user-attachments/assets/41006a4c-8f55-4ce8-a705-32cbe8c35b20" />

<img width="1307" height="176" alt="image" src="https://github.com/user-attachments/assets/3c95cbb0-b8db-4cd9-8262-0cc599ef5de9" />


```
    private ConfigurableEnvironment getOrCreateEnvironment() {
        if (this.environment != null) {
            return this.environment;
        } else {
            WebApplicationType webApplicationType = this.properties.getWebApplicationType();
            ConfigurableEnvironment environment = this.applicationContextFactory.createEnvironment(webApplicationType);
            if (environment == null && this.applicationContextFactory != ApplicationContextFactory.DEFAULT) {
                environment = ApplicationContextFactory.DEFAULT.createEnvironment(webApplicationType);
            }

            return (ConfigurableEnvironment)(environment != null ? environment : new ApplicationEnvironment());
        }
    }
```
webApplicationType 的值在构造函数中NONE，所以environment new ApplicationEnvironment;

```
Spring Boot 2.x 中：
在调用 listeners.environmentPrepared() 时，会触发 ApplicationEnvironmentPreparedEvent 事件，其中 ConfigFileApplicationListener 会监听此事件并负责加载配置文件 application.yml / application.properties。
```
```
Spring Boot 3.x 中：
配置文件的加载已经不再依赖 ConfigFileApplicationListener，而是在 prepareEnvironment() 阶段通过 ConfigDataEnvironmentPostProcessor 实现，提前完成 application.yml / application.properties 的加载。

// 1. 获取监听器集合
SpringApplicationRunListeners listeners = getRunListeners(args);

// 2. 创建环境
ConfigurableEnvironment environment = prepareEnvironment(listeners, bootstrapContext, applicationArguments);

// 3. 环境准备完成通知,会发布一个ApplicationEnvironmentPreparedEvent事件，监听这个事件的其它listener开始运行逻辑
listeners.environmentPrepared(bootstrapContext, environment);

```
spring.factories中会加载所有的监听
```
# Application Listeners
org.springframework.context.ApplicationListener=\
org.springframework.boot.ClearCachesApplicationListener,\
org.springframework.boot.builder.ParentContextCloserApplicationListener,\
org.springframework.boot.context.FileEncodingApplicationListener,\
org.springframework.boot.context.config.AnsiOutputApplicationListener,\
org.springframework.boot.context.logging.LoggingApplicationListener,\
org.springframework.boot.env.EnvironmentPostProcessorApplicationListener
```

```
LoggingApplicationListener中的监听事件，监听了ApplicationEnvironmentPreparedEvent
 public void onApplicationEvent(ApplicationEvent event) {
        if (event instanceof ApplicationStartingEvent startingEvent) {
            this.onApplicationStartingEvent(startingEvent);
        } else if (event instanceof ApplicationEnvironmentPreparedEvent environmentPreparedEvent) {
            this.onApplicationEnvironmentPreparedEvent(environmentPreparedEvent);
        } else if (event instanceof ApplicationPreparedEvent preparedEvent) {
            this.onApplicationPreparedEvent(preparedEvent);
        } else if (event instanceof ContextClosedEvent contextClosedEvent) {
            this.onContextClosedEvent(contextClosedEvent);
        } else if (event instanceof ApplicationFailedEvent) {
            this.onApplicationFailedEvent();
        }

    }
EnvironmentPostProcessorApplicationListener中也监听了ApplicationEnvironmentPreparedEvent
  public void onApplicationEvent(ApplicationEvent event) {
        if (event instanceof ApplicationEnvironmentPreparedEvent environmentPreparedEvent) {
            this.onApplicationEnvironmentPreparedEvent(environmentPreparedEvent);
        }

        if (event instanceof ApplicationPreparedEvent) {
            this.onApplicationPreparedEvent();
        }

        if (event instanceof ApplicationFailedEvent) {
            this.onApplicationFailedEvent();
        }

    }
AnsiOutputApplicationListener 实现了ApplicationEnvironmentPreparedEvent
public class AnsiOutputApplicationListener implements ApplicationListener<ApplicationEnvironmentPreparedEvent>, Ordered {
    public AnsiOutputApplicationListener() {
    }

    public void onApplicationEvent(ApplicationEnvironmentPreparedEvent event) {
        ConfigurableEnvironment environment = event.getEnvironment();
        Binder.get(environment).bind("spring.output.ansi.enabled", AnsiOutput.Enabled.class).ifBound(AnsiOutput::setEnabled);
        AnsiOutput.setConsoleAvailable((Boolean)environment.getProperty("spring.output.ansi.console-available", Boolean.class));
    }

    public int getOrder() {
        return -2147483637;
    }
}

```
在发布 ApplicationEnvironmentPreparedEvent之后依次执行监听器：
LoggingApplicationListener
AnsiOutputApplicationListener
EnvironmentPostProcessorApplicationListener
用户注册的自定义监听器
完成 application.yml 的加载和日志系统初始化

3 上下文
```
SpringApplication.run()
    |
    |-- createApplicationContext()
    |
    |-- prepareContext()
    |     |-- set environment
    |     |-- apply initializers
    |     |-- load main class config
    |
    |-- refreshContext()
    |     |-- context.refresh()
    |         |-- BeanFactory 创建与初始化
    |         |-- Bean 实例化和注入
    |         |-- ApplicationEventMulticaster 创建
    |
    |-- afterRefresh()
          |-- 发布 ApplicationReadyEvent

```
创建上下文
```
context = this.createApplicationContext();


    protected ConfigurableApplicationContext createApplicationContext() {
        return this.applicationContextFactory.create(this.properties.getWebApplicationType());
    }
默认实现是DefaultApplicationContextFactory
    private ConfigurableApplicationContext createDefaultApplicationContext() {
        return (ConfigurableApplicationContext)(!AotDetector.useGeneratedArtifacts() ? new AnnotationConfigApplicationContext() : new GenericApplicationContext());
    }


```
prepareContext
```
private void prepareContext(...) {
    // 设置环境
    context.setEnvironment(environment);

    // 设置 ApplicationContext 的 beanNameGenerator（用于生成bean的名字，@Component、@Service、@Repository 等注解bean生成的唯一名字要用到）
	//resourceLoader (加载例如application.ym)
    postProcessApplicationContext(context);

    // 执行 ApplicationContextInitializer初始化ApplicationContext 容器本身
    applyInitializers(context);

    // 广播 ApplicationContextPreparedEvent事件，上下文初始化成功
    listeners.contextPrepared(context);

    // 加载主配置类（通常是 @SpringBootApplication 类）
    this.loader.load(context, sources);
}
```
refreshContext
```
    public void refresh() throws BeansException, IllegalStateException {
    // 加锁，防止多线程同时刷新上下文
    this.startupShutdownLock.lock();

    try {
        // 记录当前线程为启动线程
        this.startupShutdownThread = Thread.currentThread();

        // 开始记录 Spring 应用启动步骤（用于性能分析等）
        StartupStep contextRefresh = this.applicationStartup.start("spring.context.refresh");

        // 1. 准备上下文环境（如设置启动时间、初始化环境变量、验证必要属性）
        this.prepareRefresh();

        // 2. 返回 DefaultListableBeanFactory，即 Bean 的核心注册和管理工
        ConfigurableListableBeanFactory beanFactory = this.obtainFreshBeanFactory();

        // 3. 为 BeanFactory 设置各种支持，如添加环境变量、ApplicationContextAware 等
        this.prepareBeanFactory(beanFactory);

        try {
            // 4. 留给子类进行自定义的 BeanFactory 修改（一般用户不会使用）
            this.postProcessBeanFactory(beanFactory);

            // 启动“Bean 后置处理器”记录
            StartupStep beanPostProcess = this.applicationStartup.start("spring.context.beans.post-process");

            // 5. 执行 BeanFactoryPostProcessor（如配置文件解析器、@Value 处理等）
            this.invokeBeanFactoryPostProcessors(beanFactory);

            // 6. 注册所有 BeanPostProcessor（如 @Autowired、@Transactional 等注解支持）
            this.registerBeanPostProcessors(beanFactory);

            // Bean 后置处理器处理完成
            beanPostProcess.end();

            // 7. 初始化国际化 MessageSource（用于 i18n）
            this.initMessageSource();

            // 8. 初始化 ApplicationEventMulticaster（事件广播器）
            this.initApplicationEventMulticaster();

            // 9. 钩子方法：留给子类（如 WebApplicationContext 初始化 Servlet 等）
            this.onRefresh();

            // 10. 注册所有 ApplicationListener（事件监听器）ApplicationEventMulticaster，来监听Spring 应用上下文生命周期事件，spring.factories中的监听是springboot启动阶段事件监听
            this.registerListeners();

            // 11. 初始化所有剩余的非懒加载单例 Bean
            this.finishBeanFactoryInitialization(beanFactory);

            // 12. 发布 ContextRefreshedEvent，完成刷新
            this.finishRefresh();
        } catch (Error | RuntimeException var12) {
            // 如果中途出错，打印警告，销毁所有已创建的 Bean，取消刷新
            if (this.logger.isWarnEnabled()) {
                this.logger.warn("Exception encountered during context initialization - cancelling refresh attempt: " + String.valueOf(var12));
            }

            this.destroyBeans();          // 销毁已创建的 Bean
            this.cancelRefresh(var12);    // 发布 ContextRefreshFailedEvent
            throw var12;                  // 向上传递异常
        } finally {
            // 结束启动步骤记录
            contextRefresh.end();
        }
    } finally {
        // 解锁，清理线程标记
        this.startupShutdownThread = null;
        this.startupShutdownLock.unlock();
    }
}

```



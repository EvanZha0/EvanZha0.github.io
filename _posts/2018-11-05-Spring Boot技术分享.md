
## Spring Boot 简介

Spring Boot是伴随着Spring4.0 产生的，其设计目的是用来简化新Spring应用的搭建、开发、部署。

![image](http://106.15.205.155:8079/springboot/menu.saveimg.savepath20180718135723.jpg)

它算不上一个全新的框架，只是一些库的集合（mvnrepository中spring-boot-starter...）,springboot的宗旨：**约定大于配置**

既然没有了xml文件，怎么去配置环境？

比如指定单个数据源，只要在application.yml中约定url、username等参数就可以了
```
spring:
  datasource:
      url: jdbc:mysql://106.15.205.155:3306/ccclubs_gwa_sys?autoReconnect=true&useUnicode=true&characterEncoding=utf-8&useSSL=false
      driver-class-name: com.mysql.jdbc.Driver
      username: root
      password: xltys1995
```

比如指定tomcat端口：

```
server:
  port: 9090
```

Spring Boot让我们的Spring应用变得更轻量化。它内置了tomcat（jetty），只需运行jar文件就能启动项目。比如：你可以仅仅依靠一个Java类来运行Spring引用。你也可以打包你的应用为jar并通过使用java –jar来运行你的Spring Web应用。


来看一个sh脚本：

![image](http://106.15.205.155:8079/springboot/menu.saveimg.savepath20180718140040.jpg)

---

## Spring Boot启动流程

首先我们来看Spring Boot项目的启动类：


```
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class Boot {

    public static void main(String[] args) {
        SpringApplication.run(Boot.class, args);
    }
}
```

启动类非常简单，主要来分析@SpringBootApplication注解，进入源码：


```
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(excludeFilters = {
		@Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
		@Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })
public @interface SpringBootApplication {
......
```

@SpringBootApplication是一个复合注解,逐个分析：

1.@SpringBootConfiguration：此注解其实就是@Configuration，表面了当前类（启动类）是一个配置类，我们可以在里面配置Bean并且注入到IOC容器。

2.@EnableAutoConfiguration：这个类是关键，用一句话概括就是它把jar文件的Bean注入Ioc容器！

我们来看它的源码：
```
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@AutoConfigurationPackage
@Import(AutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration {
...
```
通过Improt注解导入的执行类AutoConfigurationImportSelector类来实现

在AutoConfigurationImportSelector类中有一个getCandidateConfigurations()方法，他会去扫描所有第三方包下的META-INF/spring.factories文件，第三方包会把需要注入ioc的bean配置在此文件中，通过debug的方式，我们来看看一个springboot项目启动时，getCandidateConfigurations()方法加载了多少Configuration


```
Enumeration<URL> urls = (classLoader != null ?
					classLoader.getResources(FACTORIES_RESOURCE_LOCATION) :
					ClassLoader.getSystemResources(FACTORIES_RESOURCE_LOCATION));
```


![image](http://106.15.205.155:8079/springboot/menu.saveimg.savepath20180718162058.jpg)

可以看到，在项目启动时，此方法加载了110个configuration类，在这些类中，会注入更多的Bean


3.@ComponentScan：这没什么好说的，就是xml配置中的<context:component-scan>标签，意思是扫描当前包和子包下带有@Configuration，@Component，@Controller，@Service，@Repository注解的类，并且注入ioc

注解讲完了，开始解释SpringApplication.run()方法：

首先来看run方法的主要源码：


```
public ConfigurableApplicationContext run(String... args) {
		StopWatch stopWatch = new StopWatch();
		stopWatch.start();
		ConfigurableApplicationContext context = null;
		Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList<>();
		configureHeadlessProperty();
		SpringApplicationRunListeners listeners = getRunListeners(args);
		listeners.starting();
		try {
			ApplicationArguments applicationArguments = new DefaultApplicationArguments(
					args);
			ConfigurableEnvironment environment = prepareEnvironment(listeners,
					applicationArguments);
			configureIgnoreBeanInfo(environment);
			Banner printedBanner = printBanner(environment);
			context = createApplicationContext();
			exceptionReporters = getSpringFactoriesInstances(
					SpringBootExceptionReporter.class,
					new Class[] { ConfigurableApplicationContext.class }, context);
			prepareContext(context, environment, listeners, applicationArguments,
					printedBanner);
			refreshContext(context);
			afterRefresh(context, applicationArguments);
			stopWatch.stop();
			if (this.logStartupInfo) {
				new StartupInfoLogger(this.mainApplicationClass)
						.logStarted(getApplicationLog(), stopWatch);
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


##### 1.StopWatch任务计时

##### 2.configureHeadlessProperty

这是给监控用的，比如jconsole,实际上是就是设置系统属性 java.awt.headless，该属性会被设置为 true

##### 3.SpringApplicationRunListeners

它是SpringApplicationRunListener的集合，而SpringApplicationRunListener其实间接调用的是ApplicationListener接口，他们都是Spring的监听器，这一点我们可以从SpringApplicationRunListener的实现类EventPublishingRunListener中发现，利用观察者模式实现广播功能，而广播器就是SimpleApplicationEventMulticaster


```
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
```


ApplicationListener可以在Spring不同生命周期进行监听，如果我们要在不同生命周期做一些操作，可以实现ApplicationListener接口并指定生命周期事件（Event可以继承ApplicationEvent自己实现，在调用某个Bean方法时，通过ApplicationContext.publishEvent(Event)方法发布指定事件），然后重写onApplicationEvent方法，最后将实现类注入IOC容器即可。在不同生命周期有不同的事件

比如：

```
class CoolEvent extends ApplicationEvent {

        private String msg;
        /**
         * Create a new ApplicationEvent.
         *
         * @param source the object on which the event initially occurred (never {@code null})
         */
        public CoolEvent(Object source, String msg) {
            super(source);
            this.msg = msg;
        }

        public String getMsg() {
            return msg;
        }

        public void setMsg(String msg) {
            this.msg = msg;
        }
    }




    @Component
    class CoolListener implements ApplicationListener<ApplicationEvent> {


        @Override
        public void onApplicationEvent(ApplicationEvent event) {

            if (!(event instanceof CoolEvent)){
                return;
            }

            String msg = ((CoolEvent) event).getMsg();

            System.out.println(msg);

        }
    }



    
    @Component("cool")
    class CoolBean implements ApplicationContextAware {

        private ApplicationContext applicationContext;

        @Override
        public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {

            this.applicationContext = applicationContext;
        }

        public void cool(){

            CoolEvent event = new CoolEvent(applicationContext, "OK!!!");

            applicationContext.publishEvent(event);
        }

    }
```

这边就不一一展开了，可以自行研究

而SpringApplicationRunListener其实代表的就是Springioc的生命周期，它来指定周期，并且每个阶段会执行相应的时间，主要有以下几种监听：

```
    //刚执行run方法时
    void started();
     //环境建立好时候
    void environmentPrepared(ConfigurableEnvironment environment);
     //上下文建立好的时候
    void contextPrepared(ConfigurableApplicationContext context);
    //上下文载入配置时候
    void contextLoaded(ConfigurableApplicationContext context);
    //上下文刷新完成后，run方法执行完之前
    void finished(ConfigurableApplicationContext context, Throwable exception);

```

监听器结束，我们来看下一行关键代码：

##### 4.命令行参数：

```
ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
```
深入这个构造器，我们可以看到如下代码：


```
public CommandLineArgs parse(String... args) {
		CommandLineArgs commandLineArgs = new CommandLineArgs();
		for (String arg : args) {
			if (arg.startsWith("--")) {
				String optionText = arg.substring(2, arg.length());
				String optionName;
				String optionValue = null;
				if (optionText.contains("=")) {
					optionName = optionText.substring(0, optionText.indexOf('='));
					optionValue = optionText.substring(optionText.indexOf('=')+1, optionText.length());
				}
				else {
					optionName = optionText;
				}
				if (optionName.isEmpty() || (optionValue != null && optionValue.isEmpty())) {
					throw new IllegalArgumentException("Invalid argument syntax: " + arg);
				}
				commandLineArgs.addOptionArg(optionName, optionValue);
			}
			else {
				commandLineArgs.addNonOptionArg(arg);
			}
		}
		return commandLineArgs;
	}

```
有了这个方法，就能使用命令行来指定启动环境spring.profiles.active、启动tomcat端口server.port等等

接着看下面的代码：


```
ConfigurableEnvironment environment = prepareEnvironment(listeners,
					applicationArguments);
```
将上面拿到的所有监听器、命令行参数传递到prepareEnvironment函数准备运行环境，包括根据有无DispatcherServlet类判断是否为servlet环境，更改banner代码等等，可以自行研究。

##### 6.异常注入

来看下面的代码：

```
exceptionReporters = getSpringFactoriesInstances(
					SpringBootExceptionReporter.class,
					new Class[] { ConfigurableApplicationContext.class }, context);
```

进入getSpringFactoriesInstances方法，我们又看到了loadFactoryNames方法，依旧是查找META-INF/spring.factories这个文件，上面又configuration，listener，而这次是excpetion，将jar文件的excpetion也注入之后，信息录入差不多完成。

##### 7.准备环境：


```
prepareContext(context, environment, listeners, applicationArguments,printedBanner);
```

```
private void prepareContext(ConfigurableApplicationContext context,
			ConfigurableEnvironment environment, SpringApplicationRunListeners listeners,
			ApplicationArguments applicationArguments, Banner printedBanner) {
		context.setEnvironment(environment);
		postProcessApplicationContext(context);
		applyInitializers(context);
		listeners.contextPrepared(context);
		if (this.logStartupInfo) {
			logStartupInfo(context.getParent() == null);
			logStartupProfileInfo(context);
		}

		// Add boot specific singleton beans
		context.getBeanFactory().registerSingleton("springApplicationArguments",
				applicationArguments);
		if (printedBanner != null) {
			context.getBeanFactory().registerSingleton("springBootBanner", printedBanner);
		}

		// Load the sources
		Set<Object> sources = getAllSources();
		Assert.notEmpty(sources, "Sources must not be empty");
		load(context, sources.toArray(new Object[0]));
		listeners.contextLoaded(context);
	}
```
完成整个容器的创建与启动以及 bean 的注入功能。

其中postProcessApplicationContext方法对 context 进行了预设置，设置了 ResourceLoader 和 ClassLoader，并向 bean 工厂中添加了一个beanNameGenerator 。

##### 8.refreshContext()

```
private void refreshContext(ConfigurableApplicationContext context) {
		refresh(context);
		if (this.registerShutdownHook) {
			try {
				context.registerShutdownHook();
			}
			catch (AccessControlException ex) {
				// Not allowed in some environments.
			}
		}
	}
```

依旧有一个refresh方法，但在下面可以发现上下文注册了关闭钩子

java Runtime注册的ShutdownHook在JVM进程正常关闭时执行，
而spring也需要在jvm关闭时释放很多bean持有的资源，比如mysql连接池、redis连接池、清理zookeeper的临时节点、释放分布式锁等等，

传统方法都是通过xml里bean标签的destory-method方法释放资源，
在springboot中我们可以使用@PreDestroy注解，
还有一种实现DisposableBean接口，重写destroy()方法
Bean在销毁的过程中：@PreDestroy > DisposableBean > destroy-method

接下来重点介绍refresh方法：

```
@Override
	public void refresh() throws BeansException, IllegalStateException {
		synchronized (this.startupShutdownMonitor) {
			// Prepare this context for refreshing.
			prepareRefresh();

			// Tell the subclass to refresh the internal bean factory.
			ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

			// Prepare the bean factory for use in this context.
			prepareBeanFactory(beanFactory);

			try {
				// Allows post-processing of the bean factory in context subclasses.
				postProcessBeanFactory(beanFactory);

				// Invoke factory processors registered as beans in the context.
				invokeBeanFactoryPostProcessors(beanFactory);

				// Register bean processors that intercept bean creation.
				registerBeanPostProcessors(beanFactory);

				// Initialize message source for this context.
				initMessageSource();

				// Initialize event multicaster for this context.
				initApplicationEventMulticaster();

				// Initialize other special beans in specific context subclasses.
				onRefresh();

				// Check for listener beans and register them.
				registerListeners();

				// Instantiate all remaining (non-lazy-init) singletons.
				finishBeanFactoryInitialization(beanFactory);

				// Last step: publish corresponding event.
				finishRefresh();
			}

			catch (BeansException ex) {
				if (logger.isWarnEnabled()) {
					logger.warn("Exception encountered during context initialization - " +
							"cancelling refresh attempt: " + ex);
				}

				// Destroy already created singletons to avoid dangling resources.
				destroyBeans();

				// Reset 'active' flag.
				cancelRefresh(ex);

				// Propagate exception to caller.
				throw ex;
			}

			finally {
				// Reset common introspection caches in Spring's core, since we
				// might not ever need metadata for singleton beans anymore...
				resetCommonCaches();
			}
		}
	}
```
看注释可以大概了解，此时正在做大量的初始化、注册工作

而我们关注的重点就是方法finishBeanFactoryInitialization(beanFactory)，它执行饿汉式的bean加载工作。


```
protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
		// Initialize conversion service for this context.
		if (beanFactory.containsBean(CONVERSION_SERVICE_BEAN_NAME) &&
				beanFactory.isTypeMatch(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class)) {
			beanFactory.setConversionService(
					beanFactory.getBean(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class));
		}

		// Register a default embedded value resolver if no bean post-processor
		// (such as a PropertyPlaceholderConfigurer bean) registered any before:
		// at this point, primarily for resolution in annotation attribute values.
		if (!beanFactory.hasEmbeddedValueResolver()) {
			beanFactory.addEmbeddedValueResolver(strVal -> getEnvironment().resolvePlaceholders(strVal));
		}

		// Initialize LoadTimeWeaverAware beans early to allow for registering their transformers early.
		String[] weaverAwareNames = beanFactory.getBeanNamesForType(LoadTimeWeaverAware.class, false, false);
		for (String weaverAwareName : weaverAwareNames) {
			getBean(weaverAwareName);
		}

		// Stop using the temporary ClassLoader for type matching.
		beanFactory.setTempClassLoader(null);

		// Allow for caching all bean definition metadata, not expecting further changes.
		beanFactory.freezeConfiguration();

		// Instantiate all remaining (non-lazy-init) singletons.
		beanFactory.preInstantiateSingletons();
	}
```

进入beanFactory.preInstantiateSingletons()方法，发现这是一个接口，当我们查找它的实现类时发现只有一个实现类：DefaultListableBeanFactory。

查看它的preInstantiateSingletons()方法：


```
@Override
	public void preInstantiateSingletons() throws BeansException {
		if (this.logger.isDebugEnabled()) {
			this.logger.debug("Pre-instantiating singletons in " + this);
		}

		// Iterate over a copy to allow for init methods which in turn register new bean definitions.
		// While this may not be part of the regular factory bootstrap, it does otherwise work fine.
		List<String> beanNames = new ArrayList<>(this.beanDefinitionNames);

		// Trigger initialization of all non-lazy singleton beans...
		for (String beanName : beanNames) {
			RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
			if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
				if (isFactoryBean(beanName)) {
					Object bean = getBean(FACTORY_BEAN_PREFIX + beanName);
					if (bean instanceof FactoryBean) {
						final FactoryBean<?> factory = (FactoryBean<?>) bean;
						boolean isEagerInit;
						if (System.getSecurityManager() != null && factory instanceof SmartFactoryBean) {
							isEagerInit = AccessController.doPrivileged((PrivilegedAction<Boolean>)
											((SmartFactoryBean<?>) factory)::isEagerInit,
									getAccessControlContext());
						}
						else {
							isEagerInit = (factory instanceof SmartFactoryBean &&
									((SmartFactoryBean<?>) factory).isEagerInit());
						}
						if (isEagerInit) {
							getBean(beanName);
						}
					}
				}
				else {
					getBean(beanName);
				}
			}
		}

		// Trigger post-initialization callback for all applicable beans...
		for (String beanName : beanNames) {
			Object singletonInstance = getSingleton(beanName);
			if (singletonInstance instanceof SmartInitializingSingleton) {
				final SmartInitializingSingleton smartSingleton = (SmartInitializingSingleton) singletonInstance;
				if (System.getSecurityManager() != null) {
					AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
						smartSingleton.afterSingletonsInstantiated();public ConfigurableApplicationContext run(String... args) {
		StopWatch stopWatch = new StopWatch();
		stopWatch.start();
		ConfigurableApplicationContext context = null;
		Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList<>();
		configureHeadlessProperty();
		SpringApplicationRunListeners listeners = getRunListeners(args);
		listeners.starting();
		try {
			ApplicationArguments applicationArguments = new DefaultApplicationArguments(
					args);
			ConfigurableEnvironment environment = prepareEnvironment(listeners,
					applicationArguments);
			configureIgnoreBeanInfo(environment);
			Banner printedBanner = printBanner(environment);
			context = createApplicationContext();
			exceptionReporters = getSpringFactoriesInstances(
					SpringBootExceptionReporter.class,
					new Class[] { ConfigurableApplicationContext.class }, context);
			prepareContext(context, environment, listeners, applicationArguments,
					printedBanner);
			refreshContext(context);
			afterRefresh(context, applicationArguments);
			stopWatch.stop();
			if (this.logStartupInfo) {
				new StartupInfoLogger(this.mainApplicationClass)
						.logStarted(getApplicationLog(), stopWatch);
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
						return null;
					}, getAccessControlContext());
				}
				else {
					smartSingleton.afterSingletonsInstantiated();
				}
			}
		}
	}
```

发现有很多getBean()方法，点进去，在doGetBean()中，发现最重要的createBean()，这就是创建bean信息的最终方法，在实现类实现类在 AbstractAutowireCapableBeanFactory 中得以实现。

OK，代码介绍到这里已经回不去了！

我们直接回到run()


```
public ConfigurableApplicationContext run(String... args) {
		StopWatch stopWatch = new StopWatch();
		stopWatch.start();
		ConfigurableApplicationContext context = null;
		Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList<>();
		configureHeadlessProperty();
		SpringApplicationRunListeners listeners = getRunListeners(args);
		listeners.starting();
		try {
			ApplicationArguments applicationArguments = new DefaultApplicationArguments(
					args);
			ConfigurableEnvironment environment = prepareEnvironment(listeners,
					applicationArguments);
			configureIgnoreBeanInfo(environment);
			Banner printedBanner = printBanner(environment);
			context = createApplicationContext();
			exceptionReporters = getSpringFactoriesInstances(
					SpringBootExceptionReporter.class,
					new Class[] { ConfigurableApplicationContext.class }, context);
			prepareContext(context, environment, listeners, applicationArguments,
					printedBanner);
			refreshContext(context);
			afterRefresh(context, applicationArguments);
			stopWatch.stop();
			if (this.logStartupInfo) {
				new StartupInfoLogger(this.mainApplicationClass)
						.logStarted(getApplicationLog(), stopWatch);
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
这时候看到stopWatch.stop()计时结束,也说明springboot启动完成,接着listeners.started(context)监听器设置为启动状态,最后callRunners通知上下文他们的run方法

##### 总结:

spring boot启动的时候，首先会调用SpringApplication的构造函数进行初始化，调用SpringApplication的run函数，获取监听器，并设置成启动状态，后面准备环境prepareEnvironment，准备prepareContext上下文，刷新上下文refreshContext，最后调用callRunners来依次调用注册的Runner。


---
layout: post
title:  "springboot源码解读(一)启动"
categories: springboot源码解读
tags:  tech
author: Lzg
---

* content
{:toc}


**_Springboot之启动过程_**

  这篇文章会大体上解释一遍sprinboot的启动过程，后续的文章会对其中的细节进行深究。

我们都知道启动一个最简单的springboot应用是这样的：

```java
@SpringBootApplication
public class Application {

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

点进去发现是这样的：

```java
public static ConfigurableApplicationContext run(Object[] sources, String[] args) {
		return new SpringApplication(sources).run(args);
	}
  //new之后会调用初始化方法
  if (sources != null && sources.length > 0) {
			this.sources.addAll(Arrays.asList(sources));
		}
        // 判断是否是web环境，根据servlet在不在类路径上，class.forname(servlet)
		this.webEnvironment = deduceWebEnvironment();
        //  设置容器初始化器材 ApplicationContextInitializer
		setInitializers((Collection) getSpringFactoriesInstances(
				ApplicationContextInitializer.class));
        // 加载并设置applicationListener
		setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
        // 获取运行应用的Main类，方法如下
		this.mainApplicationClass = deduceMainApplicationClass();

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

上面就是一个完整的iitialize过程，其中的加载类会放在后面再聊，下面就进入到`run()`了

* * *

```java
    //计时
    StopWatch stopWatch = new StopWatch();
		stopWatch.start();
		ConfigurableApplicationContext context = null;
		FailureAnalyzers analyzers = null;
		configureHeadlessProperty();
    //创建ApplicationStartedEvent，
    //调用ApplicationListener.onApplicationEvent,设置了日志，没干太多事情
    SpringApplicationRunListeners listeners = getRunListeners(args);
		listeners.started();
		try {
			ApplicationArguments applicationArguments = new DefaultApplicationArguments(
					args);
      // 创建配置enviroment, 设置profiles,
      //创建ApplicationEnvironmentPreparedEvent事件，调用ApplicationListener
      //ConfigFileApplicationListener 配置EnvironmentPostProcessor
      //    
			ConfigurableEnvironment environment = prepareEnvironment(listeners,
					applicationArguments);
			Banner printedBanner = printBanner(environment);
      // web 创建AnnotationConfigEmbeddedWebApplicationContext
			context = createApplicationContext();
			analyzers = new FailureAnalyzers(context);
      // 排序并调用ApplicationContextInitializer
      // 设置applicationId
      // 注册bean factory postprocessor ConfigurationWarningsPostProcessor
      //CachingMetadataReaderFactoryPostProcessor
      // 设置BeanDefinitionLoader（及context）
      // 通过lister注册 PropertySourceOrderingPostProcessor

			prepareContext(context, environment, listeners, applicationArguments,
					printedBanner);
			refreshContext(context);
			afterRefresh(context, applicationArguments);
			listeners.finished(context, null);
			stopWatch.stop();
			if (this.logStartupInfo) {
				new StartupInfoLogger(this.mainApplicationClass)
						.logStarted(getApplicationLog(), stopWatch);
			}
			return context;
		}
		catch (Throwable ex) {
			handleRunFailure(context, listeners, analyzers, ex);
			throw new IllegalStateException(ex);
		}
```

接下来就是最重要的refresh方法
```java
public void refresh() throws BeansException, IllegalStateException {
		synchronized (this.startupShutdownMonitor) {
			// Prepare this context for refreshing.
			prepareRefresh();

			// Tell the subclass to refresh the internal bean factory.
			ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

			// Prepare the bean factory for use in this context.
      // 注册 ApplicationContextAwareProcessor
      // 配置依赖注入需要忽视，以及要引用的接口
      // 设置environment beans
			prepareBeanFactory(beanFactory);

			try {
				// Allows post-processing of the bean factory in context subclasses.
        //设置 WebApplicationContextServletContextAwareProcessor
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

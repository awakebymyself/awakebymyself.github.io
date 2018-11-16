---
layout: post
title:  "springboot源码解读(三)自动配置"
categories: springboot源码解读
tags:  tech
author: Lzg
---

* content
{:toc}

---

 # Springboot之自动配置

在前篇文章中我们知道了springboot是通过`BeanFactoryPostProcessor`来解析配置类将我们定义的
对象解析成spring中的`BeanDefinition` 对象放入容器当中的。
我们知道springboot的自动配置类打开一般都是这样类似的结构
```java
/**
 * {@link EnableAutoConfiguration Auto-configuration} for {@link DataSource}.
 *
 * @author Dave Syer
 * @author Phillip Webb
 * @author Stephane Nicoll
 */
@Configuration
@ConditionalOnClass({ DataSource.class, EmbeddedDatabaseType.class })
@EnableConfigurationProperties(DataSourceProperties.class)
@Import({ Registrar.class, DataSourcePoolMetadataProvidersConfiguration.class })
public class DataSourceAutoConfiguration {

```

那么它是如何做到根据实际情况为我们的应用注入bean的呢，在探究之前我们先回顾下springboot如何
解析配置类并扫描入容器的。

上篇文章我们知道通过这个类来解析的。
打开`ConfigurationClassPostProcessor`， 进入`processConfigBeanDefinitions`
首先`		String[] candidateNames = registry.getBeanDefinitionNames();(第一次进入的时候只有spring自己加的对象和启动类)
`拿到之前扫描到容器内的所有的beandefinition对象(这些对象是通过启动类的包扫描进入的)
，遍历判断是不是configuration对象。
如何判断？

看下面的代码
```
public static boolean isFullConfigurationCandidate(AnnotationMetadata metadata) {
		return metadata.isAnnotated(Configuration.class.getName());
	}
```
```
public static boolean isLiteConfigurationCandidate(AnnotationMetadata metadata) {
  // Do not consider an interface or an annotation...
  if (metadata.isInterface()) {
    return false;
  }

  // Any of the typical annotations found?
  for (String indicator : candidateIndicators) {
    if (metadata.isAnnotated(indicator)) {
      return true;
    }
  }

  // Finally, let's look for @Bean methods...
  try {
    return metadata.hasAnnotatedMethods(Bean.class.getName());
  }
  catch (Throwable ex) {
    if (logger.isDebugEnabled()) {
      logger.debug("Failed to introspect @Bean methods on class [" + metadata.getClassName() + "]: " + ex);
    }
    return false;
  }
}
```
```
private static final Set<String> candidateIndicators = new HashSet<String>(4);

	static {
		candidateIndicators.add(Component.class.getName());
		candidateIndicators.add(ComponentScan.class.getName());
		candidateIndicators.add(Import.class.getName());
		candidateIndicators.add(ImportResource.class.getName());
	}
```

拿到候选的`configCandidates`之后进入到`parse` 方法；
```java

	public void parse(Set<BeanDefinitionHolder> configCandidates) {
		this.deferredImportSelectors = new LinkedList<DeferredImportSelectorHolder>();

		for (BeanDefinitionHolder holder : configCandidates) {
			BeanDefinition bd = holder.getBeanDefinition();
			try {
				if (bd instanceof AnnotatedBeanDefinition) {
					parse(((AnnotatedBeanDefinition) bd).getMetadata(), holder.getBeanName());
				}
				else if (bd instanceof AbstractBeanDefinition && ((AbstractBeanDefinition) bd).hasBeanClass()) {
					parse(((AbstractBeanDefinition) bd).getBeanClass(), holder.getBeanName());
				}
				else {
					parse(bd.getBeanClassName(), holder.getBeanName());
				}
			}
			catch (BeanDefinitionStoreException ex) {
				throw ex;
			}
			catch (Throwable ex) {
				throw new BeanDefinitionStoreException(
						"Failed to parse configuration class [" + bd.getBeanClassName() + "]", ex);
			}
		}

		processDeferredImportSelectors();
	}

```

上面的就是循环遍历配置类，对每个进行parse， 我们跟进去看下parse啥
```java
protected void processConfigurationClass(ConfigurationClass configClass) throws IOException {
  if (this.conditionEvaluator.shouldSkip(configClass.getMetadata(), ConfigurationPhase.PARSE_CONFIGURATION)) {
    return;
  }

  ConfigurationClass existingClass = this.configurationClasses.get(configClass);
  if (existingClass != null) {
    if (configClass.isImported()) {
      if (existingClass.isImported()) {
        existingClass.mergeImportedBy(configClass);
      }
      // Otherwise ignore new imported config class; existing non-imported class overrides it.
      return;
    }
    else {
      // Explicit bean definition found, probably replacing an import.
      // Let's remove the old one and go with the new one.
      this.configurationClasses.remove(configClass);
      for (Iterator<ConfigurationClass> it = this.knownSuperclasses.values().iterator(); it.hasNext(); ) {
        if (configClass.equals(it.next())) {
          it.remove();
        }
      }
    }
  }

  // Recursively process the configuration class and its superclass hierarchy.
  SourceClass sourceClass = asSourceClass(configClass);
  do {
    sourceClass = doProcessConfigurationClass(configClass, sourceClass);
  }
  while (sourceClass != null);

  this.configurationClasses.put(configClass, configClass);
}
```

*Note：*  这个方法的第一行`shouldSkip`则判断了是否需要对这个配置类进行解析,
也就是说这个配置类是否`生效！`


第一次进入的时候只有一个`application` 对象，然后解析`ComponentScan`将扫描的bean再加入
到容器中，在注册过程中判断是否需要跳过(是否有conditional注解--解析类的注解放入map，判断map中是否有conditional注解。。 类的注解属性不是java注解包开头的，)。
在解析启动类的时候再将我们定义的bean扫描进去，再调用开头的方法递归重新解析。。。


在上面的`doProcessConfigurationClass`方法中，我们可以看到sprinbg把配置类中的`bean`方法设置到了
`configClass中`
```java
// Process individual @Bean methods
  Set<MethodMetadata> beanMethods = sourceClass.getMetadata().getAnnotatedMethods(Bean.class.getName());
  for (MethodMetadata methodMetadata : beanMethods) {
    configClass.addBeanMethod(new BeanMethod(methodMetadata, configClass));
  }

```

    // Process any @Import annotations
		processImports(configClass, sourceClass, getImports(sourceClass), true);
    将每个配置类上`@Import注解的class解析`， 如果是DeferredImportSelector 保存在类属性中后续使用
    如果是`ImportBeanDefinitionRegistrar` 则实例化并执行方法


在将所有的`Configuration`都解析完成之后，还有一步就是调用刚刚保存的所有的`DeffredImportSelector`
`EnableAutoConfigurationImportSelector`
*这个类就是实现自动配置的关键了！*

它将`META-INF/spring.factories`文件中配置的所有的类都查出来，
然后再重回解析configuration的过程当中。。
上面说过在解析配置的时候会先判断需不需要跳过这个类:

```java
if (this.conditionEvaluator.shouldSkip(configClass.getMetadata(), ConfigurationPhase.PARSE_CONFIGURATION)) {
			return;
		}
```
*如果这个配置类跳过，则里面定义的bean都不会加入到容器当中去！*
下面我们看下这个判断是否跳过这个配置类的！
```java

	/**
	 * Determine if an item should be skipped based on {@code @Conditional} annotations.
	 * @param metadata the meta data
	 * @param phase the phase of the call
	 * @return if the item should be skipped
	 */
	public boolean shouldSkip(AnnotatedTypeMetadata metadata, ConfigurationPhase phase) {
		if (metadata == null || !metadata.isAnnotated(Conditional.class.getName())) {
			return false;
		}

		if (phase == null) {
			if (metadata instanceof AnnotationMetadata &&
					ConfigurationClassUtils.isConfigurationCandidate((AnnotationMetadata) metadata)) {
				return shouldSkip(metadata, ConfigurationPhase.PARSE_CONFIGURATION);
			}
			return shouldSkip(metadata, ConfigurationPhase.REGISTER_BEAN);
		}

		List<Condition> conditions = new ArrayList<Condition>();
		for (String[] conditionClasses : getConditionClasses(metadata)) {
			for (String conditionClass : conditionClasses) {
				Condition condition = getCondition(conditionClass, this.context.getClassLoader());
				conditions.add(condition);
			}
		}

		AnnotationAwareOrderComparator.sort(conditions);

		for (Condition condition : conditions) {
			ConfigurationPhase requiredPhase = null;
			if (condition instanceof ConfigurationCondition) {
				requiredPhase = ((ConfigurationCondition) condition).getConfigurationPhase();
			}

			if (requiredPhase == null || requiredPhase == phase) {
				if (!condition.matches(this.context, metadata)) {
					return true;
				}
			}
		}

		return false;
	}

```

上面的代码主要做了两件事情，首先找到这个配置类的所有condition条件。
我以`JerseyAutoConfiguration` 为例，  这边的conditions就是`OnBeanCondition,OnWebApplicationCondition, OnClassCondition` 三个

接下来第二件事情就是循环调用condition的`match`方法。
以`OnClassCondition`为例说一下：
```java
@Override
	public final boolean matches(ConditionContext context,
			AnnotatedTypeMetadata metadata) {
		String classOrMethodName = getClassOrMethodName(metadata);
		try {
			ConditionOutcome outcome = getMatchOutcome(context, metadata);
			logOutcome(classOrMethodName, outcome);
			recordEvaluation(context, classOrMethodName, outcome);
			return outcome.isMatch();
		}
		catch (NoClassDefFoundError ex) {
			throw new IllegalStateException(
					"Could not evaluate condition on " + classOrMethodName + " due to "
							+ ex.getMessage() + " not "
							+ "found. Make sure your own configuration does not rely on "
							+ "that class. This can also happen if you are "
							+ "@ComponentScanning a springframework package (e.g. if you "
							+ "put a @ComponentScan in the default package by mistake)",
					ex);
		}
		catch (RuntimeException ex) {
			throw new IllegalStateException(
					"Error processing condition on " + getName(metadata), ex);
		}
	}
  ```
首先进入`match`方法，第一行如果是class获取className，否则就是说class#methodName
接下来就进入`getMatchOutcome`的实现,

```java
@Override
public ConditionOutcome getMatchOutcome(ConditionContext context,
    AnnotatedTypeMetadata metadata) {
  ConditionMessage matchMessage = ConditionMessage.empty();
  获取ConditionalOnClass注解的值，需要在类路径存在的类
  MultiValueMap<String, Object> onClasses = getAttributes(metadata,
      ConditionalOnClass.class);
  if (onClasses != null) {
    List<String> missing = getMatchingClasses(onClasses, MatchType.MISSING,
        context);
    if (!missing.isEmpty()) {
      return ConditionOutcome
          .noMatch(ConditionMessage.forCondition(ConditionalOnClass.class)
              .didNotFind("required class", "required classes")
              .items(Style.QUOTE, missing));
    }
    matchMessage = matchMessage.andCondition(ConditionalOnClass.class)
        .found("required class", "required classes").items(Style.QUOTE,
            getMatchingClasses(onClasses, MatchType.PRESENT, context));
  }
  MultiValueMap<String, Object> onMissingClasses = getAttributes(metadata,
      ConditionalOnMissingClass.class);
  if (onMissingClasses != null) {
    里面会通过classloader去查找这个类存在不存在classpath
    List<String> present = getMatchingClasses(onMissingClasses, MatchType.PRESENT,
        context);
    if (!present.isEmpty()) {
      return ConditionOutcome.noMatch(
          ConditionMessage.forCondition(ConditionalOnMissingClass.class)
              .found("unwanted class", "unwanted classes")
              .items(Style.QUOTE, present));
    }
    matchMessage = matchMessage.andCondition(ConditionalOnMissingClass.class)
        .didNotFind("unwanted class", "unwanted classes")
        .items(Style.QUOTE, getMatchingClasses(onMissingClasses,
            MatchType.MISSING, context));
  }
  return ConditionOutcome.match(matchMessage);
}

```

这样就能判断这个类是否符合条件了，回过头来spring的整体逻辑还是很清晰的。

到现在我们是看到了自动配置类，并将符合条件的配置类解析然后存到容器当中，但是我们知道在`Configuration`
类中配置`Bean` 的时候会有各种条件觉得是否将这个对象配置进去，那么这个又是在哪里处理的呢？

接着往下看， 猜测一下应该是在解析配置类，找到带`@Bean` 注解的方法的时候进行判断的。

我们看下`ConfigurationClassParser` 有一个成员变量`private final Map<ConfigurationClass, ConfigurationClass> configurationClasses =
			new LinkedHashMap<ConfigurationClass, ConfigurationClass>();
`
在上面的调用`doProcessConfigurationClass` 最后一行我们会发现
`  this.configurationClasses.put(configClass, configClass);
` 在一个配置类解析完成之后会把它加入到这个map中， 其中configClass保存有bean methods的信息.

让我们把视角回到刚开始的地方， 我们知道刚刚所有的一系列解析都由这个parse进入

```java
do {
  这边将所有的candidate都解析完成，包括自己定义的bean和自动配置的configuration
  parser.parse(candidates);
  parser.validate();

 这里讲所有配置类的拿出来，里面包含了所有的Bean方法
  Set<ConfigurationClass> configClasses = new LinkedHashSet<ConfigurationClass>(parser.getConfigurationClasses());
  configClasses.removeAll(alreadyParsed);

  // Read the model and create bean definitions based on its content
  if (this.reader == null) {
    this.reader = new ConfigurationClassBeanDefinitionReader(
        registry, this.sourceExtractor, this.resourceLoader, this.environment,
        this.importBeanNameGenerator, parser.getImportRegistry());
  }
   将bean存入容器， 里面会将@Bean方法，
  this.reader.loadBeanDefinitions(configClasses);
```

看下面
```java
/**
 * Read a particular {@link ConfigurationClass}, registering bean definitions
 * for the class itself and all of its {@link Bean} methods.
 */
private void loadBeanDefinitionsForConfigurationClass(ConfigurationClass configClass,
    TrackedConditionEvaluator trackedConditionEvaluator) {

  if (trackedConditionEvaluator.shouldSkip(configClass)) {
    String beanName = configClass.getBeanName();
    if (StringUtils.hasLength(beanName) && this.registry.containsBeanDefinition(beanName)) {
      this.registry.removeBeanDefinition(beanName);
    }
    this.importRegistry.removeImportingClassFor(configClass.getMetadata().getClassName());
    return;
  }

  if (configClass.isImported()) {
    registerBeanDefinitionForImportedConfigurationClass(configClass);
  }
  将配置类的bean方法注册到容器
  for (BeanMethod beanMethod : configClass.getBeanMethods()) {
    loadBeanDefinitionsForBeanMethod(beanMethod);
  }
  ImportResource
  loadBeanDefinitionsFromImportedResources(configClass.getImportedResources());
  ImportBeanDefinitionRegistrar放入
  loadBeanDefinitionsFromRegistrars(configClass.getImportBeanDefinitionRegistrars());
}
```

到这里springboot的解析入库应该算是基本完成了。
总结一下 ：
SpringBoot首先找到我们的启动类，解析类上的注解如ComponentScan, ImportResource注解然后找到这批
bean，再将这批beandefinition循序遍历，判断是否是配置类，再递归解析，将自自身自身添加到容器中去，最后再将每个配置类的bean方法,import, importResource注册到容器。

所欲在springboot启动过程中，`AbstractApplicationContext`的`	// Invoke factory processors registered as beans in the context.
				invokeBeanFactoryPostProcessors(beanFactory);` 所有的解析工作都已经完成，应用程序的bean和自动配置的bean都已经注册到容器当中了。

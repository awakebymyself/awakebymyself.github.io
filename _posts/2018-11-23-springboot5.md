---
layout: post
title:  "springboot源码解读(五)IOC"
categories: springboot源码解读
tags:  tech
author: Lzg
---

* content
{:toc}

---

# Springboot之IOC


bean初始化过程:

入口：
首先获取所有的`beanDefinitionNames`， 然后循环调用`getBean`
`AbstractBeanFactory doGetBean`， 具体代码就不贴了，跟进代码debug查看怎么实现的IOC


1. bean实例化之前调用`InstantiationAwareBeanPostProcessor的 postProcessBeforeInstantiation`,如果返回!= null（?直接生成代理？） 则调用`beanPostProcessor的postProcessAfterInitialization`

2. 下面会去找这个bean的构造器,通过`SmartInstantiationAwareBeanPostProcessor的determineCandidateConstructors`方法（see AutowiredAnnotationBeanPostProcessor）。
如果构造函数有参数的话，会通过这个构造器初始化里面的对象，完成依赖注入,否则就用默认的无参构造器初始化。
`AbstractAutowireCapableBeanFactory`

寻找后续的构造器会去找这个类的又参数的构造函数，如果没有则返回Null

```java
// Need to determine the constructor...
		Constructor<?>[] ctors = determineConstructorsFromBeanPostProcessors(beanClass, beanName);
		if (ctors != null ||
				mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_CONSTRUCTOR ||
				mbd.hasConstructorArgumentValues() || !ObjectUtils.isEmpty(args))  {
			return autowireConstructor(beanName, mbd, ctors, args);
		}
```
3.找到了构造器之后通过`autowireConstructor` 方法返回instance,bean实例完之后调用`MergedBeanDefinitionPostProcessor`, 到这一步才只是将Bean实例处理，还没有设置属性(如果是指通过自动注入,构造器注入的话引用已经设置完成)

```java
if (instanceWrapper == null) {
			instanceWrapper = createBeanInstance(beanName, mbd, args);
		}
		final Object bean = (instanceWrapper != null ? instanceWrapper.getWrappedInstance() : null);
		Class<?> beanType = (instanceWrapper != null ? instanceWrapper.getWrappedClass() : null);

		// Allow post-processors to modify the merged bean definition.
		synchronized (mbd.postProcessingLock) {
			if (!mbd.postProcessed) {
				applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
				mbd.postProcessed = true;
			}
		}
```


4.如果通过autowire设置依赖，找到`InstantiationAwareBeanPostProcessor`接口实现, 具体通过`AutowiredAnnotationBeanPostProcessor` beanPostProcessor实现， 本质还是找到需要的bean,再去容器中`getBean` 属性注入完成之后调用初始化方法

```java
protected void populateBean(String beanName, RootBeanDefinition mbd, BeanWrapper bw) {
	    ...省略
		if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
			for (BeanPostProcessor bp : getBeanPostProcessors()) {
				if (bp instanceof InstantiationAwareBeanPostProcessor) {
					InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
					if (!ibp.postProcessAfterInstantiation(bw.getWrappedInstance(), beanName)) {
						continueWithPropertyPopulation = false;
						break;
					}
				}
			}
		}
```


5.等实例初始化完成（依赖注入完成）之后 先设置bean的一系列`aware`接口

  ---> （不是合成的,非系统自己的）`beanPostProcessor`的`postProcessBeforeInitialization`

 ---> 调用`init方法, 如InitianlingBean接口`

 ---> 最后再调用`beanPostProcessor的 postProcessAfterInitialization方法。

到这一步对象的依赖注入基本是完成了。

在所有对象都完成之后，再调用其中实现了接口`SmartInitializingSingleton`的bean，执行方法。

---

### 怎么解决循环依赖？

*需要指出的是如果通过构造器注入的话，循环依赖解决不了，如果通过field自动注入的话则是可以解决的*


`DefaultListableBeanFactory`有一个成员变量`
```java
/** Names of beans that are currently in creation */
	private final Set<String> singletonsCurrentlyInCreation =
			Collections.newSetFromMap(new ConcurrentHashMap<String, Boolean>(16));`
```

在进行对象实例化之前,会将这个bean通过`beanName`设置到这个`Set`当中。

举个例子,对象A->B, 对象B-<A， 他们是通过构造器注入的，这是一个无法解决的循环依赖。

为什么？

从上文分析构造器注入的时候我们可以得知当去实例化A的时候，解析出有参数的构造器，参数是B,
再去过去B, 解析B的构造器参数再去初始化A， 这时候在获取A就会报错，在代码路径`getSingleton`
中实例化之前有一个判断
```java
protected void beforeSingletonCreation(String beanName) {
		if (!this.inCreationCheckExclusions.contains(beanName) && !this.singletonsCurrentlyInCreation.add(beanName)) {
			throw new BeanCurrentlyInCreationException(beanName);
		}
	}
```
那么这里会抛出异常,因为在最开始初始化的时候已经将A先添加进去了。


*那么为什么属性注入是可以的呢？*

回到最开始AbstractBeanFactory的`doGetBean`方法开头有如下判断

```java
final String beanName = transformedBeanName(name);
  Object bean;

  // Eagerly check singleton cache for manually registered singletons.
  Object sharedInstance = getSingleton(beanName);
  if (sharedInstance != null && args == null) {
    if (logger.isDebugEnabled()) {
      if (isSingletonCurrentlyInCreation(beanName)) {
        logger.debug("Returning eagerly cached instance of singleton bean '" + beanName +
            "' that is not fully initialized yet - a consequence of a circular reference");
      }
      else {
        logger.debug("Returning cached instance of singleton bean '" + beanName + "'");
      }
    }
    bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
  }
  ....
```

点进去`getSingleton`方法:

这里的参数boolean默认会为true

通过属性注入的时候会发现我们通过singletonFactory会直接返回这个bean的earlyReference
因为我们之前已经将beanName:A 和 ObjectFactory设置到`singletonFactories`里去了，所有这里直接返回A
```java
protected Object getSingleton(String beanName, boolean allowEarlyReference) {
		Object singletonObject = this.singletonObjects.get(beanName);
		if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
			synchronized (this.singletonObjects) {
				singletonObject = this.earlySingletonObjects.get(beanName);
				if (singletonObject == null && allowEarlyReference) {
					ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
					if (singletonFactory != null) {
						singletonObject = singletonFactory.getObject();
						this.earlySingletonObjects.put(beanName, singletonObject);
						this.singletonFactories.remove(beanName);
					}
				}
			}
		}
		return (singletonObject != NULL_OBJECT ? singletonObject : null);
	}

```

  那么问题来了，为什么通过构造器的注入的时候不行呢？
  因为在创建bean的代码路径上是这样的:

 1. `createBeanInstance`先创建bean实例

 2. 再给bean设置一个`allowEarlyReferenceBeanFactory`
 ```java
 addSingletonFactory(beanName, new ObjectFactory<Object>() {
				@Override
				public Object getObject() throws BeansException {
					return getEarlyBeanReference(beanName, mbd, bean);
				}
			});
```
3. 再去初始化bean的属性`populateBean`

如果我们通过构造器注入，那么在第一步的时候就已经去通过构造器参数再去容器中获取B, 这时还没有将A添加到
这个factoryMap中，所有上文的的getSingleton就会返回Null, 然后走到后头就会报错。

如果是通过属性注入则情况刚刚相反，我们会通过注册的`factoryBean`获取`A`的earlyReference,完成`B`的
初始化，再完成A自身的初始化。

疑问解决了，那么我们看一下怎么实现earlyReference的？
其实还是通过`SmartInstantiationAwareBeanPostProcessor`的`getEarlyBeanReference`

`	Object getEarlyBeanReference(Object bean, String beanName) throws BeansException;`

里面的实现都是直接返回这个`bean`。

综上这就是spring如何解决循环依赖的问题。



### 如何创建单例和protptype的bean

  如果是单例走`getSingleton`方法

获取单例的时候会从类成员中先回去，有则返回，没有就调用`createBean`生成-再缓存返回
```java
/** Cache of singleton objects: bean name --> bean instance */
private final Map<String, Object> singletonObjects = new ConcurrentHashMap<String, Object>(256);
```

如果是原型的
`则直接调用createBean` 生成新的


### 怎么注入环境变量的


和属性自动注入一样，拿到所有依赖的bean，去容器中寻找。
在`DefaultListableBeanFactory`的`doResolveDependency`中
有如下代码

```java
Object value = getAutowireCandidateResolver().getSuggestedValue(descriptor);
			if (value != null) {
				if (value instanceof String) {
					String strVal = resolveEmbeddedValue((String) value);
					BeanDefinition bd = (beanName != null && containsBean(beanName) ? getMergedBeanDefinition(beanName) : null);
					value = evaluateBeanDefinitionString(strVal, bd);
				}
				TypeConverter converter = (typeConverter != null ? typeConverter : getTypeConverter());
				return (descriptor.getField() != null ?
						converter.convertIfNecessary(value, type, descriptor.getField()) :
						converter.convertIfNecessary(value, type, descriptor.getMethodParameter()));
			}
```

底层会通过之前设置的`StringResolver`去寻找对应的值，然后设置进去。

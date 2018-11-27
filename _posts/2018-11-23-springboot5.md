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
```java
// Need to determine the constructor...
		Constructor<?>[] ctors = determineConstructorsFromBeanPostProcessors(beanClass, beanName);
		if (ctors != null ||
				mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_CONSTRUCTOR ||
				mbd.hasConstructorArgumentValues() || !ObjectUtils.isEmpty(args))  {
			return autowireConstructor(beanName, mbd, ctors, args);
		}
    ```

3. bean实例完之后调用`MergedBeanDefinitionPostProcessor`
,到这一步才只是将Bean实例处理，还没有设置属性(如果是指通过自动注入,构造器注入的话引用已经设置完成)
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

4. 如果通过autowire设置依赖，找到`InstantiationAwareBeanPostProcessor`接口实现, 具体通过`AutowiredAnnotationBeanPostProcessor` beanPostProcessor实现， 本质还是找到需要的bean,再去容器中`getBean`
属性注入完成之后调用初始化方法
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


5. 等实例初始化完成（依赖注入完成）之后 先设置bean的一系列`aware`接口

  ---> （不是合成的,非系统自己的）`beanPostProcessor`的`postProcessBeforeInitialization`

 ---> 调用`init方法, 如InitianlingBean接口`

 ---> 最后再调用`beanPostProcessor的 postProcessAfterInitialization方法。

到这一步对象的依赖注入基本是完成了。

在所有对象都完成之后，再调用其中实现了接口`SmartInitializingSingleton`的bean，执行方法。

 ---

 ### 怎么解决循环依赖？
`DefaultListableBeanFactory`有一个成员变量`
```java
/** Names of beans that are currently in creation */
	private final Set<String> singletonsCurrentlyInCreation =
			Collections.newSetFromMap(new ConcurrentHashMap<String, Boolean>(16));`
```

在进行对象实例化之前,会将这个bean通过`beanName`设置到这个`Set`当中。

后续在实例化对象之后，有如下一段代码:




```java
// Eagerly cache singletons to be able to resolve circular references
// even when triggered by lifecycle interfaces like BeanFactoryAware.
boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
    isSingletonCurrentlyInCreation(beanName));
if (earlySingletonExposure) {
  if (logger.isDebugEnabled()) {
    logger.debug("Eagerly caching bean '" + beanName +
        "' to allow for resolving potential circular references");
  }
  addSingletonFactory(beanName, new ObjectFactory<Object>() {
    @Override
    public Object getObject() throws BeansException {
      return getEarlyBeanReference(beanName, mbd, bean);
    }
  });
}
```

首先我们会进入这个判断语句，这里可以看到设置了一个`ObjectFactory`， 并保存在成员变量中
```java
protected void addSingletonFactory(String beanName, ObjectFactory<?> singletonFactory) {
		Assert.notNull(singletonFactory, "Singleton factory must not be null");
		synchronized (this.singletonObjects) {
			if (!this.singletonObjects.containsKey(beanName)) {
				this.singletonFactories.put(beanName, singletonFactory);
				this.earlySingletonObjects.remove(beanName);
				this.registeredSingletons.add(beanName);
			}
		}
	}
  ```

`ObjectFactory`的实现是这样的：
```java
new ObjectFactory<Object>() {
       @Override
       public Object getObject() throws BeansException {
         return getEarlyBeanReference(beanName, mbd, bean);
       }
     });

     /**
   	 * Obtain a reference for early access to the specified bean,
   	 * typically for the purpose of resolving a circular reference.
   	 * @param beanName the name of the bean (for error handling purposes)
   	 * @param mbd the merged bean definition for the bean
   	 * @param bean the raw bean instance
   	 * @return the object to expose as bean reference
   	 */
   	protected Object getEarlyBeanReference(String beanName, RootBeanDefinition mbd, Object bean) {
   		Object exposedObject = bean;
   		if (bean != null && !mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
   			for (BeanPostProcessor bp : getBeanPostProcessors()) {
   				if (bp instanceof SmartInstantiationAwareBeanPostProcessor) {
   					SmartInstantiationAwareBeanPostProcessor ibp = (SmartInstantiationAwareBeanPostProcessor) bp;
   					exposedObject = ibp.getEarlyBeanReference(exposedObject, beanName);
   					if (exposedObject == null) {
   						return null;
   					}
   				}
   			}
   		}
   		return exposedObject;
   	}
```

---
layout: post
title:  "springboot源码解读(六)AOP"
categories: springboot源码解读
tags:  tech
author: Lzg
---

* content
{:toc}

---

# Springboot之AOP

关于springAop怎么使用这里就不聊了，这篇关注的是spring如何实现的。
我们可能都知道是通过代理去实现各种非业务相关的代码的织入，那么是怎么处理的呢？

我们知道在spring的生命周期中当一个bean初始化完成之后会对他进行一些配置，例如调用init方法等等, 上午有分析到，奥秘就在其中。 在调用`BeanPostPrecessor` 的 `postProcessAfterInitialization` 方法判断当前的bean 是否符合pointCut的条件，符合就返回一个代理对象，对方法拦截。

具体到代码层面就是通过`AbstractAutoProxyCreator` 的子类 `AnnotationAwareAspectJAutoProxyCreator`

下面先贴代码，再说明过程:

```java
/**
	 * Create a proxy with the configured interceptors if the bean is
	 * identified as one to proxy by the subclass.
	 * @see #getAdvicesAndAdvisorsForBean
	 */
	@Override
	public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
		if (bean != null) {
			Object cacheKey = getCacheKey(bean.getClass(), beanName);
			if (!this.earlyProxyReferences.contains(cacheKey)) {
				return wrapIfNecessary(bean, beanName, cacheKey);
			}
		}
		return bean;
	}
```

从`wrapIfNecessary`返回的bean就是被增强了

```java
protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
		if (beanName != null && this.targetSourcedBeans.contains(beanName)) {
			return bean;
		}
		if (Boolean.FALSE.equals(this.advisedBeans.get(cacheKey))) {
			return bean;
		}
		if (isInfrastructureClass(bean.getClass()) || shouldSkip(bean.getClass(), beanName)) {
			this.advisedBeans.put(cacheKey, Boolean.FALSE);
			return bean;
		}

		// Create proxy if we have advice.
		Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);
		if (specificInterceptors != DO_NOT_PROXY) {
			this.advisedBeans.put(cacheKey, Boolean.TRUE);
			Object proxy = createProxy(
					bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
			this.proxyTypes.put(cacheKey, proxy.getClass());
			return proxy;
		}

		this.advisedBeans.put(cacheKey, Boolean.FALSE);
		return bean;
	}
```

上面的过程就做了两件事情，首先是判断这个bean是否符合条件，也就是匹配advisor，匹配的话下
会去首先找出我们的`advisor`， 不管是自己定义的亦或者`@Aspect`标注的类里面的都会找出来。

第二步则是去创建代理对象了，下面我们点进去瞧一瞧。

```java
protected Object createProxy(
			Class<?> beanClass, String beanName, Object[] specificInterceptors, TargetSource targetSource) {

		if (this.beanFactory instanceof ConfigurableListableBeanFactory) {
			AutoProxyUtils.exposeTargetClass((ConfigurableListableBeanFactory) this.beanFactory, beanName, beanClass);
		}

		ProxyFactory proxyFactory = new ProxyFactory();
		proxyFactory.copyFrom(this);

		if (!proxyFactory.isProxyTargetClass()) {
			if (shouldProxyTargetClass(beanClass, beanName)) {
				proxyFactory.setProxyTargetClass(true);
			}
			else {
				evaluateProxyInterfaces(beanClass, proxyFactory);
			}
		}

		Advisor[] advisors = buildAdvisors(beanName, specificInterceptors);
		for (Advisor advisor : advisors) {
			proxyFactory.addAdvisor(advisor);
		}

		proxyFactory.setTargetSource(targetSource);
		customizeProxyFactory(proxyFactory);

		proxyFactory.setFrozen(this.freezeProxy);
		if (advisorsPreFiltered()) {
			proxyFactory.setPreFiltered(true);
		}

		return proxyFactory.getProxy(getProxyClassLoader());
	}

```
`new`了一个proxyfactory,将advisor设置进去，设置了要代理的class， 然后获取代理对象。这里是典型的工厂模式。

```java
public Object getProxy(ClassLoader classLoader) {
		return createAopProxy().getProxy(classLoader);
	}
```
先创建了`AopProxy`， 再获取proxy对象.
实际就是`DefaultAopProxyFactory`创建`AopProxy`
```java
@Override
	public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {
		if (config.isOptimize() || config.isProxyTargetClass() || hasNoUserSuppliedProxyInterfaces(config)) {
			Class<?> targetClass = config.getTargetClass();
			if (targetClass == null) {
				throw new AopConfigException("TargetSource cannot determine target class: " +
						"Either an interface or a target is required for proxy creation.");
			}
			if (targetClass.isInterface() || Proxy.isProxyClass(targetClass)) {
				return new JdkDynamicAopProxy(config);
			}
			return new ObjenesisCglibAopProxy(config);
		}
		else {
			return new JdkDynamicAopProxy(config);
		}
	}
```
可以看出如果要代理的类不是接口，则使用cglib，否则使用jdk。 这里我们主要关注一下`cglib`。

下面就是最后一步生成`getProxy`的步骤了，先贴代码
```java
@Override
public Object getProxy(ClassLoader classLoader) {
	if (logger.isDebugEnabled()) {
		logger.debug("Creating CGLIB proxy: target source is " + this.advised.getTargetSource());
	}

	try {
		Class<?> rootClass = this.advised.getTargetClass();
		Assert.state(rootClass != null, "Target class must be available for creating a CGLIB proxy");

		Class<?> proxySuperClass = rootClass;
		if (ClassUtils.isCglibProxyClass(rootClass)) {
			proxySuperClass = rootClass.getSuperclass();
			Class<?>[] additionalInterfaces = rootClass.getInterfaces();
			for (Class<?> additionalInterface : additionalInterfaces) {
				this.advised.addInterface(additionalInterface);
			}
		}

		// Validate the class, writing log messages as necessary.
		validateClassIfNecessary(proxySuperClass, classLoader);

		// Configure CGLIB Enhancer...
		Enhancer enhancer = createEnhancer();
		if (classLoader != null) {
			enhancer.setClassLoader(classLoader);
			if (classLoader instanceof SmartClassLoader &&
					((SmartClassLoader) classLoader).isClassReloadable(proxySuperClass)) {
				enhancer.setUseCache(false);
			}
		}
		enhancer.setSuperclass(proxySuperClass);
		enhancer.setInterfaces(AopProxyUtils.completeProxiedInterfaces(this.advised));
		enhancer.setNamingPolicy(SpringNamingPolicy.INSTANCE);
		enhancer.setStrategy(new ClassLoaderAwareUndeclaredThrowableStrategy(classLoader));

		Callback[] callbacks = getCallbacks(rootClass);
		Class<?>[] types = new Class<?>[callbacks.length];
		for (int x = 0; x < types.length; x++) {
			types[x] = callbacks[x].getClass();
		}
		// fixedInterceptorMap only populated at this point, after getCallbacks call above
		enhancer.setCallbackFilter(new ProxyCallbackFilter(
				this.advised.getConfigurationOnlyCopy(), this.fixedInterceptorMap, this.fixedInterceptorOffset));
		enhancer.setCallbackTypes(types);

		// Generate the proxy class and create a proxy instance.
		return createProxyClassAndInstance(enhancer, callbacks);
	}
	catch (CodeGenerationException ex) {
		throw new AopConfigException("Could not generate CGLIB subclass of class [" +
				this.advised.getTargetClass() + "]: " +
				"Common causes of this problem include using a final class or a non-visible class",
				ex);
	}
	catch (IllegalArgumentException ex) {
		throw new AopConfigException("Could not generate CGLIB subclass of class [" +
				this.advised.getTargetClass() + "]: " +
				"Common causes of this problem include using a final class or a non-visible class",
				ex);
	}
	catch (Exception ex) {
		// TargetSource.getTarget() failed
		throw new AopConfigException("Unexpected AOP exception", ex);
	}
}
```
关于如何使用`cglib`这里不细聊了，主要就是要关注他设置的一些回调函数。

在`getCallbacks`方法里 有如下代码
```java
Callback[] mainCallbacks = new Callback[] {
			aopInterceptor,  // for normal advice
			targetInterceptor,  // invoke target without considering advice, if optimized
			new SerializableNoOp(),  // no override for methods mapped to this
			targetDispatcher, this.advisedDispatcher,
			new EqualsInterceptor(this.advised),
			new HashCodeInterceptor(this.advised)
	};
```
其中`aopInterceptor`就包含了我们程序中配置的`advisors`。到这一步完成之后一个代理对象就这么完整的生成了。在执行应用代码的时候就会先被拦截进入切面。

我们看下`aopInterceptor` 的`inteceptor`方法
```java
@Override
	public Object intercept(Object proxy, Method method, Object[] args, MethodProxy methodProxy) throws Throwable {
		Object oldProxy = null;
		boolean setProxyContext = false;
		Class<?> targetClass = null;
		Object target = null;
		try {
			if (this.advised.exposeProxy) {
				// Make invocation available if necessary.
				oldProxy = AopContext.setCurrentProxy(proxy);
				setProxyContext = true;
			}
			// May be null. Get as late as possible to minimize the time we
			// "own" the target, in case it comes from a pool...
			target = getTarget();
			if (target != null) {
				targetClass = target.getClass();
			}
			List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);
			Object retVal;
			// Check whether we only have one InvokerInterceptor: that is,
			// no real advice, but just reflective invocation of the target.
			if (chain.isEmpty() && Modifier.isPublic(method.getModifiers())) {
				// We can skip creating a MethodInvocation: just invoke the target directly.
				// Note that the final invoker must be an InvokerInterceptor, so we know
				// it does nothing but a reflective operation on the target, and no hot
				// swapping or fancy proxying.
				Object[] argsToUse = AopProxyUtils.adaptArgumentsIfNecessary(method, args);
				retVal = methodProxy.invoke(target, argsToUse);
			}
			else {
				// We need to create a method invocation...
				retVal = new CglibMethodInvocation(proxy, target, method, args, targetClass, chain, methodProxy).proceed();
			}
			retVal = processReturnType(proxy, target, method, retVal);
			return retVal;
		}
		finally {
			if (target != null) {
				releaseTarget(target);
			}
			if (setProxyContext) {
				// Restore old proxy.
				AopContext.setCurrentProxy(oldProxy);
			}
		}
	}

```
这里面最关键的一步就是`retVal = new CglibMethodInvocation(proxy, target, method, args, targetClass, chain, methodProxy).proceed();`

这个`procceed`最终会对调我们定义的`advise`方法(具体执行不细聊了，有兴趣可以自己debug看下)，执行切面代码，最后再调用业务代码，然后返回结果。


综上所述这就是spring Aop的所有的魔法，可能说的比较粗浅，不过核心的思想应该是很清楚了，包括实现的逻辑
和过程。 这对于我们更好的使用aop会提供很大的帮助。

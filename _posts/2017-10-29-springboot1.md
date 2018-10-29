---
layout: post
title:  "springboot源码解读(一)启动"
categories: springboot源码解读
tags:  tech
author: Lzg
---

* * *

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

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



bean实例化之前调用`InstantiationAwareBeanPostProcessor`,如果返回!= null 应用postProcess后置处理

否则去创建这个bean,实例完之后调用`MergedBeanDefinitionPostProcessor`

add singleFactory `SmartInstantiationAwareBeanPostProcessor` `getEarlyBeanReference` 解决递归引用

到这一步才只是将Bean实例处理，还没有设置属性

属性注入完成之后调用初始化方法

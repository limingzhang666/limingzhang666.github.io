---
title: Spring源码解析-07-关于Aware接口
copyright: true
comments: true
related_posts: true
date: 2021-04-16 22:07:00
tags: Aware接口
categories: Spring源码
---



在Bean对象的生命周期的方法中有好几个接口是Aware接口的子接口，所以弄清楚Aware接口对于理解Spring框架还是很有帮助的。

# 结论：

Aware系列接口，主要用于辅助Spring bean访问Spring容器

# Aware是什么：

```java
/**
 * A marker superinterface indicating（指示） that a bean is eligible（合适的，符合条件的，有资格的） to be notified by the
 Spring container of a particular framework object through a callback-style method.
 The actual method signature is determined(决定) by individual（个体的） subinterfaces but should
 typically consist of just one void-returning method that accepts a single argument.
 *
 * <p>Note that merely（仅仅） implementing {@link Aware} provides no default functionality.
 * Rather, processing must be done explicitly（明确的）, for example in a
 * {@link org.springframework.beans.factory.config.BeanPostProcessor}.
 * Refer to (参考，适用于){@link org.springframework.context.support.ApplicationContextAwareProcessor}
 * for an example of processing specific {@code *Aware} interface callbacks.
 *
 * @author Chris Beams
 * @author Juergen Hoeller
 * @since 3.1
 */
public interface Aware {
	// 可以发现该接口中并没有定义任何方法，所以这是个标识接口。该接口的子接口有如下：
}
```

- Aware的英文意思是“可感知”，它自身是一个空白的接口，但是它有很多实现类，所以它的功能其实就是可以让调用者获取到某些信息，例如：加载当前Bean的容器名，当前Bean在容器中的BeanName，获取一些文本信息和资源文件等等，去获取加载当前Bean的加载器信息，等等等等。
  我们也可以自定义一些XXAware，去获取自己想要的信息。
- Aware接口从字面上翻译过来是感知捕获的含义。单纯的bean（未实现Aware系列接口）是没有知觉的；实现了Aware系列接口的bean可以访问Spring容器。这些Aware系列接口增强了Spring bean的功能，但是也会造成对Spring框架的绑定，增大了与Spring框架的耦合度。（Aware是“意识到的，察觉到的”的意思，实现了Aware系列接口表明：可以意识到、可以察觉到）

## 子接口

![img](https://ask.qcloudimg.com/http-save/yehe-4919348/ztnwz2oobc.png?imageView2/2/w/1620)



## Aware系列接口的共性

1. 都以“Aware”结尾
2. 都是Aware接口的子接口，即都继承了Aware接口
3. 接口内均定义了一个set方法

```java

public interface ResourceLoaderAware extends Aware {
	void setResourceLoader(ResourceLoader resourceLoader);
}
public interface BeanFactoryAware extends Aware {
	void setBeanFactory(BeanFactory beanFactory) throws BeansException;
}
public interface ApplicationContextAware extends Aware {
	void setApplicationContext(ApplicationContext applicationContext) throws BeansException;
}
public interface BeanClassLoaderAware extends Aware {
	void setBeanClassLoader(ClassLoader classLoader);

}

```

- 每个子接口都定义了set方法。而方法中的形参是接口Aware前面的内容，也就是当前Bean需要感知的内容。所以我们需要在Bean中声明相关的成员变量来接收。

## 测试类

```java
package com.yinshi.spring.bean;

import org.springframework.beans.BeansException;
import org.springframework.beans.factory.BeanClassLoaderAware;
import org.springframework.beans.factory.BeanFactory;
import org.springframework.beans.factory.BeanFactoryAware;
import org.springframework.beans.factory.BeanNameAware;
import org.springframework.stereotype.Component;

@Component
public class AwareTestBean implements BeanNameAware, BeanFactoryAware, BeanClassLoaderAware {
	private String beanName;
	private String beanFactory;
	private String classLoader;

	@Override
	public void setBeanClassLoader(ClassLoader classLoader) {
		this.classLoader = classLoader.getName();
	}

	@Override
	public void setBeanFactory(BeanFactory beanFactory) throws BeansException {
		this.beanFactory = beanFactory.toString();
	}

	@Override
	public void setBeanName(String name) {
		this.beanName = name;
	}

	public String getBeanName() {
		return beanName;
	}

	public String getBeanFactory() {
		return beanFactory;
	}

	public String getClassLoader() {
		return classLoader;
	}

	@Override
	public String toString() {
		return "AwareTestBean{" +
				"beanName='" + beanName + '\'' +
				", beanFactory='" + beanFactory + '\'' +
				", classLoader='" + classLoader + '\'' +
				'}';
	}
}


@Test
	public void testAware() {
		AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext(MainConifgOfAutowired.class);
		AwareTestBean bean = applicationContext.getBean(AwareTestBean.class);
		System.out.println(bean.toString());
	}
```

结果：

```java
AwareTestBean{beanName='awareTestBean', beanFactory='org.springframework.beans.factory.support.DefaultListableBeanFactory@5ab956d7: defining beans [org.springframework.context.annotation.internalConfigurationAnnotationProcessor,org.springframework.context.annotation.internalAutowiredAnnotationProcessor,org.springframework.context.annotation.internalCommonAnnotationProcessor,org.springframework.context.event.internalEventListenerProcessor,org.springframework.context.event.internalEventListenerFactory,mainConifgOfAutowired,bookService,bookDao,bookController,awareTestBean,boss,car,cat,dog,myApplicationObjectSupport,myBeanPostProcessor,red,bookDao2,color]; root of factory hierarchy', classLoader='app'}

```



###### 可以看到是成功了。所以这就是Aware告知的作用。当需要获取当前Bean的一些信息，不需要硬编码或者注入的方式去拿，通过定义好的接口，直接获取

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201022183304977.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3doaXRlQmVhckNsaW1i,size_16,color_FFFFFF,t_70#pic_center)


---
title: Spring源码解析-08-关于BeanPostProcessor生命周期
copyright: true
comments: true
related_posts: true
date: 2021-04-16 23:10:46
tags:
  - BeanPostProcessor接口
  - Spring生命周期
categories: Spring源码
---

BeanPostProcessor：后置处理器
spring使用***模板模式***，在bean的创建过程中安插了许多锚点，用户寻找对应的锚点，通过重写方法介入到bean的创建过程当中。本节通过重写这些锚点，学习如何使用BeanPostProcessor、获取各类BeanAware并且理清bean的生命周期

## 创建类LifeCycleBean

```java
package com.yinshi.spring.bean;

import org.springframework.beans.BeansException;
import org.springframework.beans.factory.*;
import org.springframework.context.ApplicationContext;
import org.springframework.context.ApplicationContextAware;

public class LifeCycleBean implements BeanNameAware, BeanFactoryAware, ApplicationContextAware, InitializingBean, DisposableBean {

	private BeanFactory beanFactory;

	private ApplicationContext applicationContext;

	private String name;

	public LifeCycleBean(String name) {
		System.out.println("2. 构造方法被调用，name：" + name);
		this.name = name;
	}

	@Override
	public void setBeanName(String name) {
		System.out.println("5. BeanNameAware被调用, 获取到的beanName：" + name);

	}


	@Override
	public void setBeanFactory(BeanFactory beanFactory) throws BeansException {
		this.beanFactory = beanFactory;
		System.out.println("6. BeanFactoryAware被调用，获取到beanFactory：" + beanFactory);
	}


	@Override
	public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
		this.applicationContext = applicationContext;
		System.out.println("7. ApplicationContextAware被调用，获取到ApplicationContextAware：" + applicationContext);

	}

	@Override
	public void afterPropertiesSet() throws Exception {
		System.out.println("9. afterPropertiesSet被调用");
	}


	public void myInit() {
		System.out.println("10. myInit自定义初始化方法被调用，name：" + getName());
	}

	@Override
	public void destroy() throws Exception {
		System.out.println("13. DisposableBean被调用");
	}

	public void myDestroy() {
		System.out.println("14. destroy-method自定义销毁方法被调用");
	}

	///////////////////////////////////////////////////////////////
	public String getName() {
		return name;
	}

	public void setName(String name) {
		this.name = name;
	}

	public BeanFactory getBeanFactory() {
		return beanFactory;
	}

	public ApplicationContext getApplicationContext() {
		return applicationContext;
	}

	@Override
	public String toString() {
		return "LifeCycleBean{" +
				"beanFactory=" + beanFactory +
				", applicationContext=" + applicationContext +
				", name='" + name + '\'' +
				'}';
	}
}
```



## 创建BeanPostProcessor后置处理器

```java
package com.yinshi.spring.bean;

import org.springframework.beans.BeansException;
import org.springframework.beans.PropertyValues;
import org.springframework.beans.factory.config.InstantiationAwareBeanPostProcessor;

public class LifeCycleBeanPostProcessor implements InstantiationAwareBeanPostProcessor {
	/**
	 * 实例化之前
	 *
	 * @param beanClass the class of the bean to be instantiated
	 * @param beanName  the name of the bean
	 * @return
	 * @throws BeansException
	 */
	@Override
	public Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) throws BeansException {
		if (beanName.equals("lifeCycleBean")) {
			System.out.println("1. postProcessBeforeInstantiation被调用");
		}
		return null;
	}

	/**
	 * 实例化之后
	 */
	@Override
	public boolean postProcessAfterInstantiation(Object bean, String beanName) throws BeansException {
		if (beanName.equals("lifeCycleBean")) {
			System.out.println("3. postProcessAfterInstantiation被调用");
		}
		return true;
	}

	@Override
	public PropertyValues postProcessProperties(PropertyValues pvs, Object bean, String beanName) throws BeansException {
		if (beanName.equals("lifeCycleBean")) {
			System.out.println("4. postProcessProperties被调用");
		}
		return null;
	}

	@Override
	public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
		if (beanName.equals("lifeCycleBean")) {
			((LifeCycleBean) bean).setName("中中");
			System.out.println("8. postProcessBeforeInitialization被调用，把name改成中中");
		}
		return bean;

	}

	@Override
	public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
		if (beanName.equals("lifeCycleBean")) {
			((LifeCycleBean) bean).setName("大大");
			System.out.println("11. postProcessAfterInitialization被调用，把name改成大大");
		}
		return bean;
	}
}

```

### 配置类

```java
@ComponentScan("com.yinshi.spring.bean")
@Configuration
public class  MainConfigOfLifeCycle {
	
	//@Scope("prototype")
	@Bean(initMethod="init",destroyMethod="detory")
	public Car car(){
		return new Car();
	}

	@Bean(initMethod = "myInit", destroyMethod = "myDestroy")
	public LifeCycleBean lifeCycleBean() {
		return new LifeCycleBean("小小");
	}

}
```

## 测试类

```java
package com.yinshi.spring;

import com.yinshi.spring.bean.LifeCycleBean;
import com.yinshi.spring.config.MainConfigOfLifeCycle;
import org.junit.Test;
import org.springframework.context.annotation.AnnotationConfigApplicationContext;

public class IOCTest_LifeCycle {

	@Test
	public void test01(){
		AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext(MainConfigOfLifeCycle.class);
		System.out.println("容器创建完成...");
		applicationContext.getBean(LifeCycleBean.class);
		// 关闭容器
		applicationContext.close();

	}
}

```

## 运行结果

```java
// 控制台的输出结果

1. postProcessBeforeInstantiation被调用
2. 构造方法被调用，name：小小
3. postProcessAfterInstantiation被调用
4. postProcessProperties被调用
5. BeanNameAware被调用, 获取到的beanName：lifeCycleBean
6. BeanFactoryAware被调用，获取到beanFactory：org.springframework.beans.factory.support.DefaultListableBeanFactory@117e949d: defining beans [lifeCycleBean,lifeCycleBeanPostProcessor]; root of factory hierarchy
7. ApplicationContextAware被调用，获取到ApplicationContextAware：org.springframework.context.support.ClassPathXmlApplicationContext@71e9ddb4, started on Sat Feb 22 20:30:35 CST 2020
8. postProcessBeforeInitialization被调用，把name改成中中
9. afterPropertiesSet被调用
10. myInit自定义初始化方法被调用，name：中中
11. postProcessAfterInitialization被调用，把name改成大大
12. bean创建完成 name： 大大
13. DisposableBean被调用
14. destroy-method自定义销毁方法被调用

Process finished with exit code 0

```

# 总结

1. BeanPostProcess

   - 实例化相关： postProcessBeforeInstantiation     postProcessAfterInstantiation
   - 填充属性相关: postProcessProperties   
   - 初始化相关： postProcessBeforeInitialization`、`postProcessAfterInitialization
   - 通过重写BeanPostProcess 的方法，可以介入bean创建的不同环节。同时通过postProcessBeforeInitialization 将bean的name属性值 从“小小”改成了“中中”，又通过postProcessAfterInitialization 将“中中”改成了“大大”，成功介入了bean的创建，并且依据我们的意愿 修改了bean

2. BeanNameAware、BeanFactoryAware、ApplicationContextAware
   这3类不属于后置处理器的范畴，学名叫感知器，让bean能感知到整个容器上下文信息的接口。spring在创建过程中，通过回调子类的setBeanName, setBeanFactory, setApplicationContext实现了BeanName, BeanFactory, ApplicationContext的注入，让bean能够感知获取到spring上下文的相关信息。虽然实现的东西很牛逼，但是实现的原理一点不复杂。通过检测当前的bean是否实现相关Aware，如果实现则调用子类set方法，将当前的BeanFactory等作为参数传入，直接上源码

   ```java
   	private void invokeAwareMethods(final String beanName, final Object bean) {
   		if (bean instanceof Aware) {
   			if (bean instanceof BeanNameAware) {
   				((BeanNameAware) bean).setBeanName(beanName);
   			}
   			if (bean instanceof BeanClassLoaderAware) {
   				ClassLoader bcl = getBeanClassLoader();
   				if (bcl != null) {
   					((BeanClassLoaderAware) bean).setBeanClassLoader(bcl);
   				}
   			}
   			if (bean instanceof BeanFactoryAware) {
   				((BeanFactoryAware) bean).setBeanFactory(AbstractAutowireCapableBeanFactory.this);
   			}
   		}
   	}
   ```

3. 最后贴出总结后的流程图

   

![](/uploads/springSource/BeanLifeCycle01.jpeg)


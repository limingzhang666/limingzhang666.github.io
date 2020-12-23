---
title: Spring源码解析-03-Bean的生命周期
copyright: true
related_posts: true
date: 2020-12-23 22:25:26
tags:
  - Spring
  - Bean生命周期
categories: Spring源码
---

## Bean生命周期

- bean创建---初始化----销毁的过程

- 容器管理bean的生命周期；

  我们可以自定义初始化和销毁方法；

  容器在bean进行到当前生命周期的时候来调用我们自定义的初始化和销毁方法



### 1.构造（对象创建）

- 单实例（Singleton）：在容器启动的时候创建对象
- 多实例(prototype )：在每次获取的时候创建对象



### 2.BeanPostProcessor.postProcessBeforeInitialization

### 3.初始化

对象创建完成，并赋值好，调用初始化方法。。。

### 4.BeanPostProcessor.postProcessAfterInitialization

### 5.销毁

- 单实例：容器关闭的时候
- 多实例：容器不会管理这个bean；容器不会调用销毁方法



## BeanPostProcessor 后置处理器

 * Bean 初始化前后进行处理工作
 * 遍历得到容器中所有的BeanPostProcessor；挨个执行beforeInitialization，
 * 一但返回null，跳出for循环，不会执行后面的BeanPostProcessor.postProcessorsBeforeInitialization

```java
/**
 * 后置处理器：初始化前后进行处理工作
 * 将后置处理器加入到容器中
 * @author liming
 */
@Component
public class MyBeanPostProcessor implements BeanPostProcessor {

	@Override
	public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
		// TODO Auto-generated method stub
		System.out.println("postProcessBeforeInitialization..."+beanName+"=>"+bean);
		return bean;
	}

	@Override
	public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
		// TODO Auto-generated method stub
		System.out.println("postProcessAfterInitialization..."+beanName+"=>"+bean);
		return bean;
	}

}


```

### 原理

见： org.springframework.context.support.AbstractApplicationContext#refresh

【Bean属性赋值】populateBean(beanName, mbd, instanceWrapper);

```bash
赋值之前：
1）、拿到InstantiationAwareBeanPostProcessor后置处理器；
	postProcessAfterInstantiation()；
2）、拿到InstantiationAwareBeanPostProcessor后置处理器；
	postProcessPropertyValues()；
=====赋值之前：===
3）、应用Bean属性的值；为属性利用setter方法等进行赋值；
	applyPropertyValues(beanName, mbd, bw, pvs);
4）、【Bean初始化】initializeBean(beanName, exposedObject, mbd);
	1）、【执行Aware接口方法】invokeAwareMethods(beanName, bean);
	执行xxxAware接口的方法
		BeanNameAware\BeanClassLoaderAware\BeanFactoryAware
	2）、【执行后置处理器初始化之前】
	applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
	BeanPostProcessor.postProcessBeforeInitialization（）;
	3）、【执行初始化方法】invokeInitMethods(beanName, wrappedBean, mbd);
		1）、是否是InitializingBean接口的实现；执行接口规定的初始化；
		2）、是否自定义初始化方法；
	4）、【执行后置处理器初始化之后】applyBeanPostProcessorsAfterInitialization
						 		BeanPostProcessor.postProcessAfterInitialization()；
```

即：

```java
  initializeBean
  {
  applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
  invokeInitMethods(beanName, wrappedBean, mbd);执行自定义初始化
  applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
   }
```



##  指定Bean 初始化和销毁的方法

- 通过@Bean指定init-method和destroy-method；

```java
@ComponentScan("com.liming.bean")
@Configuration
public class MainConfigOfLifeCycle {
	
	@Bean(initMethod="init",destroyMethod="detory")
	public Car car(){
		return new Car();
	}

}
```



- 通过让Bean实现InitializingBean（定义初始化逻辑），DisposableBean（定义销毁逻辑）;

```java
@Component
public class Cat implements InitializingBean,DisposableBean {
	
	public Cat(){
		System.out.println("cat constructor...");
	}

	@Override
	public void destroy() throws Exception {
		// TODO Auto-generated method stub
		System.out.println("cat...destroy...");
	}

	@Override
	public void afterPropertiesSet() throws Exception {
		// TODO Auto-generated method stub
		System.out.println("cat...afterPropertiesSet...");
	}

}
```



- 可以使用JSR250；

@PostConstruct：在bean创建完成并且属性赋值完成；来执行初始化方法

@PreDestroy：在容器销毁bean之前通知我们进行清理工作

```java
@Component
public class Dog implements ApplicationContextAware {
	//@Autowired
	private ApplicationContext applicationContext;
	public Dog(){
		System.out.println("dog constructor...");
	}
	//对象创建并赋值之后调用
	@PostConstruct
	public void init(){
		System.out.println("Dog....@PostConstruct...");
	}
	//容器移除对象之前
	@PreDestroy
	public void detory(){
		System.out.println("Dog....@PreDestroy...");
	}
	@Override
	public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
		// TODO Auto-generated method stub
		this.applicationContext = applicationContext;
	}
	
}
```



- BeanPostProcessor【interface】：bean的后置处理器

在bean初始化前后进行一些处理工作；
 * 		postProcessBeforeInitialization:在初始化之前工作
 * 		postProcessAfterInitialization:在初始化之后工作

## 扩展

Spring底层对 BeanPostProcessor 的使用包括：

bean赋值，注入其他组件，@Autowired，生命周期注解功能，@Async,xxx BeanPostProcessor;


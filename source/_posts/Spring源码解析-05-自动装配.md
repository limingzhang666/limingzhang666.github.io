---
title: Spring源码解析-05-自动装配
copyright: true
related_posts: true
date: 2020-12-23 23:32:33
tags:
  - 自动装配
  - Spring
categories: Spring源码
---

## 自动装配

Spring利用依赖注入（DI），完成对IOC容器中中各个组件的依赖关系赋值；

### @Autowired：自动注入

- 默认优先按照类型去容器中找对应的组件:applicationContext.getBean(BookDao.class);找到就赋值
- 如果找到多个相同类型的组件，再将属性的名称作为组件的id去容器中查找 applicationContext.getBean("bookDao")
- @Qualifier("bookDao")：使用@Qualifier指定需要装配的组件的id，而不是使用属性名
- 自动装配默认一定要将属性赋值好，没有就会报错；可以使用@Autowired(required=false);
- @Primary：让Spring进行自动装配的时候，默认使用首选的bean；也可以继续使用@Qualifier指定需要装配的bean的名字



### @Resource(JSR250)

@Resource:

- 默认是按照组件名称进行装配的；

- 可以和@Autowired一样实现自动装配功能；
- 没有能支持@Primary功能
- 没有支持@Autowired（reqiured=false）;

### @Inject(JSR330)[java规范的注解]

- 需要导入javax.inject的包，

- 和Autowired的功能一样。

- 没有required=false的功能；

 ### ps:

@Autowired:Spring定义的； 

@Resource、@Inject都是java规范



## 原理



### AutowiredAnnotationBeanPostProcessor

- AutowiredAnnotationBeanPostProcessor:解析完成自动装配功能；

### @Autowired位置

@Autowired:构造器，参数，方法，属性 ，都是从容器中获取参数组件的值

- 标注在set方法位置： @Bean+方法参数；参数从容器中获取;默认不写@Autowired效果是一样的；都能自动装配

- 标在构造器上：如果组件只有一个有参构造器，这个有参构造器的@Autowired可以省略，参数位置的组件还是可以自动从容器中获取
- 放在参数位置： 

```java
//默认加在ioc容器中的组件，容器启动会调用无参构造器创建对象，再进行初始化赋值等操作
@Component
public class Boss {
	

	private Car car;
	
	//构造器要用的组件，都是从容器中获取
	public Boss(Car car){
		this.car = car;
		System.out.println("Boss...有参构造器");
	}

	public Car getCar() {
		return car;
	}

	//@Autowired 
	//标注在方法，Spring容器创建当前对象，就会调用方法，完成赋值；
	//方法使用的参数，自定义类型的值从ioc容器中获取
	public void setCar(// @Autowired
        Car car) {
		this.car = car;
	}

	@Override
	public String toString() {
		return "Boss [car=" + car + "]";
	}
	
}
```



```java
@Configuration
@ComponentScan({"com.liming.service","com.liming.dao",
	"com.liming.controller","com.liming.bean"})
public class MainConifgOfAutowired {
	
	@Primary
	@Bean("bookDao2")
	public BookDao bookDao(){
		BookDao bookDao = new BookDao();
		bookDao.setLable("2");
		return bookDao;
	}
	
	/**
	 * @Bean标注的方法创建对象的时候，方法参数的值从容器中获取
	 * @param car
	 * @return
	 */
	@Bean
	public Color color(Car car){
		Color color = new Color();
		color.setCar(car);
		return color;
	}
	

}
```



### 把Spring底层一些组件注入到自定义的Bean中

自定义组件想要使用Spring容器底层的一些组件（ApplicationContext，BeanFactory，xxx）；
 * 		自定义组件实现xxxAware；在创建对象的时候，会调用接口规定的方法注入相关组件；Aware；
 * 		把Spring底层一些组件注入到自定义的Bean中；
 * 		xxxAware：功能使用xxxProcessor；
 * 			ApplicationContextAware==》ApplicationContextAwareProcessor. invokeAwareInterfaces ；会帮忙把spring底层逐渐传入到当前bean 

```java
@Component
public class Red implements ApplicationContextAware,BeanNameAware,EmbeddedValueResolverAware {
   
   private ApplicationContext applicationContext;

   @Override
   public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
      // TODO Auto-generated method stub
      System.out.println("传入的ioc："+applicationContext);
      this.applicationContext = applicationContext;
   }

   @Override
   public void setBeanName(String name) {
      // TODO Auto-generated method stub
      System.out.println("当前bean的名字："+name);
   }

   @Override
   public void setEmbeddedValueResolver(StringValueResolver resolver) {
      // TODO Auto-generated method stub
      String resolveStringValue = resolver.resolveStringValue("你好 ${os.name} 我是 #{20*18}");
      System.out.println("解析的字符串："+resolveStringValue);
   }




}
```
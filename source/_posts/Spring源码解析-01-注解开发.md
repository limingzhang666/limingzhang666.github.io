---
title: Spring源码解析-01-注解开发
copyright: true
related_posts: true
date: 2020-12-23 21:21:41
tags:
  - Spring
  - 注解
categories: Spring源码
---

## Spring注解解析

最近在准备面试，翻出之前记录的笔记，再次回顾温习一下，快忘得差不多了。额

### @Configuration

- @Configuration ==配置文件，告诉Spring框架，这是一个配置类 
- @interface Configuration 之上，加上了

### @ComponentScans  



 @ComponentScan  value:指定要扫描的包

excludeFilters = Filter[] ：指定扫描的时候按照什么规则排除那些组件
includeFilters = Filter[] ：指定扫描的时候只需要包含哪些组件
FilterType.ANNOTATION：按照注解
FilterType.ASSIGNABLE_TYPE：按照给定的类型；
FilterType.ASPECTJ：使用ASPECTJ表达式
FilterType.REGEX：使用正则指定
FilterType.CUSTOM：使用自定义规则

```java
@Configuration  //告诉Spring这是一个配置类

@ComponentScans(
		value = {
				@ComponentScan(value="com.atguigu",includeFilters = {
/*						@Filter(type=FilterType.ANNOTATION,classes={Controller.class}),
						@Filter(type=FilterType.ASSIGNABLE_TYPE,classes{BookService.class}),*/
						@Filter(type=FilterType.CUSTOM,classes={MyTypeFilter.class})
				},useDefaultFilters = false)	
		}
		)
public class MainConfig {
	
	//给容器中注册一个Bean;类型为返回值的类型，id默认是用方法名作为id
	@Bean("person")
	public Person person01(){
		return new Person("lisi", 20);
	}

}

public class MyTypeFilter implements TypeFilter {

	/**
	 * metadataReader：读取到的当前正在扫描的类的信息
	 * metadataReaderFactory:可以获取到其他任何类信息的
	 */
	@Override
	public boolean match(MetadataReader metadataReader, MetadataReaderFactory metadataReaderFactory)
			throws IOException {
		// TODO Auto-generated method stub
		//获取当前类注解的信息
		AnnotationMetadata annotationMetadata = metadataReader.getAnnotationMetadata();
		//获取当前正在扫描的类的类信息
		ClassMetadata classMetadata = metadataReader.getClassMetadata();
		//获取当前类资源（类的路径）
		Resource resource = metadataReader.getResource();
		
		String className = classMetadata.getClassName();
		System.out.println("--->"+className);
		if(className.contains("er")){
			return true;
		}
		return false;
	}

}

```

### @Bean

- 给容器中注册一个Bean;类型为返回值的类型，id默认是用方法名作为id

```java
	@Bean("person")
	public Person person01(){
		return new Person("lisi", 20);
	}
```

### @Scope

@Scope("prototype")

系统默认是singleton   单实例

-  prototype 多实例 （IOC容器启动并不会去调用方法创建对象放在容器中，每次获取的时候才会去调用方法 创建对象）

- singleton  单实例 （默认值） ioc容器启动就会调用方法创建对象 放在ioc容器中，以后每次获取都是直接从容器中（map.get()）中拿

### @Lazy

懒加载

- 单实例bean：默认在容器启动的时候创建对象；
- 懒加载：容器启动不创建对象。第一次使用(获取)Bean创建对象，并初始化；



### @Conditional

@Conditional： 按照一定的条件 进行判断，满足条件给容器注册bean

//需要实现 Condition接口

- 使用方式

```java
//需要实现 Condition接口
public class LinuxCondition implements Condition {

    /**
     * }
     *
     * @param context  判断条件能使用的上下文 （环境）
     * @param metadata 注释信息
     * @return
     */
    @Override
    public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
        //1. 能获取到ioc 使用的beanFactory
        ConfigurableListableBeanFactory beanFactory = context.getBeanFactory();
        //2. 获取类加载器
        ClassLoader classLoader = context.getClassLoader();
        // 3. 获取当前的环境信息
        Environment environment = context.getEnvironment();
        // 4. 能获取到bean定义的注册类
        BeanDefinitionRegistry registry = context.getRegistry();

        String property = environment.getProperty("os.name");
        if (property.contains("linux")) {
            return true;
        }
        return false;
    }
    
    //使用方式
    
    @Conditional(WindowCondition.class)
    @Bean("bill")
    public Person person01(){
        return  new Person("Bill Gates",62);
    }

    @Conditional(LinuxCondition.class)
    @Bean("linus")
    public Person person02(){
        return  new Person("linus",48);
    }
```


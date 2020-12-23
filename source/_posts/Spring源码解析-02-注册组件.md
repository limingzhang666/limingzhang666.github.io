---
title: Spring源码解析-02-注册组件
copyright: true
related_posts: true
date: 2020-12-23 21:54:19
tags:
  - Spring
  - 注册组件
categories: Spring源码
---

## 注册组件



- 包扫描+组件标注注解（@Controller/@Service/@Repository/@Component）[自己写的类]
- @Bean[导入的第三方包里面的组件]
- @Import[快速给容器中导入一个组件]
    - @Import(要导入到容器中的组件)；容器中就会自动注册这个组件，id默认是全类名
    - ImportSelector:返回需要导入的组件的全类名数组；
    - ImportBeanDefinitionRegistrar:手动注册bean到容器中
- 使用Spring提供的 FactoryBean（工厂Bean）;
    - 默认获取到的是工厂bean调用getObject创建的对象
    - 要获取工厂Bean本身，我们需要给id前面加一个&， 即
         	 “&colorFactoryBean"



### @Import

使用@Import 注解，内容包含 MyImportSelector

```java
@Configuration
// 导入组件，id默认是组件的全类名（com.liming.bean.Color）
@Import({Color.class, Red.class, MyImportSelector.class})
public class MainConfig2 {
    
}
```

#### ImportSelector

```java
//自定义逻辑返回需要导入的组件
public class MyImportSelector implements ImportSelector {

	//返回值，就是到导入到容器中的组件全类名
	//AnnotationMetadata:当前标注@Import注解的类的所有注解信息
	@Override
	public String[] selectImports(AnnotationMetadata importingClassMetadata) {
		// TODO Auto-generated method stub
		//importingClassMetadata
		//方法不要返回null值
		return new String[]{"com.bean.Blue","com.bean.Yellow"};
	}

}
```

#### ImportBeanDefinitionRegistrar

使用 ImportBeanDefinitionRegistrar:手动注册bean到容器中

```java
public class MyImportBeanDefinitionRegistrar implements ImportBeanDefinitionRegistrar {

	/**
	 * AnnotationMetadata：当前类的注解信息
	 * BeanDefinitionRegistry:BeanDefinition注册类；
	 * 		把所有需要添加到容器中的bean；调用
	 * 		BeanDefinitionRegistry.registerBeanDefinition手工注册进来
	 */
	@Override
	public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
		
		boolean definition = registry.containsBeanDefinition("com.atguigu.bean.Red");
		boolean definition2 = registry.containsBeanDefinition("com.atguigu.bean.Blue");
		if(definition && definition2){
			//指定Bean定义信息；（Bean的类型，Bean。。。）
			RootBeanDefinition beanDefinition = new RootBeanDefinition(RainBow.class);
			//注册一个Bean，指定bean名
			registry.registerBeanDefinition("rainBow", beanDefinition);
		}
	}

}
```

#### FactoryBean

实现FactoryBean接口, 然后加上@Bean

```java
//创建一个Spring定义的FactoryBean
public class ColorFactoryBean implements FactoryBean<Color> {

	//返回一个Color对象，这个对象会添加到容器中
	@Override
	public Color getObject() throws Exception {
		// TODO Auto-generated method stub
		System.out.println("ColorFactoryBean...getObject...");
		return new Color();
	}

	@Override
	public Class<?> getObjectType() {
		// TODO Auto-generated method stub
		return Color.class;
	}

	//是单例？
	//true：这个bean是单实例，在容器中保存一份
	//false：多实例，每次获取都会创建一个新的bean；
	@Override
	public boolean isSingleton() {
		// TODO Auto-generated method stub
		return false;
	}

}

	@Bean
	public ColorFactoryBean colorFactoryBean(){
		return new ColorFactoryBean();
	}


	@Test
    public void testImport() {
        printBeans(applicationContext);
		// ==》》》》》 这个实际 获取的是color对象，虽然id是 colorFactoryBean
        Object colorFactoryBean2 = applicationContext.getBean("colorFactoryBean");
        Object colorFactoryBean3 = applicationContext.getBean("colorFactoryBean");
        System.out.println("colorFactoryBean==>"+colorFactoryBean2);
        System.out.println(colorFactoryBean2==colorFactoryBean3);
		// ======》 如果想要获取factoryBean本身，需要在id前面加上  & 符号
        Object colorFactoryBean4 = applicationContext.getBean("&colorFactoryBean");
        System.out.println(colorFactoryBean4);

    }

result：
colorFactoryBean
===》colorFactoryBean==>com.liming.bean.Color@159f197
true
====》com.liming.condition.ColorFactoryBean@78aab498
```


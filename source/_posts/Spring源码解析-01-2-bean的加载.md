---
title: Spring源码解析-01-02-bean的加载
copyright: true
comments: true
related_posts: true
date: 2021-03-11 23:21:48
tags: bean的加载
categories: Sprig源码解析
---



- 其实这一章，描述了很多次关于循环依赖的问题，后面需要专门去看看 。

# Bean的加载

对于加载Bean的功能，在Spring中的调用方式为：

```java
	ApplicationContext applicationContext1 = new ClassPathXmlApplicationContext("spring-config.xml");
	// MyTestBean bean = applicationContext1.getBean(MyTestBean.class);
	MyTestBean bean = applicationContext1.getBean("myTestBean");
```

现在阅读源码看看Spring代码是如何实现的

- BeanFactory

```
boolean containsBean(String name);
```

- AbstractBeanFactory

```java
@Override
public Object getBean(String name, Object... args) throws BeansException {
	return doGetBean(name, null, args, false);
}


protected <T> T doGetBean(
			String name, @Nullable Class<T> requiredType, @Nullable Object[] args, boolean typeCheckOnly)
			throws BeansException {
		// 提取对应的beanName
		String beanName = transformedBeanName(name);
		System.out.println("doGetBean beanName="+beanName);
		Object bean;

		// Eagerly check singleton cache for manually registered singletons.
		// 急切地检查手动注册的单例缓存。
		/**
		 * 检查缓存中或者实例工厂中是否有对应的实例
		 * 为什么首先会 使用这段代码呢，因为在创建单例Bean 的时候会存在依赖注入的情况，而在创建依赖的时候为了避免循环依赖，
		 * spring 创建bean的原则是不等 bean创建完成就会将bean 的ObjectFactory 提早曝光
		 * 也就是将 ObjectFactory 加入到缓存中 ，一旦下个bean 创建时候需要依赖上个bean ，则直接使用ObjectFactory
		 */

		// 直接尝试从缓存获取 或者 singletonFactories中的ObjectFactory 中获取
		Object sharedInstance = getSingleton(beanName);
		if (sharedInstance != null && args == null) {
			if (logger.isTraceEnabled()) {
				if (isSingletonCurrentlyInCreation(beanName)) {
					logger.trace("Returning eagerly cached instance of singleton bean '" + beanName +
							"' that is not fully initialized yet - a consequence of a circular reference");
				}
				else {
					logger.trace("Returning cached instance of singleton bean '" + beanName + "'");
				}
			}
			// 返回对应的实例，有时候存在诸如 BeanFactory的情况，并不是直接返回实例本身，而是返回指定方法返回的实例
			bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
		}

		else {
			//只有在单例情况才会尝试解决循环依赖，原型模式情况下，如果存在A中有B的属性，B中有Ade属性，
			// 那么当依赖注入的时候，就会产生当A还未创建完的时候，因为对于B的创建再次返回创建A，造成循环依赖，也就是下面的情况
			// isPrototypeCurrentlyInCreation(beanName)为 true
			// Fail if we're already creating this bean instance:
			// We're assumably within a circular reference.
			if (isPrototypeCurrentlyInCreation(beanName)) {
				throw new BeanCurrentlyInCreationException(beanName);
			}

			// Check if bean definition exists in this factory.
			BeanFactory parentBeanFactory = getParentBeanFactory();
			// 如果beanDefinitionMap中 也就是所有已经加载的类中 不包括beanName，则尝试从parentBeanFactory中检查
			if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
				// Not found -> check parent.
				String nameToLookup = originalBeanName(name);
				// 递归 到BeanFactory 中寻找
				if (parentBeanFactory instanceof AbstractBeanFactory) {
					return ((AbstractBeanFactory) parentBeanFactory).doGetBean(
							nameToLookup, requiredType, args, typeCheckOnly);
				}
				else if (args != null) {
					// Delegation to parent with explicit args.
					return (T) parentBeanFactory.getBean(nameToLookup, args);
				}
				else if (requiredType != null) {
					// No args -> delegate to standard getBean method.
					return parentBeanFactory.getBean(nameToLookup, requiredType);
				}
				else {
					return (T) parentBeanFactory.getBean(nameToLookup);
				}
			}
			// 如果 不是仅仅做类型检查 则是创建Bean，这里要进行记录
			if (!typeCheckOnly) {
				markBeanAsCreated(beanName);
			}

			try {
			// 将存储XML配置文件的 GernericBeanDefinition 转换为 RootBeanDefinition，
			// 如果指定beanName 是子Bean 的话，同时会合并父类的相关属性
				RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
				checkMergedBeanDefinition(mbd, beanName, args);

				// Guarantee initialization of beans that the current bean depends on.
				String[] dependsOn = mbd.getDependsOn();
				// 如果存在依赖，则需要递归实例化的bean
				if (dependsOn != null) {
					for (String dep : dependsOn) {
						if (isDependent(beanName, dep)) {
							throw new BeanCreationException(mbd.getResourceDescription(), beanName,
									"Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
						}
						// 缓存依赖调用
						registerDependentBean(dep, beanName);
						try {
							getBean(dep);
						}
						catch (NoSuchBeanDefinitionException ex) {
							throw new BeanCreationException(mbd.getResourceDescription(), beanName,
									"'" + beanName + "' depends on missing bean '" + dep + "'", ex);
						}
					}
				}

				// 实例化 依赖的bean后便可以实例化 mbd本身了
				// singleton 模式的创建
				// Create bean instance.
				if (mbd.isSingleton()) {
					sharedInstance = getSingleton(beanName, () -> {
						try {
							return createBean(beanName, mbd, args);
						}
						catch (BeansException ex) {
							// Explicitly remove instance from singleton cache: It might have been put there
							// eagerly by the creation process, to allow for circular reference resolution.
							// Also remove any beans that received a temporary reference to the bean.
							destroySingleton(beanName);
							throw ex;
						}
					});
					//
					bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
				}

				else if (mbd.isPrototype()) {
					// It's a prototype -> create a new instance.
					// prototype 模式的创建 （new）
					Object prototypeInstance = null;
					try {
						beforePrototypeCreation(beanName);
						prototypeInstance = createBean(beanName, mbd, args);
					}
					finally {
						afterPrototypeCreation(beanName);
					}
					bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
				}

				else {
					String scopeName = mbd.getScope();
					if (!StringUtils.hasLength(scopeName)) {
						throw new IllegalStateException("No scope name defined for bean ´" + beanName + "'");
					}
					Scope scope = this.scopes.get(scopeName);
					if (scope == null) {
						throw new IllegalStateException("No Scope registered for scope name '" + scopeName + "'");
					}
					try {
						Object scopedInstance = scope.get(beanName, () -> {
							beforePrototypeCreation(beanName);
							try {
								return createBean(beanName, mbd, args);
							}
							finally {
								afterPrototypeCreation(beanName);
							}
						});
						bean = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
					}
					catch (IllegalStateException ex) {
						throw new BeanCreationException(beanName,
								"Scope '" + scopeName + "' is not active for the current thread; consider " +
								"defining a scoped proxy for this bean if you intend to refer to it from a singleton",
								ex);
					}
				}
			}
			catch (BeansException ex) {
				cleanupAfterBeanCreationFailure(beanName);
				throw ex;
			}
		}

		// Check if required type matches the type of the actual bean instance.
		// 检查需要的类型是否 符合bean的实际类型
		if (requiredType != null && !requiredType.isInstance(bean)) {
			try {
				T convertedBean = getTypeConverter().convertIfNecessary(bean, requiredType);
				if (convertedBean == null) {
					throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
				}
				return convertedBean;
			}
			catch (TypeMismatchException ex) {
				if (logger.isTraceEnabled()) {
					logger.trace("Failed to convert bean '" + name + "' to required type '" +
							ClassUtils.getQualifiedName(requiredType) + "'", ex);
				}
				throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
			}
		}
		return (T) bean;
	}
```

从上面代码可以看出bean的加载经历 了一个相当复杂的过程，其中涉及各种各样的考虑。

对于加载的过程涉及的步骤大致如下：

1. 转换对应的beanName

   - 传入的参数name 可能是别名，也可以是FactoryBean,所以需要进行一系列的解析，解析的内容包括一下内容：
     - 去除Factory的修饰符， 比如 如果name是 “&aa”,name首先会去除 &，使得name=“aa”
     - 去指定alias所表示的最终beanName，例如 别名A指向名称为B的bean，则返回B； 如果别名A指向别名B，别名B指向 名称为C的bean，则返回C
2. 尝试从缓存中加载单例

   - 单例在Spring的同一个容器中 只会被创建一次，后续在获取Bean，就直接从单例缓存中获取了。

   - 当然这里也是尝试加载，首先尝试从缓存中加载，如果加载不成功则尝试从singletonFactories 中加载。
   - 因为在创建单例bean的时候会存在依赖注入的情况，而在创建依赖的时候为了避免循环依赖，在Spring中创建bean的原则是 **不等bean创建完成就会创建bean的 ObjectFactory 提早曝光加入到缓存中 ，一旦下一个bean创建时候需要依赖上一个 bean则直接使用ObjectFactory** 
3. bean的实例化

   - 如果从缓存中得到了bean 的原始状态，则需要 对bean进行实例化
   - 强调一下： 缓存中记录的只是最原始的bean 状态，并不一定是 我们最终想要的bean 。举个例子，假如 我们需要对工厂bean进行处理，那么这里得到的其实是 工厂bean的 初始状态， 但是我们真正需要的是工厂bean 中定义的factory-method方法中返回的bean ，而 getObjectForBeanInstance 就是完成这个工作的，
4. 原型模式的依赖检查

   - 只有在单例的情况下，才会尝试解决循环依赖 ，如果存在A 中有B的属性，B中有A的属性 ，那么当依赖注入的时候，就会产生当A 还未创建完成的时候因为对于B的创建再次返回创建A，造成循环依赖 ，也就是情况：if (isPrototypeCurrentlyInCreation(beanName)) =true
5. 检测parentBeanFactory

   - 从代码上看，如果缓存没有数据的话，就直接转到父类工厂上去加载了，为啥呢
   - if (parentBeanFactory != null && !containsBeanDefinition(beanName)) 
     - parentBeanFactory != null ，parentBeanFactory如果为空，则其他一切都是浮云，这个无fuck可说
     -  !containsBeanDefinition(beanName)这个判断比较重要， 她是在检测如果当前加载的XML配置文件 中不包含beanName 所对应的配置，就只能到parentBeanFactory 去尝试下了，然后再去 递归的调用getBean 方法
6.  将存储XML配置文件的GernericBeanDefinition 转换为 RootBeanDefinition
   - 因为从XML配置文件中读取到的Bean信息是存储在GernericBeanDefinition 中的，但是所有的Bean后续处理都是针对RootBeanDefinition的，所以这里需要进行一个转换，转换的同时 如果父类bean不为空的话，则会一并合并父类的属性
7. 寻找依赖
   - 因为bean的初始化过程中 可能会用到某些属性，而这些属性很可能是动态配置的，并且配置成依赖其他的bean，那么这个时候就有必要先加载依赖的bean，
   - 所以在Spring的加载顺寻中，在初始化某一个bean的时候首先会初始化这个bean所对应的依赖 
8. **针对不同的scope 进行bean的创建**
   - 在Spring中存在不同的scope，其中默认的是Singleton，但是还有些其他的配置 诸如 prototype、request之类的。这个步骤中，Spring会根据 不同的配置进行不同的初始化策略
9. 类型转换
   - 程序到这里返回bean后已经基本结束了，通常对该方法的调用参数requiredType 是为空的 ，
   - 但是可能会存在这种情况： 返回的bean 其实是个String ，但是requiredType却传入 Integer类型，那么这个时候本步骤就会起作用了，他的功能是将返回的bean转换为requiredType. 
   - 在Spring 中提供了各种各样的转换器，用户也可以自己扩展转换器来满足需求 。

## 1.1 FactoryBean的使用

- 一般情况下，Spring通过反射机制利用bean的class 属性指定实现类 来实例化bean。在某些情况下，实例化bean的过程比较复杂，如果按照传统的方式，则需要在<bean> 中提供大量的配置信息，配置方式的灵活性受限，这是采用编码的方式可能会得到一个简单的方案 。  
- spring为此通过了一个  org.springframework.beans.factory.FactoryBean 的工厂类接口 ，用户可以通过实现该接口定制实例化bean的 逻辑 
- org.springframework.beans.factory.FactoryBean 接口对于Spring框架非常重要，Spring本身就提供了70多个 FactoryBean的实现。他们隐藏了实例化一些复杂bean的细节 ，为上层应用带来了便利 。

```java
public interface FactoryBean<T> {

	/**
	 * The name of an attribute that can be
	 * {@link org.springframework.core.AttributeAccessor#setAttribute set} on a
	 * {@link org.springframework.beans.factory.config.BeanDefinition} so that
	 * factory beans can signal their object type when it can't be deduced from
	 * the factory bean class.
	 * @since 5.2
	 */
	String OBJECT_TYPE_ATTRIBUTE = "factoryBeanObjectType";

	@Nullable
    // 返回 由FactoryBean 创建的bean的实例，如果 isSingleton()返回 true，则该实例会放到Spring容器中单实例缓存池 中
	T getObject() throws Exception;

	@Nullable
    // 返回 FactoryBean 创建的bean类型
	Class<?> getObjectType();

    // 返回 由FactoryBean 创建的bean实例的作用域 是singleton 还是 prototype
	default boolean isSingleton() {
		return true;
	}
}
```

- 当配置文件中<bean>的class 属性配置的实现类是 FactoryBean是，通过getBean（）方法返回的不是FactoryBean本身 ，而是 FactoryBean#getObject()方法所返回的对象 ，相当于 Factory#getObject() 代理了getBean() 方法

```java
//@Data
public class Car {
	private int maxSpeed;
	private String brand;
	private double price;

	public int getMaxSpeed() {
		return maxSpeed;
	}

	public void setMaxSpeed(int maxSpeed) {
		this.maxSpeed = maxSpeed;
	}

	public String getBrand() {
		return brand;
	}

	public void setBrand(String brand) {
		this.brand = brand;
	}

	public double getPrice() {
		return price;
	}

	public void setPrice(double price) {
		this.price = price;
	}
}

// factoryBean
public class CarFactoryBean implements FactoryBean<Car> {
	private String carInfo;

	@Override
	public Car getObject() throws Exception {
		Car car = new Car();
		String[] infos = carInfo.split(",");
		car.setBrand(infos[0]);
		car.setMaxSpeed(Integer.valueOf(infos[1]));
		car.setPrice(Double.valueOf(infos[1]));
		return car;
	}

	@Override
	public Class<?> getObjectType() {
		return Car.class;
	}
	@Override
	public boolean isSingleton(){
		return false;
	}

	public String getCarInfo() {
		return carInfo;
	}

	public void setCarInfo(String carInfo) {
		this.carInfo = carInfo;
	}
}

// 配置文件
	<bean id="car" class="com.mazhichu.spring.factory.CarFactoryBean">
		<property name="carInfo" value="BMW,200,200000"/>
	</bean>
       
// 测试类
	@Test
	public void MyTestBeanTest() throws IOException {
		ApplicationContext applicationContext1 = new ClassPathXmlApplicationContext("spring-config.xml");
		MyTestBean bean = applicationContext1.getBean(MyTestBean.class);
		Car car = (Car)applicationContext1.getBean("car");
		System.out.println(car.toString());
        // factoryBean 需要加 &
		CarFactoryBean carFactoryBean = (CarFactoryBean)applicationContext1.getBean("&car");
		System.out.println(car.toString());
		System.out.println(carFactoryBean.toString());
	}
}
// 运行 结果
Car{maxSpeed=200, brand='BMW', price=200.0}
```

- 当调用getBean("car")时，Spring通过反射机制发现 CarFactoryBean 实现了FactoryBean接口， 这时候 Spring 容器就调用 接口方法getObject 方法返回 Car对象
- 如果希望 FactoryBean对象，则需要在使用getBean（beanName）方法时 ，在beanName 前面显示的加上"&"前缀，例如getBean（“&car”），CarFactoryBean carFactoryBean = (CarFactoryBean)applicationContext1.getBean("&car");



## 1.2 缓存中获取单例Bean

- 前面知道FactoryBean的用法后，我们就可以了解 bean加载的过程了，
- 单例在Spring的同一个容器内只会被创建一次 ，后续在获取bean 就直接从单例缓存中获取， 当然这里也只是尝试加载，
- 首先尝试从缓存中加载，然后再尝试从singletonFactories 中加载 。
- 因为在创建单例bean 的时候，会出现依赖注入的情况，而在创建依赖的时候，为了避免循环依赖 ，Spring 创建bean 的原则是 不等bean创建完成就会将创建bean的ObjectFactory提早曝光加入到缓存中，一旦下一个bean创建的时候需要依赖上个bean，则直接使用 ObjectFactory

### getSingleton逻辑

```java
	@Override
	@Nullable
	public Object getSingleton(String beanName) {
        // 参数 true，设置标识 允许早期依赖
		return getSingleton(beanName, true);
	}

/**
	 * Return the (raw) singleton object registered under the given name.
	 * <p>Checks already instantiated singletons and also allows for an early
	 * reference to a currently created singleton (resolving a circular reference).
	 * @param beanName the name of the bean to look for
	 * @param allowEarlyReference whether early references should be created or not
	 * @return the registered singleton object, or {@code null} if none found
	 */
@Nullable
	protected Object getSingleton(String beanName, boolean allowEarlyReference) {
		// Quick check for existing instance without full singleton lock
		//1. 检查缓存中是否存在单例，首先从 singletonObjects 中获取
		Object singletonObject = this.singletonObjects.get(beanName);
		if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
            // 2. 再尝试从 earlySingletonObjects 中获取
			singletonObject = this.earlySingletonObjects.get(beanName);
			if (singletonObject == null && allowEarlyReference) {
				// 3.如果为空 ，则锁定全局变量 并进行处理
				synchronized (this.singletonObjects) {
					// Consistent creation of early reference within full singleton lock
					singletonObject = this.singletonObjects.get(beanName);
					if (singletonObject == null) {
						//如果此bean正在加载，则不处理
						singletonObject = this.earlySingletonObjects.get(beanName);
						if (singletonObject == null) {
                            // 4. 获取 ObjectFactory ，通过ObjectFactory.getObject 获取bean
							// 当某些 方法需要提前初始化的时候，
							// 则会调用 addSingletonFactory 方法将对应的ObjectFactory初始化策略存储在SingletonFactoryes
							ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
							if (singletonFactory != null) {
								// 调用预先设定的getObject方法
								singletonObject = singletonFactory.getObject();
								// 记录在缓存中，earlySingletonObjects 与 singletonFactories互斥
								this.earlySingletonObjects.put(beanName, singletonObject);
								this.singletonFactories.remove(beanName);
							}
						}
					}
				}
			}
		}
		return singletonObject;
	}
```

这个方法因为设计循环依赖的检测，以及涉及很多变量的记录存取，所以让人 困惑

1. 首先尝试 从 singletonObjects 里面获取实例，
2. 如果获取不到再从 earlySIngletontonObjects里面获取，
3. 如果还获取不到，再尝试 从singletonFactories 里面获取beanName 对应的ObjectFactory ，然后调用这个 ObjectFactory 的getObject方法来创建bean ，并放到 earlySingletonObjects中去，并且从 singletonFactories 里面remove掉这个 ObjectFactory，
4. 而对于后续的所有内存操作都只为了循环依赖检测时候使用，也就是在allowEarlyReference 为 true 的情况下才会使用

### 各类Map缓存的介绍

```java
	/** Cache of singleton objects: bean name to bean instance. */
	private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);

	/** Cache of singleton factories: bean name to ObjectFactory. */
	private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<>(16);

	/** Cache of early singleton objects: bean name to bean instance. */
	private final Map<String, Object> earlySingletonObjects = new ConcurrentHashMap<>(16);

	/** Set of registered singletons, containing the bean names in registration order. */
	private final Set<String> registeredSingletons = new LinkedHashSet<>(256);
```



- singletonObjects :用于保存beanName 和创建bean实例之间的关系， beanName-> bean instance
- singletonFactories :用于保存beanName 和创建Bean的工厂之间的关系 ，beanName->objectFactory
- earlySingletonObjects： 也是保存BeanName 和创建bean 实例之间额关系，与 singletonObjects 的不同之处在于：当一个单例bean被放在这里之后，那么当bean还在创建过程中，就可以通过getBean方法获取到了， 其目的是用来检测循环引用 
- registeredSingletons ：用来保存所有已经注册的bean

## 1.3 从bean的实例中获取对象

在getBean方法中 getObjectForBeanInstance 是个高频率使用的方法，无论从缓存中获得bean 还是根据 不同的scope 策略加载bean 。

总之 我们得到bean的实例后要做第一步就是调用这个方法来检测一下正确性，其实 就是用于检测当前bean 是否是 FactoryBean类型的bean ，如果是，那么需要调用该bean对应的FactoryBean 实例中的getObject作为返回值 。

- 无论从缓存中获取到的bean还是通过不同的scope 策略加载的bean 都只是最原始的bean状态，并不一定是我们最想要的bean。 
- 举个例子，加入我们需要对工厂bean进行处理，那么这里得到的其实是工厂bean的初始状态，但是我们真正需要的是工厂bean 中定义的 factory-method 方法中返回的bean ，而 getObjectForBeanInstance 方法就是完成这个工作的 

```java
protected Object getObjectForBeanInstance(
			Object beanInstance, String name, String beanName, @Nullable RootBeanDefinition mbd) {
		// 如果指定的name 是工厂相关（以 &为前缀）且 beanInstance 又不是FactoryBean类型，则 验证不通过
		// Don't let calling code try to dereference the factory if the bean isn't a factory.
		if (BeanFactoryUtils.isFactoryDereference(name)) {
			if (beanInstance instanceof NullBean) {
				return beanInstance;
			}
			if (!(beanInstance instanceof FactoryBean)) {
				// beanInstance 又不是FactoryBean类型，则 验证不通过
				throw new BeanIsNotAFactoryException(beanName, beanInstance.getClass());
			}
			if (mbd != null) {
				mbd.isFactoryBean = true;
			}
			return beanInstance;
		}

		// Now we have the bean instance, which may be a normal bean or a FactoryBean.
		// If it's a FactoryBean, we use it to create a bean instance, unless the
		// caller actually wants a reference to the factory.
		// 现在我们有了个Bean的实例，这个实例可能会是正常的bean 或者 factoryBean
		// 如果是FactoryBean 我们使用它创建实例，但是如果 用户想要直接获取工厂实例 而不是工厂的getObject 方法对应的实例，
		// 那么传入的name 应该加入前缀 &
		if (!(beanInstance instanceof FactoryBean)) {
			return beanInstance;
		}

		// 加载FactoryBean
		Object object = null;
		if (mbd != null) {
			mbd.isFactoryBean = true;
		}
		else {
			// 尝试从缓存中加载bean
			object = getCachedObjectForFactoryBean(beanName);
		}
		if (object == null) {
			// 到这里已经明确知道  beanInstance 一定是 FactoryBean类型
			// Return bean instance from factory.
			FactoryBean<?> factory = (FactoryBean<?>) beanInstance;
			// Caches object obtained from FactoryBean if it is a singleton.

			// containsBeanDefinition 检测 beanDefinitionMap中 也就是在所有已经加载的类中检查是否定义beanName
			if (mbd == null && containsBeanDefinition(beanName)) {
				// 将存储XML配置文件的 GernericBeanDefinition 转换为RootBeanDefinition，
				// 如果指定beanName 是子bean的话，会合并父类的相关属性
				mbd = getMergedLocalBeanDefinition(beanName);
			}
			// 是否是用户定义的而不是应用程序本身定义的
			boolean synthetic = (mbd != null && mbd.isSynthetic());


			// !!!!!!重点方法
			object = getObjectFromFactoryBean(factory, beanName, !synthetic);
		}
		return object;
	}
```

- 从上面代码来看，其实这个方法并没有上面重要的信息，大多是 写辅助代码以及一些功能性的判断，
- 而真正的核心代码却委托给了  getObjectFromFactoryBean（）

### getObjectFromFactoryBean（）

- 对FactoryBean正确性的验证 
- 对非 FactoryBean 不做任何处理
- 对bean进行转换
- 将从 Factory中解析bean 的工作委托给 getObjectFromFactoryBean

```java
protected Object getObjectFromFactoryBean(FactoryBean<?> factory, String beanName, boolean shouldPostProcess) {
		// 1.如果是单例模式
		if (factory.isSingleton() && containsSingleton(beanName)) {
			synchronized (getSingletonMutex()) {
				Object object = this.factoryBeanObjectCache.get(beanName);
				if (object == null) {
					 // 1.1 就是 ObjectFactory.getObject 方法
					object = doGetObjectFromFactoryBean(factory, beanName);
					// Only post-process and store if not put there already during getObject() call above
					// (e.g. because of circular reference processing triggered by custom getBean calls)
					Object alreadyThere = this.factoryBeanObjectCache.get(beanName);
					if (alreadyThere != null) {
						object = alreadyThere;
					}
					else {
						if (shouldPostProcess) {
							if (isSingletonCurrentlyInCreation(beanName)) {
								// Temporarily return non-post-processed object, not storing it yet..
								return object;
							}
							beforeSingletonCreation(beanName);
							try {
								// 调用ObjectFactory 的后处理器
								object = postProcessObjectFromFactoryBean(object, beanName);
							}
							catch (Throwable ex) {
								throw new BeanCreationException(beanName,
										"Post-processing of FactoryBean's singleton object failed", ex);
							}
							finally {
								afterSingletonCreation(beanName);
							}
						}
						if (containsSingleton(beanName)) {
							// 1.3. 再放入缓存中
							this.factoryBeanObjectCache.put(beanName, object);
						}
					}
				}
				return object;
			}
		}
		else {
			// 2.不是单例，就可以重复创建了
			// 2.1 就是 ObjectFactory.getObject 方法
			Object object = doGetObjectFromFactoryBean(factory, beanName);
			if (shouldPostProcess) {
				try {
					// 调用ObjectFactory 的后处理器
					object = postProcessObjectFromFactoryBean(object, beanName);
				}
				catch (Throwable ex) {
					throw new BeanCreationException(beanName, "Post-processing of FactoryBean's object failed", ex);
				}
			}
			return object;
		}
	}
```

- 这个方法只做了一件事情，就是返回的bean 如果是单例的 ，那就必须保证全局唯一， 同时，也因为是单例的，所以不必重复创建 ，可以使用缓存来提高性能 。也就是说已经加载过就要记录下来以便于下次复用 ，否则就直接获取了 
- 在 doGetObjectFromFactoryBean 方法中，我们终于看到想要看到的方法，也就是 Object=factory.getObject();



### postProcessObjectFromFactoryBean(object, beanName)后处理器

- AbstractAutowireCapableBeanFactory#postProcessObjectFromFactoryBean

```java
/**
	 * Applies the {@code postProcessAfterInitialization} callback of all
	 * registered BeanPostProcessors, giving them a chance to post-process the
	 * object obtained from FactoryBeans (for example, to auto-proxy them).
	 * @see #applyBeanPostProcessorsAfterInitialization
	 */
	@Override
	protected Object postProcessObjectFromFactoryBean(Object object, String beanName) {
		return applyBeanPostProcessorsAfterInitialization(object, beanName);
	}

// 以后应用非常广泛的  XXBeanPostProcessor
	@Override
	public Object applyBeanPostProcessorsAfterInitialization(Object existingBean, String beanName)
			throws BeansException {
		Object result = existingBean;
		for (BeanPostProcessor processor : getBeanPostProcessors()) {
			Object current = processor.postProcessAfterInitialization(result, beanName);
			if (current == null) {
				return result;
			}
			result = current;
		}
		return result;
	}

```

- 在Spring 获取Bean的规则中有这样一条：  尽可能保证所有bean初始化后都会调用注册的BeanPostProcessor 的 postProcessAfterInitialization 方法进行处理

## 1.4 获取单例

之前我们知道了如何从缓存中获取单例的过程， 如果缓存中不存在已经加载的单例bean，就需要 从头开始bean的加载过程。 而spring中使用getSIngleton 的重载方法实现bean的加载过程



org/springframework/beans/factory/support/AbstractBeanFactory.java:342

- DefaultSingletonBeanRegistry#getSingleton(java.lang.String, org.springframework.beans.factory.ObjectFactory<?>)

```java
public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {
		Assert.notNull(beanName, "Bean name must not be null");
		// 全局变量需要同步
		synchronized (this.singletonObjects) {
			// 首先检查对应的bean 是否已经加载过，因为singleton模式 其实就是复用以创建的bean，所以 这一步是必须的
			Object singletonObject = this.singletonObjects.get(beanName);
			// 如果为空才可以进行 singleton 的bean 的初始化
			if (singletonObject == null) {
				if (this.singletonsCurrentlyInDestruction) {
					throw new BeanCreationNotAllowedException(beanName,
							"Singleton bean creation not allowed while singletons of this factory are in destruction " +
							"(Do not request a bean from a BeanFactory in a destroy method implementation!)");
				}
				if (logger.isDebugEnabled()) {
					logger.debug("Creating shared instance of singleton bean '" + beanName + "'");
				}
				//
                // 通过 this.singletonsCurrentlyInCreation.add(beanName) 将当前正在创建的bean 记录在缓存中。
				// 这样便可以对循环依赖进行检测
				beforeSingletonCreation(beanName);
				//
				boolean newSingleton = false;
				boolean recordSuppressedExceptions = (this.suppressedExceptions == null);
				if (recordSuppressedExceptions) {
					this.suppressedExceptions = new LinkedHashSet<>();
				}
				try {
					// 初始化 bean
					singletonObject = singletonFactory.getObject();
					newSingleton = true;
				}
				catch (IllegalStateException ex) {
					// Has the singleton object implicitly appeared in the meantime ->
					// if yes, proceed with it since the exception indicates that state.
					singletonObject = this.singletonObjects.get(beanName);
					if (singletonObject == null) {
						throw ex;
					}
				}
				catch (BeanCreationException ex) {
					if (recordSuppressedExceptions) {
						for (Exception suppressedException : this.suppressedExceptions) {
							ex.addRelatedCause(suppressedException);
						}
					}
					throw ex;
				}
				finally {
					if (recordSuppressedExceptions) {
						this.suppressedExceptions = null;
					}
					//
                    // this.singletonsCurrentlyInCreation.remove(beanName)，
					// 移除 缓存中对该bean的正在加载的状态 的记录
					afterSingletonCreation(beanName);
					//
				}
				if (newSingleton) {
					// 加入缓存 ，并删除 各类辅助状态
					addSingleton(beanName, singletonObject);
				}
			}
			return singletonObject;
		}
	}

	protected void addSingleton(String beanName, Object singletonObject) {
		synchronized (this.singletonObjects) {
			this.singletonObjects.put(beanName, singletonObject);
			this.singletonFactories.remove(beanName);
			this.earlySingletonObjects.remove(beanName);
			this.registeredSingletons.add(beanName);
		}
	}
```

上述代码其实是使用 了回调方法，使得 程序可以在 单例创建的前后做一些准备 以及处理操作， 而真正的获取单例bean的方法其实并不是在 此方法中实现的。其实 现在逻辑是在 ObjectFactory 类型的实例singletonFactory 中实现的。而这些 准备及处理操作包括如下内容：

1. 检查缓存是否已经加载过

2. 若没有加载，则记录 beanName 的正在加载状态

3. 加载单例前，记录加载状态 ，

   beforeSingletonCreation(beanName)：通过 this.singletonsCurrentlyInCreation.add(beanName) 将当前正在创建的bean 记录在缓存中。这样便可以对循环依赖进行检测
   				

4. 通过调用参数 传入的ObjectFactory 的个体Object 方法实例化bean 

5. 加载单例后的处理方法调用 

6. 将结果记录至缓存 并删除 加载bean过程中所记录的各种辅助状态 

7. 返回处理结果

## 1.5 准备创建bean

跟踪了这么多Spring源码，经历了这么多函数，或多或少发现了一些规律 ：

- 一个真正干活的函数其实是以 do开头的，比如 doGetObjectFromFactoryBean； 而 给我们错觉的函数，比如 getObjectFromFactoryBean ，其实只是从全局角度去做些 统筹的工作 。

这个规律对于createBean 也不例外，可以看看createBean做了哪些准备工作 



- org.springframework.beans.factory.support.AbstractBeanFactory#doGetBean
  - org.springframework.beans.factory.support.AbstractBeanFactory#createBean
    - org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#createBean

```
/**
	 * Central method of this class: creates a bean instance,
	 * populates the bean instance, applies post-processors, etc.
	 * @see #doCreateBean
	 */
	@Override
	protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
			throws BeanCreationException {

		if (logger.isTraceEnabled()) {
			logger.trace("Creating instance of bean '" + beanName + "'");
		}
		RootBeanDefinition mbdToUse = mbd;

		// Make sure bean class is actually resolved at this point, and
		// clone the bean definition in case of a dynamically resolved Class
		// which cannot be stored in the shared merged bean definition.

		// 1. 锁定class ，根据设置的class 属性或者根据className 来解析class
		Class<?> resolvedClass = resolveBeanClass(mbd, beanName);
		if (resolvedClass != null && !mbd.hasBeanClass() && mbd.getBeanClassName() != null) {
			mbdToUse = new RootBeanDefinition(mbd);
			mbdToUse.setBeanClass(resolvedClass);
		}

		// Prepare method overrides.
		// 2. 验证以及准备覆盖的方法
		try {
			mbdToUse.prepareMethodOverrides();
		}
		catch (BeanDefinitionValidationException ex) {
			throw new BeanDefinitionStoreException(mbdToUse.getResourceDescription(),
					beanName, "Validation of method overrides failed", ex);
		}

		try {
			// 3. 给BeanPostProcessor 一个机会来返回代理，来替代真正的实例
			// Give BeanPostProcessors a chance to return a proxy instead of the target bean instance.
			Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
			if (bean != null) {
				return bean;
			}
		}
		catch (Throwable ex) {
			throw new BeanCreationException(mbdToUse.getResourceDescription(), beanName,
					"BeanPostProcessor before instantiation of bean failed", ex);
		}

		try {
			// 4. doXXXX 才是真正做符合方法名称事情的方法
			Object beanInstance = doCreateBean(beanName, mbdToUse, args);
			if (logger.isTraceEnabled()) {
				logger.trace("Finished creating instance of bean '" + beanName + "'");
			}
			return beanInstance;
		}
		catch (BeanCreationException | ImplicitlyAppearedSingletonException ex) {
			// A previously detected exception with proper bean creation context already,
			// or illegal singleton state to be communicated up to DefaultSingletonBeanRegistry.
			throw ex;
		}
		catch (Throwable ex) {
			throw new BeanCreationException(
					mbdToUse.getResourceDescription(), beanName, "Unexpected exception during bean creation", ex);
		}
	}
```

1. 根据设置的class 属性或者根据className 来解析 Class

2. 对override 属性进行标记及验证 
3. 应用初始化前的后处理器，解析指定bean 是否存在 初始化前的短路操作 
4. 创建bean



### 实例化的前置处理

在真正调用docreate方法创建bean的实力前，使用resolveBeforeInstantiation 对 BeanDefinition 中的属性做些前置处理 。

无论其中是否有相应的逻辑实现我们都可以理解，因为真正逻辑实现前后 留有处理函数也是 可扩展的一种体现， 但是这并不是最重要的 ，在函数 中 还提供了一个 短路判断，这才是最为关键的部分 。

```java
try {
			// 3. 给BeanPostProcessor 一个机会来返回代理，来替代真正的实例
			// Give BeanPostProcessors a chance to return a proxy instead of the target bean instance.
			Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
			if (bean != null) {
				return bean;
			}
		}
```

当经过前置处理后，返回的结果如果不为空，那么会直接略过后续的Bean的创建 而直接返回结果。	

这一特性虽然很容易被忽略，但是却起着至关重要的作用，我们熟知的AOP功能 就是基于这里的判断 

```java
@Nullable
	protected Object resolveBeforeInstantiation(String beanName, RootBeanDefinition mbd) {
		Object bean = null;
		// 如果尚未被解析
		if (!Boolean.FALSE.equals(mbd.beforeInstantiationResolved)) {
			// Make sure bean class is actually resolved at this point.
			if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
				Class<?> targetType = determineTargetType(beanName, mbd);
				if (targetType != null) {
					//
					bean = applyBeanPostProcessorsBeforeInstantiation(targetType, beanName);
					//
					if (bean != null) {
						bean = applyBeanPostProcessorsAfterInitialization(bean, beanName);
					}
				}
			}
			mbd.beforeInstantiationResolved = (bean != null);
		}
		return bean;
	}
```

此方法最吸引我们的无疑 是两个方法 applyBeanPostProcessorsBeforeInstantiation  以及 applyBeanPostProcessorsAfterInitialization 。两个方法的实现非常的简单 ，无非是 对后处理器中的所有 InstantiationAwareBeanPostProcessor 类型的后处理器 进行 postProcessBeforeInstantiation 方法 和 BeanPostProcessor
















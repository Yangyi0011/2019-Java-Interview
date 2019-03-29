# 一、什么是Spring？

​	Spring是个包含一系列功能的合集，如快速开发的Spring Boot，支持微服务的Spring Cloud，支持认证与鉴权的Spring Security，Web框架Spring MVC。Spring的核心是IOC与AOP。

# 二、Spring MVC处理流程

![SpringMvc处理流程](C:/Users/Yang/Desktop/%E9%9D%A2%E8%AF%95%E5%87%86%E5%A4%87/img/SpringMvc%E5%A4%84%E7%90%86%E6%B5%81%E7%A8%8B.webp)

**SpringMVC执行流程：**

1. 用户发送请求，SpringMvc的核心控制器DispatcherServlet对请求进行拦截。
2. DispatcherServlet将拦截到请求发送给处理器映射器HandlerMapping。
3. 处理器映射器根据请求url找到具体的处理器，生成处理器执行链HandlerExecutionChain(包括处理器对象和处理器拦截器)一并返回给DispatcherServlet。
4. DispatcherServlet再根据处理器Handler获取处理器适配器HandlerAdapter，然后执行HandlerAdapter处理一系列的操作，如：参数封装，数据格式转换，数据验证等操作。
5. 执行处理器Handler(也就是Controller，也叫页面控制器)。
6. Handler（Controller）执行完成后，返回将结果与视图封装成ModelAndView返回。
7. HandlerAdapter将Handler（Controller）执行结果ModelAndView返回到DispatcherServlet。
8. DispatcherServlet将ModelAndView传给ViewReslover视图解析器。
9. ViewReslover解析后返回具体View。
10. DispatcherServlet对再View进行视图渲染（即将模型数据model填充至视图中）。
11. DispatcherServlet响应用户。

# 三、如何解决Spring的循环依赖?

## 1、什么是循环依赖？

​	循环依赖-->循环引用。--->即2个或以上bean 互相持有对方，最终形成闭环。

​	如：A依赖B，B依赖C，C又依赖A。【注意：这里不是函数的循环调用【是个死循环，除非有终结条件】，是对象相互依赖关系】

![spring循环依赖.png](C:/Users/Yang/Desktop/%E9%9D%A2%E8%AF%95%E5%87%86%E5%A4%87/img/spring%E5%BE%AA%E7%8E%AF%E4%BE%9D%E8%B5%96.png)

## 2、循环依赖的场景

1. 构造器注入的循环依赖。【**==这个Spring解决不了==**】

   ​	StudentA有参构造是StudentB。StudentB的有参构造是StudentC，StudentC的有参构造是StudentA ，这样就产生了一个循环依赖的情况，所以**在使用构造器注入实例化bean时，若是存在构造器的循环依赖，则IOC容器在实例化该bean时会报错。**

   报错原因：

   ​	**构造器注入方式**在 bean 刚被实例化时【使用有参构造】，就要注入相关的依赖，如：在实例化StudentA时，因StudentA依赖StudentB，所以IOC容器会先去实例化StudentB，而StudentB又依赖StudentC，所以IOC容器又会先去实例化StudentC，而因StudentC依赖StudentA，IOC容器一检查，发现StudentA还没构造完成……如此就形成了一个相互依赖的死循环……。

2. setter注入的循环依赖【==可以解决==】

   ​	field属性的循环依赖【setter方式 单例，默认方式-->通过递归方法找出当前Bean所依赖的Bean，然后提前缓存【会放入Cach中】起来。通过提前暴露 -->暴露一个exposedObject用于返回提前暴露的Bean。】

   **setter方式注入流程：**

   ![](C:/Users/Yang/Desktop/%E9%9D%A2%E8%AF%95%E5%87%86%E5%A4%87/img/Spring%20Bean%E7%9A%84%E7%94%9F%E5%91%BD%E5%91%A8%E6%9C%9F.png)

   图中前两步骤得知：**==Spring是先将Bean对象实例化【依赖无参构造函数】--->再设置对象属性的，==**所以即便有循环依赖，也不会出现问题（报错）；

   不报错的原因：

   ​	Spring先用构造器实例化Bean对象，然后将实例化结束的对象放到一个Map中，并且Spring提供获取这个未设置属性的实例化对象的引用方法。结合我们的实例来看，当Spring实例化了StudentA、StudentB、StudentC后，紧接着会去设置对象的属性，此时StudentA依赖StudentB，就会去Map中取出存在里面的单例StudentB对象，以此类推，就不会出来死循环的问题。

## 3、如何解决循环依赖？

   **使用无参构造器，通过==属性（字段）注入==即可。**

# 四、Spring bean 的生命周期。

1. IOC容器实例化 Bean 对象。

2. 对 bean 对象进行属性赋值（包括依赖注入）。

3. 检查Aware相关接口的实现，若有实现则给Bean注入Aware接口的相关依赖。

   Aware接口是Spring对外暴露底层组件的一种方式，实现了XXXAware接口就可以将XXX组件注入到实现类中来。如A实现了ApplicationContextAware，则Spring会将ApplicationContext【IOC容器】注入进A中，如此就可以在A里面使用ApplicationContext了。

4. BeanPostProcessor前置处理【postProcessBeforeInitialization】，在此可在bean初始化之前对bean进行相关处理。

5. 检查是否实现 InitializingBean接口，若实现则调用 afterPropertiesSet() 方法。

6. 检查是否配置有自定义的 init-method【初始化】方法，有就执行。

7. BeanPostProcessor后置处理【postProcessAfterInitialization】，在此可在bean初始化之后对bean进行相关处理。

8. 注册必要的 Destruction【销毁】相关回调接口。

9. 用户对bean的相关使用...

10. 检查是否实现DisposableBean【销毁前处理】接口，若实现则调用它的destroy方法。

11. 检查是否配置有自定义的 destroy【销毁】 方法，有则执行。

![](C:/Users/Yang/Desktop/%E9%9D%A2%E8%AF%95%E5%87%86%E5%A4%87/img/Spring%20Bean%E7%9A%84%E7%94%9F%E5%91%BD%E5%91%A8%E6%9C%9F.png)

# 五、Spring Bean 的作用域

- singleton：单例模式，Spring IoC容器中只会存在一个共享的Bean实例，无论有多少个Bean引用它，始终指向同一对象。默认IOC容器一启动就会实例化单例bean，可以指定单例bean为懒加载模式，在第一次使用到时才会实例化。
- prototype：原型模式，每次通过Spring容器获取prototype定义的bean时，容器都将创建一个新的Bean实例，每个Bean实例都有自己的属性和状态。prototype bean只有在被使用时才会实例化。
- request：在一次Http请求中，容器会返回该Bean的同一实例。而对不同的Http请求则会产生新的Bean实例，且该bean仅在当前Http Request内有效。
- session：在一次Http Session中，容器会返回该Bean的同一实例。而对不同的Session请求则会创建新的实例，该bean实例仅在当前Session内有效。
- global Session：在一个全局的Http Session中，容器会返回该Bean的同一个实例，仅在使用portlet context时有效。

# 六、Spring 核心

## 1、IOC(DI)

​	IOC【控制反转】：即**将bean的控制权从程序员手里交到Spring IOC 容器中。**以往程序员使用到一个bean时，都需要自己手动去new出来，bean的生命周期都由自己去手动管理，如创建、赋值和销毁等，使用spring后，这些操作大部分都由Sring IOC 容器去自动完成，如bean的创建、属性赋值、依赖注入等，程序员只需要定义好 bean 的成员变量和getter/setter方法就行。

​	DI【依赖注入】：在程序运行期间，若一个 bean 的成员变量引用到了另一个 bean，则 Spring IOC 容器会自动去实例化那个被引用的 bean，并将其注入到这个成员变量中以供使用。

## 2、AOP

1. AOP概念

   AOP：即面向切面编程技术，是OOP【面向对象编程】的补充和完善。

2. AOP由来

   ​	传统的OOP【面向对象】编程的执行流程是从上到下的，没有从左到右，所以在实际生产过程中难免会产生大量与业务无关的重复代码，如许多方法都需要进行日志记录、权限效验等，而AOP的出现就是将这些与业务无关的重复代码抽取出来，组成一个个切面，然后再插入到业务代码的指定位置中，以降低代码的耦合。AOP常用的场景有：权限控制、日志记录、事物处理等。

3. AOP实现

   - 动态代理：对执行消息进行截取与装饰，以取代原有对象行为的执行。
   - 静态织入：引入特定的语法创建“方面”，从而使得编译器可以在编译期间织入有关“方面”的代码。

4. Spring AOP实现

   ​	Spring AOP采用的是**==动态代理==**模式，具体实现有：

   1. jdk 反射机制：
      - 原理：通过反射机制生成代理类的字节码文件，在调用具体方法前调用InvokeHandler来处理。
      - 条件：被代理类需要有接口。
   2. cglib字节码技术：
      - 原理：利用asm开源包，将代理对象类的class文件加载进来，通过修改其字节码生成子类来处理。
      - 条件：通过继承来实现，被代理类不能为 final。
   3. Spring AOP的选择：
      1. 若目标对象实现了接口，默认会采用JDK的动态代理实现AOP，也可以强制使用CGLIB实现。
      2. 若目标对象没有实现接口，则必须采用CGLIB库。【对象类不能被 final 修饰，因为CGLIB是通过继承来实现动态代理的。】
      3. spring会自动在JDK动态代理和CGLIB之间转换。

## 3、Spring源码分析

### 1、BeanFactory的创建及预备工作

```java
Spring容器的refresh()【创建刷新】;
1、prepareRefresh() //刷新前的预处理;
	1）、initPropertySources() //初始化一些属性设置;子类自定义个性化的属性设置方法；
	2）、getEnvironment().validateRequiredProperties(); //检验属性的合法等
	3）、earlyApplicationEvents= new LinkedHashSet<ApplicationEvent>(); //保存容器中的一些早期的事件；
2、obtainFreshBeanFactory(); //获取BeanFactory；
	1）、refreshBeanFactory(); //刷新【创建】BeanFactory；
			创建了一个this.beanFactory = new DefaultListableBeanFactory();
			设置id；
	2）、getBeanFactory(); //返回刚才GenericApplicationContext创建的BeanFactory对象；
	3）、将创建的BeanFactory【DefaultListableBeanFactory】返回；
3、prepareBeanFactory(beanFactory);//BeanFactory的预准备工作（BeanFactory进行一些设置）；
	1）、设置BeanFactory的类加载器、支持表达式解析器...
	2）、添加部分BeanPostProcessor【ApplicationContextAwareProcessor】
	3）、设置忽略的自动装配的接口EnvironmentAware、EmbeddedValueResolverAware、xxx；
	4）、注册可以解析的自动装配；我们能直接在任何组件中自动注入：
			BeanFactory、ResourceLoader、ApplicationEventPublisher、ApplicationContext
	5）、添加BeanPostProcessor【ApplicationListenerDetector】
	6）、添加编译时的AspectJ；
	7）、给BeanFactory中注册一些能用的组件；
		environment【ConfigurableEnvironment】、
		systemProperties【Map<String, Object>】、
		systemEnvironment【Map<String, Object>】
4、postProcessBeanFactory(beanFactory);BeanFactory准备工作完成后进行的后置处理工作；
	1）、子类通过重写这个方法来在BeanFactory创建并预准备完成以后做进一步的设置
```

### 2、SpringBean执行流程

```java
1、invokeBeanFactoryPostProcessors(beanFactory);执行BeanFactoryPostProcessor的方法；
	BeanFactoryPostProcessor：BeanFactory的后置处理器。在BeanFactory标准初始化之后执行的；
	两个接口：BeanFactoryPostProcessor、BeanDefinitionRegistryPostProcessor
	1）、执行BeanFactoryPostProcessor的方法；
		先执行BeanDefinitionRegistryPostProcessor
		1）、获取所有的BeanDefinitionRegistryPostProcessor；
		2）、看先执行实现了PriorityOrdered优先级接口的BeanDefinitionRegistryPostProcessor、
			postProcessor.postProcessBeanDefinitionRegistry(registry)
		3）、在执行实现了Ordered顺序接口的BeanDefinitionRegistryPostProcessor；
			postProcessor.postProcessBeanDefinitionRegistry(registry)
		4）、最后执行没有实现任何优先级或者是顺序接口的BeanDefinitionRegistryPostProcessors；
			postProcessor.postProcessBeanDefinitionRegistry(registry)
    
        再执行BeanFactoryPostProcessor的方法
        1）、获取所有的BeanFactoryPostProcessor
        2）、看先执行实现了PriorityOrdered优先级接口的BeanFactoryPostProcessor、
            postProcessor.postProcessBeanFactory()
        3）、在执行实现了Ordered顺序接口的BeanFactoryPostProcessor；
            postProcessor.postProcessBeanFactory()
        4）、最后执行没有实现任何优先级或者是顺序接口的BeanFactoryPostProcessor；
            postProcessor.postProcessBeanFactory()
    
2、registerBeanPostProcessors(beanFactory);注册BeanPostProcessor（Bean的后置处理器）【 intercept bean creation】
	不同接口类型的BeanPostProcessor；在Bean创建前后的执行时机是不一样的
		BeanPostProcessor、
		DestructionAwareBeanPostProcessor、
		InstantiationAwareBeanPostProcessor、
		SmartInstantiationAwareBeanPostProcessor、
		MergedBeanDefinitionPostProcessor【internalPostProcessors】
	1）、获取所有的 BeanPostProcessor;后置处理器都默认可以通过PriorityOrdered、Ordered接口来执行优先级
	2）、先注册PriorityOrdered优先级接口的BeanPostProcessor；
		把每一个BeanPostProcessor；添加到BeanFactory中
		beanFactory.addBeanPostProcessor(postProcessor);
	3）、再注册Ordered接口的
	4）、最后注册没有实现任何优先级接口的
	5）、最终注册MergedBeanDefinitionPostProcessor；
	6）、注册一个ApplicationListenerDetector；来在Bean创建完成后检查是否是ApplicationListener，如果是
		applicationContext.addApplicationListener((ApplicationListener<?>) bean);

3、initMessageSource();初始化MessageSource组件（做国际化功能；消息绑定，消息解析）；
		1）、获取BeanFactory
		2）、看容器中是否有id为messageSource的，类型是MessageSource的组件
			如果有赋值给messageSource，如果没有自己创建一个DelegatingMessageSource；
				MessageSource：取出国际化配置文件中的某个key的值；能按照区域信息获取；
		3）、把创建好的MessageSource注册在容器中，以后获取国际化配置文件的值的时候，可以自动注入MessageSource；
			beanFactory.registerSingleton(MESSAGE_SOURCE_BEAN_NAME, this.messageSource);	
			MessageSource.getMessage(String code, Object[] args, String defaultMessage, Locale locale);

4、initApplicationEventMulticaster();初始化事件派发器；
		1）、获取BeanFactory
		2）、从BeanFactory中获取applicationEventMulticaster的ApplicationEventMulticaster；
		3）、如果上一步没有配置；创建一个SimpleApplicationEventMulticaster
		4）、将创建的ApplicationEventMulticaster添加到BeanFactory中，以后其他组件直接自动注入
		
5、onRefresh();留给子容器（子类）
		1、子类重写这个方法，在容器刷新的时候可以自定义逻辑；
		
6、registerListeners();给容器中将所有项目里面的ApplicationListener注册进来；
		1、从容器中拿到所有的ApplicationListener
		2、将每个监听器添加到事件派发器中；
		getApplicationEventMulticaster().addApplicationListenerBean(listenerBeanName);
		
		3、派发之前步骤产生的事件；
		
7、finishBeanFactoryInitialization(beanFactory);初始化所有剩下的单实例bean；
	1、beanFactory.preInstantiateSingletons();初始化后剩下的单实例bean
		1）、获取容器中的所有Bean，依次进行初始化和创建对象
		2）、获取Bean的定义信息；RootBeanDefinition
		3）、Bean !抽象的 && 单实例的 && !懒加载；
			1）、判断是否是FactoryBean；是否是实现FactoryBean接口的Bean；
			2）、不是工厂Bean。利用getBean(beanName);创建对象
				0、getBean(beanName)； ioc.getBean();
				1、doGetBean(name, null, null, false);
				2、先获取缓存中保存的单实例Bean。如果能获取到说明这个Bean之前被创建过（所有创建过的单实例Bean都会被缓存起来）
					从private final Map<String, Object> singletonObjects = new ConcurrentHashMap<String, Object>(256);获取的
				3、缓存中获取不到，开始Bean的创建对象流程；
				4、标记当前bean已经被创建
				5、获取Bean的定义信息；
				6、【获取当前Bean依赖的其他Bean;如果有按照getBean()把依赖的Bean先创建出来；】
				7、启动单实例Bean的创建流程；
					1）、createBean(beanName, mbd, args);
					2）、Object bean = resolveBeforeInstantiation(beanName, mbdToUse);让BeanPostProcessor先拦截返回代理对象；
						【InstantiationAwareBeanPostProcessor】：提前执行；
						先触发：postProcessBeforeInstantiation()；
						如果有返回值：触发postProcessAfterInitialization()；
					3）、如果前面的InstantiationAwareBeanPostProcessor没有返回代理对象；调用4）
					4）、Object beanInstance = doCreateBean(beanName, mbdToUse, args);创建Bean
						 1）、【创建Bean实例】；createBeanInstance(beanName, mbd, args);
						 	利用工厂方法或者对象的构造器创建出Bean实例；
						 2）、applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
						 	调用MergedBeanDefinitionPostProcessor的postProcessMergedBeanDefinition(mbd, beanType, beanName);
						 3）、【Bean属性赋值】populateBean(beanName, mbd, instanceWrapper);
						 	赋值之前：
						 	1）、拿到InstantiationAwareBeanPostProcessor后置处理器；
						 		postProcessAfterInstantiation()；
						 	2）、拿到InstantiationAwareBeanPostProcessor后置处理器；
						 		postProcessPropertyValues()；
						 	=====赋值之前：===
						 	3）、应用Bean属性的值；为属性利用setter方法等进行赋值；
						 		applyPropertyValues(beanName, mbd, bw, pvs);
						 4）、【Bean初始化】initializeBean(beanName, exposedObject, mbd);
						 	1）、【执行Aware接口方法】invokeAwareMethods(beanName, bean);执行xxxAware接口的方法
						 		BeanNameAware\BeanClassLoaderAware\BeanFactoryAware
						 	2）、【执行后置处理器初始化之前】applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
						 		BeanPostProcessor.postProcessBeforeInitialization（）;
						 	3）、【执行初始化方法】invokeInitMethods(beanName, wrappedBean, mbd);
						 		1）、是否是InitializingBean接口的实现；执行接口规定的初始化；
						 		2）、是否自定义初始化方法；
						 	4）、【执行后置处理器初始化之后】applyBeanPostProcessorsAfterInitialization
						 		BeanPostProcessor.postProcessAfterInitialization()；
						 5）、注册Bean的销毁方法；
					5）、将创建的Bean添加到缓存中singletonObjects；
				ioc容器就是这些Map；很多的Map里面保存了单实例Bean，环境信息。。。。；
		所有Bean都利用getBean创建完成以后；
			检查所有的Bean是否是SmartInitializingSingleton接口的；如果是；就执行afterSingletonsInstantiated()；
			
8、finishRefresh();完成BeanFactory的初始化创建工作；IOC容器就创建完成；
		1）、initLifecycleProcessor();初始化和生命周期有关的后置处理器；LifecycleProcessor
			默认从容器中找是否有lifecycleProcessor的组件【LifecycleProcessor】；如果没有new DefaultLifecycleProcessor();
			加入到容器；
			写一个LifecycleProcessor的实现类，可以在BeanFactory
			void onRefresh();
			void onClose();	
        2）、	getLifecycleProcessor().onRefresh();
            拿到前面定义的生命周期处理器（BeanFactory）；回调onRefresh()；
        3）、publishEvent(new ContextRefreshedEvent(this));发布容器刷新完成事件；
        4）、liveBeansView.registerApplicationContext(this);
```

3、Spring源码总结

```java
======总结===========
1）、Spring容器在启动的时候，先会保存所有注册进来的Bean的定义信息；
	1）、xml注册bean；<bean></bean>
	2）、注解注册Bean；@Service、@Component、@Bean、xxx
2）、Spring容器会根据这些 Bean 的定义信息在合适的时机创建这些Bean
	1）、用到这个bean的时候；利用getBean()创建bean；创建好以后保存在容器中；
	2）、统一创建剩下所有的bean的时候；finishBeanFactoryInitialization()；
3）、执行后置处理器；BeanPostProcessor
	1）、每一个bean创建完成，都会使用各种后置处理器进行处理；来增强bean的功能；
		AutowiredAnnotationBeanPostProcessor:处理自动注入
		AnnotationAwareAspectJAutoProxyCreator:来做AOP功能；
		xxx....
		增强的功能注解：
		AsyncAnnotationBeanPostProcessor
		....
4）、处理事件驱动模型；
	ApplicationListener；事件监听；
	ApplicationEventMulticaster；事件派发：
```



# 七、Spring与SpringBOOT的区别

​      `Spring Boot`是`Spring`框架的扩展，它免去了开发`Spring`应用程序所需的各种`XML配置`，SpringBoot会自动配置，并支持嵌入式的Servlet容器，使开发Spring应用变得更快，更高效。

###### 以下是`Spring Boot`中的一些特点：

 1：创建独立的`spring`应用。
  2：嵌入`Tomcat`, `Jetty` `Undertow` 而且不需要部署他们。
  3：提供的“starters” poms来简化`Maven`配置
  4：尽可能自动配置`spring`应用。
  5：提供生产指标,健壮检查和外部化配置。
  6：绝对没有代码生成和`XML`配置要求。



# 八、SpringBoot自动配置原理

1. SpringBoot启动的时候先加载主配置类【标有@SpringBootConfiguration的类】，依靠==@EnableAutoConfiguration==开启自动配置功能 。
2. 自动配置功能开启后，SpringBoot会**==将类路径下  META-INF/spring.factories 里面配置的所有EnableAutoConfiguration的值加入到IOC容器中，这些值其实就是一个个自动配置类；==**
3. 接着SpringBoot为每一个自动配置类进行自动配置功能；
4. 根据==@Conditional==的派生注解判断当前配置类是否生效，若可以生效，则这个配置类就会给容器中添加各种组件；
5. 这些组件的属性是从对应的**==xxxProperties类==**中获取的，而这些类里面的每一个属性又是根据**==@ConfigurationProperties==**注解和对应的配置文件进行绑定的；
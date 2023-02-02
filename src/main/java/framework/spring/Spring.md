# refresh核心方法

## 1、prepareRefresh

```txt
1、设置容器的启动时间。
2、设置活跃状态为true，设置关闭状态为false。
3、获取Environment对象，并加载当前系统的属性值到Environment对象中。
4、准备监听器和事件的集合对象，默认为空集合。
```

## 2、obtainFreshBeanFactory

```txt
1、创建BeanFactory。
2、设置BeanFactory的id和相关属性。
3、加载XML文件定义bena的BeanDefinition的解析，大致流程为创建xml文件的document对象，String[] -> Resource[] -> document -> BeanDefinition
```

## 3、prepareBeanFactory

```txt
1、beanFactory设置列加载器，语言处理器，属性编辑器。
2、添加ApplicationContextAwareProcessor，用于某些Aware对象的注入。
3、设置忽略接口，EnvironmentAware，EmbeddedValueResolverAware，ResourceLoaderAware，ApplicationEventPublisherAware，MessageSourceAware，ApplicationContextAware，设置忽略的原因是这些接口的实现是容器通过set方法注入的，在使用autowire注入的时候要进行忽略。
4、设置BeanFactory，ResourceLoader，ApplicationEventPublisher，ApplicationContext在注入时，如果有多个实现，使用指定对象进行注入。
5、注册BeanPostProcessor的ApplicationListenerDetector实现，用于检测bean是否实现ApplicationListener
6、注册LoadTimeWeaverAwareProcessor增加对AspectJ的支持，在java中织入分为三种方式，分为编译器织入，类加载器织入，运行期织入。aspectj提供了两种织入方式，第一种是通过特殊编译器，在编译器，将aspectj语言编写的切面类织入到java类中，第二种是类加载期织入即LoadTimeWeaverAwareProcessor
7、注册系统环境对象到beanFactory中。
```

## 4、postProcessBeanFactory

```txt
扩展方法，没有具体实现，在springMVC中注册ServletContextAwareProcessor和添加忽略依赖接口ServletContextAware，ServletConfigAware等。
```



## 5、invokeBeanFactoryPostProcessors

```txt
1、先执行前置过程中内置的BeanDefinitionRegistryPostProcessor(以下简称BDRPP)，判断参数的beanFactory是否属于BeanDefinitionRegistry。
2、执行实现PriorityOrdered接口的BDRPP，从容器中取出所有的BDRPP，并匹配出实现了PriorityOrdered的BDRPP，此过程中会进行BDRPP的实例化(getBean())，排序并注册BDRPP,最重要的执行类为ConfigurationClassPostProcessor,解析所有注解生成BeanDefinition。
3、从容器中取出所有的BDRPP，执行实现Ordered接口的BDRPP，逻辑同上。
4、从容器中取出所有的BDRPP，执行没有实现排序接口的BDRPP，逻辑同上。
5、获取容器中的所有BeanFactoryPostProcessor(以下简称BFPP)，按照。PriorityOrdered、Ordered、没有排序分类，排序，实例化(getBean())、执行postProcessBeanFactory()。
存在的问题和思考：
为什么执行完一次BDRPP需要重新获取一次(beanFactory.getBeanNamesForType())，而执行BFPP的时候不需要呢?
首先，在执行过程中可能会生成新的实现BDRPP的BeanDefinition，如先执行的实现了PriorityOrdered的BDRPP，ConfigurationClassPostProcessor会解析所有的注解生成BeanDefinition，可能存在新解析的BDRPP。其次，BDRPP可以对容器中的BeanDefinition就行新增和删除，确保每次产生的新的BDRPP会被执行到。BFPP是针对BeanFactory的增强，过程中不会产生新的BFPP。
BDRPP可以和BFPP的执行顺序调换位置吗？
不能，BDRPP是BFPP的子接口，此外，BDRPP过程中可能会伴随着新的BFPP的产生。
```

## 6、registerBeanPostProcessors

```txt
1、获取容器中的所有BeanPostProcessor(以下简称BPP)。
2、计算BPP的数量，创建BeanPostProcessorChecker(BPP实例化期间记录消息，即当bean不符合所有BPP处理的条件时)。
3、将BPP按照PriorityOrdered、internal、Ordered、nonOrdered分成四类，实现PriorityOrdered接口的BPP实例化，排序，注册。
4、实现Ordered接口的BPP实例化，筛选出MergedBeanDefinitionPostProcessor放入internal，排序，注册。
5，没有实现排序接口的BPP实例化，筛选出MergedBeanDefinitionPostProcessor放入internal，排序，注册。
6、注册internal的所有BPP。
7、添加ApplicationListenerDetector到beanFactory。
需要注意的点：
计算BPP数量+1是新加了一个BeanPostProcessorChecker。
注册时是先删除，后添加，最后的ApplicationListenerDetector的监听器是为了添加到最后。添加过程中有hasInstantiationAwareBeanPostProcessors、hasDestructionAwareBeanPostProcessors标志位，存在对应的BPP则设置为true。
InstantiationAwareBeanPostProcessor：主要处理生成代理类，懒加载，bean实例化前执行，不会按照正常的bean的实例化流程，如果前置方法生成bean，会直接执行后置方法，不会走到doCreateBean()。
DestructionAwareBeanPostProcessor：bean销毁前的回调。
存在的问题：
PriorityOrdered、Ordered、nonOrdered三类BPP注册时已经注册了internal中的BPP，为什么最后还要注册一次？
加在最后？目前还未想通 TODO...
```

## 7、initMessageSource

```txt
初始化上下文message源，国际化处理。
如果存在messageSource，进行实例化，没有则默认使用DelegatingMessageSource注册进beanFactory。没太多可说的。
```

## 8、initApplicationEventMulticaster

```txt
如果存在applicationEventMulticaster，进行实例化，没有则使用SimpleApplicationEventMulticaster注册进beanFactory，观察者模式的优化实现。
```

## 9、onRefresh

```txt
spring中为模板方法，空实现，在springboot中用于tomcat的实例化。
```

## 10、registerListeners

```txt
1、注册已经初始化的ApplicationListener。
2、通过getBeanNamesForType()获取容器中的所有ApplicationListener并注册。
3、发布早已存在的ApplicationListener。
```

## 11、finishBeanFactoryInitialization

```txt
1、通过beanFactory.getBean()初始化ConversionService(用于类型转换)。
2、注册一个默认的嵌入值解析器如果没有BeanFactoryPostProcessor，例如PropertySourcesPlaceholderConfigurer，主要用于注解属性值的解析
3、通过beanFactory.getBean()初始化LoadTimeWeaverAware(用于类型转换)。
4、禁止使用临时类加载器进行类型匹配
5、冻结所有的beanDefinition，将不能被修改或进一步的处理
6、实例化剩下的非懒加载的单例对象。

方法中整个对象创建的核心流程 getBean -> doGetBean -> createBean -> doCreateBean -> createBeanInstance -> populateBean -> initializeBean
过程中依赖的初始化，属性填充递归调用getBean
```

### 11.1、preInstantiateSingletons

```txt
1、遍历所有的beanNames，通过getBean()进行对象的初始化。
	1、合并父类的BeanDefinition，生成最终的RootBeanDefinition，BeanDefinition包含ChildBeanDefinition，GenericBeanDefinition，RootBeanDefinition，一般情况下使用GenericBeanDefinition，最终合成RootBeanDefinition。
	2、非抽象，单例，非懒加载判断，如果不是直接跳过。
	3、如果是FactoryBean，调用getBean(& + beanName)初始化FactoryBean对象，如果急切希望初始化(SmartFactoryBean中的isEagerInit()进行设置)，如果急切希望初始化，调用getBean(beanName)进行初始化，
	4、如果不是FactoryBean，直接调用getBean(beanName)。
2、遍历所有的beanNames，获取实例对象，进行SmartInitializingSingleton的后初始化回调。
```

### 11.2、getBean -> doGetBean

```txt
1、beanName转换，bean对象实现FactoryBean接口会变成&beanName，存在别名需要把别名进行转换。
2、通过getSingleton()检查是否已经注册，
3、如果已经注册，通过getObjectForBeanInstance()获取对象，这里主要是处理实现了FactoryBean接口的对象。
4、如果没有注册：
	4.1、单例对象会尝试解决循环依赖的问题，如果是原型模式下如果存在循环依赖，直接抛出异常。
	4.2、尝试在父工厂获取bean对象，获取到直接返回。
	4.3、如果存在依赖bean，遍历递归调用getBean()优先实例化依赖bean。
	4.4、如果是单例，通过getSingleton()参数ObjectFactory函数时接口的lambda表达式createBean()创建对象，创建完成调用getObjectForBeanInstance()处理是否为实现了FactoryBean接口的对象。
	4.5、如果是原型模式的对象，调用createBean()创建对象，创建前后通过当前线程设置当前创建的beanName，创建完成调用getObjectForBeanInstance()
	4.6、指定scope的bean创建，创建流程同原型模式对象一致。
	4.7、检查requiredType是否与实际Bean实例的类型匹配，如果不匹配进行类型转换。
```

### 11.3、createBean

```txt
1、解析RootBeanDefinition的Class，重新创建RootBeanDefinition。
2、验证和准备覆盖方法，创建的bean对象中包含了lookup-method和replace-method标签的时，会产生覆盖操作。
3、执行resolveBeforeInstantiation(),给BeanPostProcessors一个机会来返回代理来替代真正的实例,调用InstantiationAwareBeanPostProcessor的postProcessBeforeInstantiation和BeanPostProcessor的postProcessAfterInitialization方法来返回实例对象。AOP的代理对象不在这里产生，在initializeBean()中的postProcessAfterInitialization生成代理对象。
```

### 11.4、doCreateBean

```txt
1、执行createBeanInstance()使用对应的策略创建实例，如供给性函数式接口Supplier<?>，工厂方法，构造函数注入，无参构造初始化。
2、执行MergedBeanDefinitionPostProcessor的postProcessMergedBeanDefinition()修改合并后的BeanDefinition。
3、添加ObjectFactory<?>进三级缓存，再填充属性前添加进三级缓存，避免在填充属性时存在循环依赖问题。
4、populateBean()对bean的属性就行填充，若存在依赖与其他bean属性，则会递归初始化依赖的bean：
	4.1、执行InstantiationAwareBeanPostProcessor的postProcessAfterInstantiation()设置属性。
	4.2、属性填充，1、根据名称注入，通过getBean(beanName)获取注入属性值。2、通过类型注入，根据descriptor的依赖类型解析出与descriptor所包装的对象匹配的候选Bean对象。
	4.3、执行InstantiationAwareBeanPostProcessor的postProcessProperties()当使用注解的时候，通过这个方法来完成属性的注入。
	4.4、依赖检查。
	4.5、使用深拷贝进行属性值应用。
5、initializeBean()执行bean初始化：
	5.1、执行aware接口(BeanNameAware、BeanClassLoaderAware、beanFactoryAware).
	5.2、执行BeanPostProcessor的postProcessBeforeInitialization(),其中EnvironmentAware、EmbeddedValueResolverAware、ResourceLoaderAware、ApplicationEventPublisherAware、MessageSourceAware、ApplicationContextAware在ApplicationContextAwareProcessor的postProcessBeforeInitialization()执行回调赋值
	5.3、调用初始化方法invokeInitMethods(),先执行InitializingBean的afterPropertiesSet()的方法回调，再调用bean的init方法。
	5.4、执行BeanPostProcessor的postProcessAfterInitialization()对初始化bean进行后置处理。
6、循环依赖检查：
	6.1、从缓存中获取对象。
	6.2、缓存中获取的对象不为空，bean初始化前和初始化后对象没有被增强，将缓存中获取的对象赋值给bean。
	6.3、缓存中获取的对象不为空并且bean在初始化后被增强，bean的依赖对象是否被都已经被初始化，如果没有全部被初始化，则存在循环依赖无法被解决。
7、注册可销毁的bean到销毁bean列表中。
```

## 12、finishRefresh

```txt
完成刷新过程。
1、清空在资源加载器中的所有资源缓存。
2、通知生命周期处理器lifecycleProcessor刷新过程。
3、将刷新完成传播到生命周期处理器。
4、发布ContextRefreshedEvent事件。
5、注册当前上下文到LiveBeansView，以支持JMX服务。
```

## 13、destroyBeans

```txt
为防止bean资源占用，在异常处理中，销毁已经在前面过程中生成的单例bean
```

##14、cancelRefresh

```txt
重置active标志
```

## 15、resetCommonCaches

```txt
重置常用的反射元数据缓存
```



# AOP

## 1、AOP的概念

```txt
aspect：切面，切面由切点和通知组成，即包括横切逻辑的定义也包括连接点的定义。
pointcut：切点，每个类都拥有多个连接点，可以理解是连接点得集合。
joinpoint：连接点，程序执行的某个特定位置，如某个方法调用前后等。
weaving：织入，将增强添加到目标类的具体连接点的过程。
advice：通知，是织入到目标类连接点上的一段代码，就是增强位置和增强内容。
target：目标对象，通知织入的目标类。
aop Proxy：代理对象，即增强后产生的对象。

Advice: org.aopalliance.aop.Advice
AOP中的一个标识接口,通知和Interceptor顶级类,各种通知类型都要实现这个接口

Advisor: org.springframework.aop.Advisor
aop的顶级接口，用来管理Advice和Pointcut

Advised: org.springframework.aop.framework.Advised
SpringAOP中的核心类,它组合了Advisor和TargetSource即目标对象

Pointcut: org.springframework.aop.Pointcut
主要负责对系统的相应的Joinpoint进行捕捉，对系统中所有的对象进行Joinpoint所定义的规则进行匹配
```



## 2、AOP代理的生成

```txt
AOP代理类的生成：
在bean创建过程中的initializeBean()中的BeanPostProcessor接口的实现AbstractAutoProxyCreator调用postProcessAfterInitialization()中的wrapIfNecessary()生成代理类.
1、获取当前bean的Advices和Advisors。
	1.1、查询beanFactory中的所有Advisor
	1.2、从所有的Advices中找到合适的bean的Advisors。
	1.3、对需要代理的Advisor按照一定的规则排序。
2、创建代理对象。
	2.1、创建代理工厂(ProxyFactory),添加代理接口，增强器，设置代理的类等属性。
	2.2、创建AOP代理，有接口定义的使用JDK代理(JdkDynamicAopProxy),否则使用CGLB代理(ObjenesisCglibAopProxy)
	2.3、获取代理。
		2.3.1、CGLB代理，通过Enhancer创建代理对象。
		2.3.2、JDK代理，通过JDK的proxy生成代理对象。
3、缓存生成的代理bean的类型
```

## 3、AOP的执行过程

```txt
AOP的执行过程：
1、执行目标方法时被DynamicAdvisedInterceptor拦截。
2、通过currentInterceptorIndex每次++调用对应advice，通过++最后调用目标方法
3、递归调用父类的proceed()执行Advice执行链，执行顺序为：
ExposeInvocationInterceptor —> AspectJAroundAdvice(process前置切面) -> MethodBeforeAdviceInterceptor -> AspectJAfterAdvice -> 目标方法 -> AspectJAfterReturningAdvice -> after切面 -> aroud(后置切面)

ExposeInvocationInterceptor：
利用ThreadLocal将不同的执行MethodInvocation进行隔离：
MethodInvocation oldInvocation = invocation.get();
	invocation.set(mi);
	try {
		return mi.proceed();
	}
	finally {
		invocation.set(oldInvocation);
	}
```

# 声明式事务

## 1、事物的传播特性

```txt
PROPAGATION_REQUIRED:默认事务类型，如果没有，就新建一个事务;如果有，就加入当前事务。
PROPAGATION_REQUIRES_NEW:如果没有，就新建一个事务;如果有，就将当前事务挂起。
PROPAGATION_NESTED:如果没有，就新建一个事务;如果有，就在当前事务中嵌套其他事务。
PROPAGATION_SUPPORTS:如果没有，就以非事务方式执行;如果有，就使用当前事务。
PROPAGATION_NOT_SUPPORTED:如果没有，就以非事务方式执行;如果有，就将当前事务挂起。即无论如何不支持事务。
PROPAGATION_NEVER:如果没有，就以非事务方式执行;如果有，就抛出异常。
PROPAGATION_MANDATORY:如果没有，就抛出异常;如果有，就使用当前事务。
```

## 2、隔离级别

```txt
ISOLATION_DEFAULT： PlatfromTransactionManager默认的隔离级别，使用数据库默认的事务隔离级别。
ISOLATION_READ_UNCOMMITTED： 这是事务最低的隔离级别，它充许令外一个事务可以看到这个事务未提交的数据。这种隔离级别会产生脏读，不可重复读和幻读。
ISOLATION_READ_COMMITTED： 保证一个事务修改的数据提交后才能被另外一个事务读取。另外一个事务不能读取该事务未提交的数据。
ISOLATION_REPEATABLE_READ： 这种事务隔离级别可以防止脏读，不可重复读。但是可能出现幻读。
ISOLATION_SERIALIZABLE：这是花费最高代价但是最可靠的事务隔离级别。事务被处理为顺序执行。除了防止脏读，不可重复读外，还避免了幻读。
```


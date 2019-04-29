注：spring aop的实现代码确实很多，我先后看了很多遍才理清整体流程，这里坐下记录，但是不会像spring源码解析那样非常详细。
## 前置问题
1. 都知道spring aop是通过jdk动态代理或者cglib创建动态代理实现的，但是它是怎么做到使用创建代理方式呢？
2. @Before,@After等注解的执行顺序是如何实现的。
3. 一个方法上有多个需要被代理的注解，他们的执行顺序怎么控制。
可以先想下这几个问题，最后在回答。
## 整体流程
先大概回顾下spring获取Bean的流程
1. 加载资源
    - 可以是xml或者注解标识的basePackage，并解析资源
2. 注入资源
    - 将资源中的属性和值都映射到BeanDefinition类型的对象上，放入BeanFactory的map中
3. 获取Bean
    - 从BeanFactory中尝试获取Bean,如果获取失败就根据之前解析的BeanDefinition创建Bean,创建完成后放入另一个map
spring aop的其实也是完整参与了上述三个步骤，解析资源时aop使用的都是自定义标签，都通SPI机制，加载对应的标签解析器，
解析完成同样会放入BeanFactory中保存，获取Bean的时候通过BeanPostProcessor类型的对象，在创建Bean前或者后创建代理。
## spring aop流程梳理
1. 为什么Bean创建将的时候会被BeanPostProcessor类型的对象处理
spring 解析自定义标签(@Afpect/@Before等)时会通过SPI机制，加载对应的标签解析器
```
public void init() {
		registerBeanDefinitionParser("config", new ConfigBeanDefinitionParser());
		registerBeanDefinitionParser("aspectj-autoproxy", new AspectJAutoProxyBeanDefinitionParser());
		registerBeanDefinitionDecorator("scoped-proxy", new ScopedProxyBeanDefinitionDecorator());
		registerBeanDefinitionParser("spring-configured", new SpringConfiguredBeanDefinitionParser());
	}
```
具体每个的职责这里不展开，但是AspectJAutoProxyBeanDefinitionParser在执行解析的时候，会向容器注入
AnnotationAwareAspectJAutoProxyCreator.class这个对象，而这个对象又间接继承了SmartInstantiationAwareBeanPostProcessor，spring在创建Bean时
会遍历容器中所有BeanPostProcessor的对象，执行对应的方法，也正是因为这一步，Bean创建将的时候会被BeanPostProcessor类型的对象处理。

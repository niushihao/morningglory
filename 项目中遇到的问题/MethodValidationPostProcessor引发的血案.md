
本篇是基于参数校验不生效的问题续集，关于参数校验不生效的问题可以看上一篇【[参数校验不生效](https://github.com/niushihao/morningglory/blob/master/%E9%A1%B9%E7%9B%AE%E4%B8%AD%E9%81%87%E5%88%B0%E7%9A%84%E9%97%AE%E9%A2%98/%E5%8F%82%E6%95%B0%E6%A0%A1%E9%AA%8C%E4%B8%8D%E7%94%9F%E6%95%88.md)】
## MethodValidationPostProcessor引发的血案
上一篇说了在service中参数校验之所以不生效是因为没有校验的切入点(没有对应@Validated的PointCut),但是在项目中加入BeanPostProcessor类型的对象时还有个特别需要注意的地方。
### 问题描述
在配置文件中增加了MethodValidationPostProcessor的配置，出现部分接口参数校验生效，部分不生效，更神奇的是那些不生效的service原本正常的事务、缓存等增强也不生效了。。。

通过打断点发现增强不生效的对象确实也被代理了，但是advisors却为空,下面是我的配置文件

```
@Configuration
public class ConfigBean {

@Autowired
private ServiceA serviceA;
    
@Autowired
private ServiceB serviceB;
    
....省略N个依赖

	@Bean
	public BeanA beanA(){
	    return new BeanA();
	}
	
	... 省略N个bean
	
	//新增加的用于参数校验的后置处理器
	@Bean
	public MethodValidationPostProcessor methodValidationPostProcessor(){
	    return new MethodValidationPostProcessor();
	}


}
```
### 问题分析
执行步骤

1. spring在加载MethodValidationPostProcessor时在对应的BeanDefinition的factoryName和factoryMethod上分别设置了configBean和methodValidationPostProcessor
2. 创建Bean实例时会判断如果有factoryMethod就是使用这个方法创建实例
3. 使用factoryMethod创建实例就必须先创建ConfigBean这个实例
4. 在ConfigBean实例创建完成后会设置属性值(依赖注入)，然后又创建依赖的实例，而这些依赖的Bean他们又依赖了其他Bean...无脑进行创建。

问题定位

1. 单看上边的步骤好像看不出来与没有增强有什么关系，但是问题就在这，spring为了让我们的Bean能被增强，会先注册BeanPostProcessor类型的对象(如果增强处理器没有放入容器，我们的Bean怎么能被增强呢)，然后在循环创建所有单利且不是懒加载的Bean
2. 也就是我们的Bean在创建MethodValidationPostProcessor的时候都被提前创建了，然后他们也会执行后处理的方法，但是容器中没有后处理器的Bean,所以advisors为空。

### 根据源码回顾下流程

#### 1、容器刷新流程

```
public void refresh() throws BeansException, IllegalStateException {
		synchronized (this.startupShutdownMonitor) {
			// 1、准备刷新上下文环境
			prepareRefresh();

			// 2、初始化bean factory.
			ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

			/**	  
				* 3、对bean factory进行各种填充，容器的扩展也从这开始
				* 设置spel表达式
				* 设置几个忽略自动装配的接口
				* 设置几个自动装配的特殊规则
			*/
			prepareBeanFactory(beanFactory);

			try {
				// 4、留给子类扩展
				postProcessBeanFactory(beanFactory);

				// 5、激活BeanFactoryPostProcessors处理器
				invokeBeanFactoryPostProcessors(beanFactory);

				/**
					* 6、注册拦截bean创建的处理器，真正调用是在getBean时
					* 先于我们的业务bean的创建
				*/
				registerBeanPostProcessors(beanFactory);

				// 7、国际化处理
				initMessageSource();

				// 8、初始化应用消息广播
				initApplicationEventMulticaster();

				// 9、留给子类扩展
				onRefresh();

				// 10、在所有的bean中找到listener bean，注册到消息广播器中
				registerListeners();

				// 11、初始化剩余的单利非惰性的bean,我们的业务bean也在这一步初始化
				finishBeanFactoryInitialization(beanFactory);

				// 12、完成刷新，发出事件
				finishRefresh();
			}

		}
	}

```
#### 2、在步骤5 注册BeanPostProcessors会触发创建bean 的操作

```
// 获取所有BeanPostProcessor类型的beanName
String[] postProcessorNames = beanFactory.getBeanNamesForType(BeanPostProcessor.class, true, false);
for (String ppName : postProcessorNames) {
	if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
		// 从容器获取bean,没有就创建
		BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
		...省略N行代码
	}
	
}
```
#### 3、bean的获取逻辑：尝试从缓存中获取，获取失败就创建，我们直接看创建的逻辑

```
// 真正开始创建bean，中间部分代码省略,虽然这里有很多核心方法，但我们这里只关心createBeanInstance
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final @Nullable Object[] args)
	throws BeanCreationException {

BeanWrapper instanceWrapper = null;
if (mbd.isSingleton()) {
	instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
}
// 首先创建bean的实例
if (instanceWrapper == null) {
	instanceWrapper = createBeanInstance(beanName, mbd, args);
}

// 如果需要就提前暴露创建bean 的ObjectFactory,用来处理循环依赖
if (earlySingletonExposure) {
	if (logger.isTraceEnabled()) {
		logger.trace("Eagerly caching bean '" + beanName +
				"' to allow for resolving potential circular references");
	}
	addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
}	

// 填充属性值
populateBean(beanName, mbd, instanceWrapper);

// 调用初始化方法和后处理器方法
exposedObject = initializeBean(beanName, exposedObject, mbd);

// 注入容器
registerDisposableBeanIfNecessary(beanName, bean, mbd);

...
}
```
#### 4、创建bean实例

```
// 创建bean实例
protected BeanWrapper createBeanInstance(String beanName, RootBeanDefinition mbd, @Nullable Object[] args) {
	// 省略N行代码	
	// 如果有factoryMethodName，则使用此方法初始化
	if (mbd.getFactoryMethodName() != null) {
		return instantiateUsingFactoryMethod(beanName, mbd, args);
	}
	// 省略N行代码	
}

// 根据factoryMethodName创建bean实例
public BeanWrapper instantiateUsingFactoryMethod(
		String beanName, RootBeanDefinition mbd, @Nullable Object[] explicitArgs) {

	// 省略N行代码
	String factoryBeanName = mbd.getFactoryBeanName();
	if (factoryBeanName != null) {
		if (factoryBeanName.equals(beanName)) {
			throw new BeanDefinitionStoreException(mbd.getResourceDescription(), beanName,
					"factory-bean reference points back to the same bean definition");
		}
		// 先获取factoryBean，没有就创建。。。到这就验证了上边说的步骤
		factoryBean = this.beanFactory.getBean(factoryBeanName);
		if (mbd.isSingleton() && this.beanFactory.containsSingleton(beanName)) {
			throw new ImplicitlyAppearedSingletonException();
		}
		factoryClass = factoryBean.getClass();
		isStatic = false;
	}

```

### 解决方式

将 MethodValidationPostProcessor的配置单独放到一个配置文件。

```
@Configuration
public class ValidationConfigBean {

@Bean
public MethodValidationPostProcessor methodValidationPostProcessor(){

    return new MethodValidationPostProcessor();
}
```

### 总结

1. 出现这个问题还是对spring加载流程不是特别清楚导致的,比如解析@Configuration下@Bean标识的对象时具体解析规则、之前只关注了执行init方法前后的逻辑，没有关注创建对象实例的方法
2. 项目中的配置文件能按功能差分就拆分，如果需要依赖业务bean,就将这一部分抽成业务组件，而不是放在公共的ConfigBean中。



## dubbo SPI机制
1. 是对jdk spi的增强，配置格式是 key=value的格式，从而可以根据key做到按需加载
2. 与自适应扩展注解@Adaptive配合使用可以做到运行时根据url配置动态加载需要的实现类。
3. 有些拓展并不想在框架启动阶段被加载，而是希望在拓展方法被调用时，根据运行时参数进行加载。

## 扩展加载器 ExtensionLoader
只有一个私有的构造方法，但是提供了静态的获取实例的方法

```java
public static <T> ExtensionLoader<T> getExtensionLoader(Class<T> type) {
    if (type == null) {
        throw new IllegalArgumentException("Extension type == null");
    }
    // 必须是接口类型
    if (!type.isInterface()) {
        throw new IllegalArgumentException("Extension type (" + type + ") is not an interface!");
    }
    // 必须有@SPI注解
    if (!withExtensionAnnotation(type)) {
        throw new IllegalArgumentException("Extension type (" + type +
                ") is not an extension, because it is NOT annotated with @" + SPI.class.getSimpleName() + "!");
    }
	/**
		*根据代码逻辑看出每个type下的ExtensionLoader是个单利，但是奇怪为什么没有使用双重校验加锁的方式
		*只做了一次校验，虽然有putIfAbsent保证map中不会有脏数据，但是还是有可能创建了多个ExtensionLoader的实例
	*/
    ExtensionLoader<T> loader = (ExtensionLoader<T>) EXTENSION_LOADERS.get(type);
    if (loader == null) {
        EXTENSION_LOADERS.putIfAbsent(type, new ExtensionLoader<T>(type));
        loader = (ExtensionLoader<T>) EXTENSION_LOADERS.get(type);
    }
    return loader;
}
```
里边提供了很多扩展方法，目前我只看了两个方法，后边在补充其他的

###1. 根据扩展名加载扩展实例
#### 1.1 使用demo
```java
// 配置文件
dubbo=org.apache.dubbo.rpc.protocol.dubbo.DubboProtocol
// 获取指定实例
Protocol dubboProtocol = ExtensionLoader.getExtensionLoader(Protocol.class).getExtension("dubbo");
```
#### 1.2 获取扩展 getExtension(String name)
```java
// getExtensionLoader是静态获取实例的方法上边已经说过了，下边看下 getExtension(String name)
public T getExtension(String name) {
	// name不能为空
	if (StringUtils.isEmpty(name)) {
	    throw new IllegalArgumentException("Extension name == null");
	}
	// 如果为‘true’则根据@SPI的value值进行加载
	if ("true".equals(name)) {
	    return getDefaultExtension();
	}
	// 如果Holder为空创建一个空的Holder
	Holder<Object> holder = getOrCreateHolder(name);
	// 尝试从缓存获取
	Object instance = holder.get();
	// 这里又使用了双重校验加锁
	if (instance == null) {
	    synchronized (holder) {
	        instance = holder.get();
	        if (instance == null) {
	        	  // 缓存命中失败就创建
	            instance = createExtension(name);
	            holder.set(instance);
	        }
	    }
	}
	return (T) instance;
}
```
总结一下逻辑：就是尝试从缓存获取，如果获取失败就创建
#### 1.3 创建扩展 createExtension(String name)
```
private T createExtension(String name) {
	// 加载当前type下的所有实现，如果为空抛异常
	//这里先加载了当前type下所有实现类的全路径名，然后通过Class.forName创建其Class对象，但是并没有创建对应的实例
    Class<?> clazz = getExtensionClasses().get(name);
    if (clazz == null) {
        throw findException(name);
    }
    try {
    	 // 尝试从缓存获取实例
        T instance = (T) EXTENSION_INSTANCES.get(clazz);
        if (instance == null) {
        	  // 获取失败后创建实例
            EXTENSION_INSTANCES.putIfAbsent(clazz, clazz.newInstance());
            instance = (T) EXTENSION_INSTANCES.get(clazz);
        }
        // 暂时不清楚作用
        injectExtension(instance);
        Set<Class<?>> wrapperClasses = cachedWrapperClasses;
        /**
        	* wrapperClasses是在getExtensionClasses()的时候维护的
        	* 如果有wrapper则将刚创建的实例放入wrapper并返回wrapper典型的就是
        	* ProtocolFilterWrapper、 ProtocolListenerWrapper
        	* 
       if (CollectionUtils.isNotEmpty(wrapperClasses)) {
            for (Class<?> wrapperClass : wrapperClasses) {
                instance = injectExtension((T) wrapperClass.getConstructor(type).newInstance(instance));
            }
        }
        return instance;
    } catch (Throwable t) {
        throw new IllegalStateException("Extension instance (name: " + name + ", class: " +
                type + ") couldn't be instantiated: " + t.getMessage(), t);
    }
}
```
总结一下逻辑:

1. 加载type下所有实现类的class
2. 通过name获取指定class,创建实现类的实例
3. 如果有wrapperClasses则使用wrapper进行装饰
4. 返回

##### 1.3.1  加载type下所有实现类的class getExtensionClasses()
``` java
该方法的链路比较长，但是逻辑比较清晰，就是根当前type的全名作为路径到指定为值去加载，直接到最后的方法
/**
  * 以这个配置为例 dubbo=org.apache.dubbo.rpc.protocol.dubbo.DubboProtocol
  * extensionClasses: 缓存配置加载出来的内容 key= dubbo;value = DubboProtocol.class
  * resourceURL: 路径下的资源，即dubbo默认加载的路径+type的全路径名下的配置
  * clazz: DubboProtocol.class
  * name: dubb0
*/	
private void loadClass(Map<String, Class<?>> extensionClasses, java.net.URL resourceURL, Class<?> clazz, String name) throws NoSuchMethodException {
    // 如果当前类上有@Adaptive，缓存在cachedAdaptiveClass中    
    if (clazz.isAnnotationPresent(Adaptive.class)) {
        cacheAdaptiveClass(clazz);
    /**
    	*如果是wrapper，缓存在cachedWrapperClasses中，    		*判断条件是如果当前class如果有一个当前type为参数 的构造方法
    	* 
    } else if (isWrapperClass(clazz)) {
        cacheWrapperClass(clazz);
    // 处理类上没有@Adaptive，当前class也不是wrapper的普通扩展
    } else {
        clazz.getConstructor();
        // name如果为空就是用当前class的类名如：dubbo.protocol
        if (StringUtils.isEmpty(name)) {
            name = findAnnotationName(clazz);
            if (name.length() == 0) {报错 }
        }
		 // 维护其他的缓存
        String[] names = NAME_SEPARATOR.split(name);
        if (ArrayUtils.isNotEmpty(names)) {
            cacheActivateClass(clazz, names[0]);
            for (String n : names) {
                cacheName(clazz, n);
                saveInExtensionClass(extensionClasses, clazz, name);
            }
        }
    }
}

```

###2. 自适应扩展
####2.1 使用demo
```
// 配置文件
dubbo=org.apache.dubbo.rpc.protocol.dubbo.DubboProtocol
// 获取指定实例
Protocol dubboProtocol = ExtensionLoader.getExtensionLoader(Protocol.class).getAdaptiveExtension();
```
#### 2.2 加载自适应扩展 getAdaptiveExtension()
```java
 public T getAdaptiveExtension() {
 	/**
 		*老套路先尝试从缓存获取，这个缓存和getExtension(String name)不一样
 		* 此处是Holder<Object> cachedAdaptiveInstance
 		* getExtension(String name)是ConcurrentMap<String, Holder<Object>> cachedInstances
 	 */
    Object instance = cachedAdaptiveInstance.get();
    if (instance == null) {
        if (createAdaptiveInstanceError == null) {
            synchronized (cachedAdaptiveInstance) {
                instance = cachedAdaptiveInstance.get();
                if (instance == null) {
                    try {
                    	// 获取失败就创建
                        instance = createAdaptiveExtension();
                        cachedAdaptiveInstance.set(instance);
                    } catch (Throwable t) {抛异常}
                }
            }
        } else {抛异常}
    }

    return (T) instance;
}
```
#### 2.3 创建自适应扩展(其实就是创建了一个代理，代理方法先从URL获取extName,然后在是用扩展加载指定的实现类)
```java
// 包含三个逻辑
1. getAdaptiveExtensionClass()
2. newInstance()
3. injectExtension
private T createAdaptiveExtension() {
        
    return injectExtension((T) getAdaptiveExtensionClass().newInstance());     
}

// 看下getAdaptiveExtensionClass()
private Class<?> getAdaptiveExtensionClass() {
	 // 这个上边已经说过了，就是获取当前type下的所有实现的Class
    getExtensionClasses();
    // 再次尝试从缓存获取
    if (cachedAdaptiveClass != null) {
        return cachedAdaptiveClass;
    }
    // 创建并赋值
    return cachedAdaptiveClass = createAdaptiveExtensionClass();
}

// 创建自适应扩展
private Class<?> createAdaptiveExtensionClass() {
		 // 根据当期那type生成扩展代理的java字符串代码
        String code = new AdaptiveClassCodeGenerator(type, cachedDefaultName).generate();
        ClassLoader classLoader = findClassLoader();
        Compiler compiler = ExtensionLoader.getExtensionLoader(Compiler.class).getAdaptiveExtension();
        // 使用编译器编译并返回自适应扩展对象
        return compiler.compile(code, classLoader);
    }
```
#### 2.4 生成字符串代码的逻辑 generate()
```java
public String generate() {
    // type至少一个方法需要有@Adaptive 
    if (!hasAdaptiveMethod()) {抛异常 }

    StringBuilder code = new StringBuilder();
    
    // package信息:
    code.append(generatePackageInfo());
    
    // import信息
    code.append(generateImports());
    
    // 代理类名：type类名+$Adaptive，并继承当前type
    code.append(generateClassDeclaration());
    
    // 处理方法：包括无@Adaptive的方法，有@Adaptive的方法，并且会找URL对象
    Method[] methods = type.getMethods();
    for (Method method : methods) {
        code.append(generateMethod(method));
    }
    code.append("}");
    
   
    return code.toString();
}
```
#### 2.5 处理方法细节 generateMethod(method)
```java
private String generateMethod(Method method) {
	// 方法返回值
    String methodReturnType = method.getReturnType().getCanonicalName();
    // 方法名称
    String methodName = method.getName();
    // 构建方法内的逻辑
    String methodContent = generateMethodContent(method);
    // 方法参数
    String methodArgs = generateMethodArguments(method);
    // 方法返回异常
    String methodThrows = generateMethodThrows(method);
    return String.format(CODE_METHOD_DECLARATION, methodReturnType, methodName, methodArgs, methodThrows, methodContent);
    }
    
// 我们重点看下 generateMethodContent 构建方法内的逻辑
private String generateMethodContent(Method method) {
	// 获取方法上的Adaptive注解
    Adaptive adaptiveAnnotation = method.getAnnotation(Adaptive.class);
    StringBuilder code = new StringBuilder(512);
    // 如果没有该注解，方法逻辑就是直接抛异常
    if (adaptiveAnnotation == null) {
        return generateUnsupported(method);
    } else {
    	 // 获取URL在方法参数的位置
        int urlTypeIndex = getUrlTypeIndex(method);
        
        // 如果有URL
        if (urlTypeIndex != -1) {
            // 方法逻辑中增加 URL != null的判断
            code.append(generateUrlNullCheck(urlTypeIndex));
        // 参数中没有URL
        } else {
            // 遍历方法参数，在参数的方法中查找返回值为URL的方法，并增加非空判断和获取URL的逻辑
            code.append(generateUrlAssignmentIndirectly(method));
        }
		 // @Adaptive的value值
        String[] value = getMethodAdaptiveValue(adaptiveAnnotation);
        
		 // 方法参数中是否有Invocation类型，下边要做特殊处理
        boolean hasInvocation = hasInvocationArgument(method);
        
        // 校验Invocation不为空的逻辑
        code.append(generateInvocationArgumentNullCheck(method));
        
        /**
        	*通过value获取extName的逻辑也就是获取配置文件的key
        	*这里有特殊处理比如
        	*如果value中有protocol：
        	*    直接通过url.getProtocol()获取，获取不到会使用@SPI中的值作为默认值
        	*如果value中没有有protocol且有Invocation类型的参数：
        	*    通过url.getMethodParameter获取
        	* 如果value中没有protocol，且没有Invocation类型的参数：
        	* 	  通过url.getParameter获取   
        	*
        */ 
        code.append(generateExtNameAssignment(value, hasInvocation));
        
        // 校验extName不能为空
        code.append(generateExtNameNullCheck(value));
        
        /**
        	*此时已经从URL获取到extName,也就是配置文件的key,通过getExtetion(String name)加载扩展
        	*com.alibaba.dubbo.rpc.Protocol extension = 
        	* ExtensionLoader.getExtensionLoader(Protocol.class).getExtension(extName);
		  */
        code.append(generateExtensionAssignment());

        /**
        	*使用加载到的扩展执行方法，并返回结果
        	*return extension.refer(arg0, arg1);
		  */
        code.append(generateReturnAndInvocation(method));
    }
    
    return code.toString();
}

上边代码都是在拼接字符串
if (adaptiveAnnotation == null) {
    // $无 Adaptive 注解方法代码生成逻辑，直接抛异常
} else {
    // ${获取 URL 数据}:从方法参数，或者方法参数对象的方法中获取，获取失败抛异常
    
    // ${获取 Adaptive 注解值}
    
    // ${检测 Invocation 参数}
    
    // ${生成拓展名获取逻辑}
    
    // ${生成拓展加载与目标方法调用逻辑}
}
```
总结下generateMethodContent(Method method)：

1. 方法上无注解生成代码的逻辑是抛异常
2. 方法上有注解先找到URL对象，找不到就抛异常
3. 通过URL获取extName，如果没有默认使用@SPI value中的值
4. 通过extName 加载指定的实现类
5. 通过加载到的实现类执行方法并返回


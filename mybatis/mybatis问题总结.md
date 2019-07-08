## 问题描述：

1.mybatis大体执行流程？

2.${}和#{}的区别？

3.拦截器执行原理？

mybatis demo代码

```java
// 获取 sqlSessionFactory
SqlSessionFactory sqlSessionFactory = getSqlSessionFactory();

// 获取sqlSession
SqlSession sqlSession = sqlSessionFactory.openSession();

// 获取mapper(和下边获取Mapper方法任意一种均可)
//Object findById = sqlSession.selectOne("com.morningglory.mybatis.mapper.StudentMapper.findById", 1L);
//log.info(JSON.toJSONString(findById));

    
// 获取mapper
StudentMapper mapper = sqlSession.getMapper(StudentMapper.class);
Student student = mapper.findById(1L);
log.info(JSON.toJSONString(student));


private static SqlSessionFactory getSqlSessionFactory() throws IOException {
    String resource = "mybatis-config.xml";
    InputStream inputStream = Resources.getResourceAsStream(resource);
    SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);
    return sqlSessionFactory;
}
```
配置文件

```
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE configuration
        PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-config.dtd">
<configuration>

    <settings>
        <setting name="cacheEnabled" value="true"/>
    </settings>

    <typeAliases>
        <package name="com.morningglory.mybatis.pojo"/>
    </typeAliases>

    <plugins>
        <plugin interceptor="com.morningglory.mybatis.plugins.PrepareInterceptor" />
        <plugin interceptor="com.morningglory.mybatis.plugins.ExecutorInterceptor" />
        <plugin interceptor="com.morningglory.mybatis.plugins.ParameterInterceptor" />
    </plugins>

    <environments default="development">
        <environment id="development">
            <transactionManager type="JDBC"/>
            <dataSource type="POOLED">
                <property name="driver" value="com.mysql.jdbc.Driver"/>
                <property name="url" value="jdbc:mysql://127.0.0.1:3306/demo?characterEncoding=utf8&amp;useSSL=true"/>
                <property name="username" value="root"/>
                <property name="password" value="123456"/>
            </dataSource>
        </environment>
    </environments>

    <mappers>
        <!--<package name="mapper/*"/>-->
        <mapper resource="mapper/StudentMapper.xml" />
    </mappers>


</configuration>
```
根据demo可以大致看出执行流程：

-	获取SqlSessionFactory对象
-	根据SqlSessionFactory创建SqlSession对象
- 	直接调用SqlSession API执行或者先获取Mapper对象在执行
-  但是我们并不能很好的理解这几个对象的作用，下边通过代码看下

### 1、大体流程
#### 1.1 创建SqlSessionFactory
创建SqlSessionFactory过程中有个非常重要的步骤就是加载并解析配置文件，通过demo代码可以看出是通过new SqlSessionFactoryBuilder().build(inputStream)创建的，其中inputStream为配置文件,直接看下build方法

```java
public SqlSessionFactory build(InputStream inputStream, String environment, Properties properties) {
  // 根据配置文件流创建
  XMLConfigBuilder parser = new XMLConfigBuilder(inputStream, environment, properties);
  return build(parser.parse());
}

// 没有任何逻辑，就是创建对象
public SqlSessionFactory build(Configuration config) {
	return new DefaultSqlSessionFactory(config);
}

// parser.parse()
public Configuration parse() {
	if (parsed) {
	  throw new BuilderException("Each XMLConfigBuilder can only be used once.");
	}
	parsed = true;
	parseConfiguration(parser.evalNode("/configuration"));
	return configuration;
}
  
// parser.parse()最终调用了parseConfiguration
private void parseConfiguration(XNode root) {
try {
  // 这里具体的解析方法暂时不跟进，但是通过命名可以看出是对各个标签的解析
  // 解析properties标签
  propertiesElement(root.evalNode("properties"));
  // 解析settings标签
  Properties settings = settingsAsProperties(root.evalNode("settings"));
  loadCustomVfs(settings);
  // 解析 typeAliases标签
  typeAliasesElement(root.evalNode("typeAliases"));
  // 解析 plugins标签
  pluginElement(root.evalNode("plugins"));
  objectFactoryElement(root.evalNode("objectFactory"));
  objectWrapperFactoryElement(root.evalNode("objectWrapperFactory"));
  reflectorFactoryElement(root.evalNode("reflectorFactory"));
  settingsElement(settings);
  // 解析 environments标签
  environmentsElement(root.evalNode("environments"));
  databaseIdProviderElement(root.evalNode("databaseIdProvider"));
  // 解析typeHandler标签
  typeHandlerElement(root.evalNode("typeHandlers"));
  // 解析mappers标签
  mapperElement(root.evalNode("mappers"));
} 
}
```
所以我们可以总结下SqlSessionFactory创建流程

-	读取配置文件，调用parseConfiguration方法解析各种标签
- 	组装解析结果创建 Configuration对象(非常重要，可以类比spring的ApplicationContext)
-  通过build创建SqlSessionFactory，其实就是设置config属性的值

#### 1.2 创建SqlSession
因为SqlSession是通过SqlSessionFactory创建的，而SqlSessionFactory中也只有Configuration一个属性，所以猜想他是通过获取解析的配置信息创建的
```java
private SqlSession openSessionFromDataSource(ExecutorType execType, TransactionIsolationLevel level, boolean autoCommit) {
Transaction tx = null;
try {
  // 从配置中获取Environment(一般这里放的是jdbc的配置)
  final Environment environment = configuration.getEnvironment();
  // 创建事务工厂
  final TransactionFactory transactionFactory = getTransactionFactoryFromEnvironment(environment);
  tx = transactionFactory.newTransaction(environment.getDataSource(), level, autoCommit);
  // 创建执行器
  final Executor executor = configuration.newExecutor(tx, execType);
  // 组装上边创建的参数，并创建SqlSession
  return new DefaultSqlSession(configuration, executor, autoCommit);
} catch (Exception e) {
  closeTransaction(tx); // may have fetched a connection so lets call close()
  throw ExceptionFactory.wrapException("Error opening session.  Cause: " + e, e);
} finally {
  ErrorContext.instance().reset();
}
}
```
通过上边的代码可以总结SqlSession的创建及其作用

-	从配置中获取jdbc相关配置，并创建事务
- 	创建具体的执行器

#### 1.3 获取Mapper
```java
public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
final MapperProxyFactory<T> mapperProxyFactory = (MapperProxyFactory<T>) knownMappers.get(type);
if (mapperProxyFactory == null) {
  throw new BindingException("Type " + type + " is not known to the MapperRegistry.");
}
try {
	// 创建代理
  return mapperProxyFactory.newInstance(sqlSession);
} 
}

public T newInstance(SqlSession sqlSession) {
final MapperProxy<T> mapperProxy = new MapperProxy<T>(sqlSession, mapperInterface, methodCache);
return newInstance(mapperProxy);
}
 
protected T newInstance(MapperProxy<T> mapperProxy) {
return (T) Proxy.newProxyInstance(mapperInterface.getClassLoader(), new Class[] { mapperInterface }, mapperProxy);
} 
 
// 代理类的invoke方法
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
try {
  if (Object.class.equals(method.getDeclaringClass())) {
    return method.invoke(this, args);
  } else if (isDefaultMethod(method)) {
    return invokeDefaultMethod(proxy, method, args);
  }
} catch (Throwable t) {
  throw ExceptionUtil.unwrapThrowable(t);
}
final MapperMethod mapperMethod = cachedMapperMethod(method);
// 
return mapperMethod.execute(sqlSession, args);
}

// 指定具体调用SqlSession的哪个方法
public Object execute(SqlSession sqlSession, Object[] args) {
Object result;
switch (command.getType()) {
  case INSERT: {
  Object param = method.convertArgsToSqlCommandParam(args);
    result = rowCountResult(sqlSession.insert(command.getName(), param));
    break;
  }
  case UPDATE: {
    Object param = method.convertArgsToSqlCommandParam(args);
    result = rowCountResult(sqlSession.update(command.getName(), param));
    break;
  }
  case DELETE: {
    Object param = method.convertArgsToSqlCommandParam(args);
    result = rowCountResult(sqlSession.delete(command.getName(), param));
    break;
  }
  case SELECT:
    if (method.returnsVoid() && method.hasResultHandler()) {
      executeWithResultHandler(sqlSession, args);
      result = null;
    } else if (method.returnsMany()) {
      result = executeForMany(sqlSession, args);
    } else if (method.returnsMap()) {
      result = executeForMap(sqlSession, args);
    } else if (method.returnsCursor()) {
      result = executeForCursor(sqlSession, args);
    } else {
      Object param = method.convertArgsToSqlCommandParam(args);
      result = sqlSession.selectOne(command.getName(), param);
    }
    break;
  case FLUSH:
    result = sqlSession.flushStatements();
    break;
  default:
    throw new BindingException("Unknown execution method for: " + command.getName());
}
if (result == null && method.getReturnType().isPrimitive() && !method.returnsVoid()) {
  throw new BindingException("Mapper method '" + command.getName() 
      + " attempted to return null from a method with a primitive return type (" + method.getReturnType() + ").");
}
return result;
}
```
获取Mapper的过程

-	根据SqlSession创建MapperProxy,且改对象继承了InvocationHandler
- 	基于我们的Mapper创建代理
-  创建代理后会调用代理类的execute方法，execute方法中做了判断，会调用SqlSession对应的API
-  也可以不获取Mapper，就像demo中第一种执行的方式，其实获取Mapper就是让我们不关心SqlSession中的方法，他的原理是通过代理自动关联到对应的方法

### 1.4 SqlSession执行方法(以slectOne为例)
-	selectOne调用selectList方法
- 	根据statementId 获取MappedStatement，并执行Executor的query方法
-  创建CacheKey并尝试从缓存获取数据，获取不到则从数据库获取
-  执行预处理(设置占位符的值） SimpleExecutor.prepareStatement()
-  因为MappedStatement的statementType默认为PREPARED，所以执行PreparedStatementHandler的query()
-  PreparedStatement.excute() (jdbc的方法)
-  result处理

这只是大体流程，里边还有很多的细节没有说到，而且我们看上边的大体流程是不能回答提出的后两个问题的。。。。

### 2、 ${} 和 #{}的区别
还记得解析mappers标签的mapperElement(root.evalNode("mappers"))方法么

```java
public void parse() {
	parsePendingResultMaps();
	parsePendingCacheRefs();
	// Statement是与Sql相关的
	parsePendingStatements();
}

// 这里省略了大量代码，只看Sql的解析
public void parseStatementNode() {
	// 省略其他代码
	String lang = context.getStringAttribute("lang");
    LanguageDriver langDriver = getLanguageDriver(lang);
    // 将xml中的sql转为对象，这里不管是$还是#都是不处理的，即和xml保持一致
    SqlSource sqlSource = langDriver.createSqlSource(configuration, context, parameterTypeClass);
    
    // 省略其他代码
}

// 解析方法执行完意味着SqlSessionFactory创建成功，下边就是执行Sql前的处理，当调用SqlSource.getBoundSql时有个处理的逻辑
public BoundSql getBoundSql(Object parameterObject) {
	DynamicContext context = new DynamicContext(configuration, parameterObject);
	// 此方法将${}用ognl表达式全部替换为具体的值
	rootSqlNode.apply(context);
	SqlSourceBuilder sqlSourceParser = new SqlSourceBuilder(configuration);
	Class<?> parameterType = parameterObject == null ? Object.class : parameterObject.getClass();
	// 处理#{},并使用？做为占位符，创建ParameterMapping对象
	SqlSource sqlSource = sqlSourceParser.parse(context.getSql(), parameterType, context.getBindings());
	BoundSql boundSql = sqlSource.getBoundSql(parameterObject);
	for (Map.Entry<String, Object> entry : context.getBindings().entrySet()) {
	  boundSql.setAdditionalParameter(entry.getKey(), entry.getValue());
	}
	return boundSql;
}
```
所以$和#处理的顺序是

-	获取xml中的sql
- 	getBoundSql时将${}的值通过OGNL表达式进行替换，换成实际的值，将#{}用？替换，并生成对应的ParameterMapping
- 	预处理时通过SimpleExecutor.prepareStatement 设置对应的值
- 	所以可以看出${}是mybatis自己进行处理的，#{}是mybatis交给prepareStatement处理的
-  可以很负责任的说所有能用#{}的地方都可以用${}进行替换，前提是不考虑sql注入的问题，而有些${}却不能被#{}替换，具体可以写一个jdbc demo,比如tableName,orderBy都不能用#{},因为在处理的时候会在原值的基础上加上两个单引号。

### 3、拦截器执行原理
依然是从解析配置文件开始，pluginElement(root.evalNode("plugins"))

```java
private void pluginElement(XNode parent) throws Exception {
if (parent != null) {
  for (XNode child : parent.getChildren()) {
  // 配置文件中配的是拦截器的全路径名
    String interceptor = child.getStringAttribute("interceptor");
    Properties properties = child.getChildrenAsProperties();
    // 通过Class.forName创建实例
    Interceptor interceptorInstance = (Interceptor) resolveClass(interceptor).newInstance();
    interceptorInstance.setProperties(properties);
    // 加入到拦截器链
    configuration.addInterceptor(interceptorInstance);
  }
}
}

// 在执行具体方法时(excuter.doxxx)会通过config创建对应的Handler,这里有个代理的逻辑
public StatementHandler newStatementHandler(Executor executor, MappedStatement mappedStatement, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) {
	StatementHandler statementHandler = new RoutingStatementHandler(executor, mappedStatement, parameterObject, rowBounds, resultHandler, boundSql);
	// 遍历所有拦截器并执行其plugin方法
	statementHandler = (StatementHandler) interceptorChain.pluginAll(statementHandler);
	return statementHandler;
}

// 代理实现细节 target为创建的Handler，interceptor为我们自定义的拦截器
public static Object wrap(Object target, Interceptor interceptor) {
	Map<Class<?>, Set<Method>> signatureMap = getSignatureMap(interceptor);
	Class<?> type = target.getClass();
	Class<?>[] interfaces = getAllInterfaces(type, signatureMap);
	if (interfaces.length > 0) {
	  // 对Handler做一个代理
	  return Proxy.newProxyInstance(
	      type.getClassLoader(),
	      interfaces,
	      new Plugin(target, interceptor, signatureMap));
	}
return target;
}

// 代理的具体逻辑
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
try {
  // 根据方法匹配，如果匹配成功，就执行拦截器的方法
  Set<Method> methods = signatureMap.get(method.getDeclaringClass());
  if (methods != null && methods.contains(method)) {
    return interceptor.intercept(new Invocation(target, method, args));
  }
  // 否则直接执行目标方法
  return method.invoke(target, args);
} catch (Exception e) {
  throw ExceptionUtil.unwrapThrowable(e);
}
}
```
所以拦截器原理为:

-	通过配置文件创建所有拦截器实例
- 	在创建Handler时会通过代理创建Handler的代理类
-  代理类的逻辑就是根据方法匹配，如果匹配成功就执行拦截器方法，否则直接执行目标方法。

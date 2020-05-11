## dubbo-http协议
主要是通过集成jsonRpc来实现
### 服务导出
1.根据dubbo协议一样,先是通过代理生成Invoker对象(doInvoke方法调用Wrapper对象的doInvokeMethod方法,Wrapper对象是基于目标接口做的代理)
2.根据URL中的协议配置,由SPI实例化对应的协议对象，这里是HttpProtocol

#### 导出代码
通过AbstractProxyProtocol的export方法进行暴露(dubbo协议是重写了export方法)
```
public <T> Exporter<T> export(final Invoker<T> invoker) throws RpcException {
	// 构造serviceKey 格式为:serviceGroup/serviceName:serviceVersion:port
    final String uri = serviceKey(invoker.getUrl());

    // 尝试从缓存获取,每次暴露后会放入缓存
    Exporter<T> exporter = (Exporter<T>) exporterMap.get(uri);
    if (exporter != null) {
        // When modifying the configuration through override, you need to re-expose the newly modified service.
        if (Objects.equals(exporter.getInvoker().getUrl(), invoker.getUrl())) {
            return exporter;
        }
    }

    // 具体如何暴露由子类实现,这里可能是因为方便调用所以返回了Runable对象，但是他跟线程没有关系
    final Runnable runnable = doExport(proxyFactory.getProxy(invoker, true), invoker.getInterface(), invoker.getUrl());
    exporter = new AbstractExporter<T>(invoker) {
        @Override
        public void unexport() {
            super.unexport();
            exporterMap.remove(uri);
            if (runnable != null) {
                try {
                    runnable.run();
                } catch (Throwable t) {
                    logger.warn(t.getMessage(), t);
                }
            }
        }
    };
    exporterMap.put(uri, exporter);
    return exporter;
}
```
具体的导出实现
```
protected <T> Runnable doExport(final T impl, Class<T> type, URL url) throws RpcException {
    // 获取ip:port
    String addr = getAddr(url);
    // 尝试从缓存获取
    ProtocolServer protocolServer = serverMap.get(addr);
    if (protocolServer == null) {
        // 开启服务端的服务,根据URL的配置可以是tomcat或者jetty，同时把InternalHandler加入sevlet的service方法中
        RemotingServer remotingServer = httpBinder.bind(url, new InternalHandler(url.getParameter("cors", false)));
        // 放入缓存
        serverMap.put(addr, new ProxyProtocolServer(remotingServer));
    }
    final String path = url.getAbsolutePath();
    // 创建JsonRpcServer对象，需要制定接口实现类和接口类型
    JsonRpcServer skeleton = new JsonRpcServer(impl, type);
    // 将创建好的 JsonRpcServer放入缓存
    skeletonMap.put(path, skeleton);
    // 这里就是上边方法说的返回了Runable对象，调用unexport时才会触发删除缓存
    return () -> skeletonMap.remove(path);
}

InternalHandler是HttpProtocol的内部类，比较重要，他会放入servlet中，处理http请求
private class InternalHandler implements HttpHandler {
    // 处理跨域的一些信息
    private boolean cors;

    public InternalHandler(boolean cors) {
        this.cors = cors;
    }

    // 此方法由servlet的service触发执行(也就是http请求过来以后是由handler负责处理的)
    @Override
    public void handle(HttpServletRequest request, HttpServletResponse response)
            throws ServletException {
        String uri = request.getRequestURI();
        // 从HttpProtocol的缓存中获取JsonRpcServer,这个在导出的时候回放入
        JsonRpcServer skeleton = skeletonMap.get(uri);
        if (cors) {
            response.setHeader(ACCESS_CONTROL_ALLOW_ORIGIN_HEADER, "*");
            response.setHeader(ACCESS_CONTROL_ALLOW_METHODS_HEADER, "POST");
            response.setHeader(ACCESS_CONTROL_ALLOW_HEADERS_HEADER, "*");
        }
        if ("OPTIONS".equalsIgnoreCase(request.getMethod())) {
            response.setStatus(200);
        } else if ("POST".equalsIgnoreCase(request.getMethod())) {

            RpcContext.getContext().setRemoteAddress(request.getRemoteAddr(), request.getRemotePort());
            try {
                // 收到请求后交由JsonRpcServer处理
                skeleton.handle(request.getInputStream(), response.getOutputStream());
            } catch (Throwable e) {
                throw new ServletException(e);
            }
        } else {
            response.setStatus(500);
        }
    }

}
```
JsonRpcServer的处理流程(只截取部分代码)
```
// ObjectNode是有request.getInputStream()发序列化来的
public int handleObject(ObjectNode node, OutputStream ops){
        
        int returnCode = 0;
        // 获取请求参数，这些参数是在JspnRpcClient中传的,而且字段名都是固定的
        JsonNode jsonPrcNode    = node.get("jsonrpc");
        JsonNode methodNode     = node.get("method");
        JsonNode idNode         = node.get("id");
        JsonNode paramsNode     = node.get("params");

        // 获取请求参数的值,到这相当于取到了要调用的接口名、方法名、方法参数
        String jsonRpc      = (jsonPrcNode!=null && !jsonPrcNode.isNull()) ? jsonPrcNode.asText() : "2.0";
        String methodName   = getMethodName(methodNode);
        String serviceName  = getServiceName(methodNode);
        Object id           = parseId(idNode);

        // 先匹配方法名相同的方法
        Set<Method> methods = new HashSet<Method>();
        methods.addAll(findMethods(getHandlerInterfaces(serviceName), methodName));
        if (methods.isEmpty()) {
            writeAndFlushValue(ops, createErrorResponse(
                jsonRpc, id, -32601, "Method not found", null));
            return -32601;
        }

        // 在根据方法参数匹配,找到以后用MethodAndArgs封装起来
        MethodAndArgs methodArgs = findBestMethodByParamsNode(methods, paramsNode);
        if (methodArgs==null) {
            writeAndFlushValue(ops, createErrorResponse(
                jsonRpc, id, -32602, "Invalid method parameters", null));
            return -32602;
        }

        // 执行目标方法；getHandler(serviceName)是获取接口的实现类实例，这个是在创建JsonRpcServer时设置的
        JsonNode result = invoke(getHandler(serviceName), methodArgs.method, methodArgs.arguments);

        // 就是将result和id塞了进去
        ObjectNode response = createSuccessResponse(jsonRpc, id, result);
        
        // 通过outputstream写出
        writeAndFlushValue(ops, response);
        
        // 返回状态码
        return returnCode;
    }

```
#### 总结导出过程
1.重写doExport方法,自定义导出逻辑
2.创建RemoteServer用来接收http请求,并在servlet中加入InternalHandler,用来处理http请求
3.创建JsonRpcServer并指定接口和实现类，并使用uri作为key放入缓存
4.InternalHandler处理时先根据uri获取JsonRpcServer,然后调用它的invoke方法
5.调用过程是获取http请求的数据，获取到方法名、参数名等信息,然后通过反射调用，并将结果写完输出流去

### 服务引用
```
// refer方法就是返回Invoker对象
 protected <T> Invoker<T> protocolBindingRefer(final Class<T> type, final URL url) throws RpcException {
    // 这里又调用了doRefer方法,然后通过代理创建Invoker
    final Invoker<T> target = proxyFactory.getInvoker(doRefer(type, url), type, url);
    // 新建一个Invoker对象，重写doInvoke，由刚通过代理创建的Invoker对象执行，所以调用顺序为 Invoker -> Invoker(代理创建的) -> Wrapper
    Invoker<T> invoker = new AbstractInvoker<T>(type, url) {
        @Override
        protected Result doInvoke(Invocation invocation) throws Throwable {
            try {
                Result result = target.invoke(invocation);
                // FIXME result is an AsyncRpcResult instance.
                Throwable e = result.getException();
                if (e != null) {
                    for (Class<?> rpcException : rpcExceptions) {
                        if (rpcException.isAssignableFrom(e.getClass())) {
                            throw getRpcException(type, url, invocation, e);
                        }
                    }
                }
                return result;
            } catch (RpcException e) {
                if (e.getCode() == RpcException.UNKNOWN_EXCEPTION) {
                    e.setCode(getErrorCode(e.getCause()));
                }
                throw e;
            } catch (Throwable e) {
                throw getRpcException(type, url, invocation, e);
            }
        }
    };
    invokers.add(invoker);
    return invoker;
}
```
doRefer方法,主要是通过代理创建目标对象的代理,屏蔽远程通讯细节
```
protected <T> T doRefer(final Class<T> serviceType, URL url) throws RpcException {
    //这个对象是FactoryBean，同时继承了MethodInterceptor
    JsonProxyFactoryBean jsonProxyFactoryBean = new JsonProxyFactoryBean();
    jsonProxyFactoryBean.setServiceUrl(url.setProtocol("http").toIdentityString());
    jsonProxyFactoryBean.setServiceInterface(serviceType);

    jsonProxyFactoryBean.afterPropertiesSet();
    // 通过工厂方法获取创建的代理对象
    return (T) jsonProxyFactoryBean.getObject();
}
```
// 看下JsonProxyFactoryBean做的事情
```
// 继承了很多spring的bean
public class JsonProxyFactoryBean extends UrlBasedRemoteAccessor implements MethodInterceptor,InitializingBean,FactoryBean<Object>,
    ApplicationContextAware {

    public void afterPropertiesSet() {

        super.afterPropertiesSet();

        // 为目标接口创建代理
        proxyObject = ProxyFactory.getProxy(getServiceInterface(), this);

        // 这里省略了部分查找逻辑,与主流程无关
        objectMapper = new ObjectMapper();

        try {
            // 根据URL创建 JsonRpcHttpClient 根据ip:port创建java.net.URL
            jsonRpcHttpClient = new JsonRpcHttpClient(objectMapper, new URL(getServiceUrl()), extraHttpHeaders);
            jsonRpcHttpClient.setRequestListener(requestListener);
            jsonRpcHttpClient.setSslContext(sslContext);
            jsonRpcHttpClient.setHostNameVerifier(hostNameVerifier);
        } catch (MalformedURLException mue) {
            throw new RuntimeException(mue);
        }
    }

    public Object invoke(MethodInvocation invocation)
        throws Throwable {

        // 首先拦截到目标方法
        Method method = invocation.getMethod();
        if (method.getDeclaringClass() == Object.class && method.getName().equals("toString")) {
            return proxyObject.getClass().getName() + "@" + System.identityHashCode(proxyObject);
        }

        // 获取返回值
        Type retType = (invocation.getMethod().getGenericReturnType() != null)
            ? invocation.getMethod().getGenericReturnType()
            : invocation.getMethod().getReturnType();

        // 获取方法参数
        Object arguments = ReflectionUtil.parseArguments(
            invocation.getMethod(), invocation.getArguments(), useNamedParams);

        // 创建jsonRpcHttpClient并调用invoke方法
        return jsonRpcHttpClient.invoke(
            invocation.getMethod().getName(),
            arguments,
            retType, extraHttpHeaders);
    }
}

// jsonRpcHttpClient.invoke
public Object invoke(
        String methodName, Object argument, Type returnType,
        Map<String, String> extraHeaders)
        throws Throwable {

        // 处理链接
        HttpURLConnection con = prepareConnection(extraHeaders);
        con.connect();

        // 链接成功后获取输出流
        OutputStream ops = con.getOutputStream();
        try {
            // 调用方法，这里就是将参数封装成ObjectNode,并通过输出流写出,然后服务端接收到可以反序列化为ObjectNode,然后取方法和参数,然后执行
            super.invoke(methodName, argument, ops);
        } finally {
            ops.close();
        }

        // read and return value
        try {
            // 从输入流读取返回结果
            InputStream ips = con.getInputStream();
            try {
                // 根据返回值类型反序列化
                return super.readResponse(returnType, ips);
            } finally {
                ips.close();
            }
        } catch (IOException e) {
            throw new HttpException(readString(con.getErrorStream()), e);
        }
    }

```

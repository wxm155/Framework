# SpringMVC源码

## 1、父子容器初始化

在HttpServletBean中#init中调用FrameworkServlet#initServletBean进行父子容器的初始化，Springboot启动时不会进入，懒加载，第一次调用时进行子容器的初始化(Spring容器为父容器，SpringMVC容器为子容器)

```java
public final void init() throws ServletException {
		// 将servlet中配置的init-param参数封装到pvs变量中
		PropertyValues pvs = new ServletConfigPropertyValues(getServletConfig(), this.requiredProperties);
		if (!pvs.isEmpty()) {
			try {
              // 将当前的servlet对象转化成BeanWrapper对象，从而能够以spring的方法来将pvs注入到该beanWrapper中
				BeanWrapper bw = PropertyAccessorFactory.forBeanPropertyAccess(this);
				ResourceLoader resourceLoader = new ServletContextResourceLoader(getServletContext());
				bw.registerCustomEditor(Resource.class, new ResourceEditor(resourceLoader, getEnvironment()));
				initBeanWrapper(bw);
				bw.setPropertyValues(pvs, true);
			}
			catch (BeansException ex) {
			}
		}
		// 模板方法，SpringMVC容器初始化入口方法
		initServletBean();
}
// FrameworkServlet#initServletBean
protected final void initServletBean() throws ServletException {
		try {
          // 初始化WebApplicationContext
			this.webApplicationContext = initWebApplicationContext();
          // 模板方法
			initFrameworkServlet();
		}
		catch (ServletException | RuntimeException ex) {}
}
```

初始化SpringMVC容器

```java
protected WebApplicationContext initWebApplicationContext() {
     // 获取Spring容器，Springboot启动刷新容器
	WebApplicationContext rootContext =
			WebApplicationContextUtils.getWebApplicationContext(getServletContext());
	WebApplicationContext wac = null;
	if (this.webApplicationContext != null) {
		wac = this.webApplicationContext;
		if (wac instanceof ConfigurableWebApplicationContext) {
			ConfigurableWebApplicationContext cwac = (ConfigurableWebApplicationContext) wac;
             // 未激活，刷新容器
			if (!cwac.isActive()) {
				if (cwac.getParent() == null) {
					cwac.setParent(rootContext);
				}
              // 配置并刷新Spring容器
				configureAndRefreshWebApplicationContext(cwac);
			}
		}
	}
	if (wac == null) {
		// 从servletContext获取对应的webApplicationContext对象
		wac = findWebApplicationContext();
	}
	if (wac == null) {
		// 创建并刷新SpringMVC容器
		wac = createWebApplicationContext(rootContext);
	}
	if (!this.refreshEventReceived) {
		// 手动调用注册SpringMVC九大组件
		synchronized (this.onRefreshMonitor) {
			onRefresh(wac);
		}
	}
	if (this.publishContext) {
		// 将applicationContext设置到servletContext中
		String attrName = getServletContextAttributeName();
		getServletContext().setAttribute(attrName, wac);
	}
	return wac;
}

protected void onRefresh(ApplicationContext context) {
	initStrategies(context);
}
// 初始化九大组件
protected void initStrategies(ApplicationContext context) {
  // 初始化 MultipartResolver:主要用来处理文件上传
	initMultipartResolver(context); 
  // 初始化 LocaleResolver:主要用来处理国际化配置
	initLocaleResolver(context); 
  // 初始化 ThemeResolver:主要用来设置主题Theme
	initThemeResolver(context); 
  // 初始化 HandlerMapping:映射器，用来将对应的request跟controller进行对应
	initHandlerMappings(context); 
  // 初始化 HandlerAdapter:处理适配器
	initHandlerAdapters(context);
  // 初始化 HandlerExceptionResolver:基于HandlerExceptionResolver接口的异常处理
	initHandlerExceptionResolvers(context);
  // 初始化 RequestToViewNameTranslator:当controller处理器方法没有返回一个View对象或逻辑视图名称，并且在该方法中没有直接往response的输出流里面写数据的时候，spring将会采用约定好的方式提供一个逻辑视图名称
	initRequestToViewNameTranslator(context);
  // 初始化 ViewResolver: 将ModelAndView选择合适的视图进行渲染的处理器
	initViewResolvers(context);
  // 初始化 FlashMapManager: 提供请求存储属性，可供其他请求使用
	initFlashMapManager(context);
}
```

## 2、执行流程

SpringMVC执行流程DispatcherServlet#doDispatch

```java
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) {
	HttpServletRequest processedRequest = request;
	HandlerExecutionChain mappedHandler = null;
	boolean multipartRequestParsed = false;
  	// 获取异步管理器
	WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);
	try {
		ModelAndView mv = null;
		Exception dispatchException = null;
		try {
            // 检查是否是上传请求
			processedRequest = checkMultipart(request);
			multipartRequestParsed = (processedRequest != request);
			// 获得请求对应的HandlerExecutionChain对象（HandlerMethod和HandlerInterceptors）
			mappedHandler = getHandler(processedRequest);
			if (mappedHandler == null) {
				noHandlerFound(processedRequest, response);
				return;
			}
			// 获得当前handler对应的HandlerAdapter对象
			HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());
			// 处理GET、HEAD请求的Last-Modified,当浏览器第一次跟服务器请求资源时，服务器会在返回的请求			 // 头里包含一个last_modified的属性，代表资源最后时什么时候修改的，在浏览器以后发送请求的时			  // 候，会同时发送之前接收到的Last_modified.服务器接收到带last_modified的请求后，会跟实际			 // 资源的最后修改时间做对比，如果过期了返回新的资源，否则直接返回304表示未过期，直接使用之			 // 前缓存的结果即可
			String method = request.getMethod();
			boolean isGet = "GET".equals(method);
			if (isGet || "HEAD".equals(method)) {
				long lastModified = ha.getLastModified(request, mappedHandler.getHandler());
				if (new ServletWebRequest(request, response).checkNotModified(lastModified) && isGet) {return;}
			}
            // 执行HandlerInterceptor的前置回调
			if (!mappedHandler.applyPreHandle(processedRequest, response)) {
				return;
			}
			// 执行对应的方法，并返回视图
			mv = ha.handle(processedRequest, response, mappedHandler.getHandler());
			if (asyncManager.isConcurrentHandlingStarted()) {
				return;
			}
			applyDefaultViewName(processedRequest, mv);
          	// 执行HandlerInterceptor的后置回调
			mappedHandler.applyPostHandle(processedRequest, response, mv);
		}
        catch (Exception ex) {
        	dispatchException = ex;
        }
		catch (Throwable err) {
			dispatchException = new NestedServletException("Handler dispatch failed", err);
		}
      	// 处理返回结果，包括处理异常、渲染页面、触发Interceptor的afterCompletion
		processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);
	}
	catch (Exception ex) {
      	// 完成处理激活触发器
		triggerAfterCompletion(processedRequest, response, mappedHandler, ex);
	}
	catch (Throwable err) {
        // 完成处理激活触发器
		triggerAfterCompletion(processedRequest, response, mappedHandler,
				new NestedServletException("Handler processing failed", err));
	}
	finally {
      	// 异步请求
		if (asyncManager.isConcurrentHandlingStarted()) {
			if (mappedHandler != null) {
				mappedHandler.applyAfterConcurrentHandlingStarted(processedRequest, response);
			}
		}
		else {
          	// 删除上传请求的资源
			if (multipartRequestParsed) {
				cleanupMultipart(processedRequest);
			}
		}
	}
}
```

### 2.1、获取handler

从RequestMappingHandlerMapping中获取当前请求的handler，handler的注册通过afterPropertiesSet()进行handler的注册。

注解@RequestMapping定义的handler用的是RequestMappingHandlerMapping，实现Controller接口的handler使用的是BeanNameUrlHandlerMapping，静态资源的请求用的是SimpleUrlHandlerMapping。

实现一个handler的方式：

1、使用@RequestMapping注解；2、实现Controller接口；3、实现HttpRequestHandler接口；4、继承HttpServlet。

```java
protected HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
	if (this.handlerMappings != null) {
		for (HandlerMapping mapping : this.handlerMappings) {
          	// 获取对象handler和HandlerInterceptor包装成HandlerExecutionChain
			HandlerExecutionChain handler = mapping.getHandler(request);
			if (handler != null) {
				return handler;
			}
		}
	}
	return null;
}
// RequestMappingHandlerMapping#getHandler
public final HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
  	// 从mappingRegistry中通过路径获取HandlerMethod
	Object handler = getHandlerInternal(request);
	if (handler == null) {
		handler = getDefaultHandler();
	}
	if (handler == null) {
		return null;
	}
	// 从容器中获取handler
	if (handler instanceof String) {
		String handlerName = (String) handler;
		handler = obtainApplicationContext().getBean(handlerName);
	}
  	// 获取HandlerInterceptor包装成HandlerExecutionChain
	HandlerExecutionChain executionChain = getHandlerExecutionChain(handler, request);
  	// 处理跨域请求
	if (hasCorsConfigurationSource(handler) || CorsUtils.isPreFlightRequest(request)) {
		CorsConfiguration config = (this.corsConfigurationSource != null ? this.corsConfigurationSource.getCorsConfiguration(request) : null);
		CorsConfiguration handlerConfig = getCorsConfiguration(handler, request);
		config = (config != null ? config.combine(handlerConfig) : handlerConfig);
		executionChain = getCorsHandlerExecutionChain(request, executionChain, config);
	}
	return executionChain;
}
```

### 2.2、获取HandlerAdapter

获取不同handler对应的处理器，通过适配器模式进行适配，RequestMappingHandlerAdapter适配@RequestMapping注解定义的handler，HttpRequestHandlerAdapter适配实现HttpRequestHandler接口的handler，SimpleControllerHandlerAdapter适配实现Controller接口的handler。

```java
// 适配器模式
protected HandlerAdapter getHandlerAdapter(Object handler) throws ServletException {
	if (this.handlerAdapters != null) {
		for (HandlerAdapter adapter : this.handlerAdapters) {
			if (adapter.supports(handler)) {
				return adapter;
			}
		}
	}
	throw new ServletException("No adapter for handler [" + handler +
			"]: The DispatcherServlet configuration needs to include a HandlerAdapter that supports this handler");
}
```

### 2.3、HandlerInterceptor

SpringMVC重要拦截器，通过执行handler前置、后置及完成后进行扩展。通过HandlerExecutionChain设计的非常巧妙，使用interceptorIndex执行preHandler()进行++，如果返回false直接通过interceptorIndex倒序执行已经执行interceptors的afterCompletion()，正常执行通过interceptorIndex倒序执行afterCompletion()

```java
public interface HandlerInterceptor {
	// 前置处理
	default boolean preHandle(HttpServletRequest req, HttpServletResponse resp, Object handler){
		return true;
	}
	// 后置处理
	default void postHandle(HttpServletRequest req,HttpServletResponse resp,Object handler,
			@Nullable ModelAndView modelAndView) throws Exception {
	}
  	// 完成后回调
	default void afterCompletion(HttpServletRequest req,HttpServletResponse resp,Object handler,
			@Nullable Exception ex) throws Exception {
	}
}

// pre
boolean applyPreHandle(HttpServletRequest request, HttpServletResponse response){
	HandlerInterceptor[] interceptors = getInterceptors();
	if (!ObjectUtils.isEmpty(interceptors)) {
		for (int i = 0; i < interceptors.length; i++) {
			HandlerInterceptor interceptor = interceptors[i];
			if (!interceptor.preHandle(request, response, this.handler)) {
              	// 前置失败触发已经执行interceptors的afterCompletion
				triggerAfterCompletion(request, response, null);
				return false;
			}
			this.interceptorIndex = i;
		}
	}
	return true;
}

// post
void applyPostHandle(HttpServletRequest request, HttpServletResponse response, @Nullable ModelAndView mv){
	HandlerInterceptor[] interceptors = getInterceptors();
	if (!ObjectUtils.isEmpty(interceptors)) {
		for (int i = interceptors.length - 1; i >= 0; i--) {
			HandlerInterceptor interceptor = interceptors[i];
			interceptor.postHandle(request, response, this.handler, mv);
		}
	}
}

// afterCompletion
void triggerAfterCompletion(HttpServletRequest request, HttpServletResponse response, @Nullable Exception ex){
	HandlerInterceptor[] interceptors = getInterceptors();
	if (!ObjectUtils.isEmpty(interceptors)) {
      	// 倒序执行
		for (int i = this.interceptorIndex; i >= 0; i--) {
			HandlerInterceptor interceptor = interceptors[i];
			try {
				interceptor.afterCompletion(request, response, this.handler, ex);
			}
			catch (Throwable ex2) {
				logger.error("HandlerInterceptor.afterCompletion threw exception", ex2);
			}
		}
	}
}
```

### 2.4、执行handler

执行对应handler，并返回ModelAndView，RequestMappingHandlerAdapter通过AbstractHandlerMethodAdapter的模板方法handleInternal()执行，最终都会执行invokeHandlerMethod方法

```java
protected ModelAndView invokeHandlerMethod(HttpServletRequest request,
		HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {
	ServletWebRequest webRequest = new ServletWebRequest(request, response);
	try {
      	// 创建WebDataBinderFactory，用来创建WebDataBinder对象进行参数绑定
		WebDataBinderFactory binderFactory = getDataBinderFactory(handlerMethod);
      	// 创建ModelFactory，处理model，1、model初始化；2、处理器执行后将Model中相应的参数更新到sessionAttribute中
		ModelFactory modelFactory = getModelFactory(handlerMethod, binderFactory);
		ServletInvocableHandlerMethod invocableMethod = createInvocableHandlerMethod(handlerMethod);
      	// 设置参数处理器
		if (this.argumentResolvers != null) {
			invocableMethod.setHandlerMethodArgumentResolvers(this.argumentResolvers);
		}
      	// 设置返回值处理器
		if (this.returnValueHandlers != null) {
			invocableMethod.setHandlerMethodReturnValueHandlers(this.returnValueHandlers);
		}
		invocableMethod.setDataBinderFactory(binderFactory);
		invocableMethod.setParameterNameDiscoverer(this.parameterNameDiscoverer);
		// 创建ModelAndViewContainer对象，用于保存model和View对象
		ModelAndViewContainer mavContainer = new ModelAndViewContainer();
		mavContainer.addAllAttributes(RequestContextUtils.getInputFlashMap(request));
		modelFactory.initModel(webRequest, mavContainer, invocableMethod);
		mavContainer.setIgnoreDefaultModelOnRedirect(this.ignoreDefaultModelOnRedirect);
		AsyncWebRequest asyncWebRequest = WebAsyncUtils.createAsyncWebRequest(request, response);
		asyncWebRequest.setTimeout(this.asyncRequestTimeout);
		WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);
		asyncManager.setTaskExecutor(this.taskExecutor);
		asyncManager.setAsyncWebRequest(asyncWebRequest);
		asyncManager.registerCallableInterceptors(this.callableInterceptors);
		asyncManager.registerDeferredResultInterceptors(this.deferredResultInterceptors);
		// 异步处理
		if (asyncManager.hasConcurrentResult()) {
			Object result = asyncManager.getConcurrentResult();
			mavContainer = (ModelAndViewContainer) asyncManager.getConcurrentResultContext()[0];
			asyncManager.clearConcurrentResult();
			invocableMethod = invocableMethod.wrapConcurrentResult(result);
		}
		// 执行调用handler
		invocableMethod.invokeAndHandle(webRequest, mavContainer);
		if (asyncManager.isConcurrentHandlingStarted()) {
			return null;
		}
		return getModelAndView(mavContainer, modelFactory, webRequest);
	}
	finally {
		webRequest.requestCompleted();
	}
}

public void invokeAndHandle(ServletWebRequest webRequest, ModelAndViewContainer mavContainer,
		Object... providedArgs) throws Exception {
	// 调用父类的invokeForRequest执行请求
	Object returnValue = invokeForRequest(webRequest, mavContainer, providedArgs);
	// 处理@ResponseStatus注解
	setResponseStatus(webRequest);
	// 处理返回值，判断返回值是否为空
	if (returnValue == null) {
		// request的NotModified为true，有@ResponseStatus,RequestHandled为true，三个条件有一个成立，则设置请求处理完成并返回
		if (isRequestNotModified(webRequest) || getResponseStatus() != null || mavContainer.isRequestHandled()) {
			disableContentCachingIfNecessary(webRequest);
			mavContainer.setRequestHandled(true);
			return;
		}
	}
	// 返回值不为null，@ResponseStatus存在reason，这是请求处理完成并返回
	else if (StringUtils.hasText(getResponseStatusReason())) {
		mavContainer.setRequestHandled(true);
		return;
	}
	mavContainer.setRequestHandled(false);
	Assert.state(this.returnValueHandlers != null, "No return value handlers");
	try {
		// 使用returnValueHandlers处理返回值
		this.returnValueHandlers.handleReturnValue(
				returnValue, getReturnValueType(returnValue), mavContainer, webRequest);
	}
	catch (Exception ex) {}
}
```


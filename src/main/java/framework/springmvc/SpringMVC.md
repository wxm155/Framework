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
  	// 获取HandlerInterceptor包装HandlerExecutionChain
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
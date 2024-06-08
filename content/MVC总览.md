## 前要

​	个人对于非前后端分离不会如何返回，因此本篇内容将限定HandlerMethod被@RequestBody修饰作用的场景。另外，个人水平有限，相当MVC部分未能展现。

​	文本作为总览，将浅显地介绍从`ApplicationFilterChain.internalDoFilter(request, reponse) → HttpServlet（实际FrameworkServlet）.service(request, response)`开始的MVC代码————Tomcat调用SpringMVC提供的Servlet服务。

## 涉及模块

- HandlerExecutionChain：执行链路初始化
  - HandlerInterceptor：拦截器使用。
  - handler object：处理器对象配置及使用（这里着重介绍Handler Method）。
- @ControllerAdvice：SpringMVC提供切面功能、切入方法。
  - @ExceptionHandler：异常拦截处理
  -  @InitBinder：类型转换器PropertyEditor提供。
  -  @ModelAttribute：Model与request之间的绑定。
- HandlerMethodReturnValueHandler：ServletInvocableHandlerMethod指定方法返回对象处理器。
- HandlerMethodArgumentResolver：ServletInvocableHandlerMethod指定方法参数解析器。
- WebDataBinderFactory
  - 数据类型转换器
  - 对象字段填充数据器
  - 对象字段数据验证器。
- JSON数据读取、写入切面
  - RequestBodyAdvice：JSON读取切面
  - ResponseBodyAdvice：JSON写入切面。
- Request封装
  - MultipartResolver解析
  - 包装coyote.request。

## 涉及文章

[DispatcherServlet解析](./DispatcherServlet解析.md)

[HandlerInterceptor学习及使用](./HandlerInterceptor学习及使用.md)

[TomcatServer启动之DispatcherServlet存储](./TomcatServer启动之DispatcherServlet存储.md)

[Tomcat监听并交由DispatcherServlet-doDispatch处理的链路](Tomcat监听并交由DispatcherServlet-doDispatch处理的链路.md)

[Request组成与Multipart类型处理](Request组成与Multipart类型处理.md)

[HandlerExecutionChain - Handler调度链获取](HandlerExecutionChain - Handler调度链获取.md)

[Spring的WebMvcConfiguration设计](Spring的WebMvcConfiguration设计.md)

[SpringMVC学习笔记（二）——SpringMVC工作流程 ](https://www.cnblogs.com/worthmove/p/16653232.html)——文中的图还是能看一看的



## DispatcherServlet介绍

​	Tomcat调用Spring提供的Servlet实现类，进而调用到DispatcherServlet的过程：

<img src="picture/2024-02-27 23_21_45-DispatcherServlet调度链.drawio - draw.io.png" style="zoom:50%;" />

（从调用情况来看，貌似Request-Method请求的不同不在此处影响，唉，到底是哪里影响到了呢？）

​	DispatcherServlet处理请求的入口是doService(request, reponse)方法，不过重点还是doDispatch(request, reponse)，该方法是handler处理流程推动的开端，也是开发者功能介入的开始节点。

### DispatcherServlet的来源

文章：[TomcatServer启动之DispatcherServlet存储](./TomcatServer启动之DispatcherServlet存储.md)

> Tomcat的容器是嵌套的，StandardHost(TomcatEmbeddedContext(StandardWrapper(DispatcherServlet))))，那么单体Tomcat端口请求下，只会经由上述这条Container链路到DispatcherServlet。同时SpringMVC设计下，作为Servlet的DispatcherServlet是单体唯一的。

### doDispatch

```java
// 此方法截面提供以下内容
// request解析：文件读取、同key多value读取放置。
// handler执行链初始化：handler + 拦截器。
// handler执行支配其获取。
// 异常处理机制：processDispatchResult方法内执行，@ExceptionHandler提供。
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
   HttpServletRequest processedRequest = request;
   HandlerExecutionChain mappedHandler = null;
   boolean multipartRequestParsed = false;

   WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);

   try {
      ModelAndView mv = null;
      Exception dispatchException = null;

      try {
         // request解析
         processedRequest = checkMultipart(request);
         multipartRequestParsed = (processedRequest != request);

         // handler执行链初始化
         mappedHandler = getHandler(processedRequest);
         if (mappedHandler == null) {
            noHandlerFound(processedRequest, response);
            return;
         }

         // handler执行支配其获取。
         HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());

         // Process last-modified header, if supported by the handler.
         String method = request.getMethod();
         boolean isGet = HttpMethod.GET.matches(method);
         if (isGet || HttpMethod.HEAD.matches(method)) {
            long lastModified = ha.getLastModified(request, mappedHandler.getHandler());
            if (new ServletWebRequest(request, response).checkNotModified(lastModified) && isGet) {
               return;
            }
         }

         if (!mappedHandler.applyPreHandle(processedRequest, response)) {
            return;
         }

         // handlerAdapter执行，先进入其中看看
         mv = ha.handle(processedRequest, response, mappedHandler.getHandler());

         if (asyncManager.isConcurrentHandlingStarted()) {
            return;
         }

         applyDefaultViewName(processedRequest, mv);
         // 拦截器执行
         mappedHandler.applyPostHandle(processedRequest, response, mv);
      }
      catch (Exception ex) {
         dispatchException = ex;
      }
      catch (Throwable err) {
         // As of 4.3, we're processing Errors thrown from handler methods as well,
         // making them available for @ExceptionHandler methods and other scenarios.
         dispatchException = new NestedServletException("Handler dispatch failed", err);
      }
      // 拦截器执行 + 异常处理器执行（抛出异常才执行）
      processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);
   }
   catch (Exception ex) {
      // 拦截器执行
      triggerAfterCompletion(processedRequest, response, mappedHandler, ex);
   }
   catch (Throwable err) {
      // 同上。
      triggerAfterCompletion(processedRequest, response, mappedHandler,
            new NestedServletException("Handler processing failed", err));
   }
   finally {
      if (asyncManager.isConcurrentHandlingStarted()) {
         // Instead of postHandle and afterCompletion
         if (mappedHandler != null) {
            mappedHandler.applyAfterConcurrentHandlingStarted(processedRequest, response);
         }
      }
      else {
         // Clean up any resources used by a multipart request.
         if (multipartRequestParsed) {
            cleanupMultipart(processedRequest);
         }
      }
   }
}
```

具体内容请到以下文章了解：

[HandlerInterceptor学习及使用](./HandlerInterceptor学习及使用.md)

[Request组成与Multipart类型处理](Request组成与Multipart类型处理.md)

[HandlerExecutionChain - Handler调度链获取](HandlerExecutionChain - Handler调度链获取.md)

[Spring的WebMvcConfiguration设计](Spring的WebMvcConfiguration设计.md)——RequestMappingHandlerMapping创建和初始化知识、RequestMappingHandlerAdapter创建和初始化。



### processDispatchResult

```java
// 如dispatcher调用主干所说，这部分主要分为两部分：processHandlerException（异常处理机制--正常HandlerAdapter的平替） 和 triggerAfterCompletion（拦截器流程）。这里我们主要来探究前者的处理逻辑。
private void processDispatchResult(HttpServletRequest request, HttpServletResponse response,
      @Nullable HandlerExecutionChain mappedHandler, @Nullable ModelAndView mv,
      @Nullable Exception exception) throws Exception {

   boolean errorView = false;

   if (exception != null) {
      if (exception instanceof ModelAndViewDefiningException) {
         logger.debug("ModelAndViewDefiningException encountered", exception);
         mv = ((ModelAndViewDefiningException) exception).getModelAndView();
      }
      else {
         Object handler = (mappedHandler != null ? mappedHandler.getHandler() : null);
         // 异常处理
         mv = processHandlerException(request, response, handler, exception);
         errorView = (mv != null);
      }
   }

   // Did the handler return a view to render?
   if (mv != null && !mv.wasCleared()) {
      render(mv, request, response);
      if (errorView) {
         WebUtils.clearErrorRequestAttributes(request);
      }
   }
   else {
      if (logger.isTraceEnabled()) {
         logger.trace("No view rendering, null ModelAndView returned.");
      }
   }

   if (WebAsyncUtils.getAsyncManager(request).isConcurrentHandlingStarted()) {
      // Concurrent handling started during a forward
      return;
   }

   if (mappedHandler != null) {
      // Exception (if any) is already handled..
      // 拦截器处理。
      mappedHandler.triggerAfterCompletion(request, response, null);
   }
}
```

文章：

[exception-resolver](./Exception-resolver/exception-resolver.md)



## HandlerAdapter

### RequestMappingHandlerAdapter

handle(...) -> handleInternal(...)

#### handleInternal(HttpServletRequest, HttpServletResponse, HandlerMethod)

```java
// 本类是request进入method invoke入口的准备类，一个handler-method只会经过一次
protected ModelAndView handleInternal(HttpServletRequest request,
      HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {

   ModelAndView mav;
   // 检查本类是否具备要求：是否支持GET、POST等；是否支持session。这些要求根据本类的配置意愿来选择性判断。
   checkRequest(request);

   // Execute invokeHandlerMethod in synchronized block if required.
   // SpringMVC针对InvokeHandlerMethod，存在考虑基于同session的client发出request的同步控制。问题：session是基于会话生命周期的，那会话和request之间的关系是什么？多次的request怎么绑定相同session？
   if (this.synchronizeOnSession) {
      HttpSession session = request.getSession(false);
      if (session != null) {
         Object mutex = WebUtils.getSessionMutex(session);
         synchronized (mutex) {
            mav = invokeHandlerMethod(request, response, handlerMethod);
         }
      }
      else {
         // No HttpSession available -> no mutex necessary
         mav = invokeHandlerMethod(request, response, handlerMethod);
      }
   }
   else {
      // No synchronization on session demanded at all...
      // 非异步实际调用的。这里先研究这个。 
      mav = invokeHandlerMethod(request, response, handlerMethod);
   }

   if (!response.containsHeader(HEADER_CACHE_CONTROL)) {
      if (getSessionAttributesHandler(handlerMethod).hasSessionAttributes()) {
         applyCacheSeconds(response, this.cacheSecondsForSessionAttributeHandlers);
      }
      else {
         prepareResponse(response);
      }
   }

   return mav;
}
```

文章：

[web-method-invoke结构](./web-method-invoke结构.md)



## Controller返回类型

[SpringMVC详解二、返回值与数据在域中的保存](https://blog.csdn.net/mxcsdn/article/details/80720415)

[SpringMVC-方法四种类型返回值总结，你用过几种？](https://juejin.cn/post/6844903838332239879#heading-11)

[springMVC中controller的几种返回类型](https://blog.csdn.net/ApatheCrazyFan/article/details/80089618)

ModelAndView、String、代表redirect重定向、代表forward转发、void。这主要是返回内容与return-value-handler的适配，因此还与handler-method上的注解有关。

有空再看。


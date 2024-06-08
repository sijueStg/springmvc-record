## SpringMVC提供servlet响应request

​	这部分介绍tomcat处理请求，将请求传递给servlet处理之后的事情。Spring初始化的时候，将DispatcherServlet bean化，传递给Tomcat作为servlet实现类使用。Tomcat自从poller拉取到request至`ApplicationFilterChain.internalDoFilter(ServletRequest request, ServletResponse response)`，就将request交由SpringMVC处理，由`Servlet.service(ServletRequest, ServletResponse)`处理。这里就从service方法调用链开始研究DispatcherServlet应用。

### TomcatWebServer.service至DispatcherServlet.dispatcher调度过程

<img src="../SpringBoot/picture/2024-02-05 09_37_03-DispatcherServlet调度链.drawio - draw.io.png" style="zoom:60%;" />

### dispatcher调度主干

```java
// 这里需要提及一下HandlerExecutionChain中HandlerInterceptor调用序列。chain中拦截器存储在List中，优先级高（order值小）的在序号小的位置上。

// HandlerInterceptor执行顺序：
// applyPreHandle从优先级高的执行起，chain中interceptorIndex指向最后一次拦截器的序号。
// applyPostHandle从拦截器列表的末尾执行开始。interceptorIndex字段不在这起作用。
// afterCompletion起着finally+catch的作用，若chain中applyPreHandle执行过程中的异常抛出，则从interceptorIndex指向序号————最近一次执行成功的拦截器还是回执。若是afterCompletion非异常处理的执行过程中抛出异常，则从末尾回执。
// 不要在afterCompletion内做出可能会抛出异常的举动，尤其是ex参数传入非空的情况————处理异常的情况。
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
   // 本方法内用此引用，原因：checkMultipart(request)可能会返回包装对象，非原本request。
   HttpServletRequest processedRequest = request;
   // handler（处理器）执行链
   HandlerExecutionChain mappedHandler = null;
   boolean multipartRequestParsed = false;

   // 这个暂不做研究考虑
   WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);

   try {
      ModelAndView mv = null;
      Exception dispatchException = null;

      try {
         // multipart请求类型的request（包装）处理，以适应multipart request的需求。
         // 这里将对深层的request做解析处理，比如coyote.request.parameters.paramHashValues的数据填充，从postman的Params和Body->form-data。这是对数据流做了提取，以文本或file(文件)的形式赋值。所以针对设计者来说，应当从这一步代码执行后（不是说只有这步才是spring对request做了数据提供或修改），看springMVC对tomcat提供的request做出的修改和数据提取放置情况。
         processedRequest = checkMultipart(request);
         // 若request是multipart类型，则true。
         multipartRequestParsed = (processedRequest != request);

         // Determine handler for the current request.
         // 获取handler调度链。包含：HandlerMethod（适配request请求路径的controllerMethod） + HandlerInterceptor（处理拦截器，简称拦截器），针对handler的interceptor。handler是对处理调用controller method的一种资源集合+处理行为。
         // 在request method环境下，这里就是从this.handlerMappings列表中第一个元素类RequestMappingHandlerMapping中，获取MappingRegistry(内部类).registry字段存储着<RequestMappingInfo, MappingRegistration>（都是AbstractHandlerMethodMapping的内部类）的映射，前者与request匹配（主要是路径匹配），后者存储有handlerMethod信息。
         // getHandler(requst)的首要目的就是从适配request的MappingRegistration提取HandlerMethod。
         // 其次，向返回的HandlerExecutionChain对象注入Interceptor器。最后，就是未知的其他操作。
         mappedHandler = getHandler(processedRequest);
         if (mappedHandler == null) {
            noHandlerFound(processedRequest, response);
            return;
         }

         // Determine handler adapter for the current request.
         // 将核心MVC工作流参数化————理解为实现类参数决定handler处理流程。就目前所知的handler就HandlerMethod。这里含有诸多不知名参数，以供挖掘学习。
         // 选择适配handler的HandlerAdapter类型，这里返回了适配HandlerMethod.class(request method)的HandlerAdapter。比如RequestMappingHandlerAdapter，其内涵了arguement-resolvers和returnValue-handlers、requestResponseBodyAdvice等资源。
         HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());

         // Process last-modified header, if supported by the handler.
         // 从实际的调用中来看，貌似spring没支持关于last-modified的特性，毕竟底层方法返回值硬编码为-1。
         String method = request.getMethod();
         boolean isGet = HttpMethod.GET.matches(method);
         if (isGet || HttpMethod.HEAD.matches(method)) {
            long lastModified = ha.getLastModified(request, mappedHandler.getHandler());
            if (new ServletWebRequest(request, response).checkNotModified(lastModified) && isGet) {
               return;
            }
         }

         // 拦截器预处理，具有拦截请求作用。当某一拦截器确认拦截，调用所有拦截器的afterCompletion方法，然后返回false，停止后续handler/ha执行。afterCompletion相当于finally的作用。
         if (!mappedHandler.applyPreHandle(processedRequest, response)) {
            return;
         }

         // Actually invoke the handler.
         // 通过了所有拦截器，handler开始处理request。不同的工作流差别很大，但是目前学习的是methodHandler的处理，也就是MVC结构的handler处理。
         mv = ha.handle(processedRequest, response, mappedHandler.getHandler());

         if (asyncManager.isConcurrentHandlingStarted()) {
            return;
         }

         applyDefaultViewName(processedRequest, mv);
         // handler成功调用后，view渲染前，执行此拦截器方法。意图从参数和执行时机可见：可添加额外的model（数据模型）给mv。
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
      // 内部处理分为两部分：存在的异常处理 + 拦截器的afterCompletion方法执行。异常将反应到ModelAndView中，而afterCompletion方法将不处理异常。
      // 这里的异常处理，是this.handlerExceptionResolvers处理，其中就存在着@ControllerAdvice + @ExceptionHandler 应用的ExceptionHandlerExceptionResolver，其本质就是异常情况下，HandlerAdapter.handle(...)的平替，不过有些功能的限制就是了。问题是，@ExceptionHandler提供的异常如何抛出异常进行匹配呢，换言之，抛出异常如何和方法进行匹配。这得提到ExceptionHandlerMethodResolver的功能。有必要专门提供文件解析。
      processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);
   }
   catch (Exception ex) {
      // 用afterCompletion方法处理异常。
      triggerAfterCompletion(processedRequest, response, mappedHandler, ex);
   }
   catch (Throwable err) {
      // 同上
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



### HandlerAdapter的method激活过程

​	这里的HandlerAdapter指的是RequestMappingHandlerAdapter。其中重要的代码：

```java
// handlerAdapater.handle主要的调用方法，中间隔了几层方法调用。
// 牵涉到了异步请求（不会，暂不予以理会）、handler处理、ModelAndView创建并予以一定程度初始化
protected ModelAndView invokeHandlerMethod(HttpServletRequest request,
      HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {

   ServletWebRequest webRequest = new ServletWebRequest(request, response);
   try {
      // 用于在方法参数获取实例过程中，将意向数据转变成参数所需的类型的操作类。这里解析加载了@InitBinder标记方法。
      WebDataBinderFactory binderFactory = getDataBinderFactory(handlerMethod);
      // 鬼畜，这里解析加载了@ModelAttribute。好家伙@ControllerAdvice内支持的两个注解都在这里给捕获了。并且还想ModelFactory提供了WebDataBinderFactory。
      ModelFactory modelFactory = getModelFactory(handlerMethod, binderFactory);

      // 将handlerMethod数据copy至ServletInvocableHandlerMethod，让其作为servlet调用过程中method激活的处理对象，其是HandlerMethod继承链上的子类。
      ServletInvocableHandlerMethod invocableMethod = createInvocableHandlerMethod(handlerMethod);
      // 设置method-parameter的resolver组合器————缓存有参数resolvers，通过其间接调用适配的resolver。resolver用于从request或parameter使用的注解中获取意向数据并进行类型适配转换。
      // 这里也有将数据注入到mav的Model，比如@ReuqestBody支持的resolver，就将解析结果注入到model中。
      if (this.argumentResolvers != null) {
         invocableMethod.setHandlerMethodArgumentResolvers(this.argumentResolvers);
      }
      // 设置method-returnValue的handler组合器————缓存有returnValue的handlers，通过其间接调用适配的handler。对返回值进行操作，比如序列化，向model注入attributes、设置view等。
      if (this.returnValueHandlers != null) {
         invocableMethod.setHandlerMethodReturnValueHandlers(this.returnValueHandlers);
      }
      invocableMethod.setDataBinderFactory(binderFactory);
      invocableMethod.setParameterNameDiscoverer(this.parameterNameDiscoverer);

      // mv+response处理的容器，前者主要。
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

      if (asyncManager.hasConcurrentResult()) {
         Object result = asyncManager.getConcurrentResult();
         mavContainer = (ModelAndViewContainer) asyncManager.getConcurrentResultContext()[0];
         asyncManager.clearConcurrentResult();
         LogFormatUtils.traceDebug(logger, traceOn -> {
            String formatted = LogFormatUtils.formatValue(result, !traceOn);
            return "Resume with async result [" + formatted + "]";
         });
         invocableMethod = invocableMethod.wrapConcurrentResult(result);
      }

      // 做好了request和respose的包装，进入下一步invoke stage
      // 这里为mavContainer提供model
      invocableMethod.invokeAndHandle(webRequest, mavContainer);
      if (asyncManager.isConcurrentHandlingStarted()) {
         return null;
      }

      // 这里做了什么？
      return getModelAndView(mavContainer, modelFactory, webRequest);
   }
   finally {
      webRequest.requestCompleted();
   }
}
```

#### invokeAndHandle

```java
public void invokeAndHandle(ServletWebRequest webRequest, ModelAndViewContainer mavContainer,
      Object... providedArgs) throws Exception {

   // 这里用了method parameter resolver + method invoke
   Object returnValue = invokeForRequest(webRequest, mavContainer, providedArgs);
   setResponseStatus(webRequest);
   ......
   // 这里就是method return value handler。主要就是将返回数据加入到model。设置view？
   // 唉，又得开一个篇章学习了，不过这个handler和前面的resolver涉及跟贴切。
   this.returnValueHandlers.handleReturnValue(
            returnValue, getReturnValueType(returnValue), mavContainer, webRequest);
   ......
}
```

##### invokeForRequest

```java
public Object invokeForRequest(NativeWebRequest request, @Nullable ModelAndViewContainer mavContainer,
      Object... providedArgs) throws Exception {

   // request method的参数准备过程。这里的重点在于对@RequestBody参数的数据流读取并将其json数据格式成目标POJO。
   Object[] args = getMethodArgumentValues(request, mavContainer, providedArgs);
   if (logger.isTraceEnabled()) {
      logger.trace("Arguments: " + Arrays.toString(args));
   }
   // request method调用。后面基本上就是Method放射调用了，不用深究了。
   return doInvoke(args);
}
```

**request method parameters准备**

```java
protected Object[] getMethodArgumentValues(NativeWebRequest request, @Nullable ModelAndViewContainer mavContainer,
      Object... providedArgs) throws Exception {

   // request method的参数列表
   MethodParameter[] parameters = getMethodParameters();
   if (ObjectUtils.isEmpty(parameters)) {
      return EMPTY_ARGS;
   }

   Object[] args = new Object[parameters.length];
   for (int i = 0; i < parameters.length; i++) {
      MethodParameter parameter = parameters[i];
      // 内置方法名或构造器的参数名的解析器
      parameter.initParameterNameDiscovery(this.parameterNameDiscoverer);
      // 从参数providedArgs找与parameter代表的Class相匹配的对象。要求：对象是否是代表Class的实例。这种情况在@ExceptionHandler标记的方法被激活的过程中应用。
      args[i] = findProvidedArgument(parameter, providedArgs);
      if (args[i] != null) {
         continue;
      }
      // this.resolves是HandlerMethodArgumentResolver的注册列表的封装对象，并缓存有<MethodParameter, HandlerMethodArgumentResolver>字典的历史数据，相同的请求路径，可从历史记录中拿取对应的resolver。
      if (!this.resolvers.supportsParameter(parameter)) {
         throw new IllegalStateException(formatArgumentError(parameter, "No suitable resolver"));
      }
      try {
         // 用reoslve从request中解析出对应parameter的对象（数据流或coyote.Request.parameters中）。
         args[i] = this.resolvers.resolveArgument(parameter, mavContainer, request, this.dataBinderFactory);
      }
      catch (Exception ex) {
         // Leave stack trace for later, exception may actually be resolved and handled...
         if (logger.isDebugEnabled()) {
            String exMsg = ex.getMessage();
            if (exMsg != null && !exMsg.contains(parameter.getExecutable().toGenericString())) {
               logger.debug(formatArgumentError(parameter, exMsg));
            }
         }
         throw ex;
      }
   }
   return args;
}
```

##### handleReturnValue

​	这里类似于method-arguement-resolvers，是对返回值的handler，用于将request+returnValue选择性注入到response中去，并对mav进行一定的处理。



#### getModelAndView

​	好吧，终于到这里了，这个地方解决了，再整理一下最近学习的，可以去找工作了。

```java
private ModelAndView getModelAndView(ModelAndViewContainer mavContainer,
      ModelFactory modelFactory, NativeWebRequest webRequest) throws Exception {

   // update what? 正常情况下，就将其视作为mav缓存的map向request.attributes容器存入就是了。
   modelFactory.updateModel(webRequest, mavContainer);
   if (mavContainer.isRequestHandled()) {
      // request已经被handler处理完成，返回空。正常的不返回东西吗？
      return null;
   }
   ModelMap model = mavContainer.getModel();
   ModelAndView mav = new ModelAndView(mavContainer.getViewName(), model, mavContainer.getStatus());
   if (!mavContainer.isViewReference()) {
      mav.setView((View) mavContainer.getView());
   }
   if (model instanceof RedirectAttributes) {
      Map<String, ?> flashAttributes = ((RedirectAttributes) model).getFlashAttributes();
      HttpServletRequest request = webRequest.getNativeRequest(HttpServletRequest.class);
      if (request != null) {
         RequestContextUtils.getOutputFlashMap(request).putAll(flashAttributes);
      }
   }
   return mav;
}
```



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

#### **processHandlerException**

```java
// 
@Nullable
protected ModelAndView processHandlerException(HttpServletRequest request, HttpServletResponse response,
      @Nullable Object handler, Exception ex) throws Exception {

   // Success and error responses may use different content types
   request.removeAttribute(HandlerMapping.PRODUCIBLE_MEDIA_TYPES_ATTRIBUTE);

   // Check registered HandlerExceptionResolvers...
   ModelAndView exMv = null;
   if (this.handlerExceptionResolvers != null) {
      for (HandlerExceptionResolver resolver : this.handlerExceptionResolvers) {
         // 由resolver自己判断是否支持ex异常处理。一般这个由ExceptionHandlerExceptionResolver提供。因此接下来将介绍该resolver。另外，提一句，支持ex的resolver并不一定提供exMv。
         exMv = resolver.resolveException(request, response, handler, ex);
         // 一旦resolver提供exMv，则视为处理完毕。
         if (exMv != null) {
            break;
         }
      }
   }
   // exMv的处理。
   ......
}
```

**HandlerExceptionResolverComposite**

```java
public ModelAndView resolveException(
      HttpServletRequest request, HttpServletResponse response, @Nullable Object handler, Exception ex) {

   if (this.resolvers != null) {
      for (HandlerExceptionResolver handlerExceptionResolver : this.resolvers) {
         // 现有的分为两种：HandlerExceptionResolverComposite 和 DefaultErrorAttributes。前者和arguement-resolver设计类似，内部集成了诸多resolver，其中皆有ExceptionHandlerExceptionResolver。
         ModelAndView mav = handlerExceptionResolver.resolveException(request, response, handler, ex);
         if (mav != null) {
            return mav;
         }
      }
   }
   return null;
}
```

**HandlerExceptionResolverComposite.resolveException(...)**

```java
public ModelAndView resolveException(
      HttpServletRequest request, HttpServletResponse response, @Nullable Object handler, Exception ex) {

   if (this.resolvers != null) {
      // HandlerExceptionResolverComposite逐个resolve，直至返回mav。这里仅考虑ExceptionHandlerExceptionResolver的使用。
      for (HandlerExceptionResolver handlerExceptionResolver : this.resolvers) {
         ModelAndView mav = handlerExceptionResolver.resolveException(request, response, handler, ex);
         if (mav != null) {
            return mav;
         }
      }
   }
   return null;
}
```

**AbstractHandlerMethodExceptionResolver.resolveException(...)**

本抽象类是ExceptionHandlerExceptionResolver.super。

```java
public ModelAndView resolveException(
      HttpServletRequest request, HttpServletResponse response, @Nullable Object handler, Exception ex) {

   // 判断本resolver是否支持<request, handler>，底层不清楚干什么。关于@ControllerAdvice的basePackageClasses等参数的指定，并不在这里起作用。
   if (shouldApplyTo(request, handler)) {
      prepareResponse(ex, response);
      // 这部分就是具体类发挥作用的时刻了。
      ModelAndView result = doResolveException(request, response, handler, ex);
      if (result != null) {
         // Print debug message when warn logger is not enabled.
         if (logger.isDebugEnabled() && (this.warnLogger == null || !this.warnLogger.isWarnEnabled())) {
            logger.debug(buildLogMessage(ex, request) + (result.isEmpty() ? "" : " to " + result));
         }
         // Explicitly configured warn logger in logException method.
         logException(ex, request);
      }
      return result;
   }
   else {
      return null;
   }
}
```

**doResolveException**

```java
// AbstractHandlerMethodExceptionResolver.doResolveException(...)
protected final ModelAndView doResolveException(
      HttpServletRequest request, HttpServletResponse response, @Nullable Object handler, Exception ex) {

   HandlerMethod handlerMethod = (handler instanceof HandlerMethod ? (HandlerMethod) handler : null);
   return doResolveHandlerMethodException(request, response, handlerMethod, ex);
}
```

```java
// 这里模拟HandlerAdapter.handle(...)，有ServletInvocableHandlerMethod.invokeAndHandle(...)调用，也有ModelAndView构建准备。
protected ModelAndView doResolveHandlerMethodException(HttpServletRequest request,
      HttpServletResponse response, @Nullable HandlerMethod handlerMethod, Exception exception) {

   // 主要是定位@ExceptionHandler指定方法的命中目标。下文将介绍。
   ServletInvocableHandlerMethod exceptionHandlerMethod = getExceptionHandlerMethod(handlerMethod, exception);
   if (exceptionHandlerMethod == null) {
      return null;
   }

   // 配置本类argumentResolvers和returnValueHandlers，不过相比RequestMappingHandlerAdapter提供的arguement-resolvers和returnValue-handlers，就比较少了。这意味着异常处理的参数和返回值的解析/处理手段有限，这个需要注意。
   if (this.argumentResolvers != null) {
      exceptionHandlerMethod.setHandlerMethodArgumentResolvers(this.argumentResolvers);
   }
   if (this.returnValueHandlers != null) {
      exceptionHandlerMethod.setHandlerMethodReturnValueHandlers(this.returnValueHandlers);
   }

   ServletWebRequest webRequest = new ServletWebRequest(request, response);
   ModelAndViewContainer mavContainer = new ModelAndViewContainer();

   ArrayList<Throwable> exceptions = new ArrayList<>();
   try {
      if (logger.isDebugEnabled()) {
         logger.debug("Using @ExceptionHandler " + exceptionHandlerMethod);
      }
      // Expose causes as provided arguments as well
      // 获取<抛出异常exception，exception内层级的cause们，handlerMethod>，这些将作为适配的@ExceptionHandler标记方法 激活的参数提供。
      Throwable exToExpose = exception;
      while (exToExpose != null) {
         exceptions.add(exToExpose);
         Throwable cause = exToExpose.getCause();
         exToExpose = (cause != exToExpose ? cause : null);
      }
      Object[] arguments = new Object[exceptions.size() + 1];
      exceptions.toArray(arguments);  // efficient arraycopy call in ArrayList
      arguments[arguments.length - 1] = handlerMethod;
      // ServletInvocableHandlerMethod.invokeAndHandle(...)调用
      exceptionHandlerMethod.invokeAndHandle(webRequest, mavContainer, arguments);
   }
   catch (Throwable invocationEx) {
      // Any other than the original exception (or a cause) is unintended here,
      // probably an accident (e.g. failed assertion or the like).
      if (!exceptions.contains(invocationEx) && logger.isWarnEnabled()) {
         logger.warn("Failure in @ExceptionHandler " + exceptionHandlerMethod, invocationEx);
      }
      // Continue with default processing of the original exception...
      return null;
   }

   // ModelAndView准备。这里是什么呢？
   if (mavContainer.isRequestHandled()) {
      return new ModelAndView();
   }
   else {
      ModelMap model = mavContainer.getModel();
      HttpStatus status = mavContainer.getStatus();
      ModelAndView mav = new ModelAndView(mavContainer.getViewName(), model, status);
      mav.setViewName(mavContainer.getViewName());
      if (!mavContainer.isViewReference()) {
         mav.setView((View) mavContainer.getView());
      }
      if (model instanceof RedirectAttributes) {
         Map<String, ?> flashAttributes = ((RedirectAttributes) model).getFlashAttributes();
         RequestContextUtils.getOutputFlashMap(request).putAll(flashAttributes);
      }
      return mav;
   }
}
```

**getExceptionHandlerMethod(...)**

```java
protected ServletInvocableHandlerMethod getExceptionHandlerMethod(
      @Nullable HandlerMethod handlerMethod, Exception exception) {

   Class<?> handlerType = null;

   if (handlerMethod != null) {
      // Local exception handler methods on the controller class itself.
      // To be invoked through the proxy, even in case of an interface-based proxy.
      // 这部分貌似没啥用
      handlerType = handlerMethod.getBeanType();
      ExceptionHandlerMethodResolver resolver = this.exceptionHandlerCache.get(handlerType);
      if (resolver == null) {
         resolver = new ExceptionHandlerMethodResolver(handlerType);
         this.exceptionHandlerCache.put(handlerType, resolver);
      }
      Method method = resolver.resolveMethod(exception);
      // 有缓存数据的情况。
      if (method != null) {
         return new ServletInvocableHandlerMethod(handlerMethod.getBean(), method, this.applicationContext);
      }
      // For advice applicability check below (involving base packages, assignable types
      // and annotation presence), use target class instead of interface-based proxy.
      if (Proxy.isProxyClass(handlerType)) {
         handlerType = AopUtils.getTargetClass(handlerMethod.getBean());
      }
   }

   // 这里是关键点。<ControllerAdviceBean, ExceptionHandlerMethodResolver>是具有@ExceptionHandler使用的@ControllerAdvie标记类的解析。前者是@ControllerAdvice标记类的包装，后者是针对这个类中涉及@ExceptionHandler使用的解析————<Throwable, Method>。
   for (Map.Entry<ControllerAdviceBean, ExceptionHandlerMethodResolver> entry : this.exceptionHandlerAdviceCache.entrySet()) {
      ControllerAdviceBean advice = entry.getKey();
      // advice.isApplicableToBeanType(...)将检验@ControllerAdvice提供的参数basePackages、assignableTypes、annotations。第一个是针对Controller-class所在的位置，第二个是针对Controller-class的继承、实现关系，第三个是针对Controller-class本身使用的注解。从前向后匹配数据集，只要一满足就视为@ControllerAdvice标记类可Advice Controller-class。若都没有指定，则视为无障碍支持。
      // 不过basePackageClasses呢？它作为basePackages方便用法。basePackageClasses指定类所在包，将视为basePackages的内容。
      if (advice.isApplicableToBeanType(handlerType)) {
         ExceptionHandlerMethodResolver resolver = entry.getValue();
         // ExceptionHandlerMethodResolver中找适配exception的method，若没有，接着下一个ExceptionHandlerMethodResolver适配操作。如何支持都早springMVC目录中的ExceptionHandlerMethodResolver.md文件中介绍。
         Method method = resolver.resolveMethod(exception);
         if (method != null) {
            return new ServletInvocableHandlerMethod(advice.resolveBean(), method, this.applicationContext);
         }
      }
   }

   return null;
}
```





## 请求类型

- get
- post
- put
- delete
- options
- trace
- head ？

前四者常用。






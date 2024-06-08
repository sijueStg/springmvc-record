## 前言

​	文本介绍RequestMappingHandlerAdapter对HandlerMethod调用的流程，讲述做了哪些步骤，讲讲大致的框架。

关键字：

- handler-method指的是Controller类中request的方法。
- invocableMethod：ServletInvocableHandlerMethod实例。
- 作用域：ControllerAdvice类是handler的切面服务提供者，每个类有其自己作用的handler集合，这个范围就称为作用域。
- handler：一般指的就是Controller类。

## invokeHandlerMethod

```java
// 目标：ServletInvocableHandlerMethod.invokeAndHandle(ServletWebRequest, ModelAndViewContainer, Object...) —— <SWRequest, mavc, args>
// ServletInvocableHandlerMethod是调度Method的单元，这个method可能是handler-method、@InitBinder标记method、@ModelAttribute标记Method等。SpringMVC依赖该类反射调度HandlerMethod指向的method。
// 本方法是调度封装有handler-method的ServletInvocableHandlerMethod对象的准备方法。封装其他方法的invocableMethod有自己的准备位置。
/*
 * 准备内容：
 * ① WebDataBinderFactory：生产WebDataBinder，产品用来为data→dataType，或用来绑定data，向其属性注入数据。
 * ② ModelFactory：mavc对request的attributes绑定————要求mavc提供指定attribute，并让其向request索取绑定意向的attribute。
 * ③ mavc提供。
 * ④ 异步管理：不熟。
 */
protected ModelAndView invokeHandlerMethod(HttpServletRequest request,
      HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {

   // 封装：ServletWebRequest(HttpServletRequest, HttpServletResponse)。目的是适配HttpServletRequest，封装HttpServletResponse更多的是参数传递的便利。
   ServletWebRequest webRequest = new ServletWebRequest(request, response);
   try {
      // 创建WebDataBinderFactory工厂类，handler-method提供被作用坐标，收纳作用域可命中的@InitBinder标记方法。 
      WebDataBinderFactory binderFactory = getDataBinderFactory(handlerMethod);
      // 创建ModelFactory工厂类，handler-method提供被作用坐标，收纳作用域可命中的@ModelAttribute标记方法。
      ModelFactory modelFactory = getModelFactory(handlerMethod, binderFactory);

      // 封装：ServletInvocableHandlerMethod(handlerMethod)
      ServletInvocableHandlerMethod invocableMethod = createInvocableHandlerMethod(handlerMethod);
      if (this.argumentResolvers != null) {
      	  // 提供method参数resolver
         invocableMethod.setHandlerMethodArgumentResolvers(this.argumentResolvers);
      }
      if (this.returnValueHandlers != null) {
          // 提供方法返回值resolver
         invocableMethod.setHandlerMethodReturnValueHandlers(this.returnValueHandlers);
      }
      // 提供WebDataBinder工厂类。
      invocableMethod.setDataBinderFactory(binderFactory);
      invocableMethod.setParameterNameDiscoverer(this.parameterNameDiscoverer);

      // 制造Model-View容器，一般多用于前者，注入属性。
      ModelAndViewContainer mavContainer = new ModelAndViewContainer();
      mavContainer.addAllAttributes(RequestContextUtils.getInputFlashMap(request));
      // model中注册attribute，并向request绑定属性————model与request的属性进行绑定。
      modelFactory.initModel(webRequest, mavContainer, invocableMethod);
      mavContainer.setIgnoreDefaultModelOnRedirect(this.ignoreDefaultModelOnRedirect);

      // 不懂，暂不予理会
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

      // invocableMethod激活
      invocableMethod.invokeAndHandle(webRequest, mavContainer);
      if (asyncManager.isConcurrentHandlingStarted()) {
         return null;
      }

      // 返回mav
      return getModelAndView(mavContainer, modelFactory, webRequest);
   }
   finally {
      webRequest.requestCompleted();
   }
}
```

文章参考：

[ModelFactory学习](./ModelFactory学习/ModelFactory学习.md)——讲述了ModelFactory的创建和初始化。

[WebDataBinder.md](WebDataBinder学习.md)



## invokeAndHandle

ServletInvocableHandlerMethod.invokeAndHandle(ServletWebRequest, ModelAndViewContainer, Object...)

​	这块是复用性极高的代码块，mvc中，只要涉及method反射调用的，都会走这块。很好的代码块。

```java
/*
 * 只要具备至少具备了参数resolver、参数名discover、WebDataBinderFactory、HandlerMethod，就有调用本方法的资格。
 * @InitBinder和@ModelAttribute没有返回值解析，只有@Exceptionhandler进行封装的时候会安排。
 */
public void invokeAndHandle(ServletWebRequest webRequest, ModelAndViewContainer mavContainer,
      Object... providedArgs) throws Exception {

   // 激活HandlerMethod封装method。proviededArgs是方法参数优先提供者。
   Object returnValue = invokeForRequest(webRequest, mavContainer, providedArgs);
   setResponseStatus(webRequest);

   // 返回空值或有异常，则给mavc设置handler finish标记
   if (returnValue == null) {
      /*
       * ①
       * ② handler-method被@ResponseStatus标记的解析情况。
       * ③ mavContainer已经被setRequestHandled(true)
       */
      if (isRequestNotModified(webRequest) || getResponseStatus() != null || mavContainer.isRequestHandled()) {
         disableContentCachingIfNecessary(webRequest);
         mavContainer.setRequestHandled(true);
         return;
      }
   }
   // 这个应该是有异常的意思吧。。。
   else if (StringUtils.hasText(getResponseStatusReason())) {
      mavContainer.setRequestHandled(true);
      return;
   }

   mavContainer.setRequestHandled(false);
   Assert.state(this.returnValueHandlers != null, "No return value handlers");
   try {
      // 启用返回值处理器。
      this.returnValueHandlers.handleReturnValue(
            returnValue, getReturnValueType(returnValue), mavContainer, webRequest);
   }
   catch (Exception ex) {
      if (logger.isTraceEnabled()) {
         logger.trace(formatErrorForReturnValue(returnValue), ex);
      }
      throw ex;
   }
}
```

文章：

[HandlerMethodReturnValueHandler](./return-value-handler/HandlerMethodReturnValueHandler.md)



### 不处理返回值情况

这里不看也行，一般不会进到这里面来。

#### isRequestNotModified

```java
private boolean isRequestNotModified(ServletWebRequest webRequest) {
   return webRequest.isNotModified();
}
public boolean isNotModified() {
    return this.notModified;
}
```

下面将对ServletWebRequest.notModified何时以及如何修改进行判断。

该字段修改自三个private方法ServletWebRequest.validateIfUnmodifiedSince、ServletWebRequest.validateIfNoneMatch、ServletWebRequest.validateIfModifiedSince，而它们都仅应用在ServletWebRequest.checkNotModified。

DispatcherServlet.doDispatch中调用了ServletWebRequest.checkNotModified。

```java
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
    ......................
    String method = request.getMethod();
    boolean isGet = HttpMethod.GET.matches(method);
    // 只有Get或Head请求才会进来。
    if (isGet || HttpMethod.HEAD.matches(method)) {
        // RequestHandleMappingAdapter只会返回-1。
        long lastModified = ha.getLastModified(request, mappedHandler.getHandler());
        // 调用ServletWebRequest.checkNotModified(null, lastModified)
        // etag为空或空字符串，lastModified为-1，永远到不了三个被修改的方法。
        if (new ServletWebRequest(request, response).checkNotModified(lastModified) && isGet) {
            return;
        }
    }
    ......................
}
```

姑且就将ServletWebRequest.notModified视为false来对待，即isRequestNotModified(webRequest)==false。

#### getResponseStatus

HandlerMethod.evaluateResponseStatus()中修改HandlerMethod.responseStatus。这个方法只有在HandlerMethod的构造函数中使用，因此是在spring启动解析出@RequestMapping标记方法——handler-method时就调用了。

文章：[@ResponseStatus](./@ResponseStatus使用.md)

不用@ResponseStatus，因此默认HandlerMethod.responseStatus==null。

#### mavContainer.isRequestHandled

​	这里应该讲究的是参数解析时候是否不是设置为true了。目前只看到ServletResponseMethodArgumentResolver是这样的，即参数存在ServletResponse、OutputStream、Writer类型。唉，感觉现在在做不用功。







## getMethodArgumentValues

ServletInvocableHandlerMethod.invokeForRequest(...)→

InvocableHandlerMethod.getMethodArgumentValues(...)

```java
// 提供HandlerMethod所需的参数
protected Object[] getMethodArgumentValues(NativeWebRequest request, @Nullable ModelAndViewContainer mavContainer,
      Object... providedArgs) throws Exception {

   // HandlerMethod.HandlerMethodParameter，用于解析handlermethod的参数，其对获取参数的注解进行了加强，貌似能穿透到泛型上。
   MethodParameter[] parameters = getMethodParameters();
   if (ObjectUtils.isEmpty(parameters)) {
      return EMPTY_ARGS;
   }

   Object[] args = new Object[parameters.length];
   // 解析每个参数
   for (int i = 0; i < parameters.length; i++) {
      MethodParameter parameter = parameters[i];
      // 提供参数名discover
      parameter.initParameterNameDiscovery(this.parameterNameDiscoverer);
      // 从现成的参数提供集合中查找能匹配上parameter的对象。要求：对象是parameter所需求的本身或子类。
      args[i] = findProvidedArgument(parameter, providedArgs);
      if (args[i] != null) {
         continue;
      }
      if (!this.resolvers.supportsParameter(parameter)) {
         throw new IllegalStateException(formatArgumentError(parameter, "No suitable resolver"));
      }
      try {
         // 未能在providedArgs中找到合适的。
         // 使用参数resolver提供适配parameter的对象。数据那么从request中获取，那么从parameter的注解上获取。
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

文章拓展：

[AbstractNamedValueMethodArgumentResolver](./arguement-resolver/AbstractNamedValueMethodArgumentResolver.md)

[ServletModelAttributeMethodProcessor](./arguement-resolver/ServletModelAttributeMethodProcessor.md)

[HandlerMethodArgumentResolver介绍](./arguement-resolver/HandlerMethodArgumentResolver介绍.md)

[others](./arguement-resolver/others.md)



## getModelAndView

```java
private ModelAndView getModelAndView(ModelAndViewContainer mavContainer,
      ModelFactory modelFactory, NativeWebRequest webRequest) throws Exception {

   modelFactory.updateModel(webRequest, mavContainer);
   if (mavContainer.isRequestHandled()) {
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

只有在mavContainer.isRequestHandled()设置为false的时候处理。

```java
public void invokeAndHandle(ServletWebRequest webRequest, ModelAndViewContainer mavContainer,
      Object... providedArgs) throws Exception {
    .............
    mavContainer.setRequestHandled(false);
    this.returnValueHandlers.handleReturnValue(
            returnValue, getReturnValueType(returnValue), mavContainer, webRequest);
    .............
}
```

可以看见，调用返回值前mavContainer.requestHandled被设置成了false，也就是需要在return-value-handler处理中不设置true才会返回视图。这个是前后端分离的思想吗？这里记录一下各个handler的情况吧。

- AsyncTaskMethodReturnValueHandler

  WebAsyncTask类型返回值处理。无返回值时设置mavc.requestHandled=true。

- CallableMethodReturnValueHandler

  Callable类型返回值处理。无返回值时设置mavc.requestHandled=true。

- DeferredResultMethodReturnValueHandler

  DeferredResult、ListenableFuture、CompletionStage类型返回值处理。无返回值时设置mavc.requestHandled=true。

- HttpEntityMethodProcessor

  无返回值时设置mavc.requestHandled=true。

- HttpHeadersReturnValueHandler

  直接设置为true。

- ModelAndViewMethodReturnValueHandler

  无返回值时设置mavc.requestHandled=true。

- RequestResponseBodyMethodProcessor

  直接设置为true。

- ResponseBodyEmitterReturnValueHandler

  无返回值，或者从返回值是ResponseEntity类型的并且从responseEntity.getBody()无获取值得情况下，设置时设置mavc.requestHandled=true

- StreamingResponseBodyReturnValueHandler

  无返回值时设置mavc.requestHandled=true。

- others

  没有设置情况，也就是都是false。

从网上的瞟了一眼得数据看，前后分离，后端只是返回前端需要的数据，无需渲染view。

```java
if (mavContainer.isRequestHandled()) {
    return null;
}
```

是代表前后端分离吗？视图已经备注里，意思是不用渲染？

算了，就姑且这样认为吧，仅讨论前后端分离的。


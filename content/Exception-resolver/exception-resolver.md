## 参考：

类之间的关系：[DispatcherServlet](./materials/DispatcherServlet.xmind)-HandlerExceptionResolver

[ExceptionHandlerMethodResolver](./ExceptionHandlerMethodResolver.md)

## 异常处理入口

```java
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



## **ExceptionHandlerExceptionResolver**解析

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
   // 进入
   return doResolveHandlerMethodException(request, response, handlerMethod, ex);
}
```

**重要地方**

```java
// 准备ServletInvocableHandlerMethod。
protected ModelAndView doResolveHandlerMethodException(HttpServletRequest request,
      HttpServletResponse response, @Nullable HandlerMethod handlerMethod, Exception exception) {

   // 找到适配的、优先级最高的ControllerAdvice，进行ServletInvocableHandlerMethod(@ExceptionHanlder标记方法)封装。
   ServletInvocableHandlerMethod exceptionHandlerMethod = getExceptionHandlerMethod(handlerMethod, exception);
   if (exceptionHandlerMethod == null) {
      return null;
   }

   // 异常方法参数的resolver比较少，注意。
   if (this.argumentResolvers != null) {
      exceptionHandlerMethod.setHandlerMethodArgumentResolvers(this.argumentResolvers);
   }
   if (this.returnValueHandlers != null) {
      exceptionHandlerMethod.setHandlerMethodReturnValueHandlers(this.returnValueHandlers);
   }

   ServletWebRequest webRequest = new ServletWebRequest(request, response);
   // 异常方法基本在享受等价的handler-method处理，mavc都是自己提供的。之前用的mavc弃用了。
   ModelAndViewContainer mavContainer = new ModelAndViewContainer();

   ArrayList<Throwable> exceptions = new ArrayList<>();
   try {
      if (logger.isDebugEnabled()) {
         logger.debug("Using @ExceptionHandler " + exceptionHandlerMethod);
      }
      // Expose causes as provided arguments as well
      // 获取<抛出异常exception，exception内层级的cause们，handlerMethod>，作为@ExceptionHandler标记方法的现成参数对象提供。
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

**重要**

```java
protected ServletInvocableHandlerMethod getExceptionHandlerMethod(
      @Nullable HandlerMethod handlerMethod, Exception exception) {

   Class<?> handlerType = null;

   if (handlerMethod != null) {
      // Local exception handler methods on the controller class itself.
      // To be invoked through the proxy, even in case of an interface-based proxy.
      handlerType = handlerMethod.getBeanType();
      ExceptionHandlerMethodResolver resolver = this.exceptionHandlerCache.get(handlerType);
      if (resolver == null) {
         // 构造ExceptionHandlerMethodResolver，解析handlerType-controller类的@ExceptionHandler。
         resolver = new ExceptionHandlerMethodResolver(handlerType);
         this.exceptionHandlerCache.put(handlerType, resolver);
      }
      Method method = resolver.resolveMethod(exception);
      // 不同于@InitBinder或@ModelAtrribute将全局放在优先级较高的位置上，@ExceptionHandler将局部先执行。若Controller类中有合适的适配的@ExceptionHandler标记方法，则就启用了。
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
      // advice.isApplicableToBeanType(...)将检验@ControllerAdvice提供的参数basePackages、assignableTypes、annotations。第一个是针对Controller类所在的位置，第二个是针对Controller类的继承、实现关系，第三个是针对Controller类本身使用的注解。从前向后匹配数据集，只要一满足就视为ControllerAdvice类命中Controller类。若都没有指定，则视为无障碍支持。
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


## 前言

DispatcherServlet.handlerMappings提供了HandlerMapping实现类，它们依据Request，提供对应的handler+其他处理流程。

这里先对RequestMappingHandlerMapping进行分析。

## 注册位置

文章：[Spring的WebMvcConfiguration设计](Spring的WebMvcConfiguration设计.md)

> 文章介绍了Mvc所需组件的创建位置（至少有部分是这样的）

RequestMappingHandlerMapping在WebMvcConfigurationSupport类内做了bean化处理



## RequestMappingHandlerMapping

文章：

[Spring的WebMvcConfiguration设计](Spring的WebMvcConfiguration设计.md)

> 介绍了RequestMappingHandlerMapping组件创建和初始化过程，初始化---解析Controller类，并将其做handler-method相关信息注册。

### 主干方法

```java
public final HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
   // 依据request，获取HandlerMethod，其包含handler(Controller instance)和handler-method，能支持反射调用。在AbstractHandlerMethodMapping中实现。
   Object handler = getHandlerInternal(request);
   if (handler == null) {
      // 当请求路径不存在的时候，可以给定HandlerMethod，让其去请求特殊方法，以解决请求映射不到handler-method的情况。这个需要自行配合。不难，但是得考虑设计是否合理。对了，又BasicErrorController，不应该由这个来处理。
      handler = getDefaultHandler();
   }
   // 若没有映射到，又没提供默认HandlerMethod，则说明此HandlerMapping不提供request所需的HandlerExecutionChain。
   if (handler == null) {
      return null;
   }
   // Bean name or resolved handler?
   if (handler instanceof String) {
      String handlerName = (String) handler;
      handler = obtainApplicationContext().getBean(handlerName);
   }

   // Ensure presence of cached lookupPath for interceptors and others
   if (!ServletRequestPathUtils.hasCachedPath(request)) {
      initLookupPath(request);
   }

   // 一般到这里就可以了。这里面提供了拦截器服务，将this.adaptedInterceptors逐个添加给executionChain。
   HandlerExecutionChain executionChain = getHandlerExecutionChain(handler, request);

   if (logger.isTraceEnabled()) {
      logger.trace("Mapped to " + handler);
   }
   else if (logger.isDebugEnabled() && !DispatcherType.ASYNC.equals(request.getDispatcherType())) {
      logger.debug("Mapped to " + executionChain.getHandler());
   }

   if (hasCorsConfigurationSource(handler) || CorsUtils.isPreFlightRequest(request)) {
      CorsConfiguration config = getCorsConfiguration(handler, request);
      if (getCorsConfigurationSource() != null) {
         CorsConfiguration globalConfig = getCorsConfigurationSource().getCorsConfiguration(request);
         config = (globalConfig != null ? globalConfig.combine(config) : config);
      }
      if (config != null) {
         config.validateAllowCredentials();
      }
      executionChain = getCorsHandlerExecutionChain(request, executionChain, config);
   }

   // 返回<request, HandlerMethod, Interceptors>调度资源。
   return executionChain;
}
```

### AbstractHandlerMethodMapping.getHandlerInternal(HttpServletRequest)

```java
protected HandlerMethod getHandlerInternal(HttpServletRequest request) throws Exception {
   // 获取request请求路径url
   String lookupPath = initLookupPath(request);
   this.mappingRegistry.acquireReadLock();
   try {
      // 依据请求路径获取HandlerMethod。
      // this.registry - <RequestMappingInfo, MappingRegistration(.....)>
      // this.pathLookup - <handler-method-path/url, List<RequestMappingInfo>>
      // 就凭上述两个内置属性就能获取，就不去细想了，没太多时间。
      HandlerMethod handlerMethod = lookupHandlerMethod(lookupPath, request);
      return (handlerMethod != null ? handlerMethod.createWithResolvedBean() : null);
   }
   finally {
      this.mappingRegistry.releaseReadLock();
   }
}
```
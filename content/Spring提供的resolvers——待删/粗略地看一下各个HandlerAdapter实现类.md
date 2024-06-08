## 来源

​	还是得从DispatcherServlet.doDispatch说起。获取、执行：

```java
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
    ......
    HandlerExecutionChain mappedHandler = null;
    ......
    mappedHandler = getHandler(processedRequest);
    ......
    HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());
    ......
    mv = ha.handle(processedRequest, response, mappedHandler.getHandler());
    ......
}
```

下面提供两个get获取地方法：

```java
protected HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
   if (this.handlerMappings != null) {
      for (HandlerMapping mapping : this.handlerMappings) {
         // 显然HandlerMapping与request之间存在适配关系。
         HandlerExecutionChain handler = mapping.getHandler(request);
         if (handler != null) {
            return handler;
         }
      }
   }
   return null;
}
```

```java
protected HandlerAdapter getHandlerAdapter(Object handler) throws ServletException {
   if (this.handlerAdapters != null) {
      for (HandlerAdapter adapter : this.handlerAdapters) {
         // 显然HandlerAdapter与handler存在适配关系。
         if (adapter.supports(handler)) {
            return adapter;
         }
      }
   }
   throw new ServletException("No adapter for handler [" + handler +
         "]: The DispatcherServlet configuration needs to include a HandlerAdapter that supports this handler");
}
```

上述的获取关系已经在DispatcherServlet的画布1中展示出来了，以下补充HandlerAdapter与HandlerMethod的具体适配过程。



**RequestMappingHandlerAdapter**

```java
// 用了父类的
public final boolean supports(Object handler) {
   return (handler instanceof HandlerMethod && supportsInternal((HandlerMethod) handler));
}
```

**HttpRequestHandlerAdapter**

```java
public boolean supports(Object handler) {
   return (handler instanceof HttpRequestHandler);
}
```

**HandlerFunctionAdapter**

```java
public boolean supports(Object handler) {
   return handler instanceof HandlerFunction;
}
```

**SimpleControllerHandlerAdapter**

```java
public boolean supports(Object handler) {
   return (handler instanceof Controller);
}
```


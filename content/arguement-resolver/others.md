## HandlerMethodArgumentResolverComposite

本类屏蔽了request+method-parameter与具体调用resolver的直接关联，本类以逐个遍历resolvers，以检阅受支持resolver，找到后缓存——预热。注意：resolver先支持先匹配，因此resolver之间的先后顺序有要求，当自定义resolver时要注意位置。

```java
public boolean supportsParameter(MethodParameter parameter) {
   return getArgumentResolver(parameter) != null;
}
private HandlerMethodArgumentResolver getArgumentResolver(MethodParameter parameter) {
    HandlerMethodArgumentResolver result = this.argumentResolverCache.get(parameter);
    if (result == null) {
        for (HandlerMethodArgumentResolver resolver : this.argumentResolvers) {
            // 由此看，resolver的先后关系，有关系。除非乱序也可以，不过看到了源码则不然，实际上的确有顺序上的考究。
            if (resolver.supportsParameter(parameter)) {
                result = resolver;
                this.argumentResolverCache.put(parameter, result);
                break;
            }
        }
    }
    return result;
}
```

```java
// 使用环境
public Object resolveArgument(MethodParameter parameter, @Nullable ModelAndViewContainer mavContainer,
      NativeWebRequest webRequest, @Nullable WebDataBinderFactory binderFactory) throws Exception {

   HandlerMethodArgumentResolver resolver = getArgumentResolver(parameter);
   if (resolver == null) {
      throw new IllegalArgumentException("Unsupported parameter type [" +
            parameter.getParameterType().getName() + "]. supportsParameter should be called first.");
   }
   return resolver.resolveArgument(parameter, mavContainer, webRequest, binderFactory);
}
```

## RequestParamMapMethodArgumentResolver

```java
// 受此resolver支持，method parameter代表的参数的签名必须如此：
// @RequestParam Map.class
public boolean supportsParameter(MethodParameter parameter) {
   RequestParam requestParam = parameter.getParameterAnnotation(RequestParam.class);
   return (requestParam != null && Map.class.isAssignableFrom(parameter.getParameterType()) &&
         !StringUtils.hasText(requestParam.name()));
}
```

```java
// MultiValueMap.class、Part.class和普通数据。它们数据存放数据的位置在webRequest的不同位置。
// 这里有两块高度重合的代码块，前者针对MultiValueMap，后者针对Regular Map，这区别在于容器的选择，后者能兼容前者，但后者不能。
public Object resolveArgument(MethodParameter parameter, @Nullable ModelAndViewContainer mavContainer,
      NativeWebRequest webRequest, @Nullable WebDataBinderFactory binderFactory) throws Exception {

   ResolvableType resolvableType = ResolvableType.forMethodParameter(parameter);

   // 根据method-parameter代表参数的Map类型决定执行哪一个逻辑代码块。
   if (MultiValueMap.class.isAssignableFrom(parameter.getParameterType())) {
      // MultiValueMap
      Class<?> valueType = resolvableType.as(MultiValueMap.class).getGeneric(1).resolve();
      if (valueType == MultipartFile.class) {
          // 只提取MultipartFile在webReqeust内独立于RequestFacade。
         MultipartRequest multipartRequest = MultipartResolutionDelegate.resolveMultipartRequest(webRequest);
         return (multipartRequest != null ? multipartRequest.getMultiFileMap() : new LinkedMultiValueMap<>(0));
      }
      else if (valueType == Part.class) {
         HttpServletRequest servletRequest = webRequest.getNativeRequest(HttpServletRequest.class);
         if (servletRequest != null && MultipartResolutionDelegate.isMultipartRequest(servletRequest)) {
            // 读取connector.request.parts
            Collection<Part> parts = servletRequest.getParts();
            LinkedMultiValueMap<String, Part> result = new LinkedMultiValueMap<>(parts.size());
            for (Part part : parts) {
               result.add(part.getName(), part);
            }
            return result;
         }
         return new LinkedMultiValueMap<>(0);
      }
      else {
         // 感觉这里不会经过
         // connector.reqeuest.parameterMap
         Map<String, String[]> parameterMap = webRequest.getParameterMap();
         // 这里用的容器不一样，将普通数据放置到了MultiValueMap的容器上。
         MultiValueMap<String, String> result = new LinkedMultiValueMap<>(parameterMap.size());
         parameterMap.forEach((key, values) -> {
            for (String value : values) {
               result.add(key, value);
            }
         });
         return result;
      }
   }

   else {
      // Regular Map
      // 上面是.as(MultiValueMap.class)，这里是.asMap()。貌似也就这个区别了。
      Class<?> valueType = resolvableType.asMap().getGeneric(1).resolve();
      if (valueType == MultipartFile.class) {
         MultipartRequest multipartRequest = MultipartResolutionDelegate.resolveMultipartRequest(webRequest);
         return (multipartRequest != null ? multipartRequest.getFileMap() : new LinkedHashMap<>(0));
      }
      else if (valueType == Part.class) {
         HttpServletRequest servletRequest = webRequest.getNativeRequest(HttpServletRequest.class);
         if (servletRequest != null && MultipartResolutionDelegate.isMultipartRequest(servletRequest)) {
            Collection<Part> parts = servletRequest.getParts();
            LinkedHashMap<String, Part> result = CollectionUtils.newLinkedHashMap(parts.size());
            for (Part part : parts) {
               if (!result.containsKey(part.getName())) {
                  result.put(part.getName(), part);
               }
            }
            return result;
         }
         return new LinkedHashMap<>(0);
      }
      else {
         Map<String, String[]> parameterMap = webRequest.getParameterMap();
         // 这里用的容器不一样
         Map<String, String> result = CollectionUtils.newLinkedHashMap(parameterMap.size());
         parameterMap.forEach((key, values) -> {
            if (values.length > 0) {
               result.put(key, values[0]);
            }
         });
         return result;
      }
   }
}
```



## PathVariableMapMethodArgumentResolver

```java
// 受此resolver支持，method parameter代表的参数的签名必须如此：
// @PathVariable Map.class
public boolean supportsParameter(MethodParameter parameter) {
   PathVariable ann = parameter.getParameterAnnotation(PathVariable.class);
   return (ann != null && Map.class.isAssignableFrom(parameter.getParameterType()) &&
         !StringUtils.hasText(ann.value()));
}
```

```java
// 从connector.Request.attributes中返回k-HandlerMapping.URI_TEMPLATE_VARIABLES_ATTRIBUTE的v值。v值是requestMapping.name中{}起来的内容 - request-url对应的片段的k-v键值对Map。
public Object resolveArgument(MethodParameter parameter, @Nullable ModelAndViewContainer mavContainer,
      NativeWebRequest webRequest, @Nullable WebDataBinderFactory binderFactory) throws Exception {

   @SuppressWarnings("unchecked")
   Map<String, String> uriTemplateVars =
         (Map<String, String>) webRequest.getAttribute(
               HandlerMapping.URI_TEMPLATE_VARIABLES_ATTRIBUTE, RequestAttributes.SCOPE_REQUEST);

   if (!CollectionUtils.isEmpty(uriTemplateVars)) {
      return new LinkedHashMap<>(uriTemplateVars);
   }
   else {
      return Collections.emptyMap();
   }
}
```



## RequestHeaderMapMethodArgumentResolver

```java
// 受此resolver支持，method parameter代表的参数的签名必须如此：
// @RequestHeader("可有可无") Map.class
public boolean supportsParameter(MethodParameter parameter) {
   return (parameter.hasParameterAnnotation(RequestHeader.class) &&
         Map.class.isAssignableFrom(parameter.getParameterType()));
}
```

```java
// 
public Object resolveArgument(MethodParameter parameter, @Nullable ModelAndViewContainer mavContainer,
      NativeWebRequest webRequest, @Nullable WebDataBinderFactory binderFactory) throws Exception {

   Class<?> paramType = parameter.getParameterType();
   // Multipart内容
   if (MultiValueMap.class.isAssignableFrom(paramType)) {
      MultiValueMap<String, String> result;
      if (HttpHeaders.class.isAssignableFrom(paramType)) {
         result = new HttpHeaders();
      }
      else {
         result = new LinkedMultiValueMap<>();
      }
      // 来源数据：coyote.request.headers
      for (Iterator<String> iterator = webRequest.getHeaderNames(); iterator.hasNext();) {
         String headerName = iterator.next();
         String[] headerValues = webRequest.getHeaderValues(headerName);
         if (headerValues != null) {
            // 就是换个容器。
            for (String headerValue : headerValues) {
               result.add(headerName, headerValue);
            }
         }
      }
      return result;
   }
   else {
      // regular map
      Map<String, String> result = new LinkedHashMap<>();
      for (Iterator<String> iterator = webRequest.getHeaderNames(); iterator.hasNext();) {
         String headerName = iterator.next();
         String headerValue = webRequest.getHeader(headerName);
         if (headerValue != null) {
            result.put(headerName, headerValue);
         }
      }
      return result;
   }
}
```



## MatrixVariableMapMethodArgumentResolver

```java
// 受此resolver支持，method parameter代表的参数的签名必须如此：
// @MatrixVariable(pathVar="可有可无") Map.class
public boolean supportsParameter(MethodParameter parameter) {
   MatrixVariable matrixVariable = parameter.getParameterAnnotation(MatrixVariable.class);
   return (matrixVariable != null && Map.class.isAssignableFrom(parameter.getParameterType()) &&
         !StringUtils.hasText(matrixVariable.name()));
}
```

```java
// pathVar可有可无，若无，则提取全部的矩阵序列的矩阵变量，将他们的映射关系扁平化（忽略矩阵序列-pathVar这个隔阂），二维变一维。
public Object resolveArgument(MethodParameter parameter, @Nullable ModelAndViewContainer mavContainer,
      NativeWebRequest request, @Nullable WebDataBinderFactory binderFactory) throws Exception {

   @SuppressWarnings("unchecked")
   Map<String, MultiValueMap<String, String>> matrixVariables =
         (Map<String, MultiValueMap<String, String>>) request.getAttribute(
               HandlerMapping.MATRIX_VARIABLES_ATTRIBUTE, RequestAttributes.SCOPE_REQUEST);

   if (CollectionUtils.isEmpty(matrixVariables)) {
      return Collections.emptyMap();
   }

   MultiValueMap<String, String> map = new LinkedMultiValueMap<>();
   MatrixVariable ann = parameter.getParameterAnnotation(MatrixVariable.class);
   Assert.state(ann != null, "No MatrixVariable annotation");
   String pathVariable = ann.pathVar();

   if (!pathVariable.equals(ValueConstants.DEFAULT_NONE)) {
      MultiValueMap<String, String> mapForPathVariable = matrixVariables.get(pathVariable);
      if (mapForPathVariable == null) {
         return Collections.emptyMap();
      }
      map.putAll(mapForPathVariable);
   }
   else {
      for (MultiValueMap<String, String> vars : matrixVariables.values()) {
         vars.forEach((name, values) -> {
            for (String value : values) {
               map.add(name, value);
            }
         });
      }
   }

   // 这列做出的行为：若method-parameter代表Map<String, String>，是一对一的单一映射，那么选中MultiValueMap中value代表的List中的first eleme，作为key的一对一映射对象。
   return (isSingleValueMap(parameter) ? map.toSingleValueMap() : map);
}
```

```java
private boolean isSingleValueMap(MethodParameter parameter) {
   if (!MultiValueMap.class.isAssignableFrom(parameter.getParameterType())) {
      ResolvableType[] genericTypes = ResolvableType.forMethodParameter(parameter).getGenerics();
      if (genericTypes.length == 2) {
         return !List.class.isAssignableFrom(genericTypes[1].toClass());
      }
   }
   return false;
}
```

## PrincipalMethodArgumentResolver

认作弃用。

```java
// method parmeter代表Principal.class
// 并且必须使用注解，不然ServletRequestMethodArgumentResolver会较早而判断选中
// 感觉这个resolver被弃用了，
// 不知道如何传递，request才会收纳，附带Principal对象。
public boolean supportsParameter(MethodParameter parameter) {
   return Principal.class.isAssignableFrom(parameter.getParameterType());
}
```

```java
// 
public Object resolveArgument(MethodParameter parameter, @Nullable ModelAndViewContainer mavContainer,
      NativeWebRequest webRequest, @Nullable WebDataBinderFactory binderFactory) throws Exception {

   HttpServletRequest request = webRequest.getNativeRequest(HttpServletRequest.class);
   if (request == null) {
      throw new IllegalStateException("Current request is not of type HttpServletRequest: " + webRequest);
   }

   Principal principal = request.getUserPrincipal();
   if (principal != null && !parameter.getParameterType().isInstance(principal)) {
      throw new IllegalStateException("Current user principal is not of type [" +
            parameter.getParameterType().getName() + "]: " + principal);
   }

   return principal;
}
```

## ContinuationHandlerMethodArgumentResolver

涉及kotlin.coroutines.Continuation，不予理会

```java
public boolean supportsParameter(MethodParameter parameter) {
   return "kotlin.coroutines.Continuation".equals(parameter.getParameterType().getName());
}
```

```java
public Object resolveArgument(MethodParameter parameter, @Nullable ModelAndViewContainer mavContainer,
      NativeWebRequest webRequest, @Nullable WebDataBinderFactory binderFactory) throws Exception {

   return null;
}
```

## UriComponentsBuilderMethodArgumentResolver

​	创建关于UriComponents.class构造者类。暂不予理会。

```java
// method parameter代表类型是UriComponentsBuilder.class或ServletUriComponentsBuilder.class
public boolean supportsParameter(MethodParameter parameter) {
   Class<?> type = parameter.getParameterType();
   return (UriComponentsBuilder.class == type || ServletUriComponentsBuilder.class == type);
}
```

```java
public Object resolveArgument(MethodParameter parameter, @Nullable ModelAndViewContainer mavContainer,
      NativeWebRequest webRequest, @Nullable WebDataBinderFactory binderFactory) throws Exception {

   HttpServletRequest request = webRequest.getNativeRequest(HttpServletRequest.class);
   Assert.state(request != null, "No HttpServletRequest");
   return ServletUriComponentsBuilder.fromServletMapping(request);
}
```

## ServletRequestMethodArgumentResolver

```java
// 不赘述了，还是前往类注释看即可。拦截了好多常用的。
public boolean supportsParameter(MethodParameter parameter) {
   Class<?> paramType = parameter.getParameterType();
   return (WebRequest.class.isAssignableFrom(paramType) ||
         ServletRequest.class.isAssignableFrom(paramType) ||
         MultipartRequest.class.isAssignableFrom(paramType) ||
         HttpSession.class.isAssignableFrom(paramType) ||
         (pushBuilder != null && pushBuilder.isAssignableFrom(paramType)) ||
         (Principal.class.isAssignableFrom(paramType) && !parameter.hasParameterAnnotations()) ||
         InputStream.class.isAssignableFrom(paramType) ||
         Reader.class.isAssignableFrom(paramType) ||
         HttpMethod.class == paramType ||
         Locale.class == paramType ||
         TimeZone.class == paramType ||
         ZoneId.class == paramType);
}
```

```java
public Object resolveArgument(MethodParameter parameter, @Nullable ModelAndViewContainer mavContainer,
      NativeWebRequest webRequest, @Nullable WebDataBinderFactory binderFactory) throws Exception {

   Class<?> paramType = parameter.getParameterType();

   // 三个WebRequest实现、继承链上的Class。ServletWebRequest是目前顶层（最外层）的request封装类
   // 返回最外一层的request
   // WebRequest / NativeWebRequest / ServletWebRequest
   if (WebRequest.class.isAssignableFrom(paramType)) {
      if (!paramType.isInstance(webRequest)) {
         throw new IllegalStateException(
               "Current request is not of type [" + paramType.getName() + "]: " + webRequest);
      }
      return webRequest;
   }

   // ServletRequest和HttpServletRequest是一个继承链上的；MultipartRequest和MultipartHttpServletRequest是另一个继承链上的。
   // 前者是不包含文件传输的情况下的request封装类的接口，后者是包含文件的情况下的reqeust封装类的泛型。
   // 返回第二层的request。
   // ServletRequest / HttpServletRequest / MultipartRequest / MultipartHttpServletRequest
   if (ServletRequest.class.isAssignableFrom(paramType) || MultipartRequest.class.isAssignableFrom(paramType)) {
      return resolveNativeRequest(webRequest, paramType);
   }

   // 剩余类型的解析，剩余的都在此中解析
   // HttpServletRequest required for all further argument types
   // HttpServletRequest是RequestFacade的接口和MultipartHttpServletRequest的父接口。
   return resolveArgument(paramType, resolveNativeRequest(webRequest, HttpServletRequest.class));
}

private <T> T resolveNativeRequest(NativeWebRequest webRequest, Class<T> requiredType) {
   T nativeRequest = webRequest.getNativeRequest(requiredType);
   if (nativeRequest == null) {
      throw new IllegalStateException(
            "Current request is not of type [" + requiredType.getName() + "]: " + webRequest);
   }
   return nativeRequest;
}

@Nullable
private Object resolveArgument(Class<?> paramType, HttpServletRequest request) throws IOException {
   // parameterType-HttpSession		获取session然后向上造型
   if (HttpSession.class.isAssignableFrom(paramType)) {
      HttpSession session = request.getSession();
      if (session != null && !paramType.isInstance(session)) {
         throw new IllegalStateException(
               "Current session is not of type [" + paramType.getName() + "]: " + session);
      }
      return session;
   }
   // pushBuilder尝试获取javax.servlet.http.PushBuilder 的Class，否则null。
   // 推送构造，不知道用于什么，怎么用。
   else if (pushBuilder != null && pushBuilder.isAssignableFrom(paramType)) {
      return PushBuilderDelegate.resolvePushBuilder(request, paramType);
   }
   // connector.request中有usingReader和usingInputStream两个互斥的布尔类型，只有request支持的哪一种才行得通，不然会抛出异常。
   // 字节输入流，new CoyoteInputStream(connector.request.inputBuffer)封装
   else if (InputStream.class.isAssignableFrom(paramType)) {
      InputStream inputStream = request.getInputStream();
      if (inputStream != null && !paramType.isInstance(inputStream)) {
         throw new IllegalStateException(
               "Request input stream is not of type [" + paramType.getName() + "]: " + inputStream);
      }
      return inputStream;
   }
   // 字符输入流，new CoyoteReader(connector.request.inputBuffer)封装
   else if (Reader.class.isAssignableFrom(paramType)) {
      Reader reader = request.getReader();
      if (reader != null && !paramType.isInstance(reader)) {
         throw new IllegalStateException(
               "Request body reader is not of type [" + paramType.getName() + "]: " + reader);
      }
      return reader;
   }
   // 不赘述了，PrincipalMethodArgumentResolver中提到过了。
   else if (Principal.class.isAssignableFrom(paramType)) {
      Principal userPrincipal = request.getUserPrincipal();
      if (userPrincipal != null && !paramType.isInstance(userPrincipal)) {
         throw new IllegalStateException(
               "Current user principal is not of type [" + paramType.getName() + "]: " + userPrincipal);
      }
      return userPrincipal;
   }
   // coyote.request.methodMB.toString()
   else if (HttpMethod.class == paramType) {
      return HttpMethod.resolve(request.getMethod());
   }
   // 区域，不熟。
   else if (Locale.class == paramType) {
      return RequestContextUtils.getLocale(request);
   }
   // 时区。这个时区指什么呢？请求者的时区，还是响应请求时的系统时区？因该是后者。
   else if (TimeZone.class == paramType) {
      TimeZone timeZone = RequestContextUtils.getTimeZone(request);
      return (timeZone != null ? timeZone : TimeZone.getDefault());
   }
   // 时区ID，暂不予理会。
   else if (ZoneId.class == paramType) {
      TimeZone timeZone = RequestContextUtils.getTimeZone(request);
      return (timeZone != null ? timeZone.toZoneId() : ZoneId.systemDefault());
   }

   // Should never happen...
   throw new UnsupportedOperationException("Unknown parameter type: " + paramType.getName());
}
```

```java
private static class PushBuilderDelegate {

   @Nullable
   public static Object resolvePushBuilder(HttpServletRequest request, Class<?> paramType) {
      PushBuilder pushBuilder = request.newPushBuilder();
      if (pushBuilder != null && !paramType.isInstance(pushBuilder)) {
         throw new IllegalStateException(
               "Current push builder is not of type [" + paramType.getName() + "]: " + pushBuilder);
      }
      return pushBuilder;

   }
}
```

## RequestPartMethodArgumentResolver

```java
// 使用了@RequestPart 或者 method-parameter代表类型是MultipartFile.class或Part.class本身或其数组、Collection。
public boolean supportsParameter(MethodParameter parameter) {
   if (parameter.hasParameterAnnotation(RequestPart.class)) {
      return true;
   }
   else {
      if (parameter.hasParameterAnnotation(RequestParam.class)) {
         return false;
      }
      return MultipartResolutionDelegate.isMultipartArgument(parameter.nestedIfOptional());
   }
}
```

```java
public Object resolveArgument(MethodParameter parameter, @Nullable ModelAndViewContainer mavContainer,
      NativeWebRequest request, @Nullable WebDataBinderFactory binderFactory) throws Exception {

   HttpServletRequest servletRequest = request.getNativeRequest(HttpServletRequest.class);
   Assert.state(servletRequest != null, "No HttpServletRequest");

   RequestPart requestPart = parameter.getParameterAnnotation(RequestPart.class);
   boolean isRequired = ((requestPart == null || requestPart.required()) && !parameter.isOptional());

   String name = getPartName(parameter, requestPart);
   parameter = parameter.nestedIfOptional();
   Object arg = null;

   Object mpArg = MultipartResolutionDelegate.resolveMultipartArgument(name, parameter, servletRequest);
   if (mpArg != MultipartResolutionDelegate.UNRESOLVABLE) {
      arg = mpArg;
   }
   else {
      try {
         // 这段解析HttpInputMessage没看懂，放着先，现在去学习一下绑定。
         HttpInputMessage inputMessage = new RequestPartServletServerHttpRequest(servletRequest, name);
         arg = readWithMessageConverters(inputMessage, parameter, parameter.getNestedGenericParameterType());
         if (binderFactory != null) {
            WebDataBinder binder = binderFactory.createBinder(request, arg, name);
            if (arg != null) {
               validateIfApplicable(binder, parameter);
               if (binder.getBindingResult().hasErrors() && isBindExceptionRequired(binder, parameter)) {
                  throw new MethodArgumentNotValidException(parameter, binder.getBindingResult());
               }
            }
            if (mavContainer != null) {
               mavContainer.addAttribute(BindingResult.MODEL_KEY_PREFIX + name, binder.getBindingResult());
            }
         }
      }
      catch (MissingServletRequestPartException | MultipartException ex) {
         if (isRequired) {
            throw ex;
         }
      }
   }

   if (arg == null && isRequired) {
      if (!MultipartResolutionDelegate.isMultipartRequest(servletRequest)) {
         throw new MultipartException("Current request is not a multipart request");
      }
      else {
         throw new MissingServletRequestPartException(name);
      }
   }
   return adaptArgumentIfNecessary(arg, parameter);
}
```


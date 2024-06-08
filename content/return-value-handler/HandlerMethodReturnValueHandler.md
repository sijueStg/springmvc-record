放我把所有的的handler的support罗列一下吧。

## HandlerMethodReturnValueHandler

```java
boolean supportsReturnType(MethodParameter returnType);
```

```java
void handleReturnValue(@Nullable Object returnValue, MethodParameter returnType,
      ModelAndViewContainer mavContainer, NativeWebRequest webRequest) throws Exception;
```

## ViewNameMethodReturnValueHandler

```java
// 处理无返回对象或者文本（String）的情况
public boolean supportsReturnType(MethodParameter returnType) {
   Class<?> paramType = returnType.getParameterType();
   return (void.class == paramType || CharSequence.class.isAssignableFrom(paramType));
}
```

## MapMethodProcessor

```java
// 返回类型是Map.class
public boolean supportsReturnType(MethodParameter returnType) {
    return Map.class.isAssignableFrom(returnType.getParameterType());
}
```

## ViewMethodReturnValueHandler

```java
public boolean supportsReturnType(MethodParameter returnType) {
   return View.class.isAssignableFrom(returnType.getParameterType());
}
```

## StreamingResponseBodyReturnValueHandler

```java
// 处理返回返回类型是StreamingResponseBody.class或ResponseEntity.class
// 前者与异步Stream流有关；后者与spring.web的RestTemplate有关。
public boolean supportsReturnType(MethodParameter returnType) {
   if (StreamingResponseBody.class.isAssignableFrom(returnType.getParameterType())) {
      return true;
   }
   else if (ResponseEntity.class.isAssignableFrom(returnType.getParameterType())) {
      Class<?> bodyType = ResolvableType.forMethodParameter(returnType).getGeneric().resolve();
      return (bodyType != null && StreamingResponseBody.class.isAssignableFrom(bodyType));
   }
   return false;
}
```

## DeferredResultMethodReturnValueHandler

```java
// 返回类型是 DeferredResult.class 或 ListenableFuture.class 或 CompletionStage.class。陌生，可怕。
public boolean supportsReturnType(MethodParameter returnType) {
   Class<?> type = returnType.getParameterType();
   return (DeferredResult.class.isAssignableFrom(type) ||
         ListenableFuture.class.isAssignableFrom(type) ||
         CompletionStage.class.isAssignableFrom(type));
}
```

## HandlerMethodReturnValueHandlerComposite

handler选择器。

```java
public void handleReturnValue(@Nullable Object returnValue, MethodParameter returnType,
      ModelAndViewContainer mavContainer, NativeWebRequest webRequest) throws Exception {

   // 选择handler。
   HandlerMethodReturnValueHandler handler = selectHandler(returnValue, returnType);
   if (handler == null) {
      throw new IllegalArgumentException("Unknown return value type: " + returnType.getParameterType().getName());
   }
   handler.handleReturnValue(returnValue, returnType, mavContainer, webRequest);
}
```

```java
private HandlerMethodReturnValueHandler getReturnValueHandler(MethodParameter returnType) {
   for (HandlerMethodReturnValueHandler handler : this.returnValueHandlers) {
      if (handler.supportsReturnType(returnType)) {
         return handler;
      }
   }
   return null;
}
```

## HttpHeadersReturnValueHandler

```java
// 返回类型是HttpHeaders.class
public boolean supportsReturnType(MethodParameter returnType) {
   return HttpHeaders.class.isAssignableFrom(returnType.getParameterType());
}
```

## CallableMethodReturnValueHandler

```java
// 返回类型是Callable.class
public boolean supportsReturnType(MethodParameter returnType) {
   return Callable.class.isAssignableFrom(returnType.getParameterType());
}
```

## ModelMethodProcessor

```java
// 返回类型是Model.class
public boolean supportsReturnType(MethodParameter returnType) {
   return Model.class.isAssignableFrom(returnType.getParameterType());
}
```

## ModelAttributeMethodProcessor

```java
// 返回的类型上使用了@ModelAttribute，或者当前handler认定不需要类型标记有注解且返回类型不是简单类型————涉及基础类型及其包装类，它们的数组和Collection，还有一些基底的类，还是自己看一看比较好。
public boolean supportsReturnType(MethodParameter returnType) {
   return (returnType.hasMethodAnnotation(ModelAttribute.class) ||
         (this.annotationNotRequired && !BeanUtils.isSimpleProperty(returnType.getParameterType())));
}
```

## ResponseBodyEmitterReturnValueHandler

```java
// 判断返回类型是不是ResponseEntity.class，若是则获取其泛型类型，若不是则返回本身类型。获得的类型是ResponseBodyEmitter.class 或 响应类型（超出理解范畴了）。
public boolean supportsReturnType(MethodParameter returnType) {
   Class<?> bodyType = ResponseEntity.class.isAssignableFrom(returnType.getParameterType()) ?
         ResolvableType.forMethodParameter(returnType).getGeneric().resolve() :
         returnType.getParameterType();

   return (bodyType != null && (ResponseBodyEmitter.class.isAssignableFrom(bodyType) ||
         this.reactiveHandler.isReactiveType(bodyType)));
}
```

## ModelAndViewMethodReturnValueHandler

```java
// 返回类型是 ModelAndView.class
public boolean supportsReturnType(MethodParameter returnType) {
   return ModelAndView.class.isAssignableFrom(returnType.getParameterType());
}
```

## ModelAndViewResolverMethodReturnValueHandler

```java
// 用到了直接返回true。这是一定是放在最后的，用于保底的。
public boolean supportsReturnType(MethodParameter returnType) {
   return true;
}
```

## AbstractMessageConverterMethodProcessor

### *RequestResponseBodyMethodProcessor

就拿你下手了解一下。

```java
// 调用方法所在类被@ResponseBody注解，或者调用方法被@ResponseBody注解。
public boolean supportsReturnType(MethodParameter returnType) {
   // .getContainingClass()所在Controller.class
   return (AnnotatedElementUtils.hasAnnotation(returnType.getContainingClass(), ResponseBody.class) ||
         returnType.hasMethodAnnotation(ResponseBody.class));
}
```

```java
public void handleReturnValue(@Nullable Object returnValue, MethodParameter returnType,
      ModelAndViewContainer mavContainer, NativeWebRequest webRequest)
      throws IOException, HttpMediaTypeNotAcceptableException, HttpMessageNotWritableException {

   mavContainer.setRequestHandled(true);
   ServletServerHttpRequest inputMessage = createInputMessage(webRequest);
   ServletServerHttpResponse outputMessage = createOutputMessage(webRequest);

   // Try even with null return value. ResponseBodyAdvice could get involved.
   // 切面Advice起了作用。
   writeWithMessageConverters(returnValue, returnType, inputMessage, outputMessage);
}
```

**writeWithMessageConverters**

```java
protected <T> void writeWithMessageConverters(@Nullable T value, MethodParameter returnType,
      ServletServerHttpRequest inputMessage, ServletServerHttpResponse outputMessage)
      throws IOException, HttpMediaTypeNotAcceptableException, HttpMessageNotWritableException {

   Object body;
   // value所对应的类Class
   Class<?> valueType;
   // 可能是ParameterizedTypeImpl，也可能是ResolvableType，具备解析value的泛型信息（还有个所属来源，这是可能出现的情况，不是不然，一般就是ParameterizedTypeImpl.class）。
   Type targetType;

   // 填充body、valueType、targetType。
   if (value instanceof CharSequence) {
      body = value.toString();
      valueType = String.class;
      targetType = String.class;
   }
   else {
      body = value;
      // value==null时，采用returnType记录的返回类型（方法声明的返回类型）
      valueType = getReturnValueType(body, returnType);
      // getGenericType(returnType)：有泛型参数则返回诸如ParameterizedTypeImpl类型的实例，该Type专门用来描述具有泛型参数的类型。没有则直接返回Class即可，它也是Type的一种。
      // returnType.getContainingClass()：method所在class。
      targetType = GenericTypeResolver.resolveType(getGenericType(returnType), returnType.getContainingClass());
   }

   // 返回类型时InputStreamResource.class或是Resource.class继承链上的。
   if (isResourceType(value, returnType)) {
      outputMessage.getHeaders().set(HttpHeaders.ACCEPT_RANGES, "bytes");
      if (value != null && inputMessage.getHeaders().getFirst(HttpHeaders.RANGE) != null &&
            outputMessage.getServletResponse().getStatus() == 200) {
         Resource resource = (Resource) value;
         try {
            List<HttpRange> httpRanges = inputMessage.getHeaders().getRange();
            outputMessage.getServletResponse().setStatus(HttpStatus.PARTIAL_CONTENT.value());
            body = HttpRange.toResourceRegions(httpRanges, resource);
            valueType = body.getClass();
            targetType = RESOURCE_REGION_LIST_TYPE;
         }
         catch (IllegalArgumentException ex) {
            outputMessage.getHeaders().set(HttpHeaders.CONTENT_RANGE, "bytes */" + resource.contentLength());
            outputMessage.getServletResponse().setStatus(HttpStatus.REQUESTED_RANGE_NOT_SATISFIABLE.value());
         }
      }
   }

   // 这部分内容就是获取目标的selectedMediaType。
   MediaType selectedMediaType = null;
   // MediaType是从request.headers中拿出来的Content-Type字段数据的解析封装对象。MediaType是MimeType的子类。
   // 就MimeType所述，其代表MIME Type，其用于诸如HTTP等互联网协议中。
   MediaType contentType = outputMessage.getHeaders().getContentType();
   // 查看Content-Type字段是不是具体的，是具体的就是预设的意思。抽象的标准是MimeType的type和subtype字段是不是等价于"*"（这代表通配符），若subtype.startWith("*+")也算抽象。
   boolean isContentTypePreset = contentType != null && contentType.isConcrete();
   if (isContentTypePreset) {
      if (logger.isDebugEnabled()) {
         logger.debug("Found 'Content-Type:" + contentType + "' in response");
      }
      selectedMediaType = contentType;
   }
   else {
      // 不是预设的情况下，应用Content-Type通配符，进行匹配。
      HttpServletRequest request = inputMessage.getServletRequest();
      List<MediaType> acceptableTypes;
      try {
         // 用于从request中提取另一种形式存储的MimeType数据。headers.Accept中的数据为主。不过提取策略很多而且还可能具有嵌套形式。有过滤、有固定非request源头的固定给定MediaType。
         acceptableTypes = getAcceptableMediaTypes(request);
      }
      catch (HttpMediaTypeNotAcceptableException ex) {
         int series = outputMessage.getServletResponse().getStatus() / 100;
         if (body == null || series == 4 || series == 5) {
            if (logger.isDebugEnabled()) {
               logger.debug("Ignoring error response content (if any). " + ex);
            }
            return;
         }
         throw ex;
      }
      // 这里从三个地方获取MedieaType。① org.springframework.web.servlet.HandlerMapping.producibleMediaTypes中获取 ② this.messageConverters中获取支持valueType的Mediea（具体怎么支持就不深究了）。③ MediaType.ALL —— "*"。仅从一个地方获取，先从序号小的判断起。
      // HttpMessageConverter messageConverter用于将request->reponse的数据转换、转移。其明确有支持的MimeType集合，比如<type, subtype>->{<application, json>, <application, +*json>, <*, *>}
      List<MediaType> producibleTypes = getProducibleMediaTypes(request, valueType, targetType);

      if (body != null && producibleTypes.isEmpty()) {
         throw new HttpMessageNotWritableException(
               "No converter found for return value of type: " + valueType);
      }
      List<MediaType> mediaTypesToUse = new ArrayList<>();
      // 做连个MediaType集合的相互比较匹配。<requestedType, producibleType> -> newType(可能是原对象requestedType)，将其添加到mediaTypesToUse，以供使用。
      for (MediaType requestedType : acceptableTypes) {
         for (MediaType producibleType : producibleTypes) {
            if (requestedType.isCompatibleWith(producibleType)) {
               mediaTypesToUse.add(getMostSpecificMediaType(requestedType, producibleType));
            }
         }
      }
      if (mediaTypesToUse.isEmpty()) {
         if (logger.isDebugEnabled()) {
            logger.debug("No match for " + acceptableTypes + ", supported: " + producibleTypes);
         }
         if (body != null) {
            throw new HttpMediaTypeNotAcceptableException(producibleTypes);
         }
         return;
      }

      // 排序，这对后面的判断有影响。
      MediaType.sortBySpecificityAndQuality(mediaTypesToUse);

      // 选择优先级高的符合要求的medieaType。
      for (MediaType mediaType : mediaTypesToUse) {
         // 具体的
         if (mediaType.isConcrete()) {
            selectedMediaType = mediaType;
            break;
         }
         // <type, subtype> in {<*, *>, <application, *>}中，则返回<application, octet-stream>
         else if (mediaType.isPresentIn(ALL_APPLICATION_MEDIA_TYPES)) {
            selectedMediaType = MediaType.APPLICATION_OCTET_STREAM;
            break;
         }
      }

      if (logger.isDebugEnabled()) {
         logger.debug("Using '" + selectedMediaType + "', given " +
               acceptableTypes + " and supported " + producibleTypes);
      }
   }

   if (selectedMediaType != null) {
      // 移除Media.parameters中的k-q。
      selectedMediaType = selectedMediaType.removeQualityValue();
      for (HttpMessageConverter<?> converter : this.messageConverters) {
         GenericHttpMessageConverter genericConverter = (converter instanceof GenericHttpMessageConverter ?
               (GenericHttpMessageConverter<?>) converter : null);
         // 判断给定的converter是否支持selectedMediaType，selectedMediaType要求的charset是否在json编码提供的范围内，以及是否根据<valueType, selectedMediaType>能拿到ObjectMapper————ObjectMapper是用来读取和写入JSON的，POJO或是Jackson的JsonNode风格，大概@RequestBody和@ResponseBody都是如此支持的。
         if (genericConverter != null ?
               ((GenericHttpMessageConverter) converter).canWrite(targetType, valueType, selectedMediaType) :
               converter.canWrite(valueType, selectedMediaType)) {
            // RequestBodyAdvice执行。
            body = getAdvice().beforeBodyWrite(body, returnType, selectedMediaType,
                  (Class<? extends HttpMessageConverter<?>>) converter.getClass(),
                  inputMessage, outputMessage);
            if (body != null) {
               Object theBody = body;
               LogFormatUtils.traceDebug(logger, traceOn ->
                     "Writing [" + LogFormatUtils.formatValue(theBody, !traceOn) + "]");
               addContentDispositionHeader(inputMessage, outputMessage);
               // genericConverter支持的MimeType传输数据。比如applicatio/json，将body对象json格式化输出给response的outputStream。
               if (genericConverter != null) {
                  genericConverter.write(body, targetType, selectedMediaType, outputMessage);
               }
               else {
                  ((HttpMessageConverter) converter).write(body, selectedMediaType, outputMessage);
               }
            }
            else {
               if (logger.isDebugEnabled()) {
                  logger.debug("Nothing to write: null body");
               }
            }
            // 进入了此if，即结束。
            return;
         }
      }
   }
   
   // selectedMediaType==null 或 messageConverters中没有支持value向reponse can-write的，就进行以下代码：来报错。（所以正常使用无需关心）
   if (body != null) {
      Set<MediaType> producibleMediaTypes =
            (Set<MediaType>) inputMessage.getServletRequest()
                  .getAttribute(HandlerMapping.PRODUCIBLE_MEDIA_TYPES_ATTRIBUTE);

      if (isContentTypePreset || !CollectionUtils.isEmpty(producibleMediaTypes)) {
         throw new HttpMessageNotWritableException(
               "No converter for [" + valueType + "] with preset Content-Type '" + contentType + "'");
      }
      throw new HttpMediaTypeNotAcceptableException(getSupportedMediaTypes(body.getClass()));
   }
}
```

### HttpEntityMethodProcessor

```java
// 返回类型是HttpEntity.class但不是RequestEntity.class
public boolean supportsReturnType(MethodParameter returnType) {
   return (HttpEntity.class.isAssignableFrom(returnType.getParameterType()) &&
         !RequestEntity.class.isAssignableFrom(returnType.getParameterType()));
}
```

## AsyncTaskMethodReturnValueHandler

```java
// 返回类型是WebAsyncTask.class
public boolean supportsReturnType(MethodParameter returnType) {
   return WebAsyncTask.class.isAssignableFrom(returnType.getParameterType());
}
```






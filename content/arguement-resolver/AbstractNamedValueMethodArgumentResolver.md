## 设计理念

​	依据method-parameter要求，从request获取数据并将其构建成其所需类型对象的过程就是本类设计的内容。桥接了request请求和invoke method的中间服务。

​	named-value提取了method-parameter本身的信息：要求从request获取意向数据的信息；激活method是否必须要本参数对象；method-parameter提供的默认意向数据。各个实现类根据自己的需求构建named-value，并且将以此向request中获取意向数据。

​	WebDataBinder用于将获取的意向数据转换成method-parameter所需类型的对象（注入基本数据类型，数组）。比如意向数据是基本类型，method-parameter所需类型使其集合或数组形式，那么就构建集合或数组的默认提供类型，然后添加意向数据进入其中。切记：Map.class 和 数组、Collection.class不太一样，注意使用Map.class作method parameter所需类型。

## 字段

```java
// 使用configurableBeanFactory的能力。它还是有很多能力的。
@Nullable
private final ConfigurableBeanFactory configurableBeanFactory;

// 不清楚其具体作用
@Nullable
private final BeanExpressionContext expressionContext;

// 缓存request method使用的resolver
private final Map<MethodParameter, NamedValueInfo> namedValueInfoCache = new ConcurrentHashMap<>(256);
```



## 模板方法

```java
// 此抽象类实现类resolver，用来解析request + request method parameter的入口。本方法放置了多个hook方法，以满足子类特质功能的需求。
// 此方法大致步骤（此类resolver的解析步骤）：准备named-value；解析named-value.name（${}-占位符替换方向，#{}-SpEL语法方向）；依据named-value + method-parameter从request中获取（构建参数对象）意向数据源；若前一步骤未获取意向数据，接受named-value提供的默认值作为意向数据；将意向数据构建成request-method-parameter所需类型的对象；处理解析后的数据（一般这里不该数据，而且仅就一个子类实现，用于了缓存数据）
// 进入此方法的都是method-parameter初步满足resolver解析支持的。
public final Object resolveArgument(
    // 描述request method的信息，一般按照其要求，从reqeust中获取解析数据。
    MethodParameter parameter, 
    // 待研究
    @Nullable ModelAndViewContainer mavContainer,
    // request
    NativeWebRequest webRequest, 
    // 解析从request或注解中获取的意向数据
    @Nullable WebDataBinderFactory binderFactory) throws Exception {

   // 构建<name, required-flag, default-value>数据结构。name：实现类意图从request获取数据的method-parameter特征（reqeust的attributes或parameters或headers中的key值）；required-flag：method-parameter是否能获取意向数据并转变成所需类型的对象；default-value：在无法从request获取意向数据的时候，由注解提供的意向数据。
   NamedValueInfo namedValueInfo = getNamedValueInfo(parameter);
   MethodParameter nestedParameter = parameter.nestedIfOptional();

   // 对named-value.name作占位符解析替换（${})和SpEL语法解析（#{})
   Object resolvedName = resolveEmbeddedValuesAndExpressions(namedValueInfo.name);
   if (resolvedName == null) {
      throw new IllegalArgumentException(
            "Specified name must not resolve to null: [" + namedValueInfo.name + "]");
   }

   // 由实现类从request中获取符合method-paramter的意向数据。子类实现。
   Object arg = resolveName(resolvedName.toString(), nestedParameter, webRequest);
   // 未能从request获取意向数据情况处理
   if (arg == null) {
      // 将named-value.defaultValue作为意向数据。
      if (namedValueInfo.defaultValue != null) {
         // 对named-value.name作placeholder解析替换（${})和SpEL语法解析（#{})
         arg = resolveEmbeddedValuesAndExpressions(namedValueInfo.defaultValue);
      }
      else if (namedValueInfo.required && !nestedParameter.isOptional()) {
         handleMissingValue(namedValueInfo.name, nestedParameter, webRequest);
      }
      // 若method parameter所需类型是布尔类型，且arg=null，自动提供false作为意向值。
      arg = handleNullValue(namedValueInfo.name, arg, nestedParameter.getNestedParameterType());
   }
   // 若arg!=null，且arg.equals("")，且提供默认值的。将named-value.default-value作占位符解析替换（${})和SpEL语法解析（#{})。
   else if ("".equals(arg) && namedValueInfo.defaultValue != null) {
      arg = resolveEmbeddedValuesAndExpressions(namedValueInfo.defaultValue);
   }

   if (binderFactory != null) {
      // WebDataBinder也要从reqeust获取数据吗，不然为何如此构建呢？
      // build对象ServletRequestDataBinderFactory。WebDataBinderFactory是RequestMappingHandlerAdapter.invokeHandlerMethod中构建。是对当前请求中的HandlerMethod进行包装、解析、构建。
      WebDataBinder binder = binderFactory.createBinder(webRequest, null, namedValueInfo.name);
      try {
         // 将目前解析得出的arg，做与parameter.getParameterType()相匹配的数据处理。比如arg="123,321"，parameter.getParameterType()=Ljava.lang.String，就数据转换成["123", "321"]。先放着，之后再看看，文本有几种解析方式，显然是依据request method parameter type来决定解析方式。
         // 将意向数据转换成method-parameter所需类型的对象。要学习一下。
         arg = binder.convertIfNecessary(arg, parameter.getParameterType(), parameter);
      }
      catch (ConversionNotSupportedException ex) {
         throw new MethodArgumentConversionNotSupportedException(arg, ex.getRequiredType(),
               namedValueInfo.name, parameter, ex.getCause());
      }
      catch (TypeMismatchException ex) {
         throw new MethodArgumentTypeMismatchException(arg, ex.getRequiredType(),
               namedValueInfo.name, parameter, ex.getCause());
      }
      // Check for null value after conversion of incoming argument value
      // 对于吊诡现象的处理。
      if (arg == null && namedValueInfo.defaultValue == null &&
            namedValueInfo.required && !nestedParameter.isOptional()) {
         handleMissingValueAfterConversion(namedValueInfo.name, nestedParameter, webRequest);
      }
   }

   // 获取resolved value后的处理内容。这个只有PathVariableMethodArgumentResolver子类实现了，其他都是本抽象类的空步骤。
   handleResolvedValue(arg, namedValueInfo.name, parameter, mavContainer, webRequest);

   return arg;
}
```

存在复用的代码块，独立成方法，以供多次调用：

```java
// 就注释而言，功能：解析注解指定的值，要求具备placeholders and expressions解析的功能。（前者是${}，后者SpEL）。先placeholders解析，后expressions解析。
// 本方法本着适用则解析应用，否则就原值返回————通常都应实现这种设计。
private Object resolveEmbeddedValuesAndExpressions(String value) {
   if (this.configurableBeanFactory == null || this.expressionContext == null) {
      return value;
   }
   // placeholders解析
   // 调用configurableBeanFactory内置的内嵌值resolver解析value————即从Environment中获取对应的K-V，进行替换${}。
   String placeholdersResolved = this.configurableBeanFactory.resolveEmbeddedValue(value);
   // expression解析————主要是SpEL解析
   // #{}包含的内容即SpEL解析内容，比如"sd #{1+1} 12"就解析成"sd 2 12"
   BeanExpressionResolver exprResolver = this.configurableBeanFactory.getBeanExpressionResolver();
   if (exprResolver == null) {
      return value;
   }
   // expression解析方法调用，也是最后结果返回的出口。
   return exprResolver.evaluate(placeholdersResolved, this.expressionContext);
}
```



## 待子类实现方法

- ```java
  // 标记①
  boolean supportsParameter(MethodParameter parameter);
  ```

- ```java
  // 标记②
  // 这个方法必须与updateNamedValueInfo结合起来看待，该方法在name.length=0 & defaultValue=ValueConstants.DEFAULT_NONE的情况下，修改成name=parameter.name & defaultValue=null。
  protected abstract NamedValueInfo createNamedValueInfo(MethodParameter parameter);
  ```

- ```java
  // 标记③
  protected abstract Object resolveName(String name, MethodParameter parameter, NativeWebRequest request)
        throws Exception;
  ```

- ```java
  // 标记④
  // 所有的实现都在抛异常，只不过，抛出的异常不同罢了
  protected void handleMissingValue(String name, MethodParameter parameter, NativeWebRequest request)
        throws Exception
  ```

- ```java
  // 标记⑤
  // 所有的实现都在抛异常，只不过，抛出的异常不同罢了
  protected void handleMissingValue(String name, MethodParameter parameter) throws ServletException
  ```

- ```java
  // 标记⑥
  // 所有的实现都在抛异常，只不过，抛出的异常不同罢了
  protected void handleMissingValueAfterConversion(String name, MethodParameter parameter, NativeWebRequest request)
  			throws Exception
  ```

  ```java
  // 标记⑦ PathVariableMethodArgumentResolver仅有实现需求
  protected void handleResolvedValue(@Nullable Object arg, String name, MethodParameter parameter,
        @Nullable ModelAndViewContainer mavContainer, NativeWebRequest webRequest)
  ```

  



## RequestHeaderMethodArgumentResolver

### ①

```java
// 受当前resolver支持，必须使用@RequestHeader，并且request method parameter不能是Map类型。在确定parameter接收单个的情况下用String，不确定的情况下用String[]。
public boolean supportsParameter(MethodParameter parameter) {
   return (parameter.hasParameterAnnotation(RequestHeader.class) &&
         !Map.class.isAssignableFrom(parameter.nestedIfOptional().getNestedParameterType()));
}
```

### ②

```java
// 最终得到：<ann.name/resolvedParameter.name, anno.required, anno.defaultValue>
protected NamedValueInfo createNamedValueInfo(MethodParameter parameter) {
   RequestHeader ann = parameter.getParameterAnnotation(RequestHeader.class);
   Assert.state(ann != null, "No RequestHeader annotation");
   return new RequestHeaderNamedValueInfo(ann);
}
```

### ③

```java
// 读取coyote.Request.headers中k=name的value，从 request.getHeaderValues(name)的底层编码方式来看，headers中可以存在同k的键值对。
protected Object resolveName(String name, MethodParameter parameter, NativeWebRequest request) throws Exception {
   String[] headerValues = request.getHeaderValues(name);
   if (headerValues != null) {
      return (headerValues.length == 1 ? headerValues[0] : headerValues);
   }
   else {
      return null;
   }
}
```

### ④

延用父类

### ⑤

```java
// 直接抛异常
protected void handleMissingValue(String name, MethodParameter parameter) throws ServletRequestBindingException {
   throw new MissingRequestHeaderException(name, parameter);
}
```

### ⑥

```java
// 直接抛异常
protected void handleMissingValueAfterConversion(
    String name, MethodParameter parameter, NativeWebRequest request) throws Exception {

    throw new MissingRequestHeaderException(name, parameter, true);
}
```

### ⑦

延用父类



## RequestAttributeMethodArgumentResolver

### ①

```java
// 受当前resolver支持，必须使用@RequestAttribute。
public boolean supportsParameter(MethodParameter parameter) {
   return parameter.hasParameterAnnotation(RequestAttribute.class);
}
```

### ②

```java
// 最终创建：<ann.name/resolvedParameter.name, ann.required, null>
protected NamedValueInfo createNamedValueInfo(MethodParameter parameter) {
   RequestAttribute ann = parameter.getParameterAnnotation(RequestAttribute.class);
   Assert.state(ann != null, "No RequestAttribute annotation");
   return new NamedValueInfo(ann.name(), ann.required(), ValueConstants.DEFAULT_NONE);
}
```

### ③

```java
// 从四种数据集合中获取，依照以下数字大小，依次尝试获取。想来一般从②中获取。
// ① connector.Request.specialAttributes - 静态Map字段，获取connector.Request.SpecialAttributeAdapter实现类对象。
// ② connector.Request.attributes中获取
// ③ coyote.Request.attributes中获取
// ④ 关于TSL配置信息中获取，指定了某些动作coyote.Request.attributes中新增了数据，并将指定指定的coyote.Request.attribute复用在connector.Request.attributes中，依据name在connector.Request.attributes再尝试获取一次。
// 这个不好直接给定啊。
protected Object resolveName(String name, MethodParameter parameter, NativeWebRequest request){
   return request.getAttribute(name, RequestAttributes.SCOPE_REQUEST);
}
```

### ⑤

```java
// 直接抛出异常
protected void handleMissingValue(String name, MethodParameter parameter) throws ServletException {
   throw new ServletRequestBindingException("Missing request attribute '" + name +
         "' of type " +  parameter.getNestedParameterType().getSimpleName());
}
```

### ⑥

延用父类

### ⑦

延用父类



## RequestParamMethodArgumentResolver

### ①

```java
// 实际中，本类实例成两个resolvers使用，区别在于this.useDefaultResolution布尔值的赋值。
// this.useDefaultResolution=false：使用@RequestParam + 非map parameter；使用@RequestParam("指定名字") + Map parameter（这个无法实现，springmvc内置屏蔽了这个————偷懒了，没有去看最初抛出异常的信息，发现是“无法解析，上传文件超过limit”！醉了！！）; MultipartFile.class或Part.class parameter
// this.useDefaultResolution=true：parameter是基本数据类型或其包装类，以及它们的一维数组；parameter是Enum.class、CharSequence.class、Number.class、Date.class、Temporal.class、URI.class、URL.class、Locale.class、Class.class类型。
// @RequestPart为什么在这里不支持呢，其有什么特性需要支持？之后需要学习。
public boolean supportsParameter(MethodParameter parameter) {
   if (parameter.hasParameterAnnotation(RequestParam.class)) {
      if (Map.class.isAssignableFrom(parameter.nestedIfOptional().getNestedParameterType())) {
         RequestParam requestParam = parameter.getParameterAnnotation(RequestParam.class);
         return (requestParam != null && StringUtils.hasText(requestParam.name()));
      }
      else {
         return true;
      }
   }
   else {
      if (parameter.hasParameterAnnotation(RequestPart.class)) {
         return false;
      }
      parameter = parameter.nestedIfOptional();
      if (MultipartResolutionDelegate.isMultipartArgument(parameter)) {
         return true;
      }
      else if (this.useDefaultResolution) {
         return BeanUtils.isSimpleProperty(parameter.getNestedParameterType());
      }
      else {
         return false;
      }
   }
}
```

### ②

```java
// 最终创建：
// 使用@RequstPram：<ann.name/resolved-parameter.name, ann.required, ann.defaultValue>
// 不使用~~：<resolved-parameter.name, false, null>
protected NamedValueInfo createNamedValueInfo(MethodParameter parameter) {
   RequestParam ann = parameter.getParameterAnnotation(RequestParam.class);
   return (ann != null ? new RequestParamNamedValueInfo(ann) : new RequestParamNamedValueInfo());
}
```

### ③

```java
// 目前就仅用当作用来解析非文件parameter的情况就行
protected Object resolveName(String name, MethodParameter parameter, NativeWebRequest request) throws Exception {
   HttpServletRequest servletRequest = request.getNativeRequest(HttpServletRequest.class);

   // 这个请求方式，我仍未找到，可能是postman没有提供。
   // 可以从request内嵌的request是HttpServletRequest.class实现类情况
   if (servletRequest != null) {
      // 方法内用来解析request method parameter是Part.class、MultipartFile.class的或者它们的数组或Collections容器类型的情况。若不是上述情况，则返回MultipartResolutionDelegate.UNRESOLVABLE。
      Object mpArg = MultipartResolutionDelegate.resolveMultipartArgument(name, parameter, servletRequest);
      if (mpArg != MultipartResolutionDelegate.UNRESOLVABLE) {
         return mpArg;
      }
   }

   Object arg = null;
   // 这里同上，query param中无法传输文件，body中传输文件的话，这个resolver又不受支持调用。
   MultipartRequest multipartRequest = request.getNativeRequest(MultipartRequest.class);
   if (multipartRequest != null) {
      List<MultipartFile> files = multipartRequest.getFiles(name);
      if (!files.isEmpty()) {
         arg = (files.size() == 1 ? files.get(0) : files);
      }
   }
   if (arg == null) {
      // 从coyote.request.parameters中找
      String[] paramValues = request.getParameterValues(name);
      if (paramValues != null) {
         arg = (paramValues.length == 1 ? paramValues[0] : paramValues);
      }
   }
   return arg;
}
```

### ④

```java
protected void handleMissingValue(String name, MethodParameter parameter, NativeWebRequest request)
      throws Exception {

   handleMissingValueInternal(name, parameter, request, false);
}
```

### ⑤

延用父类

### ⑥

```java
protected void handleMissingValueAfterConversion(
      String name, MethodParameter parameter, NativeWebRequest request) throws Exception {

   handleMissingValueInternal(name, parameter, request, true);
}
```

④、⑤、⑥共享使用

```java
// 都抛出异常，只不过，根据request和request method parameter是否是涉及文件的，以此来区别使用抛出异常。
protected void handleMissingValueInternal(
      String name, MethodParameter parameter, NativeWebRequest request, boolean missingAfterConversion)
      throws Exception {

   HttpServletRequest servletRequest = request.getNativeRequest(HttpServletRequest.class);
   if (MultipartResolutionDelegate.isMultipartArgument(parameter)) {
      if (servletRequest == null || !MultipartResolutionDelegate.isMultipartRequest(servletRequest)) {
         throw new MultipartException("Current request is not a multipart request");
      }
      else {
         throw new MissingServletRequestPartException(name);
      }
   }
   else {
      throw new MissingServletRequestParameterException(name,
            parameter.getNestedParameterType().getSimpleName(), missingAfterConversion);
   }
}
```

### ⑦

延用父类



## AbstractCookieValueMethodArgumentResolver

### ①

```java
// 只要确认参数是否使用注解@CookieValue。大概是本次resolver解析方向单一导致的。
public boolean supportsParameter(MethodParameter parameter) {
   return parameter.hasParameterAnnotation(CookieValue.class);
}
```

### ②

```java
// 最终形成：<ann.name/resolved-parameter.name, ann.required, ann.defaultValue>
protected NamedValueInfo createNamedValueInfo(MethodParameter parameter) {
   CookieValue annotation = parameter.getParameterAnnotation(CookieValue.class);
   Assert.state(annotation != null, "No CookieValue annotation");
   return new CookieValueNamedValueInfo(annotation);
}
```

### ③

待子类实现

### ④

延用父类

### ⑤

```java
// 抛异常
protected void handleMissingValue(String name, MethodParameter parameter) throws ServletRequestBindingException {
   throw new MissingRequestCookieException(name, parameter);
}
```

### ⑥

```java
// 抛异常
protected void handleMissingValueAfterConversion(
      String name, MethodParameter parameter, NativeWebRequest request) throws Exception {

   throw new MissingRequestCookieException(name, parameter, true);
}
```

### ⑦

延用父类



### ServletCookieValueMethodArgumentResolver

### ③

```java
// 要么headers中添加k-cookie(key不区分大小写) v-"key=value"。
// 要么在专门添加cookies的弹窗内添加，“Type a domain name”是域名的意思，本地填写localhost，不要写127.0.0.1。内容填写格式有要求——每块内容用";"相隔，每个位置有固定的内容填写。
protected Object resolveName(String cookieName, MethodParameter parameter,
      NativeWebRequest webRequest) throws Exception {

   HttpServletRequest servletRequest = webRequest.getNativeRequest(HttpServletRequest.class);
   Assert.state(servletRequest != null, "No HttpServletRequest");

   // 从connector.request.cookies中获取name=cookieName的Cookie对象。
   Cookie cookieValue = WebUtils.getCookie(servletRequest, cookieName);
   // 若request method parameter是Cookie.class类型，则直接返回cookieValue
   if (Cookie.class.isAssignableFrom(parameter.getNestedParameterType())) {
      return cookieValue;
   }
   else if (cookieValue != null) {
      // 貌似传输时可能存在编码，这里采用进行解码处理。因此不推荐直接使用Cookie.class作parameter进行获取cookie。
      return this.urlPathHelper.decodeRequestString(servletRequest, cookieValue.getValue());
   }
   else {
      return null;
   }
}
```

### 



## SessionAttributeMethodArgumentResolver

### ①

```java
// 同cookie的resolver一样，session的resolver同样解析方向单一。request method parameter使用注解@SessionAttribute
public boolean supportsParameter(MethodParameter parameter) {
   return parameter.hasParameterAnnotation(SessionAttribute.class);
}
```

### ②

```java
// 最终目标：<ann.name/resolved-parameter.name, ann.required, null>
protected NamedValueInfo createNamedValueInfo(MethodParameter parameter) {
   SessionAttribute ann = parameter.getParameterAnnotation(SessionAttribute.class);
   Assert.state(ann != null, "No SessionAttribute annotation");
   return new NamedValueInfo(ann.name(), ann.required(), ValueConstants.DEFAULT_NONE);
}
```

### ③

```java
// coyote.request.attribute
protected Object resolveName(String name, MethodParameter parameter, NativeWebRequest request) {
   return request.getAttribute(name, RequestAttributes.SCOPE_SESSION);
}
```

### ④

延用父类

### ⑤

```java
// 抛出异常，烦死了，拿不到session。这个实践先搁置吧。
protected void handleMissingValue(String name, MethodParameter parameter) throws ServletException {
   throw new ServletRequestBindingException("Missing session attribute '" + name +
         "' of type " + parameter.getNestedParameterType().getSimpleName());
}
```

### ⑥

延用父类



## MatrixVariableMethodArgumentResolver

关键字：

- 矩阵信息：矩阵变量 + 所关联的路径片段。
- 路径片段：request url中去除举证变量的“/”之间的文本
- 矩阵变量：路径片段后用“;”分隔的k-v键值对。
- 矩阵序列：单个路径片段 + 其所附着的举证变量。

### ①

```java
// 受此resolver支持，必须满足以下任意一种情况：
// ① @MatrixVariable("content") Map.class —— 貌似这个抽象类实现的子类都不支持Map.class，或者说历史版本支持，但是又开发了非继承此首次此抽象类的但支持相同共功能的Map.class用法的新的resolver。不要使用该策略。
// ② @MatrixVariable(可有可无) 非Map.class
public boolean supportsParameter(MethodParameter parameter) {
   if (!parameter.hasParameterAnnotation(MatrixVariable.class)) {
      return false;
   }
   if (Map.class.isAssignableFrom(parameter.nestedIfOptional().getNestedParameterType())) {
      MatrixVariable matrixVariable = parameter.getParameterAnnotation(MatrixVariable.class);
      return (matrixVariable != null && StringUtils.hasText(matrixVariable.name()));
   }
   return true;
}
```

### ②

```java
// 最终创建：<ann.name/resolved-parameter.name, ann.required, ann.defaultValue>
// @MartixVariable还有一个字段信息：pathVar，其代表指定的矩阵信息跟在那个路径片段后，换言之，索要那个矩阵序列的矩阵信息。
protected NamedValueInfo createNamedValueInfo(MethodParameter parameter) {
   MatrixVariable ann = parameter.getParameterAnnotation(MatrixVariable.class);
   Assert.state(ann != null, "No MatrixVariable annotation");
   return new MatrixVariableNamedValueInfo(ann);
}
```

### ③

```java
// spring mvc底层应该将request method的映射路径文本进行了解析，对{}包含的内容进行了解析，当然{}应该局限在/之间，一般映射路径应该包含/之间的整块内容：/con/{text}/sdf。
// 就@MartixVariable的使用意图，矩阵变量是附着在路径片段（片段指/之间的内容）k-v键值对集合，每个片段+矩阵变量集合 是矩阵信息的一个序列，多个序列就组成一个矩阵。
protected Object resolveName(String name, MethodParameter parameter, NativeWebRequest request) throws Exception {
   // 这个语句同RequestAttributeMethodArgumentResolver.resolveName的功能一致，具体可前往查看，这里不重复赘述。
   // MultiValueMap<String, String> 其实是Map<String, List<String>>
   Map<String, MultiValueMap<String, String>> pathParameters = (Map<String, MultiValueMap<String, String>>)
         request.getAttribute(HandlerMapping.MATRIX_VARIABLES_ATTRIBUTE, RequestAttributes.SCOPE_REQUEST);
   if (CollectionUtils.isEmpty(pathParameters)) {
      return null;
   }

   MatrixVariable ann = parameter.getParameterAnnotation(MatrixVariable.class);
   Assert.state(ann != null, "No MatrixVariable annotation");
   String pathVar = ann.pathVar();
   List<String> paramValues = null;

   // 若pathVar指定，那就方便了，<pathVar, name>确定了矩阵序列的key对应的value。
   if (!pathVar.equals(ValueConstants.DEFAULT_NONE)) {
      if (pathParameters.containsKey(pathVar)) {
         paramValues = pathParameters.get(pathVar).get(name);
      }
   }
   else {
      boolean found = false;
      paramValues = new ArrayList<>();
      // 若未指定路径序列（pathVar未给定），则遍历矩阵序列求得。若从不同的矩阵序列中找到相同的key，则抛出异常。因此不推荐这种。
      for (MultiValueMap<String, String> params : pathParameters.values()) {
         if (params.containsKey(name)) {
            if (found) {
               String paramType = parameter.getNestedParameterType().getName();
               throw new ServletRequestBindingException(
                     "Found more than one match for URI path parameter '" + name +
                     "' for parameter type [" + paramType + "]. Use 'pathVar' attribute to disambiguate.");
            }
            paramValues.addAll(params.get(name));
            found = true;
         }
      }
   }

   if (CollectionUtils.isEmpty(paramValues)) {
      return null;
   }
   else if (paramValues.size() == 1) {
      return paramValues.get(0);
   }
   else {
      return paramValues;
   }
}
```

### ④

延用父类

### ⑤

```java
// 抛出异常
protected void handleMissingValue(String name, MethodParameter parameter) throws ServletRequestBindingException {
   throw new MissingMatrixVariableException(name, parameter);
}
```

### ⑥

```java
// 抛出异常
protected void handleMissingValueAfterConversion(
      String name, MethodParameter parameter, NativeWebRequest request) throws Exception {

   throw new MissingMatrixVariableException(name, parameter, true);
}
```

### 注意

参考：[SpringBoot @MatrixVariable注解 矩阵变量](https://blog.csdn.net/qq_40646143/article/details/115240461)

springMVC默认不开启矩阵解析功能，需要自行修改配置。

```java
@Configuration
public class MyWebMvcConfigurer implements WebMvcConfigurer {

    // 参考：https://blog.csdn.net/qq_40646143/article/details/115240461
    @Override
    public void configurePathMatch(PathMatchConfigurer configurer) {
        UrlPathHelper urlPathHelper = new UrlPathHelper();
        //不移除举证变量后面的内容
        urlPathHelper.setRemoveSemicolonContent(false);
        urlPathHelper.setAlwaysUseFullPath(true);
        configurer.setUrlPathHelper(urlPathHelper);
    }

//    @Bean // WebConfigurer
//    public WebMvcConfigurer webMvcConfigurer() {
//        return new WebMvcConfigurer() {
//            @Override
//            public void configurePathMatch(PathMatchConfigurer configurer) {
//                UrlPathHelper urlPathHelper = new UrlPathHelper();
//                urlPathHelper.setRemoveSemicolonContent(false);
//                configurer.setUrlPathHelper(urlPathHelper);
//            }
//        };
//    }
}
```



## ExpressionValueMethodArgumentResolver

### ①

```java
// 受此resolver支持，需用注解@Value。
public boolean supportsParameter(MethodParameter parameter) {
   return parameter.hasParameterAnnotation(Value.class);
}
```

### ②

```java
// 最终创建：<@Value, false, ann.value>
protected NamedValueInfo createNamedValueInfo(MethodParameter parameter) {
   PathVariable ann = parameter.getParameterAnnotation(PathVariable.class);
   Assert.state(ann != null, "No PathVariable annotation");
   return new PathVariableNamedValueInfo(ann);
}
```

### ③

```java
// 不从request中获取数据，@Value的意图是用Envirionment等系统资源或SpEL语法解析文本。这个方法在抽象父类的resolveEmbeddedValuesAndExpressions中实现类。
protected Object resolveName(String name, MethodParameter parameter, NativeWebRequest webRequest) throws Exception {
   // No name to resolve
   return null;
}
```

### ④

延用父类

### ⑤

```java
// 抛出异常
protected void handleMissingValue(String name, MethodParameter parameter) throws ServletException {
   throw new UnsupportedOperationException("@Value is never required: " + parameter.getMethod());
}
```

### ⑥

延用父类



## PathVariableMethodArgumentResolver

### ①

```java
// 受当前resolver支持，必须使用@PathVariable，并且给定有意义的path-value。
// 没有现成的PropertyEditor实现类支持String转Map的————TypeConverterDelegate.convertIfNecessary(...)调用中差不多就是这样子的。因此不要这样用：@PathVariable("name") Map<String, String> pathVariables。这样子写既不雅观，也没有效用价值。
public boolean supportsParameter(MethodParameter parameter) {
   if (!parameter.hasParameterAnnotation(PathVariable.class)) {
      return false;
   }
   if (Map.class.isAssignableFrom(parameter.nestedIfOptional().getNestedParameterType())) {
      PathVariable pathVariable = parameter.getParameterAnnotation(PathVariable.class);
      return (pathVariable != null && StringUtils.hasText(pathVariable.value()));
   }
   return true;
}
```

### ②

```java
// 最终结果：<ann.name/resolvedParameter.name, ann.required, null>
protected NamedValueInfo createNamedValueInfo(MethodParameter parameter) {
   PathVariable ann = parameter.getParameterAnnotation(PathVariable.class);
   Assert.state(ann != null, "No PathVariable annotation");
   return new PathVariableNamedValueInfo(ann);
}
```

### ③

```java
// 从connector.Request.attributes中读取k-HandlerMapping.URI_TEMPLATE_VARIABLES_ATTRIBUTE的v值。v值是requestMapping.name中{}起来的内容 - request-url对应的片段的k-v键值对Map。
protected Object resolveName(String name, MethodParameter parameter, NativeWebRequest request) throws Exception {
   Map<String, String> uriTemplateVars = (Map<String, String>) request.getAttribute(
         HandlerMapping.URI_TEMPLATE_VARIABLES_ATTRIBUTE, RequestAttributes.SCOPE_REQUEST);
   return (uriTemplateVars != null ? uriTemplateVars.get(name) : null);
}
```

### ④

```java
// 直接抛异常
protected void handleMissingValueAfterConversion(
      String name, MethodParameter parameter, NativeWebRequest request) throws Exception {

   throw new MissingPathVariableException(name, parameter, true);
}
```

### ⑤

```java
// 直接抛异常
protected void handleMissingValue(String name, MethodParameter parameter) throws ServletRequestBindingException {
   throw new MissingPathVariableException(name, parameter);
}
```

### ⑥

```java
// 向connector.Request.attributes写入View.PATH_VARIABLES - HashMap的容器，缓存name-arg键值对。不知道用在哪里了？
protected void handleResolvedValue(
    @Nullable Object arg, String name, MethodParameter parameter,
    @Nullable ModelAndViewContainer mavContainer, 
    NativeWebRequest request) {

   String key = View.PATH_VARIABLES;
   int scope = RequestAttributes.SCOPE_REQUEST;
   Map<String, Object> pathVars = (Map<String, Object>) request.getAttribute(key, scope);
   if (pathVars == null) {
      pathVars = new HashMap<>();
      request.setAttribute(key, pathVars, scope);
   }
   pathVars.put(name, arg);
}
```



## 意向数据转变成method-parameter所需类型对象

```java
// WebDataBinderFactory来源：RequestMappingHandlerAdapter.invokeHandlerMethod。
// 适配器，DispatherServlet使用HandlerMethod的适配器————也可以理解为中间应用场景。DispatcherServlet提供response+request+handlerMethod给RequestMappingHandlerAdapter，让其invoke handlerMethod代表的method。
// handlerMethod中处理了 方法参数解析+方法调用+返回值处理（mav方面）。
protected ModelAndView invokeHandlerMethod(HttpServletRequest request,
      HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {

   ServletWebRequest webRequest = new ServletWebRequest(request, response);
   ......
   WebDataBinderFactory binderFactory = getDataBinderFactory(handlerMethod);
   ......
   // handlerMethod是HandlerMethod对象，这里用ServletInvocableHandlerMethod将handlerMethod的属性全部引用复制过来。HandlerMethod->InvocableHandlerMethod->ServletInvocableHandlerMethod是实现链路上的。
   ServletInvocableHandlerMethod invocableMethod = createInvocableHandlerMethod(handlerMethod);
   ......
    
   invocableMethod.setDataBinderFactory(binderFactory);
   ......
   invocableMethod.invokeAndHandle(webRequest, mavContainer);
   ......
}
```



```java
// WebDataBinder也要从reqeust获取数据吗，不然为何如此构建呢？
// build对象ServletRequestDataBinderFactory。WebDataBinderFactory是RequestMappingHandlerAdapter.invokeHandlerMethod中构建。是对当前请求中的HandlerMethod进行包装、解析、构建。
WebDataBinder binder = binderFactory.createBinder(webRequest, null, namedValueInfo.name);
try {
    // 将目前解析得出的arg，做与parameter.getParameterType()相匹配的数据处理。比如arg="123,321"，parameter.getParameterType()=Ljava.lang.String，就数据转换成["123", "321"]。先放着，之后再看看，文本有几种解析方式，显然是依据request method parameter type来决定解析方式。
    // 将意向数据转换成method-parameter所需类型的对象。要学习一下。
    arg = binder.convertIfNecessary(arg, parameter.getParameterType(), parameter);
}
```


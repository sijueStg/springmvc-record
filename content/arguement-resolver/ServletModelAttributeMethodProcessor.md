## 简介

​	ServletModelAttributeMethodProcessor存在基类ModelAttributeMethodProcessor，因此需要先后介绍两个类，后者先介绍。

​	ServletModelAttributeMethodProcessor被用于参数和返回值解析，目前本篇先根据参数解析分析，但得给返回值解析的文本提供层级空间。

​	ServletModelAttributeMethodProcessor参数解析是针对@ModelAttribute参数使用，和保底作用——解析未使用参数的且自定义类参数的情况。



## ModelAttributeMethodProcessor

### 内部类FieldAwareConstructorParameter

​	这个类的存在，就是为了覆盖父类MethodParameter的getParameterAnnotations方法。

```java
// 这个重复方法，是为了将构造器参数和对标的类内参数的注解进行合并，以Set<Annotation>.toArray的形式存储。因此，应当让构造参数代表的nameValue.name与对标的field保持一致。
public Annotation[] getParameterAnnotations() {
   Annotation[] anns = this.combinedAnnotations;
   if (anns == null) {
      // 构造器参数标记的注解。
      anns = super.getParameterAnnotations();
      try {
         // 若构造参数提供的nameValue.name与类内字段是同一个，则将其注解提取。否则抛出异常。不过这个异常将被忽视。
         Field field = getDeclaringClass().getDeclaredField(this.parameterName);
         Annotation[] fieldAnns = field.getAnnotations();
         if (fieldAnns.length > 0) {
            List<Annotation> merged = new ArrayList<>(anns.length + fieldAnns.length);
            merged.addAll(Arrays.asList(anns));
            for (Annotation fieldAnn : fieldAnns) {
               boolean existingType = false;
               for (Annotation ann : anns) {
                  if (ann.annotationType() == fieldAnn.annotationType()) {
                     existingType = true;
                     break;
                  }
               }
               if (!existingType) {
                  merged.add(fieldAnn);
               }
            }
            anns = merged.toArray(new Annotation[0]);
         }
      }
      catch (NoSuchFieldException | SecurityException ex) {
         // ignore
      }
      this.combinedAnnotations = anns;
   }
   return anns;
}
```

### 方法

#### supportsParameter

```java
// 子类ServletModelAttributeMethodProcessor是作为两个resolver对象在使用，
// ① 解析参数带有注解ModelAttribute的情况。
// ② 作为保底resolver，一般解析无特殊注解使用且参数类型是自定义的。
// this.annotationNotRequired的布尔值用来屏蔽两种resolver的使用。
public boolean supportsParameter(MethodParameter parameter) {
   return (parameter.hasParameterAnnotation(ModelAttribute.class) ||
         (this.annotationNotRequired && !BeanUtils.isSimpleProperty(parameter.getParameterType())));
}
```

#### resolveArgument

```java
// parameter代表的类，不能用List等容器包含自定义类，不然会报错的。
public final Object resolveArgument(MethodParameter parameter, @Nullable ModelAndViewContainer mavContainer,
      NativeWebRequest webRequest, @Nullable WebDataBinderFactory binderFactory) throws Exception {

   Assert.state(mavContainer != null, "ModelAttributeMethodProcessor requires ModelAndViewContainer");
   Assert.state(binderFactory != null, "ModelAttributeMethodProcessor requires WebDataBinderFactory");

   // ModelFactory解析name的方式：注解提供name；注解未提供name或没有注解，则解析类型，一般就是驼峰格式的类名
   String name = ModelFactory.getNameForParameter(parameter);
   ModelAttribute ann = parameter.getParameterAnnotation(ModelAttribute.class);
   if (ann != null) {
      // 解析@ModelAttribute提供的绑定属性————对container.noBinding的集合进行添加或移除。
      mavContainer.setBinding(name, ann.binding());
   }

   Object attribute = null;
   // 这东西与WebDataBinder有关。
   BindingResult bindingResult = null;

   // 绑定有属性，则将该属性拿出来。
   if (mavContainer.containsAttribute(name)) {
      attribute = mavContainer.getModel().get(name);
   }
   else {
      try {
         // 选取parameter的代表类的某个构造函数，解析其参数，从request提取数据进行赋值————内容其实request.parameter或request.文件区。需要注意的是，这种不想json，只能解析一层的，不要瞎嵌套。
         attribute = createAttribute(name, parameter, binderFactory, webRequest);
      }
      catch (BindException ex) {
         if (isBindExceptionRequired(parameter)) {
            // No BindingResult parameter -> fail with BindException
            throw ex;
         }
         // Otherwise, expose null/empty value and associated BindingResult
         if (parameter.getParameterType() == Optional.class) {
            attribute = Optional.empty();
         }
         else {
            attribute = ex.getTarget();
         }
         bindingResult = ex.getBindingResult();
      }
   }

   // 上述操作中，只有attribute创建过程中出现绑定异常的时候，bindingResult才会被给予实例提供。当然，抛出其他异常的时候，这一步也就不用进行了。
   // 总归来说，这个if语句块内执行在正常attribute获取的场景下。
   if (bindingResult == null) {
      // Bean property binding and validation;
      // skipped in case of binding failure on construction.
      // 好家伙，好学习提供了target的WebDataBinder使用了。
      // 这么说，<targetObject, targetName>提供的时候是在基于绑定信息？提供什么绑定信息呢？
      WebDataBinder binder = binderFactory.createBinder(webRequest, attribute, name);
      if (binder.getTarget() != null) {
         if (!mavContainer.isBindingDisabled(name)) {
            // 这个接口是提供给子类的hook。待学。这里还会绑定数据，要学习的。
            // 这个其实借用BeanWrapperImpl，解析target。先用MutablePropertyValues收集request的数据，然后遍历其中每个PropertyValue，找target的对标setter，执行数据绑定。具体的就不分析。当然获取的数据在注入target对标字段时，还是会调用convertIfNecessary方法，将数据转换成目标数据对象
            bindRequestParameters(binder, webRequest);
         }
         // 这一块比较困惑，是验证注解的使用？
         // javax的@Valid、Spring的@Validated、自定义注解-以Valid开头命名。如何进行验证呢？
         // spring封装了hibernate的validator，不做具体的了解，但是我需要知道它会检测什么注解。
         validateIfApplicable(binder, parameter);
         if (binder.getBindingResult().hasErrors() && isBindExceptionRequired(binder, parameter)) {
            throw new BindException(binder.getBindingResult());
         }
      }
      // Value type adaptation, also covering java.util.Optional
      // 对于自定义来说，不能用List等容器，会出现异常报错。就当作这里是因为Optional的包装吧。。。。。。
      if (!parameter.getParameterType().isInstance(attribute)) {
         attribute = binder.convertIfNecessary(binder.getTarget(), parameter.getParameterType(), parameter);
      }
      bindingResult = binder.getBindingResult();
   }

   // Add resolved attribute and BindingResult at the end of the model
   // 目前看起来就只有：<parameter.nameValue.name, parameter解析出来的对象>、<org.springframework.validation.BindingResult.hello, BeanPropertyBindingResult————表示对Hello的数据填充/绑定效果>，具体不讨论了。
   Map<String, Object> bindingResultModel = bindingResult.getModel();
   mavContainer.removeAttributes(bindingResultModel);
   mavContainer.addAllAttributes(bindingResultModel);

   return attribute;
}
```

#### createAttribute

```java
// 这个将被子类覆盖。只有子类不能通过自己的形式获取attribute对象值时，才会调用父类方法——本方法。
protected Object createAttribute(String attributeName, MethodParameter parameter,
      WebDataBinderFactory binderFactory, NativeWebRequest webRequest) throws Exception {

   MethodParameter nestedParameter = parameter.nestedIfOptional();
   Class<?> clazz = nestedParameter.getNestedParameterType();

   // 存在kotlin获取构造函数，这里不考虑。
   /*
   	* 构造函数分为public和非public。当仅存在一个public构造函数的情况下，则返回它；若没有public构造函数且只有一个非public的构造函数，则返回该构造函数；若是其他则返回无参构造函数（不区分public等修饰符）。因此时刻为对象提供无参构造函数是非常好的，否则就得用适配类进行包裹了。
   	*/
   Constructor<?> ctor = BeanUtils.getResolvableConstructor(clazz);
   Object attribute = constructAttribute(ctor, attributeName, parameter, binderFactory, webRequest);
   if (parameter != nestedParameter) {
      attribute = Optional.of(attribute);
   }
   return attribute;
}
```

#### constructAttribute

```java
protected Object constructAttribute(Constructor<?> ctor, String attributeName, MethodParameter parameter,
      WebDataBinderFactory binderFactory, NativeWebRequest webRequest) throws Exception {

   // 推荐优良品种（选择）
   if (ctor.getParameterCount() == 0) {
      return BeanUtils.instantiateClass(ctor);
   }
   
   // 这里其实就是构造器参数列表 → 代表字符串数组。参数转换成代表的字符串。@ConstructorProperties提供了预设的转换结果，直接、清楚。若没有该注解支持，则提取构造函数的参数名。
   String[] paramNames = BeanUtils.getParameterNames(ctor);
   Class<?>[] paramTypes = ctor.getParameterTypes();
   Object[] args = new Object[paramTypes.length];
   WebDataBinder binder = binderFactory.createBinder(webRequest, null, attributeName);
   // 这两个字段WebDataBinder的两个常量字段提取出来了——"_"、"!"
   String fieldDefaultPrefix = binder.getFieldDefaultPrefix();
   String fieldMarkerPrefix = binder.getFieldMarkerPrefix();
   boolean bindingFailure = false;
   Set<String> failedParams = new HashSet<>(4);

   for (int i = 0; i < paramNames.length; i++) {
      String paramName = paramNames[i];
      // 从这里看，本resolver解析methodParameter，是将其目标构造函数的参数列表为依据，解析出nameValue.name集合，从request的parameterValues中获取相关数据。
      Class<?> paramType = paramTypes[i];
      Object value = webRequest.getParameterValues(paramName);
 	
      // 若提取出来的数据是单元素数组，则提取出来重新赋值。
      if (ObjectUtils.isArray(value) && Array.getLength(value) == 1) {
         value = Array.get(value, 0);
      }

      if (value == null) {
         if (fieldDefaultPrefix != null) {
            value = webRequest.getParameter(fieldDefaultPrefix + paramName);
         }
         if (value == null) {
            // 为什么拿到了却不用呢？奇怪。
            if (fieldMarkerPrefix != null && webRequest.getParameter(fieldMarkerPrefix + paramName) != null) {
               // 依据paramType是布尔类型、数组类型和其他分别获取不同空值：布尔类型-false；数组类型-0空间数组；其他：null。
               value = binder.getEmptyValue(paramType);
            }
            else {
               // 没有，说明可能是不是文本参数，而是文件参数，这个方法中进行提取尝试。
               value = resolveConstructorArgument(paramName, paramType, webRequest);
            }
         }
      }

      try {
         // FieldAwareConstructorParameter对MethodParameter获取Parameter的注解的方法进行了覆盖，将与paramName同名的field上的注解给合并在了一起。
         MethodParameter methodParam = new FieldAwareConstructorParameter(ctor, i, paramName);
         if (value == null && methodParam.isOptional()) {
            args[i] = (methodParam.getParameterType() == Optional.class ? Optional.empty() : null);
         }
         else {
            // ConversionService使用的converter.convert的使用参数就包含了 将methodParam解析为TypeDescriptor对象的情况。反而是PropertyEditor则没有那么丰富的数据使用。需要依据上下文的，看来得用ConversionService。
            args[i] = binder.convertIfNecessary(value, paramType, methodParam);
         }
      }
      catch (TypeMismatchException ex) {
         // 记录绑定失败的信息，唉，得学BindingResult。
         ex.initPropertyName(paramName);
         args[i] = null;
         failedParams.add(paramName);
         binder.getBindingResult().recordFieldValue(paramName, paramType, value);
         binder.getBindingErrorProcessor().processPropertyAccessException(ex, binder.getBindingResult());
         bindingFailure = true;
      }
   }

   // 处理出现绑定失败的情况。等以后再学吧。
   if (bindingFailure) {
      BindingResult result = binder.getBindingResult();
      for (int i = 0; i < paramNames.length; i++) {
         String paramName = paramNames[i];
         if (!failedParams.contains(paramName)) {
            Object value = args[i];
            result.recordFieldValue(paramName, paramTypes[i], value);
            validateValueIfApplicable(binder, parameter, ctor.getDeclaringClass(), paramName, value);
         }
      }
      if (!parameter.isOptional()) {
         try {
            Object target = BeanUtils.instantiateClass(ctor, args);
            throw new BindException(result) {
               @Override
               public Object getTarget() {
                  return target;
               }
            };
         }
         catch (BeanInstantiationException ex) {
            // swallow and proceed without target instance
         }
      }
      // 最后倒是要抛出异常的。。。
      throw new BindException(result);
   }

   // 好了，获取到参数args了，进行实例化。
   return BeanUtils.instantiateClass(ctor, args);
}
```

#### resolveConstructorArgument

```java
// 这个会被子类覆盖，先解析看看。
// 这里就是提取paramName作为nameValue.name向request的文件内容提取————MultipartFile或Part。
public Object resolveConstructorArgument(String paramName, Class<?> paramType, NativeWebRequest request)
      throws Exception {

   MultipartRequest multipartRequest = request.getNativeRequest(MultipartRequest.class);
   if (multipartRequest != null) {
      List<MultipartFile> files = multipartRequest.getFiles(paramName);
      if (!files.isEmpty()) {
         return (files.size() == 1 ? files.get(0) : files);
      }
   }
   else if (StringUtils.startsWithIgnoreCase(
         request.getHeader(HttpHeaders.CONTENT_TYPE), MediaType.MULTIPART_FORM_DATA_VALUE)) {
      HttpServletRequest servletRequest = request.getNativeRequest(HttpServletRequest.class);
      if (servletRequest != null && HttpMethod.POST.matches(servletRequest.getMethod())) {
         List<Part> parts = StandardServletPartUtils.getParts(servletRequest, paramName);
         if (!parts.isEmpty()) {
            return (parts.size() == 1 ? parts.get(0) : parts);
         }
      }
   }
   return null;
}
```



## ServletModelAttributeMethodProcessor

感觉没有加什么特别的内容。。。。。。

### 方法

#### createAttribute

```java
// 感觉不同于父类，父类中用到了binder.getFieldDefaultPrefix()、binder.getFieldMarkerPrefix()对attributeName进行包装获取，这里是直接的。
protected final Object createAttribute(String attributeName, MethodParameter parameter,
      WebDataBinderFactory binderFactory, NativeWebRequest request) throws Exception {

   String value = getRequestValueForAttribute(attributeName, request);
   if (value != null) {
      Object attribute = createAttributeFromRequestValue(
            value, attributeName, parameter, binderFactory, request);
      if (attribute != null) {
         return attribute;
      }
   }

   // 上述没有获取到什么，或者获得的无法转换，调用父类的方法。
   return super.createAttribute(attributeName, parameter, binderFactory, request);
}
```

#### resolveConstructorArgument

```java
// 给父类获取的方式添加了一种。
public Object resolveConstructorArgument(String paramName, Class<?> paramType, NativeWebRequest request)
      throws Exception {

   Object value = super.resolveConstructorArgument(paramName, paramType, request);
   if (value != null) {
      return value;
   }
   ServletRequest servletRequest = request.getNativeRequest(ServletRequest.class);
   // 不是获取文件类型构造器参数。那还能从哪里获取呢？Spring提供的特殊的parameter：k - HandlerMapping.URI_TEMPLATE_VARIABLES_ATTRIBUTE。
   if (servletRequest != null) {
      String attr = HandlerMapping.URI_TEMPLATE_VARIABLES_ATTRIBUTE;
      @SuppressWarnings("unchecked")
      Map<String, String> uriVars = (Map<String, String>) servletRequest.getAttribute(attr);
      return uriVars.get(paramName);
   }
   return null;
}
```

#### getRequestValueForAttribute

```java
// 
protected String getRequestValueForAttribute(String attributeName, NativeWebRequest request) {
   Map<String, String> variables = getUriTemplateVariables(request);
   String variableValue = variables.get(attributeName);
   if (StringUtils.hasText(variableValue)) {
      return variableValue;
   }
   String parameterValue = request.getParameter(attributeName);
   if (StringUtils.hasText(parameterValue)) {
      return parameterValue;
   }
   return null;
}
```



#### getUriTemplateVariables

```java
// spring提供指定位置属性k - HandlerMapping.URI_TEMPLATE_VARIABLES_ATTRIBUTE。
protected final Map<String, String> getUriTemplateVariables(NativeWebRequest request) {
   @SuppressWarnings("unchecked")
   Map<String, String> variables = (Map<String, String>) request.getAttribute(
         HandlerMapping.URI_TEMPLATE_VARIABLES_ATTRIBUTE, RequestAttributes.SCOPE_REQUEST);
   return (variables != null ? variables : Collections.emptyMap);
}
```



#### 



## 处理验证

接收参数处理验证注解的也只有两个参数解析类了：ModelAttributeMethodProcessor、AbstractMessageConverterMethodArgumentResolver（实现类：RequestPartMethodArgumentResolver、RequestResponseBodyMethodProcessor）毕竟它们可以实现自定义对象的解析。

后续总结的时候记得加入以下两篇文章关注内容：

[@Validated和@Valid区别：Spring validation验证框架对入参实体进行嵌套验证必须在相应属性（字段）加上@Valid而不是@Validated](https://blog.csdn.net/qq_27680317/article/details/79970590)

[SpingBoot项目使用@Validated和@Valid参数校验](https://juejin.cn/post/7213634349233111100#heading-4)

​	从ModelAttributeMethodProcessor.resolveArgument的语句validateIfApplicable(binder, parameter)执行中分析。

```java
protected void validateIfApplicable(WebDataBinder binder, MethodParameter parameter) {
   for (Annotation ann : parameter.getParameterAnnotations()) {
      // 分组功能，这个是用来支持验证注解的使用场景。
      Object[] validationHints = ValidationAnnotationUtils.determineValidationHints(ann);
      if (validationHints != null) {
         binder.validate(validationHints);
         break;
      }
   }
}
```

DataBinder.validate(.)

```java
public void validate(Object... validationHints) {
   Object target = getTarget();
   Assert.state(target != null, "No target to validate");
   BindingResult bindingResult = getBindingResult();
   // Call each validator with the same binding result
   // Validators是WebDataBinderFactory在创建时，由一个初始化类提供的。
   for (Validator validator : getValidators()) {
      if (!ObjectUtils.isEmpty(validationHints) && validator instanceof SmartValidator) {
         ((SmartValidator) validator).validate(target, bindingResult, validationHints);
      }
      else if (validator != null) {
         validator.validate(target, bindingResult);
      }
   }
}
```

ValidatorAdapter.validate(..)

```java
public void validate(Object target, Errors errors) {
   this.target.validate(target, errors);
}
```

SpringValidatorAdapter.validate(..)

```java
public void validate(Object target, Errors errors) {
   if (this.targetValidator != null) {
      // processConstraintViolations处理约束异常；targetValidator处理约束检查。其中后者，Spring交由hibernate的ValidatorImpl类检查约束。
      processConstraintViolations(this.targetValidator.validate(target), errors);
   }
}
```

```java
public final <T> Set<ConstraintViolation<T>> validate(T object, Class<?>... groups) {
   Contracts.assertNotNull( object, MESSAGES.validatedObjectMustNotBeNull() );
   sanityCheckGroups( groups );

   @SuppressWarnings("unchecked")
   Class<T> rootBeanClass = (Class<T>) object.getClass();
   // 提供object的验证注解信息。
   BeanMetaData<T> rootBeanMetaData = beanMetaDataManager.getBeanMetaData( rootBeanClass );

   if ( !rootBeanMetaData.hasConstraints() ) {
      return Collections.emptySet();
   }

   // 构建解析环境，提供解析目标对象object。rootBeanClass是object的Class。rootBeanMetaData是hibernate提供的用来描述rootBeanClass的验证注解的对象。
   // OK，到此为止，后续的先不看了，意义不大。
   BaseBeanValidationContext<T> validationContext = getValidationContextBuilder().forValidate( rootBeanClass, rootBeanMetaData, object );

   ValidationOrder validationOrder = determineGroupValidationOrder( groups );
   BeanValueContext<?, Object> valueContext = ValueContexts.getLocalExecutionContextForBean(
         validatorScopedContext.getParameterNameProvider(),
         object,
         validationContext.getRootBeanMetaData(),
         PathImpl.createRootPath()
   );

   return validateInContext( validationContext, valueContext, validationOrder );
}
```



failingConstraintViolations




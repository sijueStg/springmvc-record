## WebDataBinderFactory的构建

​	WebDataBinderFactory初步看下来，其构建主要提供两样东西：初步适配的@InitBinder标记的方法的包装类InvocableHandlerMethod、RequestMappingHandlerAdapter.webBindingInitializer（用于给全部将要构建的WebDataBinder提供数据）。

```java
// 创建WebDataBinderFactory instance，选择命中的@InitBinder标记方法进行封装创建。
// this.initBinderCache 和 this.initBinderAdviceCache，前者是局部的，后者是全局的。前者是从@Controller标记的类中提取@InitBinder标记的方法，这些方法也只能作用于该类中的@RequestMapping标记的方法。后者是@ControllerAdvice提供的，能作用域全局或指定范围内的包内的类。
private WebDataBinderFactory getDataBinderFactory(HandlerMethod handlerMethod) throws Exception {
   Class<?> handlerType = handlerMethod.getBeanType();
   Set<Method> methods = this.initBinderCache.get(handlerType);
   if (methods == null) {
      // 给handlerType做解析，并提供缓存服务。局部的。
      methods = MethodIntrospector.selectMethods(handlerType, INIT_BINDER_METHODS);
      this.initBinderCache.put(handlerType, methods);
   }
   List<InvocableHandlerMethod> initBinderMethods = new ArrayList<>();
   // RequestMappingHandlerAdapter.initBinderAdviceCache - Map<ControllerAdviceBean, Set<Method>>映射，选择命中handler-method的@InitBinder标记的方法。
   this.initBinderAdviceCache.forEach((controllerAdviceBean, methodSet) -> {
      // 判断：handlerType是否在本@ControllerAdvice标记类作用范围内。
      if (controllerAdviceBean.isApplicableToBeanType(handlerType)) {
         Object bean = controllerAdviceBean.resolveBean();
         for (Method method : methodSet) {
            initBinderMethods.add(createInitBinderMethod(bean, method));
         }
      }
   });
   for (Method method : methods) {
      Object bean = handlerMethod.getBean();
      initBinderMethods.add(createInitBinderMethod(bean, method));
   }
   // 创建WebDataBinderFactory的实现类实例。
   // initBinderMethods将作为WebDataBinderFactory实例化时的数据。
   return createDataBinderFactory(initBinderMethods);
}
```

```java
private InvocableHandlerMethod createInitBinderMethod(Object bean, Method method) {
   InvocableHandlerMethod binderMethod = new InvocableHandlerMethod(bean, method);
   if (this.initBinderArgumentResolvers != null) {
   /* 专门提供给@InitBinder标记方法使用的参数resolver，有点少，注意 */       binderMethod.setHandlerMethodArgumentResolvers(this.initBinderArgumentResolvers);
   }
   // 添加@InitBinder标记方法的参数解析所需的WebDataBinder的制造工厂。共用初始化环境this.webBindingInitializer。
   // 绑定DefaultDataBinderFactory，是因为其方法initBinder是空的，不会让@InitBinder标记方法的参数解析提供者WebDataBinder，经历@IinitBinder标记的方法的过滤，不然就会导致无线循环，且没有意义。可看InitBinderDataBinderFactory实现initBinder的意义。
   binderMethod.setDataBinderFactory(new DefaultDataBinderFactory(this.webBindingInitializer));
   // 参数名发现？
   binderMethod.setParameterNameDiscoverer(this.parameterNameDiscoverer);
   return binderMethod;
}
```

```java
protected InitBinderDataBinderFactory createDataBinderFactory(List<InvocableHandlerMethod> binderMethods)
      throws Exception {

   // getWebBindingInitializer()提供RequestMappingHandlerAdapter.this.webBindingInitializer。
   return new ServletRequestDataBinderFactory(binderMethods, getWebBindingInitializer());
}
```

## WebDataBinderFactory构建WebDataBinder

​	WebDataBinder就是为此次MethodParameter代表的参数提供意向数据的转型服务。

DefaultDataBinderFactory.createBinder

```java
// 工厂创建WebDataBinder。
public final WebDataBinder createBinder(
      NativeWebRequest webRequest, @Nullable Object target, String objectName) throws Exception {

   // 实例化WebDataBinder实现类的方法委托，留个hook，给子类选择什么实现类的空间。 objectName是获取意向数据中构建的nameValue.name。
   WebDataBinder dataBinder = createBinderInstance(target, objectName, webRequest);
   if (this.initializer != null) {
      // 将dataBinder委托给WebBindingInitializer进行初始化信息。其中就有将环境中的ConversionService交给dataBinder。
      this.initializer.initBinder(dataBinder, webRequest);
   }
   // 给不同WebBindingInitializer初始化，提供相同初始化作用的空间。
   // 实际上只有InitBinderDataBinderFactory进行有意义的实现，其用来激活@InitBinder标记方法，提供datBinder，其是实质本意就是向dataBinder做出修改，比如添加PropertyEditor。
   initBinder(dataBinder, webRequest);
   return dataBinder;
}
```

InitBinderDataBinderFactory.initBinder

```java
public void initBinder(WebDataBinder dataBinder, NativeWebRequest request) throws Exception {
   // 让dataBinder经由符合要求的binderMethod的处理。一般性就要求注册PropertyEditor。不过怎么样多样化操作，还是有必要学习的。
   for (InvocableHandlerMethod binderMethod : this.binderMethods) {
      // 查看@InitBinder指明作用的nameValue.name集合里，是否包含此次解析的methodParamter的nameValue.name。若未指定，则说明无限制。
      if (isBinderMethodApplicable(binderMethod, dataBinder)) {
         // 这告诉我们@InitBinder标记的方法不能有返回值，只能用void。方法所需的参数可以从<request, dataBinder, 所用注解提供值>中提供。
         Object returnValue = binderMethod.invokeForRequest(request, null, dataBinder);
         if (returnValue != null) {
            throw new IllegalStateException(
                  "@InitBinder methods must not return a value (should be void): " + binderMethod);
         }
      }
   }
}
```

InitBinderDataBinderFactory.isBinderMethodApplicable

```java
protected boolean isBinderMethodApplicable(HandlerMethod initBinderMethod, WebDataBinder dataBinder) {
   InitBinder ann = initBinderMethod.getMethodAnnotation(InitBinder.class);
   // 这里表明，的确要求是@InitBinder标记的方法。
   Assert.state(ann != null, "No InitBinder annotation");
   String[] names = ann.value();
   // dataBinder.getObjectName是指此次解析的methodParameter的nameValue.name。
   return (ObjectUtils.isEmpty(names) || ObjectUtils.containsElement(names, dataBinder.getObjectName()));
}
```







```java
public final Object resolveArgument(MethodParameter parameter, @Nullable ModelAndViewContainer mavContainer,
			NativeWebRequest webRequest, @Nullable WebDataBinderFactory binderFactory) throws Exception {
   ......
if (binderFactory != null) {
   WebDataBinder binder = binderFactory.createBinder(webRequest, null, namedValueInfo.name);
   try {
      // 数据转换
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
   if (arg == null && namedValueInfo.defaultValue == null &&
         namedValueInfo.required && !nestedParameter.isOptional()) {
      handleMissingValueAfterConversion(namedValueInfo.name, nestedParameter, webRequest);
   }
}
```



## WebDataBinder服务方式

​	WebDataBinder提供的服务主要分为数据转型、数据绑定和验证服务。

​	数据转型是指，arguement-resolver已经从request或parameter注解中获取意向数据，但考虑该数据类型无法与paremeter-type匹配，由此进行数据转型。这是经常用到的。

​	数据绑定是指，针对已经提供的target，request.parameters判断每个能否匹配target中的内部字段——例如，<hero.nickname, jjking>将匹配target.hero.nickname——并在成功的情况下给予通过setter的赋值。若匹配过程中是深度的，则应当构造中间对象——一般就是给予无参构造函数，若只有有参构造函数呢，这是个有意思的话题。

​	数据验证是指验证提供的target中内置字段是否符合提供的注解的含义，比如@NotNull、@NotBlank。spring提供的validator是基于hibernate的validator进行的二次架构。

参考：

[WebBinder相关类关系图](./WebBinder.xmind)

### 数据转型

​	以AbstractNamedValueMethodArgumentResolver.resolveArgument(MethodParameter, ModelAndViewContainer, NativeWebRequest, WebDataBinderFactory)为使用背景，讨论WebDataBinder的数据转型操作：

```java
// 提供数据转换的服务有两种，一种是PropertyEditor，另一种是ConversionService提供的。前者应用于字符串转换成特殊类型，后者就是普遍转换的了，是系统内数据转换的共识。
// <propertyName, oldValue>对于数据转型来说是<null, null>，姑且先就这样解析。

// <requiredType, propertyName/propertyPath>是指定解析requiredType中propertyPath指定位置的属性。propertyPath可能是requiredType内嵌类型或其内嵌类型的属性。propertyName==null，则是解析requiredType整个类型的意思。
// typeDescriptor可能是用methodparameter或requiredType中的一种解析，当前分析的肯定是前者。
// 暂时就不理解propertyName的用法了，毕竟不清楚其值的格式是什么？
public <T> T convertIfNecessary(
    @Nullable String propertyName, 
    @Nullable Object oldValue, 
    @Nullable Object newValue,
    @Nullable Class<T> requiredType, 
    @Nullable TypeDescriptor typeDescriptor) throws IllegalArgumentException {

   // 依据<requiredType, propertyName>从注册容器中提取适配的PropertyEditor。一般就从this.customEditorsForPath或this.customEditors中找。
   PropertyEditor editor = this.propertyEditorRegistry.findCustomEditor(requiredType, propertyName);

   // 解析异常对象，记录异常信息
   ConversionFailedException conversionAttemptEx = null;

   // 提供WebConversionService，其内提供了大量基础的Converter。当然，自定义也可以。
   ConversionService conversionService = this.propertyEditorRegistry.getConversionService();
   // 如此看优先使用的还是Editoir。这里再没有editor却又ConversionService启动后者服务。
   if (editor == null && conversionService != null && newValue != null && typeDescriptor != null) {
      // 对newValue进行TypeDescriptor进行解析。
      TypeDescriptor sourceTypeDesc = TypeDescriptor.forObject(newValue);
      // conversionService判断sourceTypeDesc代表内容->typeDescriptor代表内容的可行性。
      if (conversionService.canConvert(sourceTypeDesc, typeDescriptor)) {
         try {
            return (T) conversionService.convert(newValue, sourceTypeDesc, typeDescriptor);
         }
         catch (ConversionFailedException ex) {
            // fallback to default conversion logic below
            conversionAttemptEx = ex;
         }
      }
   }

   // 这个重新赋值的意思，是强调newValue存在被ConversionService使用但转型失败了的情况。后续需要考虑这个，不过一般的，ConversionService也不会修改newValue。
   Object convertedValue = newValue;

   // 转换的结果现在不返回，后面处理了什么？
   // Value not of required type?
   // 有PropertyEditor 或者 没有editor且convertedValue不能向上造型成requiredType————给出的value不能简单地通过向上造型来提供转型。
   if (editor != null || (requiredType != null && !ClassUtils.isAssignableValue(requiredType, convertedValue))) {
      // requiredType是集合类型，且被抓换数据是字符串。则尝试将convertedValue分割造成字符串数组。
      if (typeDescriptor != null && requiredType != null && Collection.class.isAssignableFrom(requiredType) &&
            convertedValue instanceof String) {
         // 获取泛型元素的类型描述
         TypeDescriptor elementTypeDesc = typeDescriptor.getElementTypeDescriptor();
         // 元素类型给定，且是Class或枚举，则用","分隔convertedValue，获取字符串数组。这是意图全路径转Class.class或枚举元素了。
         if (elementTypeDesc != null) {
            Class<?> elementType = elementTypeDesc.getType();
            if (Class.class == elementType || Enum.class.isAssignableFrom(elementType)) {
               convertedValue = StringUtils.commaDelimitedListToStringArray((String) convertedValue);
            }
         }
      }
      // 若<requiredType, propertyName>找不到，则用仅用requiredType找。
      if (editor == null) {
         // Spring默认提供的，可以自己去看。
         editor = findDefaultEditor(requiredType);
      }
      // doConvertValue设置的目的之一：依据convertedValue是否是文本，选择性调用editor的setAsText或setValue。当用setAsText，oldValue会被setValue执行，放入容器中，因此newValue注入是否考虑到editor内置的旧属性需要关注，一般的就是覆盖，没有旧状态考虑。
      convertedValue = doConvertValue(oldValue, convertedValue, requiredType, editor);
   }
    
   // 若上述没有editor得到应用，则肯定没有响应的editor了。
   // 下面就是考虑editor失败只能启动ConversionService；或者直接的ConversonService使用也失败了，需要考虑convertedValue是否是Map、Collection、Array的特殊容器对象了，这种情况需要把每个元素领出来单独转型。
   boolean standardConversion = false;

   if (requiredType != null) {
       
      if (convertedValue != null) {
         // 顶级类Object需求，直接返回
         if (Object.class == requiredType) {
            return (T) convertedValue;
         }
         // 需求：convertedValue -> 数组
         else if (requiredType.isArray()) {
            // Array required -> apply appropriate conversion of elements.
            if (convertedValue instanceof String && Enum.class.isAssignableFrom(requiredType.getComponentType())) {
               convertedValue = StringUtils.commaDelimitedListToStringArray((String) convertedValue);
            }
            // componentType：requiredType.getComponentType()。按照convertedValue实际类型在[集合、数组、其他]选项中，匹配不同操作。前两者都是将convertedValue的每一个元素转换成componentType，其他就是将convertedValue自身转换成componentType。元素转换结果用数组存储返回。
            return (T) convertToTypedArray(convertedValue, propertyName, requiredType.getComponentType());
         }
         // 下面两个判断convertedValue是否是Collection或Map来处理，若不是则返回原值。转换后的结果还需要处理
         // 需求：convertedValue -> 集合。不直接返回。
         else if (convertedValue instanceof Collection) {
            // 将convertedValue的每个元素转换成requiredType所需要的元素类型。
            convertedValue = convertToTypedCollection(
                  (Collection<?>) convertedValue, propertyName, requiredType, typeDescriptor);
            standardConversion = true;
         }
         // 需求：convertedValue -> Map。不直接返回。
         else if (convertedValue instanceof Map) {
            // 将convertedValue的每个K-V转换成requiredType所需要的K-V类型。
            convertedValue = convertToTypedMap(
                  (Map<?, ?>) convertedValue, propertyName, requiredType, typeDescriptor);
            standardConversion = true;
         }
         
         // 如果convertedValue是单元素数组，把元素拎出来。
         if (convertedValue.getClass().isArray() && Array.getLength(convertedValue) == 1) {
            convertedValue = Array.get(convertedValue, 0);
            standardConversion = true;
         }
         
         // 需求：convertedValue（基本数据类型的包装类）-> 文本（String）
         if (String.class == requiredType && ClassUtils.isPrimitiveOrWrapper(convertedValue.getClass())) {
            // We can stringify any primitive value...
            return (T) convertedValue.toString();
         }
         // convertedValue是String类型，且requiredType不是String类型。则用requiredType的单参字符串类型的构造函数创建转型。这个好用，可以常用。
         else if (convertedValue instanceof String && !requiredType.isInstance(convertedValue)) {
            // conversionAttemptEx==null -> 之前未有异常抛出；requiredType非接口、非枚举。
            if (conversionAttemptEx == null && !requiredType.isInterface() && !requiredType.isEnum()) {
               try {
                  Constructor<T> strCtor = requiredType.getConstructor(String.class);
                  return BeanUtils.instantiateClass(strCtor, convertedValue);
               }
               catch (NoSuchMethodException ex) {
                  // proceed with field lookup
                  if (logger.isTraceEnabled()) {
                     logger.trace("No String constructor found on type [" + requiredType.getName() + "]", ex);
                  }
               }
               catch (Exception ex) {
                  if (logger.isDebugEnabled()) {
                     logger.debug("Construction via String failed for type [" + requiredType.getName() + "]", ex);
                  }
               }
            }
            String trimmedValue = ((String) convertedValue).trim();
            if (requiredType.isEnum() && trimmedValue.isEmpty()) {
               // It's an empty enum identifier: reset the enum value to null.
               return null;
            }
            // 枚举转换，暂不学。
            convertedValue = attemptToConvertStringToEnum(requiredType, trimmedValue, convertedValue);
            standardConversion = true;
         }
         else if (convertedValue instanceof Number && Number.class.isAssignableFrom(requiredType)) {
            convertedValue = NumberUtils.convertNumberToTargetClass(
                  (Number) convertedValue, (Class<Number>) requiredType);
            standardConversion = true;
         }
      }
      else {
         // convertedValue == null
         if (requiredType == Optional.class) {
            convertedValue = Optional.empty();
         }
      }

      // 以下内容基本就是为了抛出准备的。可以不看了
      if (!ClassUtils.isAssignableValue(requiredType, convertedValue)) {
         if (conversionAttemptEx != null) {
            // Original exception from former ConversionService call above...
            throw conversionAttemptEx;
         }
         else if (conversionService != null && typeDescriptor != null) {
            // ConversionService not tried before, probably custom editor found
            // but editor couldn't produce the required type...
            // ConversionService未用，可能是由于找到了editor导致，先用了editor，但是editor无法实现转换效果，因此启用ConversionService试试。
            TypeDescriptor sourceTypeDesc = TypeDescriptor.forObject(newValue);
            if (conversionService.canConvert(sourceTypeDesc, typeDescriptor)) {
               return (T) conversionService.convert(newValue, sourceTypeDesc, typeDescriptor);
            }
         }

         // Definitely doesn't match: throw IllegalArgumentException/IllegalStateException
         StringBuilder msg = new StringBuilder();
         msg.append("Cannot convert value of type '").append(ClassUtils.getDescriptiveType(newValue));
         msg.append("' to required type '").append(ClassUtils.getQualifiedName(requiredType)).append('\'');
         if (propertyName != null) {
            msg.append(" for property '").append(propertyName).append('\'');
         }
         if (editor != null) {
            msg.append(": PropertyEditor [").append(editor.getClass().getName()).append(
                  "] returned inappropriate value of type '").append(
                  ClassUtils.getDescriptiveType(convertedValue)).append('\'');
            throw new IllegalArgumentException(msg.toString());
         }
         else {
            msg.append(": no matching editors or conversion strategy found");
            throw new IllegalStateException(msg.toString());
         }
      }
   }

   if (conversionAttemptEx != null) {
      if (editor == null && !standardConversion && requiredType != null && Object.class != requiredType) {
         throw conversionAttemptEx;
      }
      logger.debug("Original ConversionService attempt failed - ignored since " +
            "PropertyEditor based conversion eventually succeeded", conversionAttemptEx);
   }

   return (T) convertedValue;
}
```

```java
// convertIfNecessary是对外提供数据转型的接口，其重载了多个以适配多种环境，它们的区别在于提供对目标类型的表述：
/*
 * ① <Class<T>> - 所需类型Class
 * ② <Class<T>, MethodParameter> - 所需类型Class和目标参数信息的封装对象
 * ③ <Class<T>, Field> - 这应该是数据绑定一块的内容
 * ④ <Class<T>, TypeDescriptor> - 所需类型Class和类型描述TypeDescriptor对象，后面都会统一用TypeDescriptor类型分装MethodParameter、Field的必要信息。
 */
public <T> T convertIfNecessary(@Nullable Object value, @Nullable Class<T> requiredType,
      @Nullable MethodParameter methodParam) throws TypeMismatchException {

   // getTypeConverter将会返回TypeConverter-类型转换服务提供者，convertIfNecessary(...)执行服务。
   return getTypeConverter().convertIfNecessary(value, requiredType, methodParam);
}
```

```java
// 提供了三种类型转换服务：DirectFieldAccessor、BeanWrapperImpl、SimpleTypeConverter，其中后者两者常用。前两者用于数据绑定，最后者用于数据转型。显然，前两者与BindingResult绑定，BindingResult相应的存在两种：DirectFieldBindingResult、BeanPropertyBindingResult，它们选择依据this.directFieldAccess选择创建。
/*
 * 三种类型转换服务：
 * DirectFieldBindingResult-DirectFieldAccessor
 * BeanPropertyBindingResult-BeanWrapperImpl
 * SimpleTypeConverter
 */
protected TypeConverter getTypeConverter() {
   if (getTarget() != null) {
      /*
       * 依据this.directFieldAccess选择性提供：
       * DirectFieldBindingResult-DirectFieldAccessor(target)
       * BeanPropertyBindingResult-BeanWrapperImpl(target)
       */
      return getInternalBindingResult().getPropertyAccessor();
   }
   else {
      return getSimpleTypeConverter();
   }
}
```

在[WebBinder相关类关系图](./WebBinder.xmind)-convertIfNecessary进行路线图表中已经展示了`convertIfNecessary`执行链，其中最重要的一块在TypeConverterDelegate类上，下面就介绍它：

```java
@Nullable
public <T> T convertIfNecessary(@Nullable String propertyName, @Nullable Object oldValue, @Nullable Object newValue,
      @Nullable Class<T> requiredType, @Nullable TypeDescriptor typeDescriptor) throws IllegalArgumentException {

   // Custom editor for this type?
   PropertyEditor editor = this.propertyEditorRegistry.findCustomEditor(requiredType, propertyName);

   ConversionFailedException conversionAttemptEx = null;

   // No custom editor but custom ConversionService specified?
   ConversionService conversionService = this.propertyEditorRegistry.getConversionService();
   if (editor == null && conversionService != null && newValue != null && typeDescriptor != null) {
      TypeDescriptor sourceTypeDesc = TypeDescriptor.forObject(newValue);
      if (conversionService.canConvert(sourceTypeDesc, typeDescriptor)) {
         try {
            return (T) conversionService.convert(newValue, sourceTypeDesc, typeDescriptor);
         }
         catch (ConversionFailedException ex) {
            // fallback to default conversion logic below
            conversionAttemptEx = ex;
         }
      }
   }

   Object convertedValue = newValue;

   // Value not of required type?
   if (editor != null || (requiredType != null && !ClassUtils.isAssignableValue(requiredType, convertedValue))) {
      if (typeDescriptor != null && requiredType != null && Collection.class.isAssignableFrom(requiredType) &&
            convertedValue instanceof String) {
         TypeDescriptor elementTypeDesc = typeDescriptor.getElementTypeDescriptor();
         if (elementTypeDesc != null) {
            Class<?> elementType = elementTypeDesc.getType();
            if (Class.class == elementType || Enum.class.isAssignableFrom(elementType)) {
               convertedValue = StringUtils.commaDelimitedListToStringArray((String) convertedValue);
            }
         }
      }
      if (editor == null) {
         editor = findDefaultEditor(requiredType);
      }
      convertedValue = doConvertValue(oldValue, convertedValue, requiredType, editor);
   }

   boolean standardConversion = false;

   if (requiredType != null) {
      // Try to apply some standard type conversion rules if appropriate.

      if (convertedValue != null) {
         if (Object.class == requiredType) {
            return (T) convertedValue;
         }
         else if (requiredType.isArray()) {
            // Array required -> apply appropriate conversion of elements.
            if (convertedValue instanceof String && Enum.class.isAssignableFrom(requiredType.getComponentType())) {
               convertedValue = StringUtils.commaDelimitedListToStringArray((String) convertedValue);
            }
            return (T) convertToTypedArray(convertedValue, propertyName, requiredType.getComponentType());
         }
         else if (convertedValue instanceof Collection) {
            // Convert elements to target type, if determined.
            convertedValue = convertToTypedCollection(
                  (Collection<?>) convertedValue, propertyName, requiredType, typeDescriptor);
            standardConversion = true;
         }
         else if (convertedValue instanceof Map) {
            // Convert keys and values to respective target type, if determined.
            convertedValue = convertToTypedMap(
                  (Map<?, ?>) convertedValue, propertyName, requiredType, typeDescriptor);
            standardConversion = true;
         }
         if (convertedValue.getClass().isArray() && Array.getLength(convertedValue) == 1) {
            convertedValue = Array.get(convertedValue, 0);
            standardConversion = true;
         }
         if (String.class == requiredType && ClassUtils.isPrimitiveOrWrapper(convertedValue.getClass())) {
            // We can stringify any primitive value...
            return (T) convertedValue.toString();
         }
         else if (convertedValue instanceof String && !requiredType.isInstance(convertedValue)) {
            if (conversionAttemptEx == null && !requiredType.isInterface() && !requiredType.isEnum()) {
               try {
                  Constructor<T> strCtor = requiredType.getConstructor(String.class);
                  return BeanUtils.instantiateClass(strCtor, convertedValue);
               }
               catch (NoSuchMethodException ex) {
                  // proceed with field lookup
                  if (logger.isTraceEnabled()) {
                     logger.trace("No String constructor found on type [" + requiredType.getName() + "]", ex);
                  }
               }
               catch (Exception ex) {
                  if (logger.isDebugEnabled()) {
                     logger.debug("Construction via String failed for type [" + requiredType.getName() + "]", ex);
                  }
               }
            }
            String trimmedValue = ((String) convertedValue).trim();
            if (requiredType.isEnum() && trimmedValue.isEmpty()) {
               // It's an empty enum identifier: reset the enum value to null.
               return null;
            }
            convertedValue = attemptToConvertStringToEnum(requiredType, trimmedValue, convertedValue);
            standardConversion = true;
         }
         else if (convertedValue instanceof Number && Number.class.isAssignableFrom(requiredType)) {
            convertedValue = NumberUtils.convertNumberToTargetClass(
                  (Number) convertedValue, (Class<Number>) requiredType);
            standardConversion = true;
         }
      }
      else {
         // convertedValue == null
         if (requiredType == Optional.class) {
            convertedValue = Optional.empty();
         }
      }

      if (!ClassUtils.isAssignableValue(requiredType, convertedValue)) {
         if (conversionAttemptEx != null) {
            // Original exception from former ConversionService call above...
            throw conversionAttemptEx;
         }
         else if (conversionService != null && typeDescriptor != null) {
            // ConversionService not tried before, probably custom editor found
            // but editor couldn't produce the required type...
            TypeDescriptor sourceTypeDesc = TypeDescriptor.forObject(newValue);
            if (conversionService.canConvert(sourceTypeDesc, typeDescriptor)) {
               return (T) conversionService.convert(newValue, sourceTypeDesc, typeDescriptor);
            }
         }

         // Definitely doesn't match: throw IllegalArgumentException/IllegalStateException
         StringBuilder msg = new StringBuilder();
         msg.append("Cannot convert value of type '").append(ClassUtils.getDescriptiveType(newValue));
         msg.append("' to required type '").append(ClassUtils.getQualifiedName(requiredType)).append('\'');
         if (propertyName != null) {
            msg.append(" for property '").append(propertyName).append('\'');
         }
         if (editor != null) {
            msg.append(": PropertyEditor [").append(editor.getClass().getName()).append(
                  "] returned inappropriate value of type '").append(
                  ClassUtils.getDescriptiveType(convertedValue)).append('\'');
            throw new IllegalArgumentException(msg.toString());
         }
         else {
            msg.append(": no matching editors or conversion strategy found");
            throw new IllegalStateException(msg.toString());
         }
      }
   }

   if (conversionAttemptEx != null) {
      if (editor == null && !standardConversion && requiredType != null && Object.class != requiredType) {
         throw conversionAttemptEx;
      }
      logger.debug("Original ConversionService attempt failed - ignored since " +
            "PropertyEditor based conversion eventually succeeded", conversionAttemptEx);
   }

   return (T) convertedValue;
}
```

很麻烦啊，终于知道为什么json格式传输那么爽了。还好，相同格式数据传入不会改变类型转换器检索、使用路径，即使编程失败了，也可以调整到合适，不会应该运行中改变检索、使用路径，还是可以的。



### 数据绑定

#### 使用背景

​	这里使用的背景在ServletModelAttributeMethodProcessor(annotationNotRequired=true)的场景中，其用来解析自定义的POJO。

```java
public final Object resolveArgument(
    MethodParameter parameter, 
    @Nullable ModelAndViewContainer mavContainer,
    NativeWebRequest webRequest, 
    @Nullable WebDataBinderFactory binderFactory) throws Exception {
    ..........................
    // 构造parameter代表类型的对象时没有报错
    if (bindingResult == null) {
        // 创建用于数据绑定的WebDataBinder，TypeConverter启用BeanWrapperImpl
        WebDataBinder binder = binderFactory.createBinder(webRequest, attribute, name);
        if (binder.getTarget() != null) {
            // 好嘛，bindingDisabled在这里用到了。。。
            if (!mavContainer.isBindingDisabled(name)) {
                // 这是数据绑定服务启动的入口
                bindRequestParameters(binder, webRequest);
            }
            validateIfApplicable(binder, parameter);
            if (binder.getBindingResult().hasErrors() && isBindExceptionRequired(binder, parameter)) {
                throw new BindException(binder.getBindingResult());
            }
        }
        // Value type adaptation, also covering java.util.Optional
        if (!parameter.getParameterType().isInstance(attribute)) {
            attribute = binder.convertIfNecessary(binder.getTarget(), parameter.getParameterType(), parameter);
        }
        bindingResult = binder.getBindingResult();
    }
    .......................
}
```

**WebRequestDataBinder.bind(.)**

```java
protected void bindRequestParameters(WebDataBinder binder, NativeWebRequest request) {
    ServletRequest servletRequest = request.getNativeRequest(ServletRequest.class);
    Assert.state(servletRequest != null, "No ServletRequest");
    ServletRequestDataBinder servletBinder = (ServletRequestDataBinder) binder;
    servletBinder.bind(servletRequest);
}
```

#### 提供绑定数据

```java
// 分为两部分：准备MutablePropertyValues 和 将mpv作为数据源进行数据绑定
public void bind(ServletRequest request) {
    // 构建过程中，已经把文本参数coyoteRequest.parmeters提取出来注入了。
    MutablePropertyValues mpvs = new ServletRequestParameterPropertyValues(request);
    MultipartRequest multipartRequest = WebUtils.getNativeRequest(request, MultipartRequest.class);
    // 将文件部分绑定
    if (multipartRequest != null) {
        bindMultipart(multipartRequest.getMultiFileMap(), mpvs);
    }
    // 奇怪，为什么会往这里走呢？
    else if (StringUtils.startsWithIgnoreCase(request.getContentType(), MediaType.MULTIPART_FORM_DATA_VALUE)) {
        HttpServletRequest httpServletRequest = WebUtils.getNativeRequest(request, HttpServletRequest.class);
        if (httpServletRequest != null && HttpMethod.POST.matches(httpServletRequest.getMethod())) {
            StandardServletPartUtils.bindParts(httpServletRequest, mpvs, isBindEmptyMultipartFiles());
        }
    }
    addBindValues(mpvs, request);
    // 数据绑定
    doBind(mpvs);
}
```

#### 执行绑定前置准备

​	配置允许字段、不允许字段等操作。

```java
protected void doBind(MutablePropertyValues mpvs) {
   // fieldDefaultPrefix - "!" 前缀处理
   checkFieldDefaults(mpvs);
   // fieldMarkerPrefix - "_" 前缀处理
   checkFieldMarkers(mpvs);
   // 处理name后缀"[]"情况，考虑将后缀去掉的newName加入（条件是可写入，应当是是否又对标setter），并且一定移除原本pv
   adaptEmptyArrayIndices(mpvs);
   // 进一步
   super.doBind(mpvs);
}
```

```java
protected void doBind(MutablePropertyValues mpvs) {
   // DataBinder提供allowedFields和disallowedFields数组，来许可或pause，一般都为空，无需判断。  
   checkAllowedFields(mpvs);
   // 检查是否具备要求的requiredFields，若不具备则向bindingErrorProcessor添加异常FieldError。
   checkRequiredFields(mpvs);
   // 好了可以开始应用了
   applyPropertyValues(mpvs);
}
```

#### 选择TypeConverter支持数据绑定

```java
protected void applyPropertyValues(MutablePropertyValues mpvs) {
   try {
      // Bind request parameters onto target object.
      // 这里探讨：BeanWrapperImpl.setPropertyValues(mpvs, this.ignoreUnknownFields, this.ignoreInvalidFields)
      // this.ignoreUnknownFields指的是是否忽视字段绑定时字段不在；this.ignoreInvalidFields指的是进行对象的校验时是否忽视
      // this.ignoreUnknownFields与@RequestBody的可忽视比较：都可以没有任何field可填充，但是JSON必须得有{}。
      // 构造函数这块，前者可以任意存在，只需要setter即可，但是json必须得有无参构造函数，内置对象也必须空参构造函数。
      getPropertyAccessor().setPropertyValues(mpvs, isIgnoreUnknownFields(), isIgnoreInvalidFields());
   }
   catch (PropertyBatchUpdateException ex) {
      // Use bind error processor to create FieldErrors.
      for (PropertyAccessException pae : ex.getPropertyAccessExceptions()) {
         getBindingErrorProcessor().processPropertyAccessException(pae, getInternalBindingResult());
      }
   }
}
```

```java
public void setPropertyValues(PropertyValues pvs, boolean ignoreUnknown, boolean ignoreInvalid)
      throws BeansException {

   List<PropertyAccessException> propertyAccessExceptions = null;
   List<PropertyValue> propertyValues = (pvs instanceof MutablePropertyValues ?
         ((MutablePropertyValues) pvs).getPropertyValueList() : Arrays.asList(pvs.getPropertyValues()));

   // 这个向反射调用的访问权限，先后都得操作。什么意思呢？
   if (ignoreUnknown) {
      this.suppressNotWritablePropertyException = true;
   }
   try {
      for (PropertyValue pv : propertyValues) {
         // setPropertyValue may throw any BeansException, which won't be caught
         // here, if there is a critical failure such as no matching field.
         // We can attempt to deal only with less serious exceptions.
         try {
            // 进入
            setPropertyValue(pv);
         }
         catch (NotWritablePropertyException ex) {
            if (!ignoreUnknown) {
               throw ex;
            }
            // Otherwise, just ignore it and continue...
         }
         catch (NullValueInNestedPathException ex) {
            if (!ignoreInvalid) {
               throw ex;
            }
            // Otherwise, just ignore it and continue...
         }
         catch (PropertyAccessException ex) {
            if (propertyAccessExceptions == null) {
               propertyAccessExceptions = new ArrayList<>();
            }
            propertyAccessExceptions.add(ex);
         }
      }
   }
   finally {
      if (ignoreUnknown) {
         this.suppressNotWritablePropertyException = false;
      }
   }

   // If we encountered individual exceptions, throw the composite exception.
   if (propertyAccessExceptions != null) {
      PropertyAccessException[] paeArray = propertyAccessExceptions.toArray(new PropertyAccessException[0]);
      throw new PropertyBatchUpdateException(paeArray);
   }
}
```

#### 数据绑定

```java
// 分为两个步骤：创建字段匹配的前置路径上的空对象；字段匹配位置上进行数据绑定
public void setPropertyValue(PropertyValue pv) throws BeansException {
   PropertyTokenHolder tokens = (PropertyTokenHolder) pv.resolvedTokens;
   if (tokens == null) {
      String propertyName = pv.getName();
      AbstractNestablePropertyAccessor nestedPa;
      try {
         // 打通propertyName设置之前的内置字段，比如hero.name，就必须把hero先创建。
         nestedPa = getPropertyAccessorForPropertyPath(propertyName);
      }
      catch (NotReadablePropertyException ex) {
         throw new NotWritablePropertyException(getRootClass(), this.nestedPath + propertyName,
               "Nested property in path '" + propertyName + "' does not exist", ex);
      }
      tokens = getPropertyNameTokens(getFinalPath(nestedPa, propertyName));
      if (nestedPa == this) {
         pv.getOriginalPropertyValue().resolvedTokens = tokens;
      }
      // 向目标字段绑定数据pv
      nestedPa.setPropertyValue(tokens, pv);
   }
   else {
      setPropertyValue(tokens, pv);
   }
}
```

##### 逐个深入创建空对象，铺实目标字段前的空对象。

```java
protected AbstractNestablePropertyAccessor getPropertyAccessorForPropertyPath(String propertyPath) {
   // 比如hero.name, 获取nestedProperty=hero nestedPath=name
   int pos = PropertyAccessorUtils.getFirstNestedPropertySeparatorIndex(propertyPath);
   // Handle nested properties recursively.
   if (pos > -1) {
      String nestedProperty = propertyPath.substring(0, pos);
      String nestedPath = propertyPath.substring(pos + 1);
      // 进入
      AbstractNestablePropertyAccessor nestedPa = getNestedPropertyAccessor(nestedProperty);
      // 调用自身，可以看到这是个递归，是对字段的内置字段进行赋值。即对hero的name字段
      return nestedPa.getPropertyAccessorForPropertyPath(nestedPath);
   }
   else {
      // 若是最后一个字段：这个字段是可以借助ConversionService或PropertyEditor解析文本/文件转换成当前BeanWrapperImpl解析的字段（对象）
      return this;
   }
}
```

```java
private AbstractNestablePropertyAccessor getNestedPropertyAccessor(String nestedProperty) {
   if (this.nestedPropertyAccessors == null) {
      this.nestedPropertyAccessors = new HashMap<>();
   }
   // Get value of bean property.
   // 这个就不深入了，简单介绍一下PropertyToeknHolder：<actualName, canonicalName, keys>，貌似是对key-value一类进行解析的。
   PropertyTokenHolder tokens = getPropertyNameTokens(nestedProperty);
   String canonicalName = tokens.canonicalName;
   Object value = getPropertyValue(tokens);
   if (value == null || (value instanceof Optional && !((Optional<?>) value).isPresent())) {
      // 允许自动增长————即允许达到匹配为之前可创建空对象
      if (isAutoGrowNestedPaths()) {
         // 创建空对象，这里就不继续了，累了，太长太深记录不太合适
         value = setDefaultValue(tokens);
      }
      else {
         throw new NullValueInNestedPathException(getRootClass(), this.nestedPath + canonicalName);
      }
   }

   // Lookup cached sub-PropertyAccessor, create new one if not found.
   // <String, AbstractNestablePropertyAccessor>：key代表当前wrappedObject的内置字段名canonicalName，value代表给canonicalName提供数据绑定的BeanWrapperImpl。
   AbstractNestablePropertyAccessor nestedPa = this.nestedPropertyAccessors.get(canonicalName);
   if (nestedPa == null || nestedPa.getWrappedInstance() != ObjectUtils.unwrapOptional(value)) {
      if (logger.isTraceEnabled()) {
         logger.trace("Creating new nested " + getClass().getSimpleName() + " for property '" + canonicalName + "'");
      }
      // 
      nestedPa = newNestedPropertyAccessor(value, this.nestedPath + canonicalName + NESTED_PROPERTY_SEPARATOR);
      // Inherit all type-specific PropertyEditors.
      copyDefaultEditorsTo(nestedPa);
      copyCustomEditorsTo(nestedPa, canonicalName);
      this.nestedPropertyAccessors.put(canonicalName, nestedPa);
   }
   else {
      if (logger.isTraceEnabled()) {
         logger.trace("Using cached nested property accessor for property '" + canonicalName + "'");
      }
   }
   // 返回下一步要求解析的BeanWrapperImpl--内置字段（对象）
   return nestedPa;
}
```



```java
private AbstractNestablePropertyAccessor getNestedPropertyAccessor(String nestedProperty) {
   if (this.nestedPropertyAccessors == null) {
      this.nestedPropertyAccessors = new HashMap<>();
   }
   // Get value of bean property.
   PropertyTokenHolder tokens = getPropertyNameTokens(nestedProperty);
   String canonicalName = tokens.canonicalName;
   Object value = getPropertyValue(tokens);
   if (value == null || (value instanceof Optional && !((Optional<?>) value).isPresent())) {
      if (isAutoGrowNestedPaths()) {
         value = setDefaultValue(tokens);
      }
      else {
         throw new NullValueInNestedPathException(getRootClass(), this.nestedPath + canonicalName);
      }
   }

   // Lookup cached sub-PropertyAccessor, create new one if not found.
   AbstractNestablePropertyAccessor nestedPa = this.nestedPropertyAccessors.get(canonicalName);
   if (nestedPa == null || nestedPa.getWrappedInstance() != ObjectUtils.unwrapOptional(value)) {
      if (logger.isTraceEnabled()) {
         logger.trace("Creating new nested " + getClass().getSimpleName() + " for property '" + canonicalName + "'");
      }
      nestedPa = newNestedPropertyAccessor(value, this.nestedPath + canonicalName + NESTED_PROPERTY_SEPARATOR);
      // Inherit all type-specific PropertyEditors.
      copyDefaultEditorsTo(nestedPa);
      copyCustomEditorsTo(nestedPa, canonicalName);
      this.nestedPropertyAccessors.put(canonicalName, nestedPa);
   }
   else {
      if (logger.isTraceEnabled()) {
         logger.trace("Using cached nested property accessor for property '" + canonicalName + "'");
      }
   }
   return nestedPa;
}
```

##### 向目标字段绑定数据pv

```java
protected void setPropertyValue(PropertyTokenHolder tokens, PropertyValue pv) throws BeansException {
   if (tokens.keys != null) {
   	  // 这个应该是map一类的把。。
      processKeyedProperty(tokens, pv);
   }
   else {
      // 进入
      processLocalProperty(tokens, pv);
   }
}
```

```java
private void processLocalProperty(PropertyTokenHolder tokens, PropertyValue pv) {
    PropertyHandler ph = getLocalPropertyHandler(tokens.actualName);
    // 假定存在setter，不然除却rootObject的那一层字段，都得被抛出异常。
    Object oldValue = null;
    try {
        Object originalValue = pv.getValue();
        Object valueToApply = originalValue;
        if (!Boolean.FALSE.equals(pv.conversionNecessary)) {
            if (pv.isConverted()) {
                valueToApply = pv.getConvertedValue();
            }
            else {
                if (isExtractOldValueForEditor() && ph.isReadable()) {
                    try {
                        oldValue = ph.getValue();
                    }
                    catch (Exception ex) {
                        if (ex instanceof PrivilegedActionException) {
                            ex = ((PrivilegedActionException) ex).getException();
                        }
                        if (logger.isDebugEnabled()) {
                            logger.debug("Could not read previous value of property '" +
                                         this.nestedPath + tokens.canonicalName + "'", ex);
                        }
                    }
                }
                // 文本数据转换，当然也可能是文件
                valueToApply = convertForProperty(
                    tokens.canonicalName, oldValue, originalValue, ph.toTypeDescriptor());
            }
            // 记录是否进行了数据转换。
            pv.getOriginalPropertyValue().conversionNecessary = (valueToApply != originalValue);
        }
        // 调用存在的setter方法。
        ph.setValue(valueToApply);
    }
    ...................// 异常捕获处理
}
```

##### 总结

​	BeanWrapperImpl解析rootObject或者wrapperedObject，每个字段都有。

​	给定了一个深链链的字段，则采用递归的方式，一节一节的先将空字段创建出来，最后再进行setter设置。

​	数据绑定支持额外的敏感字段屏蔽、必要字段需要的条件。

​	BeanWrapperImp解析必定还支持key-value。自己要试试看。



### 数据验证

​	这个地方主打的就是实现，不深入研究了，毕竟是二次设计框架。

研究背景还是ServletModelAttributeMethodProcessor(annotationNotRequired=true)

```java
public final Object resolveArgument(
    MethodParameter parameter, 
    @Nullable ModelAndViewContainer mavContainer,
    NativeWebRequest webRequest, 
    @Nullable WebDataBinderFactory binderFactory) throws Exception {
    ..........................
    // 构造parameter代表类型的对象时没有报错
    if (bindingResult == null) {
        // 创建用于数据绑定的WebDataBinder，TypeConverter启用BeanWrapperImpl
        WebDataBinder binder = binderFactory.createBinder(webRequest, attribute, name);
        if (binder.getTarget() != null) {
            // 好嘛，bindingDisabled在这里用到了。。。
            if (!mavContainer.isBindingDisabled(name)) {
                // 这是数据绑定服务启动的入口
                bindRequestParameters(binder, webRequest);
            }
            // 数据验证。
            validateIfApplicable(binder, parameter);
            if (binder.getBindingResult().hasErrors() && isBindExceptionRequired(binder, parameter)) {
                throw new BindException(binder.getBindingResult());
            }
        }
        // Value type adaptation, also covering java.util.Optional
        if (!parameter.getParameterType().isInstance(attribute)) {
            attribute = binder.convertIfNecessary(binder.getTarget(), parameter.getParameterType(), parameter);
        }
        bindingResult = binder.getBindingResult();
    }
    .......................
}
```

```java
protected void validateIfApplicable(WebDataBinder binder, MethodParameter parameter) {
   for (Annotation ann : parameter.getParameterAnnotations()) {
      Object[] validationHints = ValidationAnnotationUtils.determineValidationHints(ann);
      if (validationHints != null) {
         binder.validate(validationHints);
         break;
      }
   }
}
```

后续可前往：

[ServletModelAttributeMethodProcessor](./arguement-resolver/ServletModelAttributeMethodProcessor.md)————看内容

[WebDataBinder提供的数据校验机制使用](./WebDataBinder学习/WebDataBinder提供的数据校验机制使用.md)







## PropertyEditorRegistrySupport

### PropertyEditorRegistry接口实现

**registerCustomEditor**

```java
public void registerCustomEditor(Class<?> requiredType, PropertyEditor propertyEditor) {
   registerCustomEditor(requiredType, null, propertyEditor);
}
```

```java
// this.customEditorsForPath和this.customEditors是两个不一样需求的PropertEditor容器。
// this.customEditorsForPath - <propertyPath, CustomEditorHolder(propertyEditor, requiredType)>，propertyPath是具有字段路径匹配的需求————这只有在数据绑定的环境下应用。
// this.customEditors - <requiredType, propertyEditor>，一般用在数据转型，依据转型的目标提供PropertyEditor，因此一种类型仅能提供一个PorpetyEditor————String → requiredType。
public void registerCustomEditor(@Nullable Class<?> requiredType, @Nullable String propertyPath, PropertyEditor propertyEditor) {
   if (requiredType == null && propertyPath == null) {
      throw new IllegalArgumentException("Either requiredType or propertyPath is required");
   }
   if (propertyPath != null) {
      if (this.customEditorsForPath == null) {
         this.customEditorsForPath = new LinkedHashMap<>(16);
      }
      this.customEditorsForPath.put(propertyPath, new CustomEditorHolder(propertyEditor, requiredType));
   }
   else {
      if (this.customEditors == null) {
         this.customEditors = new LinkedHashMap<>(16);
      }
      this.customEditors.put(requiredType, propertyEditor);
      this.customEditorCache = null;
   }
}
```

**findCustomEditor**

```java
// 立下使用规矩吧，requiredType不能空传，propertyPath用来区分数据转型和数据绑定。
public PropertyEditor findCustomEditor(@Nullable Class<?> requiredType, @Nullable String propertyPath) {
   Class<?> requiredTypeToUse = requiredType;
   if (propertyPath != null) {
      if (this.customEditorsForPath != null) {
         // 从this.customEditorsForPath中依据propertyPath选中CustomEditorHolder，从其看是否可以拿出PropertyEditor，进一步匹配需求：当requiredType指定，需匹配requiredType与CustomEditorHolder.registeredType的实现或继承关系，一般就是前者更抽象；当requiredType未指定，则只有CustomEditorHolder.registeredType非数组非集合的情况下能获取。
         // 规则能简化获取逻辑，背景要求：CustomEditorHolder.registeredType如是给定CustomEditorHolder.PropertyEditor将会返回的最抽象类型（可能会有多种类型的实例以Object的存储），requiredType不能给空。那么获取规则应当简化为：requiredType是CustomEditorHolder.PropertyEditor的抽象。感觉源码中CustomEditorHolder的判断显得很怪。
         // 看源码来理解比较方便。
         PropertyEditor editor = getCustomEditor(propertyPath, requiredType);
         // 这里的涉及对propertyPath包含"[]"的剔出匹配，比如pre[ss].suffix → pre.suffix的形式，这个不做考虑，没有这方便的场景想象。
         if (editor == null) {
            List<String> strippedPaths = new ArrayList<>();
            addStrippedPropertyPaths(strippedPaths, "", propertyPath);
            for (Iterator<String> it = strippedPaths.iterator(); it.hasNext() && editor == null;) {
               String strippedPath = it.next();
               editor = getCustomEditor(strippedPath, requiredType);
            }
         }
         if (editor != null) {
            return editor;
         }
      }
      // 没找到且requiredType为空，则用propertyPath衍生出requiredType，不知道是怎么搞得。不管他，反正定个规则：requiredType必须非空传入。
      if (requiredType == null) {
         requiredTypeToUse = getPropertyType(propertyPath);
      }
   }
   // 数据转型服务下的，找this.customEditors拿取适配的。感觉里面的isAssignableFrom匹配机制怪怪的，顺序貌似颠倒了。
   return getCustomEditor(requiredTypeToUse);
}
```



## BeanWrapperImpl

### 字段解析

<wrappedObject, nestedPath, rootObject>

- wrappedObject：当前分析的（字段）对象或rootObject。
- nestedPath：wrappedObject所代表字段路径
- rootObject：根对象，即parameter对应对象。

比如：

<JiKing$Hero, "hero.", JiKing>

Jiking有内置字段hero，其类型是Jiking.Hero。



this.nestedPropertyAccessors

​	<String, AbstractNestablePropertyAccessor>：即当前wrappedObject内置的字段名（驼峰格式）与其字段（对象）被提供数据绑定服务的BeanWrapperImpl。






## ModelFactory构建

RequestMappingHandlerAdapter.invokeHandlerMethod(...) -> RequestMappingHandlerAdapter.getModelFactory(...)

```java
// ModelFactory的创建，加载了命中handler-method的@ModelAttribute标记方法，并为每个方法提供InvocableHandlerMethod封装。
// ModelFactory(InvocableHandlerMethod(@ModelAttribute命中method, binderFactory), binderFactory, sessionAttrHandler)
private ModelFactory getModelFactory(HandlerMethod handlerMethod, WebDataBinderFactory binderFactory) {
   // SessionAttributesHandler持有mavc应当绑定request属性的name集合，以及检索或存入request.attributes的能力。
   SessionAttributesHandler sessionAttrHandler = getSessionAttributesHandler(handlerMethod);
   Class<?> handlerType = handlerMethod.getBeanType();
   // this.modelAttributeCache是局部提供的@ModelAttribute标记的方法，该局部是指当前访问方法所在的类————@Controller所标记的类，handlerType。
   Set<Method> methods = this.modelAttributeCache.get(handlerType);
   if (methods == null) {
      // 挑选handler-method所在类的类内@ModelAttribute标记方法。这是局部的。
      // MODEL_ATTRIBUTE_METHODS是个过滤器，过滤使用@ModelAttribute，未用@RequestMapping的类内方法。
      methods = MethodIntrospector.selectMethods(handlerType, MODEL_ATTRIBUTE_METHODS);
      this.modelAttributeCache.put(handlerType, methods);
   }
   List<InvocableHandlerMethod> attrMethods = new ArrayList<>();
   // 挑选ControllerAdvice类内命中的@ModelAttribute标记方法。全局的。
   // 全局优先于局部加入容器attrMethods。
   this.modelAttributeAdviceCache.forEach((controllerAdviceBean, methodSet) -> {
      // 判断：ControllerAdvice类是否命中handlerType
      if (controllerAdviceBean.isApplicableToBeanType(handlerType)) {
         Object bean = controllerAdviceBean.resolveBean();
         for (Method method : methodSet) {
            attrMethods.add(createModelAttributeMethod(binderFactory, bean, method));
         }
      }
   });
   // 局部加入容器attrMethods。
   for (Method method : methods) {
      Object bean = handlerMethod.getBean();
      attrMethods.add(createModelAttributeMethod(binderFactory, bean, method));
   }
   return new ModelFactory(attrMethods, binderFactory, sessionAttrHandler);
}
```

```java
private SessionAttributesHandler getSessionAttributesHandler(HandlerMethod handlerMethod) {
   // RequestMappingHandlerAdapter注册在RequestMappingHandlerMapping中，它们都是单体JVM唯一的存在。
   // RequestMappingHandlerAdapter.this.sessionAttributesHandlerCache存储<handlerType, SessionAttributesHandler(this.sessionAttributeStore)>。this.sessionAttributeStore是无状态的代理类，其代理了操作WebRequest的attribute的功能。
   return this.sessionAttributesHandlerCache.computeIfAbsent(
         handlerMethod.getBeanType(),
         type -> new SessionAttributesHandler(type, this.sessionAttributeStore));
}
```

```java
// 为命中的@ModelAttribute标记的方法提供InvocableHandlerMethod封装。
private InvocableHandlerMethod createModelAttributeMethod(WebDataBinderFactory factory, Object bean, Method method) {
   InvocableHandlerMethod attrMethod = new InvocableHandlerMethod(bean, method);
   if (this.argumentResolvers != null) {
       // 有完整的参数resolver配置：目前spring提供的有27个，2个重复类但不同作用。
     attrMethod.setHandlerMethodArgumentResolvers(this.argumentResolvers);
   }
   // 构造函数或者方法声明位置的参数名获取。
   attrMethod.setParameterNameDiscoverer(this.parameterNameDiscoverer);
   // 不同于WebDataBinderFactory构建中，对方法的包装类提供的DefaultDataBinderFactory，这里提供的是其子类链中的ServletRequestDataBinderFactory。
   // 上述描述意味着，@ModelAttribute标记方法的参数，将会经由@InitBinder标记方法的处理，将享受与@RequestMapping标记方法的参数的意向数据转换的同等待遇。
   // 当然，无需担心什么幂等性问题，这个不存在，@InitBinder的存在是提供对WebDataBinder的处理，每次数据转换都会新建一个，在该对象基础上进行操作。
   attrMethod.setDataBinderFactory(factory);
   return attrMethod;
}
```

```java
/*
 * this.modelMethods：用ModelMethod封装，之前用InvocableHandlerMethod封装@ModelAttribute标记方法的对象。提供dependencies依赖数据。
 * this.dataBinderFactory：绑定DataBinder。
 * this.sessionAttributesHandler：这个记录着本次访问handlerMethod所在的Controller所要求Model绑定request.session的匹配，以及提供访问request.session属性的能力。
 */
public ModelFactory(@Nullable List<InvocableHandlerMethod> handlerMethods,
      WebDataBinderFactory binderFactory, SessionAttributesHandler attributeHandler) {

   if (handlerMethods != null) {
      // ModelMethod对InvocableHandlerMethod的封装，解析参数列表要求mavc提供attributes的名字集合。
      for (InvocableHandlerMethod handlerMethod : handlerMethods) {
         this.modelMethods.add(new ModelMethod(handlerMethod));
      }
   }
   this.dataBinderFactory = binderFactory;
   this.sessionAttributesHandler = attributeHandler;
}
```

```java
public ModelMethod(InvocableHandlerMethod handlerMethod) {
   this.handlerMethod = handlerMethod;
   for (MethodParameter parameter : handlerMethod.getMethodParameters()) {
      // dependencies记录了参数列表中@ModelAttribute要求的名字的集合，这用来给定@ModelAttribute标记方法启动的相对优先级。优先级：① dependencies全部符合；② 存储顺序——@Order提供。两个条件共用，前者优先于后者。
      if (parameter.hasParameterAnnotation(ModelAttribute.class)) {
         this.dependencies.add(getNameForParameter(parameter));
      }
   }
}
```

涉及：

[ParameterNameDiscoverer学习](./ParameterNameDiscoverer学习.md)



## ModelFactory.initModel

​	填充Model。

```java
// 主要目标：利用@ModelAttribute和@ModelAttributes将mavc绑定request的属性，并给予数据copy。
// 绑定的行为基本是mavc未有指定属性的情况下绑定（argument-resolver中采取的是覆盖）。采取这个策略是有原因的，避免重复的添加/覆盖。这是由于以下讲述的原因：
// sessionAttributesHandler是从RequestMappingHandlerAdapter中缓存的，是跨Request。该对象内的knownAttributeNames的存储是动态增加的，会从第三块填充位置——handlerMethod的标记有@ModelAttribute的parameter中拿取，因此在历史数据存储的情况下，存在重复绑定的时候。当然，命名冲突了也是一个原因。
public void initModel(NativeWebRequest request, ModelAndViewContainer container, HandlerMethod handlerMethod)
      throws Exception {

   // 第一块填充位置。
   // Retrieve "known" session attributes listed as @SessionAttributes
   // 本次请求方法所在的类上提供的@SessionAttributes指明的属性名，将它们从本次request中提取出来。对指名却不存在的情况不敏感。
   Map<String, ?> sessionAttributes = this.sessionAttributesHandler.retrieveAttributes(request);
   container.mergeAttributes(sessionAttributes);
   // 第二块填充位置。
   // @ModelAttribute标记方法执行过程中提供：arguement-resolver提供 + 方法内绑定操作 + returnValue-resolver。
   invokeModelAttributeMethods(request, container);

   // 第三块填充位置。
   // 从handlerMethod的参数列表进行@ModelAttribute解析，返回每个注解位置参数应当绑定session中的哪些属性。
   for (String name : findSessionAttributeArguments(handlerMethod)) {
      if (!container.containsAttribute(name)) {
         Object value = this.sessionAttributesHandler.retrieveAttribute(request, name);
         if (value == null) {
            throw new HttpSessionRequiredException("Expected session attribute '" + name + "'", name);
         }
         container.addAttribute(name, value);
      }
   }
}
```

```java
private void invokeModelAttributeMethods(NativeWebRequest request, ModelAndViewContainer container)
      throws Exception {

   // 激活@ModelAttribute标记的方法，但是激活后会移除。
   // 那parameter.name - returnValue的k-v进行绑定。name一般就取决于@ModelAttribute提供的name。
   while (!this.modelMethods.isEmpty()) {
      // 挑选一个合适的ModelMethod激活。挑选过程：先挑选ModelMethod.dependencies全部在container/Model中存在的先激活，若没有再挑选顺序存储的第一个。每次挑选都会经过上述过程。
      // 挑选中ModelMethod，就会将其删除。
      InvocableHandlerMethod modelMethod = getNextModelMethod(container).getHandlerMethod();
      ModelAttribute ann = modelMethod.getMethodAnnotation(ModelAttribute.class);
      Assert.state(ann != null, "No ModelAttribute annotation");
      // 查看ModelAndView中是否存在@ModelAttribute指定的属性
      if (container.containsAttribute(ann.name())) {
         if (!ann.binding()) {
            // 进制绑定的含义未知晓。
            container.setBindingDisabled(ann.name());
         }
         continue;
      }

      // 进入调用@ModelAttribute标记方法的过程。
      Object returnValue = modelMethod.invokeForRequest(request, container);
      // 当方法返回值不存在时，直接进行下一个。
      if (modelMethod.isVoid()) {
         if (StringUtils.hasText(ann.value())) {
            if (logger.isDebugEnabled()) {
               logger.debug("Name in @ModelAttribute is ignored because method returns void: " +
                     modelMethod.getShortLogMessage());
            }
         }
         continue;
      }

      // 对于返回类型的名字的定义：@ModelAttribute提供的value/name属性；类名的驼峰格式化（首个大写字母小写化）。前者提供的情况下，使用注解的优先。
      String returnValueName = getNameForReturnValue(returnValue, modelMethod.getReturnType());
      // 话说这个非绑定到底是什么意思？
      if (!ann.binding()) {
         container.setBindingDisabled(returnValueName);
      }
      if (!container.containsAttribute(returnValueName)) {
         container.addAttribute(returnValueName, returnValue);
      }
   }
}
```

```java
private List<String> findSessionAttributeArguments(HandlerMethod handlerMethod) {
   List<String> result = new ArrayList<>();
   for (MethodParameter parameter : handlerMethod.getMethodParameters()) {
      if (parameter.hasParameterAnnotation(ModelAttribute.class)) {
         // 如同getNameForReturnValue(.)，一般都是从@ModelAttribute获取返回值。
         String name = getNameForParameter(parameter);
         Class<?> paramType = parameter.getParameterType();
         // this.sessionAttributesHandler中存储有从@SessionAttributes注释提供的绑定信息要求，当
         if (this.sessionAttributesHandler.isHandlerSessionAttribute(name, paramType)) {
            result.add(name);
         }
      }
   }
   return result;
}
```

 
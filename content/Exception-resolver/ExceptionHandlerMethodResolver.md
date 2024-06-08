## 	类的简单介绍

```java
// 对给定类(ControllerAdvice或Controller类)的@ExceptionHandler使用进行解析，提供resolveMethod(ex)匹配，并返回适配的方法。适配原则在最后有表明。
```

## 字段

```java
   /**
    * A filter for selecting {@code @ExceptionHandler} methods.
    * 提供用来过滤得到给定Class中标记有@ExceptionHandler的method。
    */
   public static final MethodFilter EXCEPTION_HANDLER_METHODS = method ->
         AnnotatedElementUtils.hasAnnotation(method, ExceptionHandler.class);

   // this.noMatchingExceptionHandler反射获取method，不过目前看来没什么用。
   private static final Method NO_MATCHING_EXCEPTION_HANDLER_METHOD;

   static {
      try {
         NO_MATCHING_EXCEPTION_HANDLER_METHOD =
               ExceptionHandlerMethodResolver.class.getDeclaredMethod("noMatchingExceptionHandler");
      }
      catch (NoSuchMethodException ex) {
         throw new IllegalStateException("Expected method not found: " + ex);
      }
   }


   // @ControllerAdvice+@ExceptionHandler提供的Method的解析结果。
   private final Map<Class<? extends Throwable>, Method> mappedMethods = new HashMap<>(16);

   // 历史使用过程中，抛出异常在mappedMethods中找到的适配method。潜在的前提：抛出异常所在的Controller-method受@ControllerAdvice指定支持。即一样的异常在不同@ControllerAdvice受理类，有不同的处理方式。
   private final Map<Class<? extends Throwable>, Method> exceptionLookupCache = new ConcurrentReferenceHashMap<>(16);


```

## 方法

**主要从resolveMethod开始看起**，这是外部调用的入口。

### **ExceptionHandlerMethodResolver**

```java
   // 对@ControllerAdvice指定类进行@ExceptionHandler解析。显然，当解析结果中mappedMethods为空，则必然不用handlerType，也就是不用当前构造对象。
   public ExceptionHandlerMethodResolver(Class<?> handlerType) {
      for (Method method : MethodIntrospector.selectMethods(handlerType, EXCEPTION_HANDLER_METHODS)) {
         for (Class<? extends Throwable> exceptionType : detectExceptionMappings(method)) {
            addExceptionMapping(exceptionType, method);
         }
      }
   }
```

### **detectExceptionMappings**

```java
   // @ExceptionHandler使用时提供的异常，有两种情况：注解提供的异常，方法的异常参数。前者有了后者就不用，后者相当于时默认值，若两种都没有则需要抛出@ExceptionHandler使用异常。
   @SuppressWarnings("unchecked")
   private List<Class<? extends Throwable>> detectExceptionMappings(Method method) {
      List<Class<? extends Throwable>> result = new ArrayList<>();
      detectAnnotationExceptionMappings(method, result);
      // 若@ExceptionHandler不指定异常，拿其标记的方法的异常参数作为默认选中异常，不过如此并不是很推荐。对了，如此，那么异常参数如何传递给正确方法的参数呢，呃呃，又是个问题，唉。
      if (result.isEmpty()) {
         for (Class<?> paramType : method.getParameterTypes()) {
            if (Throwable.class.isAssignableFrom(paramType)) {
               result.add((Class<? extends Throwable>) paramType);
            }
         }
      }
      if (result.isEmpty()) {
         throw new IllegalStateException("No exception types mapped to " + method);
      }
      return result;
   }
```

### **detectAnnotationExceptionMappings**

```java
// 这个应该设计成static方法啊，用来指定容器，收集@ExceptionHandler指定的异常。好家伙，原来@ExceptionHandler能指定异常集合。
private void detectAnnotationExceptionMappings(Method method, List<Class<? extends Throwable>> result) {
    ExceptionHandler ann = AnnotatedElementUtils.findMergedAnnotation(method, ExceptionHandler.class);
    Assert.state(ann != null, "No ExceptionHandler annotation");
    result.addAll(Arrays.asList(ann.value()));
}
```

### **addExceptionMapping**

```java
// springMVC初始化时解析@ControllerAdvice+@ExceptionHandler时用到的。
private void addExceptionMapping(Class<? extends Throwable> exceptionType, Method method) {
    Method oldMethod = this.mappedMethods.put(exceptionType, method);
    if (oldMethod != null && !oldMethod.equals(method)) {
        throw new IllegalStateException("Ambiguous @ExceptionHandler method mapped for [" +
                                        exceptionType + "]: {" + oldMethod + ", " + method + "}");
    }
}
```

### **hasExceptionMappings**

```java
// 看是否有@ControllerAdvice+@ExceptionHandler编程method提供。
public boolean hasExceptionMappings() {
    return !this.mappedMethods.isEmpty();
}
```

### **resolveMethod**

```java
   // 使用入口。获取与exception相匹配的Method。先考虑@ExceptionHandler指定异常在抛出异常exception的继承链路上，若都没有则考虑@ExceptionHandler指定异常在抛出异常exception深层异常(cause异常)的继承链路上。同一链路上，看继承关系上和异常exception的距离。
   @Nullable
   public Method (Exception exception) {
      return resolveMethodByThrowable(exception);
   }
```

### **resolveMethodByThrowable**

```java
   // 递归形式设计
   @Nullable
   public Method resolveMethodByThrowable(Throwable exception) {
      Method method = resolveMethodByExceptionType(exception.getClass());
      if (method == null) {
         // 本身异常找不到Method，那么找深层异常的匹配的Method。
         Throwable cause = exception.getCause();
         if (cause != null) {
            method = resolveMethodByThrowable(cause);
         }
      }
      // 出口情况：method == null && no-cause-class || method != null
      return method;
   }
```

### **resolveMethodByExceptionType**

```java
   @Nullable
   public Method resolveMethodByExceptionType(Class<? extends Throwable> exceptionType) {
      // exceptionLookupCache是历史记录吗？
      Method method = this.exceptionLookupCache.get(exceptionType);
      if (method == null) {
         // 搜索不动产（@ControllerAdvice+@ExceptionHandler解析结果）数据
         method = getMappedMethod(exceptionType);
         // 看样子是极了。
         this.exceptionLookupCache.put(exceptionType, method);
      }
      return (method != NO_MATCHING_EXCEPTION_HANDLER_METHOD ? method : null);
   }
```

### **getMappedMethod**

```java
   /**
    * Return the {@link Method} mapped to the given exception type, or
    * {@link #NO_MATCHING_EXCEPTION_HANDLER_METHOD} if none.
    */
   private Method getMappedMethod(Class<? extends Throwable> exceptionType) {
      // 收集符合要求的Method的集合
      List<Class<? extends Throwable>> matches = new ArrayList<>();
      for (Class<? extends Throwable> mappedException : this.mappedMethods.keySet()) {
         // 看来@ExceptionHandler给定异常在抛出异常exceptionType的继承链上的也会考虑。
         if (mappedException.isAssignableFrom(exceptionType)) {
            matches.add(mappedException);
         }
      }
      if (!matches.isEmpty()) {
         if (matches.size() > 1) {
            // ExceptionDepthComparator提供的优先逻辑：两个比较异常e1、e2，计算它们距离坐标异常exceptionType的距离（距离指的是继承的距离，直接的父类是1，本身是0），距离越小的优先级越高，序列越靠前。不在继承链上的给定优先级Integer.MAX_VALUE。
            matches.sort(new ExceptionDepthComparator(exceptionType));
         }
         return this.mappedMethods.get(matches.get(0));
      }
      else {
         // 没有匹配结果，返回本类给定的方法————无用、空方法。
         return NO_MATCHING_EXCEPTION_HANDLER_METHOD;
      }
   }
```

### **noMatchingExceptionHandler**

```java
   /**
    * For the {@link #NO_MATCHING_EXCEPTION_HANDLER_METHOD} constant.
    */
   @SuppressWarnings("unused")
   private void noMatchingExceptionHandler() {
   }
```



## @ControllerAdvice+@ExceptionHandler使用技巧和注意事项

**演示代码**

```java
@ControllerAdvice(
        basePackages = "com.example.demo"
)
public class DemoControllerAdvice {

    @ExceptionHandler(RuntimeException.class)
    public String commonHandler(RuntimeException re){
        System.out.println("look");
        return "commonHandler";
    }

    @ExceptionHandler
    public String takeParameter(SubException se){
        System.out.println("look");
        return "takeParameter";
    }

    @ExceptionHandler
    public String totalParamForm(UnsupportedOperationException ue,
                                 HandlerMethod handlerMethod,
                                 ServletRequest servletRequest,
                                 ServletResponse servletResponse){
        System.out.println("look");
        return "totalParamForm";
    }
}
```

### @ExceptionHandler整体的异常提供

ExceptionHandlerExceptionResolver解析@ExceptionHandler标记的异常有两种提取方式：注解中给定；方法参数中提取异常参数。前者若给定了，后者不提取；前者未给，向后者提取。若两种方式都没提取到异常，会在SpringMVC初始化时刻，报异常。

### ControllerAdvice类内显示异常要求

@ExceptionHandler标记位之间提取的异常，必须不同、没有异常类交集。不然也会在编译期间抛出异常。因此，要提取异常类的cause，是否有必要将其作为参数，让resolver注入。

### @ExceptionHandler指定异常建议

不要用向上造型来接收具体异常，想接收什么类型就用什么类型接收。ExceptionHandlerExceptionResolver挑选@ExceptionHandler，是看谁持有最接近抛出异常的异常类。

### ControllerAdvice设计建议

dispatch过程中抛出的异常（一般都是request-method中跑出来的），只会挑选一个@ControllerAdvice标记类作为Advice处理，因此没有必要让@ControllerAdvice之间存在作用域交集。不过可以有一个托底的Advice。

托底Advice。@Order提供的优先级可以作为Advice处理异常的先后顺序依据，数值越小，优先级越大，没提供@Order则视为最低优先级。@ControllerAdvice若未提供范围，则视为全局支持。托底Advice一般就是无@Order，无范围选择。
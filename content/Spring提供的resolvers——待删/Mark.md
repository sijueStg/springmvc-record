# 不明朗

## RedirectAttributesMethodArgumentResolver

```java
// method-parameter是RedirectAttributes的继承/实现链上
// Model的特制接口，重定向时attributes的提供。
public boolean supportsParameter(MethodParameter parameter) {
   return RedirectAttributes.class.isAssignableFrom(parameter.getParameterType());
}
```

```java
public Object resolveArgument(MethodParameter parameter, @Nullable ModelAndViewContainer mavContainer,
      NativeWebRequest webRequest, @Nullable WebDataBinderFactory binderFactory) throws Exception {

   Assert.state(mavContainer != null, "RedirectAttributes argument only supported on regular handler methods");

   ModelMap redirectAttributes;
   if (binderFactory != null) {
      // 这个要学的，但是现在还不是时候
      DataBinder dataBinder = binderFactory.createBinder(webRequest, null, DataBinder.DEFAULT_OBJECT_NAME);
      redirectAttributes = new RedirectAttributesModelMap(dataBinder);
   }
   else {
      redirectAttributes  = new RedirectAttributesModelMap();
   }
   mavContainer.setRedirectModel(redirectAttributes);
   return redirectAttributes;
}
```



## MapMethodProcessor

```java
// 受此resolver支持，method parameter代表的参数的签名必须如此：
// Map.class，没有任何注解使用
public boolean supportsParameter(MethodParameter parameter) {
   return (Map.class.isAssignableFrom(parameter.getParameterType()) &&
         parameter.getParameterAnnotations().length == 0);
}
```

```java
public Object resolveArgument(MethodParameter parameter, @Nullable ModelAndViewContainer mavContainer,
      NativeWebRequest webRequest, @Nullable WebDataBinderFactory binderFactory) throws Exception {

   Assert.state(mavContainer != null, "ModelAndViewContainer is required for model exposure");
   return mavContainer.getModel();
}
```

## ModelMethodProcessor

```java
public boolean supportsParameter(MethodParameter parameter) {
   return Model.class.isAssignableFrom(parameter.getParameterType());
}
```

```java
public Object resolveArgument(MethodParameter parameter, @Nullable ModelAndViewContainer mavContainer,
      NativeWebRequest webRequest, @Nullable WebDataBinderFactory binderFactory) throws Exception {

   Assert.state(mavContainer != null, "ModelAndViewContainer is required for model exposure");
   return mavContainer.getModel();
}
```




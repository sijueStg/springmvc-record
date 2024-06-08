## 前言

​	目前在RequestMappingHandlerAdapter中只有两个类在用：LocalVariableTableParameterNameDiscoverer和StandardReflectionParameterNameDiscoverer。这里就介绍这两个类即可。

## ParameterNameDiscoverer

```java
// 提供方法或构造器的参数名称——声明的参数名。
public interface ParameterNameDiscoverer {
	@Nullable
	String[] getParameterNames(Method method);
    @Nullable
	String[] getParameterNames(Constructor<?> ctor);
}
```

## LocalVariableTableParameterNameDiscoverer

​	从缓存中找出匹配名字：

```java
Map<Class<?>, Map<Executable, String[]>> parameterNamesCache
```

涉及了bridgeMethod.isBridge()的判断，不懂就放着吧。

一般方法传入就当作获取null返回值即可。



## StandardReflectionParameterNameDiscoverer

​	标准的通过反射获取。

```java
public class StandardReflectionParameterNameDiscoverer implements ParameterNameDiscoverer {

   @Override
   @Nullable
   public String[] getParameterNames(Method method) {
      return getParameterNames(method.getParameters());
   }

   @Override
   @Nullable
   public String[] getParameterNames(Constructor<?> ctor) {
      return getParameterNames(ctor.getParameters());
   }

   // Parameter描述着参数声明所在位置和自身信息。
   @Nullable
   private String[] getParameterNames(Parameter[] parameters) {
      String[] parameterNames = new String[parameters.length];
      for (int i = 0; i < parameters.length; i++) {
         Parameter param = parameters[i];
         // 不清楚什么场合下，该方法返回false。可能注入函数式接口的这种实现方法不具有吧。。。
         if (!param.isNamePresent()) {
            return null;
         }
         parameterNames[i] = param.getName();
      }
      return parameterNames;
   }

}
```
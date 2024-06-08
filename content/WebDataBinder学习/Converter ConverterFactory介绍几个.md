## 模板

转换：<sourceType, targetType>

convert作用：

## spring提供converter所在包

```txt
org.springframework.core.convert.support
org.springframework.boot.convert
```

## Spring提供Formatter所在包

```txt
org.springframework.format
```



## StringToNumberConverterFactory

提供Converter：StringToNumber

转换：String-> Number

convert作用：

```java
public T convert(String source) {
    if (source.isEmpty()) {
        return null;
    }
    // targetType可以指定在：[Byte、Short、Integer、Long、BigInteger、Float、Double、BigDecimal]中。将文本代表的数值转换成具体的Number子类。
    return NumberUtils.parseNumber(source, this.targetType);
}
```

看样子太多了，主要图的就是类型转换标准。

## ObjectToStringConverter

转换：Object -> String

convert：

```java
public String convert(Object source) {
    return source.toString();
}
```

用例：

```java
converterRegistry.addConverter(Number.class, String.class, new ObjectToStringConverter());
```

## IdToEntityConverter

 有趣的设计。

转化：

convert设计：

```java
public Object convert(@Nullable Object source, TypeDescriptor sourceType, TypeDescriptor targetType) {
   if (source == null) {
      return null;
   }
   // 找targetType的单参、方法名为"find+targetType类名"、返回类型为targetType的static方法————实际上就是targeType是提供了构造自己实例的工厂方法。
   Method finder = getFinder(targetType.getType());
   Assert.state(finder != null, "No finder method");
   // 在conversionService中寻找converter，实现sourceType -> findMethod.parameterType。将转换结果对象作为findMethod的参数。有深意的是调用方法的对象是source，也就是说，source是targeType本身或其子类——source.instanceof(targetType)？离谱
   Object id = this.conversionService.convert(
         source, sourceType, TypeDescriptor.valueOf(finder.getParameterTypes()[0]));
   return ReflectionUtils.invokeMethod(finder, source, id);
}
```

新奇点：

- converter应用conversionService自己，谨慎使用，防止获取到自己。
- sourceType -> targetType可以不是直接的转换。



好了，姑且这块内容到此结束，接下来前去看WebBinder在MVC中的应用。
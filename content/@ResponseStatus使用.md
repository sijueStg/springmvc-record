## 参考：

[Spring中@ResponseStatus注解的作用](https://blog.csdn.net/Thinkingcao/article/details/110875494)

## 简述作用位置

evaluateResponseStatus方法是HandlerMethod类的方法，仅会在HandlerMethod的构造函数中使用。其用来解析@ResponseStatus。

```java
// 与@ResponseStatus注解有关。该注解
private void evaluateResponseStatus() {
   ResponseStatus annotation = getMethodAnnotation(ResponseStatus.class);
   if (annotation == null) {
      annotation = AnnotatedElementUtils.findMergedAnnotation(getBeanType(), ResponseStatus.class);
   }
   if (annotation != null) {
      String reason = annotation.reason();
      String resolvedReason = (StringUtils.hasText(reason) && this.messageSource != null ?
            this.messageSource.getMessage(reason, null, reason, LocaleContextHolder.getLocale()) :
            reason);

      this.responseStatus = annotation.code();
      this.responseStatusReason = resolvedReason;
   }
}	
```

HandlerMethod包装反射调用的类方法，比如@InitBinder标记的方法、@RequestMapping标记的方法等。



感觉不好用，@ResponseStatus的存在会干扰mvc的运行，一般还是在@ExceptionHandler提取异常内容好用些。所以暂时不学习。
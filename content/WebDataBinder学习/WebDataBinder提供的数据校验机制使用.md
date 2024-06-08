## 前言

​	Spring提供的WebDataBinder的数据绑定服务是基于Hibernate的validator的二次开发，且只需要关注切面式编程，注重实践即可————主要是若要拓展校验功能，可能需要开发hibernate的java代码，感觉代价不小。

参考：

[@Validated和@Valid区别：Spring validation验证框架对入参实体进行嵌套验证必须在相应属性（字段）加上@Valid而不是@Validated](https://blog.csdn.net/qq_27680317/article/details/79970590)

[SpingBoot项目使用@Validated和@Valid参数校验](https://juejin.cn/post/7213634349233111100#heading-4)

## 校验注解

需要提供依赖：

```xml
<dependency>
   <groupId>org.springframework.boot</groupId>
   <artifactId>spring-boot-starter-validation</artifactId>
</dependency>
```

注解位置：

javax.validation.constraints.*。

应用：只要放在字段上即可。

## @Validated和@Valid

@Validated：用在方法入参上无法单独提供嵌套验证功能。不能用在成员属性（字段）上，也无法提示框架进行嵌套验证。能配合嵌套验证注解@Valid进行嵌套验证。

@Valid：用在方法入参上无法单独提供嵌套验证功能。**能够用在成员属性（字段）上，提示验证框架进行嵌套验证**。能配合嵌套验证注解@Valid进行嵌套验证。

## 分组校验

​	当一个POJO有不同的校验标准时，就不能共用同一套验证内容。因此需要不同验证场景下对POJO内的验证注解挑选支持。分组校验就是提供的规则。

规则：

```java
public @interface NotNull {

   String message() default "{javax.validation.constraints.NotNull.message}";

   Class<?>[] groups() default { };

   Class<? extends Payload>[] payload() default { };

   /**
    * Defines several {@link NotNull} annotations on the same element.
    *
    * @see javax.validation.constraints.NotNull
    */
   @Target({ METHOD, FIELD, ANNOTATION_TYPE, CONSTRUCTOR, PARAMETER, TYPE_USE })
   @Retention(RUNTIME)
   @Documented
   @interface List {

      NotNull[] value();
   }
}
```

每个验证注解都须至少支持如上形式，其中groups就是用来支持分组，看如下图：

<img src="picture/2024-03-03 03_16_20-验证图画草稿.xmind.png" style="zoom:50%;" />

默认分组就是x.validation.groups.Default时，每个验证注解都可以给定任意分组Class集合，比如给定FirstAspect，那么当前组相当于给定的到Default之间的所有组，即FirstAspect == (FirstAspect, ValidationGroup1, Default)。

@Validated提供字段能指明出启用哪些组的校验。



## 嵌套验证

@Valid支持，标记在给定字段上，也解析该字段的内置字段的验证注解。


## 作用

​	就类注释所述，TypeDescriptor提供类型转换服务期间`类型描述单元`，其具有类型使用场景的上下文数据缓存能力，且能针对arrays或generic collection（generic代表泛型）型进行解析。关键是`上下文范围`是多少	

依据构造函数，可见描述类型的三元素：<resolvableType, type, annotatedElement>

- resolvableType：ResolvableType类型数据，这个能兼容很多种类型。感觉还得在学一次，哭了。。。。
- type：Class<?>。
- annotatedElement：这个就是上下文注解了，主要是指定类型声明（如Field、MethodParameter等）位置上的注解。


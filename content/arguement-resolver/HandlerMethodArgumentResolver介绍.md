## HandlerMethodArgumentResolver介绍

​	本接口主要用于从request中解析出数据，并转变成parameter代表的类的实例。因为数据与传输的目的多种多样，resolver也有很多，目前看有21种，本段内容将尽力介绍它们。（任重而道远啊）。

### 大致作用

- PathVariableMapMethodArgumentResolver

- ErrorsMethodArgumentResolver

- AbstractNamedValueMethodArgumentResolver

  将request的named value（注入header、request parameters等）数据解析成request method parameter。

  - RequestHeaderMethodArgumentResolver
  - RequestAttributeMethodArgumentResolver
  - RequestParamMethodArgumentResolver
  - AbstractCookieValueMethodArgumentResolver
    - ServletCookieValueMethodArgumentResolver
  - SessionAttributeMethodArgumentResolver
  - MatrixVariableMethodArgumentResolver
  - ExpressionValueMethodArgumentResolver
  - PathVariableMethodArgumentResolver

- RequestHeaderMapMethodArgumentResolver

- ServletResponseMethodArgumentResolver

- SessionStatusMethodArgumentResolver

- RequestParamMapMethodArgumentResolver

- PrincipalMethodArgumentResolver

- ContinuationHandlerMethodArgumentResolver

- AbstractMessageConverterMethodArgumentResolver

  - RequestPartMethodArgumentResolver

- UriComponentsBuilderMethodArgumentResolver

- ServletRequestMethodArgumentResolver

- RedirectAttributesMethodArgumentResolver

- MatrixVariableMapMethodArgumentResolver


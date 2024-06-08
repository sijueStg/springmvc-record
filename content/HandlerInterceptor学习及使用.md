## 介绍

​	Spring提供HandlerInterceptor接口，为`handler执行、rendering the view`这两个执行点作为三个切点处理：preHandle、postHandle、afterCompletion。

**preHandle**：在handler执行前处理，当返现异常时，则会拦截handler执行。

**postHandle**：当且仅当拦截器通过、handler顺利执行，此处拦截器方法执行，本部分用于专注于ModelAndView处理，向View暴露数据，以供render(mv, request, response)——不过有相当一部分mv返回null，这个不是很清楚。

**afterCompletion**：显然是善后工作方法，资源清理等功能在此执行。



## 拦截器不同时期执行次序

```java
// 以下是拦截器执行次序的简化展现
// mappedHandler是HandlerExecutionChain对象变量，封装着拦截器的执行。
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
    try {
        try{
            if (!mappedHandler.applyPreHandle(processedRequest, response)) 			  {
                return;
            }
        	mv = ha.handle(processedRequest, response, mappedHandler.getHandler());
        	mappedHandler.applyPostHandle(processedRequest, response, mv);
        } 
        catch (e){
            // 异常捕获、记录
        }
        // 异常处理
        render(mv, request, response);
    } catch(e){
        triggerAfterCompletion(processedRequest, response, mappedHandler, ex);
    }
}
```

- 正常情况下：

  HandlerExecutionChain用List分装拦截器HandlerInterceptor，坐标越小，执行优先级越高。

- 存在拦截器拦截

  若是拦截器的拦截确认，则不会执行后续handler执行、拦截器其他方法，直接返回。

- 异常发生：

  HandlerExecutionChain封装有interceptorIndex计数器，其用来表明上一次拦截器执行的坐标。interceptorIndex+1=执行过的拦截器。

  applyPostHandle：

  ```java
  for (int i = this.interceptorList.size() - 1; i >= 0; i--) {...}
  ```

  afterCompletion：

  ```java
  for (int i = this.interceptorIndex; i >= 0; i--) {...}
  ```

  - applyPreHandle执行异常

    一般不会。执行已经执行过applyPreHandle方法的拦截器的triggerAfterCompletion。

  - handler执行异常、applyPostHandle执行异常、render异常。

    由于拦截器未拦截，所有拦截器都执行过applyPreHandle，因此它们都执行afterCompletion。

## 拦截器的注册

文章：[Spring的WebMvcConfiguration设计](Spring的WebMvcConfiguration设计.md)

> 提供的文章介绍了为什么配置一个bean化的WebMvcConfigurer实现类，即可提供。

```java
@Component
public class WebRequestInterceptorConfig implements WebMvcConfigurer {

    @Resource
    private List<HandlerInterceptor> handlerInterceptorList;

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        // no need order排序，注册器完全拉去后，向外提供会做该处理。
//        handlerInterceptorList.sort(Comparator.comparingInt(pre -> operateNull(OrderUtils.getOrder(pre.getClass()))));
//        getOrderedInterceptorRegistry(handlerInterceptorList);
        handlerInterceptorList.forEach(registry::addInterceptor);

    }

    private void getOrderedInterceptorRegistry(List<HandlerInterceptor> interceptors){
        interceptors.sort(AnnotationAwareOrderComparator.INSTANCE);
    }

    private Integer operateNull(Integer order){
        return order==null ? Integer.MAX_VALUE : order;
    }
}
```

由@Order提供，提供数值越小，优先级越高，未提供则认为优先级末位。applyPreHandle：大 → 小/前 → 后；applyPostHandle、triggerAfterCompletion：小 → 大 / 后 → 前。



## 深入了解

​	后续就是对于方法的实际场景使用了，这个考验实战，我经验不足，以后填充。


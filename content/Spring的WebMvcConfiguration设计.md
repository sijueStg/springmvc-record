## 前言

​	这里介绍Spring提供的WebMvcConfiguration设计，其巧妙地将`配置加载`的过程分离成`配置环境`和`注册器`两个组成部分。配置环境提供配置；注册器向配置环境索取配置。

​	配置环境和注册器两者的组合依赖于DelegatingWebMvcConfiguration与WebMvcConfigurerComposite的使用。前者应用后者，缓存配置环境；后者应用WebMvcConfigurer，为配置器提供配置。

​	WebMvcConfigurer是各种注册器拉取配置的环境，配置环境理应是不变的，全局环境中WebMvcConfigurer集合，将为空的注册器提供完整的配置。

​	上述的类关系，在[Spring的WebMvcConfiguration设计](./materials/Spring的WebMvcConfiguration设计.xmind)文档中清晰地关联起来。

## WebMvcConfigurer加载和应用

​	WebMvcConfigurer是个接口，为各种注册器提供配置，将实现类加载到WebMvcConfigurerComposite，只需要将该类bean化即可。同时，这里将介绍DelegatingWebMvcConfiguration和WebMvcConfigurerComposite的来历和使用。

### **WebMvcConfigurer加载**

```java
@Configuration(proxyBeanMethods = false)
public class DelegatingWebMvcConfiguration extends WebMvcConfigurationSupport {

   private final WebMvcConfigurerComposite configurers = new WebMvcConfigurerComposite();

   // @Autowired标记在方法上，该方法将在本类bean实例化期间调用，在构造函数和setter之间时刻调用。其看似是构造函数和setter作用的中庸，不想让字段公开，但是字段初始化依赖于外部。因此该类型method，不推荐bean化以外的时间段调用。
   @Autowired(required = false)
   // 将bean化的WebMvcConfigurer实现类加入到this.configurers。
   public void setConfigurers(List<WebMvcConfigurer> configurers) {
      if (!CollectionUtils.isEmpty(configurers)) {
         this.configurers.addWebMvcConfigurers(configurers);
      }
   }
   ...............
}
```

### **注册器拉取配置**

​	WebMvcConfigurationSupport是DelegatingWebMvcConfiguration的父类，其提供诸多使用指定注册器的应用者组件，这些组件将应用到MVC架构中承担相应的责任。

```java
public class WebMvcConfigurationSupport implements ApplicationContextAware, ServletContextAware {
	// 获取得到完整配置的注册器。hook接口，留给子类实现。
    // 为了节约空间，将共同的protected void去除，{}也去除。
    // 拦截器-这个是本次展开讨论的点，其他都没有遇见过。
    addInterceptors(InterceptorRegistry registry)
    // 路径匹配器？
    configurePathMatch(PathMatchConfigurer configurer)
    configureContentNegotiation(ContentNegotiationConfigurer configurer)
    addViewControllers(ViewControllerRegistry registry)
    addResourceHandlers(ResourceHandlerRegistry registry) 
    configureDefaultServletHandling(DefaultServletHandlerConfigurer configurer)
    addFormatters(FormatterRegistry registry)
    addArgumentResolvers(List<HandlerMethodArgumentResolver> argumentResolvers)
    addReturnValueHandlers(List<HandlerMethodReturnValueHandler> returnValueHandlers)
    configureMessageConverters(List<HttpMessageConverter<?>> converters)
    extendMessageConverters(List<HttpMessageConverter<?>> converters)
    configureAsyncSupport(AsyncSupportConfigurer configurer)
    configureHandlerExceptionResolvers(List<HandlerExceptionResolver> exceptionResolvers)
    extendHandlerExceptionResolvers(List<HandlerExceptionResolver> exceptionResolvers)
    configureViewResolvers(ViewResolverRegistry registry)
    addCorsMappings(CorsRegistry registry)
}
```

**DelegatingWebMvcConfiguration实现**

```java
@Configuration(proxyBeanMethods = false)
public class DelegatingWebMvcConfiguration extends WebMvcConfigurationSupport {

   private final WebMvcConfigurerComposite configurers = new WebMvcConfigurerComposite();
    
   @Override
	protected void addInterceptors(InterceptorRegistry registry) {
		this.configurers.addInterceptors(registry);
	}
   .......................
}
```

**WebMvcConfigurerComposite关联`配置环境`和`注册器`**

```java
@Override
public void addInterceptors(InterceptorRegistry registry) {
   for (WebMvcConfigurer delegate : this.delegates) {
      // 注册器前往配置环境拉取数据。
      delegate.addInterceptors(registry);
   }
}
```



## 注册器应用者组件

​	WebMvcConfigurationSupport提供了hook，自然也要用到，除了外部调用，子本身提供了一些组件（本人不喜欢如此紧密，怕设计不周，出现意外）。

```java

```



## 组件学习

### RequestMappingHandlerMapping

WebMvcConfigurationSupport内提供了一些组件，比如RequestMappingHandlerMapping，这里记录它们的创建和初始化。

```java
// 学到了，居然可以让子类做@Configuration，父类方法@Bean。配置具有类的穿透性检查。
public class WebMvcConfigurationSupport implements ApplicationContextAware, ServletContextAware {
	
    // 提供了一种HandlerMapping
    @Bean
	public RequestMappingHandlerMapping requestMappingHandlerMapping(
			@Qualifier("mvcContentNegotiationManager") ContentNegotiationManager contentNegotiationManager,
			@Qualifier("mvcConversionService") FormattingConversionService conversionService,
			@Qualifier("mvcResourceUrlProvider") ResourceUrlProvider resourceUrlProvider) {

		RequestMappingHandlerMapping mapping = createRequestMappingHandlerMapping();
		mapping.setOrder(0);
         // 添加拦截器。
		mapping.setInterceptors(getInterceptors(conversionService, resourceUrlProvider));
		mapping.setContentNegotiationManager(contentNegotiationManager);
		mapping.setCorsConfigurations(getCorsConfigurations());

		PathMatchConfigurer pathConfig = getPathMatchConfigurer();
		if (pathConfig.getPatternParser() != null) {
			mapping.setPatternParser(pathConfig.getPatternParser());
		}
		else {
			mapping.setUrlPathHelper(pathConfig.getUrlPathHelperOrDefault());
			mapping.setPathMatcher(pathConfig.getPathMatcherOrDefault());

			Boolean useSuffixPatternMatch = pathConfig.isUseSuffixPatternMatch();
			if (useSuffixPatternMatch != null) {
				mapping.setUseSuffixPatternMatch(useSuffixPatternMatch);
			}
			Boolean useRegisteredSuffixPatternMatch = pathConfig.isUseRegisteredSuffixPatternMatch();
			if (useRegisteredSuffixPatternMatch != null) {
				mapping.setUseRegisteredSuffixPatternMatch(useRegisteredSuffixPatternMatch);
			}
		}
		Boolean useTrailingSlashMatch = pathConfig.isUseTrailingSlashMatch();
		if (useTrailingSlashMatch != null) {
			mapping.setUseTrailingSlashMatch(useTrailingSlashMatch);
		}
		if (pathConfig.getPathPrefixes() != null) {
			mapping.setPathPrefixes(pathConfig.getPathPrefixes());
		}

		return mapping;
	}
    
    // 拦截器缓存唯一且不会重复拉取配置。
    protected final Object[] getInterceptors(
			FormattingConversionService mvcConversionService,
			ResourceUrlProvider mvcResourceUrlProvider) {

		if (this.interceptors == null) {
			InterceptorRegistry registry = new InterceptorRegistry();
             // hook接口，拉取registry从配置环境中拉去了全部的配置。
			addInterceptors(registry);
             // 添加了两个特殊的拦截器。
			registry.addInterceptor(new ConversionServiceExposingInterceptor(mvcConversionService));
			registry.addInterceptor(new ResourceUrlProviderExposingInterceptor(mvcResourceUrlProvider));
             // 获取拦截器，每个拦截器都用InterceptorRegistration包装，不过返回的时候用了mappedInterceptor包装，并进行了一定的关联配置处理。
			this.interceptors = registry.getInterceptors();
		}
		return this.interceptors.toArray();
	}
    .......................
}
```

RequestMappingHandlerMapping的继承链路上抽象类AbstractHandlerMethodMapping实现InitializingBean接口，

AbstractHandlerMethodMapping.afterPropertiesSet()

​	这里面提供了this.corsLookup、this.pathLookUp、this.nameLookUp、this.registry的初始化。

```java
public void afterPropertiesSet() {
   initHandlerMethods();
}
```

```java
protected void initHandlerMethods() {
   for (String beanName : getCandidateBeanNames()) {
      if (!beanName.startsWith(SCOPED_TARGET_NAME_PREFIX)) {
         // 挑选合适的bean做处理。
         processCandidateBean(beanName);
      }
   }
   // 只是打了个日志，没做什么处理。
   handlerMethodsInitialized(getHandlerMethods());
}
```

```java
// this.detectHandlerMethodsInAncestorContexts==false，执行后者。
// SpringBoot中提供AnnotationConfigServletWebServerApplicationContext环境中DefaultListableBeanFactory提供的beanType是Object.class的bean instances的name数组。
// Object.class显然是包含了所有的bean。bean会根据其继承、实现链路上的接口和实现类作为相关容器的唯一性来存储，即bean将会存储在k-接口、k-父类、k-父类的接口等容器上，spring这点也很奈斯。
protected String[] getCandidateBeanNames() {
   return (this.detectHandlerMethodsInAncestorContexts ?
         BeanFactoryUtils.beanNamesForTypeIncludingAncestors(obtainApplicationContext(), Object.class) :
         obtainApplicationContext().getBeanNamesForType(Object.class));
}
```

```java
protected void processCandidateBean(String beanName) {
   Class<?> beanType = null;
   try {
      // 依据beanName获取相关bean instance的beanType（本身类型）
      beanType = obtainApplicationContext().getType(beanName);
   }
   catch (Throwable ex) {
      // An unresolvable bean type, probably from a lazy bean - let's ignore it.
      if (logger.isTraceEnabled()) {
         logger.trace("Could not resolve type for bean '" + beanName + "'", ex);
      }
   }
   // isHandler的意思是：要求beanType被@Controller和@RequestMapping标记，即beanType是Controller类。
   if (beanType != null && isHandler(beanType)) {
      detectHandlerMethods(beanName);
   }
}
```

解析handler：将url与handler-method提取适配。

```java
// 目前也只有上面那个能调用它
protected void detectHandlerMethods(Object handler) {
   // 若handler是个非String对象，则将其直接视为Controller类。
   Class<?> handlerType = (handler instanceof String ?
         obtainApplicationContext().getType((String) handler) : handler.getClass());

   if (handlerType != null) {
      // 处理CGLIB问题。
      Class<?> userType = ClassUtils.getUserClass(handlerType);
      // 实际就是提供<Method, RequestMappingInfo>，RequestMappingInfo描述handler-method的信息。
      Map<Method, T> methods = MethodIntrospector.selectMethods(userType,
            (MethodIntrospector.MetadataLookup<T>) method -> {
               try {
                  // 依据<handler-method, handlerTye>获取RequestMappingInfo
                  return getMappingForMethod(method, userType);
               }
               catch (Throwable ex) {
                  throw new IllegalStateException("Invalid mapping on handler class [" +
                        userType.getName() + "]: " + method, ex);
               }
            });
      if (logger.isTraceEnabled()) {
         logger.trace(formatMappings(userType, methods));
      }
      else if (mappingsLogger.isDebugEnabled()) {
         mappingsLogger.debug(formatMappings(userType, methods));
      }
      methods.forEach((method, mapping) -> {
         Method invocableMethod = AopUtils.selectInvocableMethod(method, userType);
         // 向this.mappingRegistry注册<handlerName, handler-method, RequestMappingInfo>
         registerHandlerMethod(handler, invocableMethod, mapping);
      });
   }
}
```

```java
protected void registerHandlerMethod(Object handler, Method method, T mapping) {
   this.mappingRegistry.register(mapping, handler, method);
}
```

```java
......................
public void register(T mapping, Object handler, Method method) {
   this.readWriteLock.writeLock().lock();
   try {
      // HandlerMethod(handlerName, handler-method)
      HandlerMethod handlerMethod = createHandlerMethod(handler, method);
      validateMethodMapping(handlerMethod, mapping);

      Set<String> directPaths = AbstractHandlerMethodMapping.this.getDirectPaths(mapping);
      // this.pathLookup - <handler-method-path/url, List<RequestMappingInfo>>
      for (String path : directPaths) {
         this.pathLookup.add(path, mapping);
      }

      String name = null;
      if (getNamingStrategy() != null) {
         // 这个策略就是handlerName提取每个大写字母 + “#” + handler-method-name，其用来给HandlerMethod命名，显然这种命名不能保证唯一。
         name = getNamingStrategy().getName(handlerMethod, mapping);
         // this.nameLookup - <handler-method-new-name, List<HandlerMethod>
         addMappingName(name, handlerMethod);
      }

      // 未知。
      CorsConfiguration corsConfig = initCorsConfiguration(handler, method, mapping);
      if (corsConfig != null) {
         corsConfig.validateAllowCredentials();
         this.corsLookup.put(handlerMethod, corsConfig);
      }

      // 提供this.registry - <RequestMappingInfo, MappingRegistration(.....)>
      this.registry.put(mapping,
            new MappingRegistration<>(mapping, handlerMethod, directPaths, name, corsConfig != null));
   }
   finally {
      this.readWriteLock.writeLock().unlock();
   }
}
......................
```



### RequestMappingHandlerAdapter

​	WebMvcConfigurationSupport -> DelegatingWebMvcConfiguration -> EnableWebMvcConfiguration这是一个连续的继承链路，RequestMappingHandlerAdapter的bean化就在EnableWebMvcConfiguration配置类中。

关注点：

- this.argumentResolvers
- this.returnValueHandlers
- this.parameterNameDiscoverer
- this.ignoreDefaultModelOnRedirect
- this.asyncRequestTimeout
- this.taskExecutor
- this.callableInterceptors
- this.deferredResultInterceptors

```java
@AutoConfiguration(after = { DispatcherServletAutoConfiguration.class, TaskExecutionAutoConfiguration.class,
      ValidationAutoConfiguration.class })
@ConditionalOnWebApplication(type = Type.SERVLET)
@ConditionalOnClass({ Servlet.class, DispatcherServlet.class, WebMvcConfigurer.class })
@ConditionalOnMissingBean(WebMvcConfigurationSupport.class)
@AutoConfigureOrder(Ordered.HIGHEST_PRECEDENCE + 10)
public class WebMvcAutoConfiguration {

	/**
	 * Configuration equivalent to {@code @EnableWebMvc}.
	 */
	@Configuration(proxyBeanMethods = false)
	@EnableConfigurationProperties(WebProperties.class)
	public static class EnableWebMvcConfiguration extends DelegatingWebMvcConfiguration implements ResourceLoaderAware {

		private final Resources resourceProperties;

		private final WebMvcProperties mvcProperties;

		private final WebProperties webProperties;

		private final ListableBeanFactory beanFactory;

		private final WebMvcRegistrations mvcRegistrations;

		private ResourceLoader resourceLoader;

		public EnableWebMvcConfiguration(WebMvcProperties mvcProperties, WebProperties webProperties,
				ObjectProvider<WebMvcRegistrations> mvcRegistrationsProvider,
				ObjectProvider<ResourceHandlerRegistrationCustomizer> resourceHandlerRegistrationCustomizerProvider,
				ListableBeanFactory beanFactory) {
			this.resourceProperties = webProperties.getResources();
			this.mvcProperties = mvcProperties;
			this.webProperties = webProperties;
			this.mvcRegistrations = mvcRegistrationsProvider.getIfUnique();
			this.beanFactory = beanFactory;
		}

         // 关注点：
		@Bean
		@Override
		public RequestMappingHandlerAdapter requestMappingHandlerAdapter(
				@Qualifier("mvcContentNegotiationManager") ContentNegotiationManager contentNegotiationManager,
				@Qualifier("mvcConversionService") FormattingConversionService conversionService,
				@Qualifier("mvcValidator") Validator validator) {
             // 离谱了，拿父类的来获得一个instance先。
			RequestMappingHandlerAdapter adapter = super.requestMappingHandlerAdapter(contentNegotiationManager,
					conversionService, validator);
			adapter.setIgnoreDefaultModelOnRedirect(
					this.mvcProperties == null || this.mvcProperties.isIgnoreDefaultModelOnRedirect());
			return adapter;
		}
    }
}
```

**WebMvcConfigurationSupport**

```java
public class WebMvcConfigurationSupport implements ApplicationContextAware, ServletContextAware {
	.....................
	@Bean
	public RequestMappingHandlerAdapter requestMappingHandlerAdapter(
			@Qualifier("mvcContentNegotiationManager") ContentNegotiationManager contentNegotiationManager,
			@Qualifier("mvcConversionService") FormattingConversionService conversionService,
			@Qualifier("mvcValidator") Validator validator) {

		RequestMappingHandlerAdapter adapter = createRequestMappingHandlerAdapter();
         // 添加参数contentNegotiationManager
		adapter.setContentNegotiationManager(contentNegotiationManager);
         // 添加bean化的HttpMessageConverter实现类集合
		adapter.setMessageConverters(getMessageConverters());
        // 添加ConfigurableWebBindingInitializer(conversionService, validator, DelegatingWebMvcConfiguration.getMessageCodesResolver())——我记得validator待学的
adapter.setWebBindingInitializer(getConfigurableWebBindingInitializer(conversionService, validator));
         // 看清楚，下面配置字段明显就是收纳WebMvcConfigurer提供的自定义的。
         // 添加方法参数resolver。这里由WebMvcConfigurer实现了提供。
		adapter.setCustomArgumentResolvers(getArgumentResolvers());
		// 添加方法返回值resolver。这里由WebMvcConfigurer实现了提供。
         adapter.setCustomReturnValueHandlers(getReturnValueHandlers());

         // jackson2支持。唉，这两个我没遇见过。
		if (jackson2Present) {
			adapter.setRequestBodyAdvice(Collections.singletonList(new JsonViewRequestBodyAdvice()));
			adapter.setResponseBodyAdvice(Collections.singletonList(new JsonViewResponseBodyAdvice()));
		}

         // AsyncSupportConfigurer，这个类貌似是用来支持并发request handling的。内容由WebMvcConfigurer实现了提供。
		AsyncSupportConfigurer configurer = getAsyncSupportConfigurer();
		if (configurer.getTaskExecutor() != null) {
             // 任务执行框架-线程池，ThreadPoolTaskExecutor。
			adapter.setTaskExecutor(configurer.getTaskExecutor());
		}
		if (configurer.getTimeout() != null) {
             // 注释上，没看懂超时时间怎么用的。没关系invokeHandlerMethod方法中会要求你学习到的。
			adapter.setAsyncRequestTimeout(configurer.getTimeout());
		}
        // 注册List<CallableProcessingInterceptor>，元素是拦截器一类。
adapter.setCallableInterceptors(configurer.getCallableInterceptors());
        // 注册List<DeferredResultProcessingInterceptor>，元素是拦截器一类。
adapter.setDeferredResultInterceptors(configurer.getDeferredResultInterceptors());

		return adapter;
	}
}
```

同样的，RequestMappingHandlerAdapter也实现了InitializingBean接口。

```java
public void afterPropertiesSet() {
   // Do this first, it may add ResponseBody advice beans
   // 这里主要是提供对@ControllerAdvice解析的支持。
   initControllerAdviceCache();

   if (this.argumentResolvers == null) {
      // this.getDefaultArgumentResolvers()提供，齐全的，用于解析@ModelAndView、@InitBinder和HandlerMethod方法的参数。
      List<HandlerMethodArgumentResolver> resolvers = getDefaultArgumentResolvers();
      this.argumentResolvers = new HandlerMethodArgumentResolverComposite().addResolvers(resolvers);
   }
   if (this.initBinderArgumentResolvers == null) {
      // 这个偏少，专门给@InitBinder标记方法的解析参数。
      List<HandlerMethodArgumentResolver> resolvers = getDefaultInitBinderArgumentResolvers();
      this.initBinderArgumentResolvers = new HandlerMethodArgumentResolverComposite().addResolvers(resolvers);
   }
   // 返回参数解析器
   if (this.returnValueHandlers == null) {
      List<HandlerMethodReturnValueHandler> handlers = getDefaultReturnValueHandlers();
      this.returnValueHandlers = new HandlerMethodReturnValueHandlerComposite().addHandlers(handlers);
   }
}
```

```java
private void initControllerAdviceCache() {
   if (getApplicationContext() == null) {
      return;
   }

   // 获取@ControllerAdvice标记类的封装
   // ControllerAdviceBean(beanName, beanFactory, @ControllerAdvice注解代理实现类)
   List<ControllerAdviceBean> adviceBeans = ControllerAdviceBean.findAnnotatedBeans(getApplicationContext());

   List<Object> requestResponseBodyAdviceBeans = new ArrayList<>();

   // 这里对@ControllerAdvice做解析支持，其支持四种作用，这里提供三种，另一种“异常”在RequestMappingHandlerMapping得到解析支持。
   for (ControllerAdviceBean adviceBean : adviceBeans) {
      Class<?> beanType = adviceBean.getBeanType();
      if (beanType == null) {
         throw new IllegalStateException("Unresolvable type for ControllerAdviceBean: " + adviceBean);
      }
      // 获取使用@ModelAttribute却未用@RequestMapping的方法。
      Set<Method> attrMethods = MethodIntrospector.selectMethods(beanType, MODEL_ATTRIBUTE_METHODS);
      if (!attrMethods.isEmpty()) {
         this.modelAttributeAdviceCache.put(adviceBean, attrMethods);
      }
      // 获取使用@InitBinder的方法。
      Set<Method> binderMethods = MethodIntrospector.selectMethods(beanType, INIT_BINDER_METHODS);
      if (!binderMethods.isEmpty()) {
         this.initBinderAdviceCache.put(adviceBean, binderMethods);
      }
      // 同时是RequestBodyAdvice或ResponseBodyAdvice的实现类
      if (RequestBodyAdvice.class.isAssignableFrom(beanType) || ResponseBodyAdvice.class.isAssignableFrom(beanType)) {
         requestResponseBodyAdviceBeans.add(adviceBean);
      }
   }

   if (!requestResponseBodyAdviceBeans.isEmpty()) {
      this.requestResponseBodyAdvice.addAll(0, requestResponseBodyAdviceBeans);
   }

   ........................ 日志.........................
}
```





## BeanPostProcessor支持

有一部分类，比如RequestMappingHandlerMapping类将会在此类中得到支持。

```java
class ApplicationContextAwareProcessor implements BeanPostProcessor

    // 这个方法将在afterPropertiesSet方法之前开始。
	@Override
	@Nullable
	public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
     	// RequestMappingHandlerMapping是实现了EmbeddedValueResolverAware和ApplicationContextAware接口的，因此还需要做处理。。。。。。
		if (!(bean instanceof EnvironmentAware || bean instanceof EmbeddedValueResolverAware ||
				bean instanceof ResourceLoaderAware || bean instanceof ApplicationEventPublisherAware ||
				bean instanceof MessageSourceAware || bean instanceof ApplicationContextAware ||
				bean instanceof ApplicationStartupAware)) {
			return bean;
		}

		AccessControlContext acc = null;

		if (System.getSecurityManager() != null) {
			acc = this.applicationContext.getBeanFactory().getAccessControlContext();
		}

		if (acc != null) {
			AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
				invokeAwareInterfaces(bean);
				return null;
			}, acc);
		}
		else {
             // 处理。
			invokeAwareInterfaces(bean);
		}

		return bean;
	}

    private void invokeAwareInterfaces(Object bean) {
        if (bean instanceof EmbeddedValueResolverAware) {
            // 调用RequestMappingHandlerMapping实现的方法。
            ((EmbeddedValueResolverAware) bean).setEmbeddedValueResolver(this.embeddedValueResolver);
        }
        if (bean instanceof ApplicationContextAware) {
             // 调用RequestMappingHandlerMapping实现的方法。这个里面是实现了@ExceptionHandler解析支持。
			((ApplicationContextAware) bean).setApplicationContext(this.applicationContext);
		}
        ..................................
    }
}
```

AbstractHandlerMapping.initApplicationContext()

```java
protected void initApplicationContext() throws BeansException {
   extendInterceptors(this.interceptors);
   detectMappedInterceptors(this.adaptedInterceptors);
   // 进入
   initInterceptors();
}
```

```java
protected void initInterceptors() {
   if (!this.interceptors.isEmpty()) {
      for (int i = 0; i < this.interceptors.size(); i++) {
         // 这个this.interceptors哪里来的呢？
         Object interceptor = this.interceptors.get(i);
         if (interceptor == null) {
            throw new IllegalArgumentException("Entry number " + i + " in interceptors array is null");
         }
         this.adaptedInterceptors.add(adaptInterceptor(interceptor));
      }
   }
}
```



## custom arguement-resolver和return-value-handler优先级

​	参数resolver和返回值handler都是在RequestMappingHandlerAdapter环境中引用到的，也是在那里配置的。

- this.argumentResolvers：供handler-method和@ModelAttribute标记方法使用
- this.initBinderArgumentResolvers：供@InitBinder标记方法使用
- this.returnValueHandlers：供handler-method使用。

```java
public void afterPropertiesSet() {
   // Do this first, it may add ResponseBody advice beans
   initControllerAdviceCache();

   if (this.argumentResolvers == null) {
      List<HandlerMethodArgumentResolver> resolvers = getDefaultArgumentResolvers();
      this.argumentResolvers = new HandlerMethodArgumentResolverComposite().addResolvers(resolvers);
   }
   if (this.initBinderArgumentResolvers == null) {
      // 就你提供的最少，那你做样例了。
      // custom的resolver是有摆放位置的，这决定了其在spring提供的resolver中的位置是相对不变的。
      List<HandlerMethodArgumentResolver> resolvers = getDefaultInitBinderArgumentResolvers();
      this.initBinderArgumentResolvers = new HandlerMethodArgumentResolverComposite().addResolvers(resolvers);
   }
   if (this.returnValueHandlers == null) {
      List<HandlerMethodReturnValueHandler> handlers = getDefaultReturnValueHandlers();
      this.returnValueHandlers = new HandlerMethodReturnValueHandlerComposite().addHandlers(handlers);
   }
}
```

```java
private List<HandlerMethodArgumentResolver> getDefaultInitBinderArgumentResolvers() {
   List<HandlerMethodArgumentResolver> resolvers = new ArrayList<>(20);

   // Annotation-based argument resolution
   resolvers.add(new RequestParamMethodArgumentResolver(getBeanFactory(), false));
   resolvers.add(new RequestParamMapMethodArgumentResolver());
   resolvers.add(new PathVariableMethodArgumentResolver());
   resolvers.add(new PathVariableMapMethodArgumentResolver());
   resolvers.add(new MatrixVariableMethodArgumentResolver());
   resolvers.add(new MatrixVariableMapMethodArgumentResolver());
   resolvers.add(new ExpressionValueMethodArgumentResolver(getBeanFactory()));
   resolvers.add(new SessionAttributeMethodArgumentResolver());
   resolvers.add(new RequestAttributeMethodArgumentResolver());

   // Type-based argument resolution
   resolvers.add(new ServletRequestMethodArgumentResolver());
   resolvers.add(new ServletResponseMethodArgumentResolver());

   // Custom arguments
   // 只能在自己的螺蛳壳里做道场了。。。优先级仅限在自定义排，外部的优先级相对位置已经确定死了。每种自定义要做到截留，不要全部类型考虑处理，这是ServletModelAttributeMethodProcessor(true)该干的活。
   if (getCustomArgumentResolvers() != null) {
      resolvers.addAll(getCustomArgumentResolvers());
   }

   // Catch-all
   resolvers.add(new PrincipalMethodArgumentResolver());
   resolvers.add(new RequestParamMethodArgumentResolver(getBeanFactory(), true));

   return resolvers;
}
```
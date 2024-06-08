## 总览

​	这里对下文的内容做一些概论。

1. 注册TomcatEmbeddedContext(TomcatStarter(**ServletContextInitializer[]**))，**ServletContextInitializer[]**主要的一个元素由ServletWebServerApplicationContext.getSelfInitializer提供。

2. Tomcat.start启动后，容器等资源也随之初始化，其中Tomcat的类StandardContext将调用startInternal方法，启动资源所需的资源，上述的`ServletContextInitializer[]`资源在`某个未知阶段`存储在其字段initializers上。激活`ServletContextInitializer，调用ServletWebServerApplicationContext提供实现的方法selfInitialize(ServletContext)。

3. 方法执行过程中，DispatcherServlet，执行其函数式接口ServletContextInitializer实现的方法onStartup，将其注册在servletContext环境中，以StandardWrapper(DispatcherServlet)的形式存储在环境中，而全局状态下，StandardHost(TomcatEmbeddedContext(StandardWrapper)))，由此，单个Tomcat端口兼容容器下，DispatcherServlet全局唯一。

   不知道Tomcat的集群是不是与JVM启动的数量绑定，若是的话，一个Tomcat就拥有一个DispatcherServlet，不是的话就麻烦了，得学。



## DispatcherServletRegistrationBean提供

​	最为关键的DispatcherServlet就是DispatcherServletRegistrationBean父类的内部属性。该类由Spring bean化提供，以下代码展示了他们的关系。

`DispatcherServletAutoConfiguration.DispatcherServletRegistrationConfiguration`提供了`DispatcherServletRegistrationBean`，该类向Tomcat提供了Servlet：DispatcherServlet。

```java
@AutoConfigureOrder(Ordered.HIGHEST_PRECEDENCE)
@AutoConfiguration(after = ServletWebServerFactoryAutoConfiguration.class)
@ConditionalOnWebApplication(type = Type.SERVLET)
@ConditionalOnClass(DispatcherServlet.class)
public class DispatcherServletAutoConfiguration {
    
    @Configuration(proxyBeanMethods = false)
	@Conditional(DefaultDispatcherServletCondition.class)
	@ConditionalOnClass(ServletRegistration.class)
	@EnableConfigurationProperties(WebMvcProperties.class)
	protected static class DispatcherServletConfiguration {

         // 全局单例DispatcherServlet提供。
		@Bean(name = DEFAULT_DISPATCHER_SERVLET_BEAN_NAME)
		public DispatcherServlet dispatcherServlet(WebMvcProperties webMvcProperties) {
			DispatcherServlet dispatcherServlet = new DispatcherServlet();
			dispatcherServlet.setDispatchOptionsRequest(webMvcProperties.isDispatchOptionsRequest());
			dispatcherServlet.setDispatchTraceRequest(webMvcProperties.isDispatchTraceRequest());
			dispatcherServlet.setThrowExceptionIfNoHandlerFound(webMvcProperties.isThrowExceptionIfNoHandlerFound());
			dispatcherServlet.setPublishEvents(webMvcProperties.isPublishRequestHandledEvents());
			dispatcherServlet.setEnableLoggingRequestDetails(webMvcProperties.isLogRequestDetails());
			return dispatcherServlet;
		}
         .................................

	}

    @Configuration(proxyBeanMethods = false)
    @Conditional(DispatcherServletRegistrationCondition.class)
    @ConditionalOnClass(ServletRegistration.class)
    @EnableConfigurationProperties(WebMvcProperties.class)
    @Import(DispatcherServletConfiguration.class)
    protected static class DispatcherServletRegistrationConfiguration {

       @Bean(name = DEFAULT_DISPATCHER_SERVLET_REGISTRATION_BEAN_NAME)
       @ConditionalOnBean(value = DispatcherServlet.class, name = DEFAULT_DISPATCHER_SERVLET_BEAN_NAME)
       // DispatcherServletRegistrationBean bean化。
       // 将DispatcherServlet的bean-object inject。
       public DispatcherServletRegistrationBean dispatcherServletRegistration(DispatcherServlet dispatcherServlet,
             WebMvcProperties webMvcProperties, ObjectProvider<MultipartConfigElement> multipartConfig) {
          DispatcherServletRegistrationBean registration = new DispatcherServletRegistrationBean(dispatcherServlet,
                webMvcProperties.getServlet().getPath());
          registration.setName(DEFAULT_DISPATCHER_SERVLET_BEAN_NAME);
          registration.setLoadOnStartup(webMvcProperties.getServlet().getLoadOnStartup());
          multipartConfig.ifAvailable(registration::setMultipartConfig);
          return registration;
       }
    }
}
```

简单表示，就是DispatcherServletRegistrationBean(DispatcherServlet)。



## DispatcherServletRegistrationBean实现函数接口ServletContextInitializer的使用

可前往文章[ServletContextInitializer设计理念](../SpringBoot/ServletContextInitializer设计理念.md)学习使用背景。

以下调用过程不会总是衔接的，会存在跳跃性：

SpringApplication.refreshContext(ConfigurableApplicationContext) →

ConfigurableApplicationContext.refresh()——method invoker：AnnotationConfigServletWebServerApplicationContext →

ServletWebServerApplicationContext.createWebServer()

```java
private void createWebServer() {
   WebServer webServer = this.webServer;
   ServletContext servletContext = getServletContext();
   if (webServer == null && servletContext == null) {
      StartupStep createWebServer = this.getApplicationStartup().start("spring.boot.webserver.create");
      // ServletWebServerFactoryConfiguration.EmbeddedTomcat.tomcatServletWebServerFactory()提供bean化的TomcatServletWebServerFactory实例。
      ServletWebServerFactory factory = getWebServerFactory();
      createWebServer.tag("factory", factory.getClass().toString());
      // 这一步提供TomcatWebServer，这个过程中干了非常多的事情，
      // getSelfInitializer()提供的ServletContextInitializer实现实际上是被javax.servlet.ServletContainerInitializer 的Spring实现TomcatStarter给封装到数组中了。Tomcat用ServletContainerInitializer对ServletContext进行部分（尤其是外部）的初始化工作。
      this.webServer = factory.getWebServer(getSelfInitializer());
      createWebServer.end();
      getBeanFactory().registerSingleton("webServerGracefulShutdown",
            new WebServerGracefulShutdownLifecycle(this.webServer));
      getBeanFactory().registerSingleton("webServerStartStop",
            new WebServerStartStopLifecycle(this, this.webServer));
   }
   else if (servletContext != null) {
      try {
         getSelfInitializer().onStartup(servletContext);
      }
      catch (ServletException ex) {
         throw new ApplicationContextException("Cannot initialize servlet context", ex);
      }
   }
   initPropertySources();
}
```

getSelfInitializer：很棒的技巧，将类内同参数列表的方法作为目标函数式接口的实现，即ServletContextInitializer的实现类隐藏了selfInitialize方法，向外界提供服务。其另一种效用是“**延迟作用**”，当方法调用环境准备完善后，才调用。

```java
// 当前方法所在类 ServletWebServerApplicationContext
private org.springframework.boot.web.servlet.ServletContextInitializer getSelfInitializer() {
   return this::selfInitialize;
}

private void selfInitialize(ServletContext servletContext) throws ServletException {
   prepareWebApplicationContext(servletContext);
   registerApplicationScope(servletContext);
   WebApplicationContextUtils.registerEnvironmentBeans(getBeanFactory(), servletContext);
   for (ServletContextInitializer beans : getServletContextInitializerBeans()) {
      beans.onStartup(servletContext);
   }
}
```

等之后我们在讨论这个“延迟方法”什么时候调用。

ServletWebServerFactory.getWebServer(ServletContextInitializer...)——method invoker：TomcatServletWebServerFactory。

1. TomcatServletWebServerFactory.prepareContext(Host, ServletContextInitializer[])
2. TomcatServletWebServerFactory.configureContext(Context, ServletContextInitializer[])——context：TomcatEmbeddedContext；其中一个目的：构建TomcatEmbeddedContext(TomcatStarter(**ServletContextInitializer[]**))
3. Host.addChild(Container)——method invoker：StandardHost，来自Tomcat.getHost()，即Tomcat的virtual Host（只需记住Tomcat和Host一对一即可）；Container：TomcatEmbeddedContext。**添加Container到StandardHost作为child Container，每个子容器将作为web request的处理者**。
4. TomcatStarter(ServletContextInitializer[])——类注释讲解其作用：用来**触发ServletContainerInitializer**们。
5. 由上述步骤，可想见prepareContext方法中，其中一个意图就是向Tomcat.Host中添加TomcatEmbeddedContext(TomcatStarter(**ServletContextInitializer[]**))。

TomcatServletWebServerFactory.getTomcatWebServer(Tomcat)

1. TomcatWebServer.initialize()

2. Tomcat.start()——这一步后，tomcat各种资源开始初始化了。

3. StandardServer.start()：实际调用method所在类LifecycleBase。这个过程中许多资源都通过LifecycleBase.start()实现，因此混杂不堪。

   ..................................................................

4. StandardContext.startInternal()——method invoker：TomcatEmbeddedContext。这个方法很长，截取目标代码块：

   ```java
   protected synchronized void startInternal() throws LifecycleException {
       ......................
       // 这里的k-v最主要的就是：k：TomcatStarter(ServletContextInitializer[])。ServletContextInitializer[]其中最重要的就是ServletWebServerApplicationContext.getSelfInitializer提供的。
       for (Map.Entry<ServletContainerInitializer, Set<Class<?>>> entry :
           initializers.entrySet()) {
           try {
               // this.context.getFacade()。context：ApplicationContext；getFacade()：ApplicationContextFacade。
               entry.getKey().onStartup(entry.getValue(),
                       getServletContext());
           } catch (ServletException e) {
               log.error(sm.getString("standardContext.sciFail"), e);
               ok = false;
               break;
           }
       }
   }
   ```

5. TomcatStarter.onStartup(Set<Class<?>>, ServletContext)

   ```java
   public void onStartup(Set<Class<?>> classes, ServletContext servletContext) throws ServletException {
       for (ServletContextInitializer initializer : this.initializers) {
           initializer.onStartup(servletContext);
       }
       。。。。。。。。。。
   }
   ```

6. ServletWebServerApplicationContext.selfInitialize(ServletContext)

   ```java
   // 经由this.getSelfInitializer方法作用，本方法作为函数式接口的内部实现。
   private void selfInitialize(ServletContext servletContext) throws ServletException {
      prepareWebApplicationContext(servletContext);
      registerApplicationScope(servletContext);
      WebApplicationContextUtils.registerEnvironmentBeans(getBeanFactory(), servletContext);
      for (ServletContextInitializer beans : getServletContextInitializerBeans()) {
         beans.onStartup(servletContext);
      }
   }
   ```

   ```java
   protected Collection<ServletContextInitializer> getServletContextInitializerBeans() {
      // getBeanFactory()：DefaultListableBeanFactory
      return new ServletContextInitializerBeans(getBeanFactory());
   }
   ```

   ```java
   // 本类实现Iterator，作为for可循环基础。核心就是sortedList装着什么呢？
   public Iterator<ServletContextInitializer> iterator() {
      return this.sortedList.iterator();
   }
   ```

   ```java
   public ServletContextInitializerBeans(ListableBeanFactory beanFactory,
         Class<? extends ServletContextInitializer>... initializerTypes) {
      this.initializers = new LinkedMultiValueMap<>();
      this.initializerTypes = (initializerTypes.length != 0) ? Arrays.asList(initializerTypes)
            : Collections.singletonList(ServletContextInitializer.class);
      // 这里向initializers添加DispatcherServletRegistrationBean————该方法主旨就是向beanFactory找ServletContextInitializer接口实现类的单例。
      addServletContextInitializerBeans(beanFactory);
      addAdaptableBeans(beanFactory);
      // initializers-Map<Class<?>, List<ServletContextInitializer>>，将value提取，并扁平化存储---Steam的flatMap作用。
      List<ServletContextInitializer> sortedInitializers = this.initializers.values().stream()
            .flatMap((value) -> value.stream().sorted(AnnotationAwareOrderComparator.INSTANCE))
            .collect(Collectors.toList());
      // sortedList，迭代器式存储。
      this.sortedList = Collections.unmodifiableList(sortedInitializers);
      logMappings(this.initializers);
   }
   ```

7. RegistrationBean.onStartup(ServletContext)——method invoker：DispatcherServletRegistrationBean；servletContex：ApplicationContextFacade

   ```java
   public final void onStartup(ServletContext servletContext) throws ServletException {
      String description = getDescription();
      if (!isEnabled()) {
         logger.info(StringUtils.capitalize(description) + " was not registered (disabled)");
         return;
      }
      // 开始了！开始了！要将DispathcherServlet注册给某个对象了。
      register(description, servletContext);
   }
   ```

8. DynamicRegistrationBean.register(String, ServletContext)

9. DynamicRegistrationBean.addRegistration(String, ServletContext)

10. ApplicationContextFacade.addServlet(String, Servlet)——method invoker：所在方法参数ServletContext---ApplicationContextFacade；Servlet：this.servlet---DispatcherServlet。

11. ApplicationContext.addServlet(String, Servlet)——method invoker：ApplicationContextFacade.context --- ApplicationContext。

12. ApplicationContext.addServlet(String, String, Servlet, Map<String,String>)。

    ```java
    private ServletRegistration.Dynamic addServlet(String servletName, String servletClass,
            Servlet servlet, Map<String,String> initParams) throws IllegalStateException {
        ................
        // context实际是TomcatEmbeddedContext。查看是否有对标容器。遗憾，第一次进入，没找到。
        Wrapper wrapper = (Wrapper) context.findChild(servletName);
    
        // Assume a 'complete' ServletRegistration is one that has a class and
        // a name
        if (wrapper == null) {
            // 首次进入创建一个子Container。<servletName, StandardWrapper>，StandardWrapper是刚new的。  
            wrapper = context.createWrapper();
            wrapper.setName(servletName);
            context.addChild(wrapper);
        } else {
            if (wrapper.getName() != null &&
                    wrapper.getServletClass() != null) {
                if (wrapper.isOverridable()) {
                    wrapper.setOverridable(false);
                } else {
                    return null;
                }
            }
        }
    
        ServletSecurity annotation = null;
        if (servlet == null) {
            wrapper.setServletClass(servletClass);
            Class<?> clazz = Introspection.loadClass(context, servletClass);
            if (clazz != null) {
                annotation = clazz.getAnnotation(ServletSecurity.class);
            }
        } else {
            // 好了目标达到了，DispatcherServlet送到了StandardWrapper里了。后续使用DispatcherServlet也是从StandardWrapper拿取。
            wrapper.setServletClass(servlet.getClass().getName());
            wrapper.setServlet(servlet);
            if (context.wasCreatedDynamicServlet(servlet)) {
                annotation = servlet.getClass().getAnnotation(ServletSecurity.class);
            }
        }
    
        if (initParams != null) {
            for (Map.Entry<String, String> initParam: initParams.entrySet()) {
                wrapper.addInitParameter(initParam.getKey(), initParam.getValue());
            }
        }
    
        // 封装了<StandardWrapper, TomcatEmbeddedContext>是为了干什么呢？貌似没存起来，应该是对封住内容做处理适配器。
        ServletRegistration.Dynamic registration =
                new ApplicationServletRegistration(wrapper, context);
        if (annotation != null) {
            registration.setServletSecurity(new ServletSecurityElement(annotation));
        }
        return registration;
    }
    ```

## Tomcat容器嵌套

StandardHost(TomcatEmbeddedContext(StandardWrapper)))

ApplicationContextFacade(ApplicationContext(StandardHost)？——这个不是，ApplicationContextFacade和ApplicationContext都是ServletContext，不是Container，只不过前者嵌套后者。而ApplicationContext又嵌套StandardContext。

不过StandardContext又反过来包含ApplicationContext。唉难办，循环嵌套，比较复杂喽。
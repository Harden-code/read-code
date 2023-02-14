#### Spring MVC笔记

`org.springframework.web.servlet.DispatcherServlet` 该类为Spring MVC处理HTTP请求的核心类。继承Servlet规范里的`javax.servlet.http.HttpServlet`，利用Servlet生命周期初始化的init方法（`org.springframework.web.servlet.HttpServletBean#init`）。创建WebApplicationContext并refresh容器。该Servlet里initStrategies方法初始化Mvc的核心组件

HandlerMapping HandlerAdapter等

> org.springframework.web.servlet.FrameworkServlet#configureAndRefreshWebApplicationContext
>
> org.springframework.web.servlet.FrameworkServlet#onRefresh

org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter



@ResponseBody关键处理判断

org.springframework.web.method.support.ModelAndViewContainer#isRequestHandled

org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter#getModelAndView

待补充。。。。



####  Locale 如何在 Servelt 以及 Spring Web MVC 获取并且存储到 ThreadLocal

#### Spring MVC Locale 中的应用

##### 如何解析HTTP请求得到Locale

1. 在Servlet规范中的ServletRequest接口定义了`getLocale()`方法，可以根据具体实现查看

   ```JAVA
   org.apache.catalina.connector.Request#getLocale
   @Override
   public Locale getLocale() {

     if (!localesParsed) {
       parseLocales();
     }

     if (locales.size() > 0) {
       return locales.get(0);
     }

     return defaultLocale;
   }

   protected void parseLocales() {
   	....
     Enumeration<String> values = getHeaders("accept-language");
   	....
   }
   ```

2. Spring内建实现 `org.springframework.web.servlet.LocaleResolver`  可查看具体实现类`org.springframework.web.servlet.i18n.AcceptHeaderLocaleResolver`

   ​

##### 解析得到国际化(Locale对象)如何在MVC的整个处理过程中使用

在框架中提供了`org.springframework.context.i18n.LocaleContextHolder`对象，该对象里通过`ThreadLocal`

来存储单条请求的`LocaleContext` 



##### `LocaleContextHolder`存储`LocaleContext` 的时机

1. 通过`org.springframework.web.filter.RequestContextFilter`拦截器，进行拦截解析后进行存放

   ```java
   @Override
   protected void doFilterInternal(
     HttpServletRequest request, HttpServletResponse response, FilterChain filterChain)
     throws ServletException, IOException {

     ServletRequestAttributes attributes = new ServletRequestAttributes(request, response);
     initContextHolders(request, attributes);

     try {
       filterChain.doFilter(request, response);
     }
     finally {
       //清除Holder里存放的Locale
       resetContextHolders();
       if (logger.isTraceEnabled()) {
         logger.trace("Cleared thread-bound request context: " + request);
       }
       attributes.requestCompleted();
     }
   }
   ```

2. 在FrameworkServlet (DispatcherServlet父类) 中的处理请求方法里都会有个processRequest方法 ，处理LocaleContext和RequestAttributes的存储

   ```JAVA
   protected final void processRequest(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
   		Throwable failureCause = null;

   		LocaleContext previousLocaleContext = LocaleContextHolder.getLocaleContext();//1
   		LocaleContext localeContext = buildLocaleContext(request);//2

   		RequestAttributes previousAttributes = RequestContextHolder.getRequestAttributes();
   		ServletRequestAttributes requestAttributes = buildRequestAttributes(request, response, previousAttributes);

   		...
   		finally {
   			resetContextHolders(request, previousLocaleContext, previousAttributes);
   			if (requestAttributes != null) {
   				requestAttributes.requestCompleted();
   			}
   			logResult(request, response, failureCause, asyncManager);
   			publishRequestHandledEvent(request, response, startTime, failureCause);
   		}
   	}
   ```

   > 1 处获取的到LocaleContext对象，由上可知是在 RequestContextFilter拦截器里进行设置的。但是在处理Spring在处理实现时，DispatcherServlet会重写2的buildLocaleContext方法，通过Spring内建的`LocaleResolver`接口重新创建一个新的LocaleContext放在Holder中。等的模板处理完后，又恢复上一个previousLocaleContext设置在Holder中



#### 理解 Locale 如何在 View Render 中配合使用，需要模板引擎来整合

通过上述的LocaleContextHolder对象获取Locale在View中使用






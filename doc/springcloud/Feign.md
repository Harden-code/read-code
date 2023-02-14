#### `feign.Feign.Builder`  

Feign的构造对象



#### `feign.Target`  

 接口描述的HTTP的调用原信息



#### `feign.Feign`  

定义的接口最终通过动态代理的方式生成`ReflectiveFeign`执行HTTP的通信

- `feign.ReflectiveFeign`   

  ```java
  @Override
  public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    if ("equals".equals(method.getName())) {
      try {
        Object otherHandler =
            args.length > 0 && args[0] != null ? Proxy.getInvocationHandler(args[0]) : null;
        return equals(otherHandler);
      } catch (IllegalArgumentException e) {
        return false;
      }
    } else if ("hashCode".equals(method.getName())) {
      return hashCode();
    } else if ("toString".equals(method.getName())) {
      return toString();
    }
   //委派Map<Method, MethodHandler -> SynchronousMethodHandler
    return dispatch.get(method).invoke(args);
  }
  ```





#### `feign.InvocationHandlerFactory` 

 创建JDK的代理拦截对象工厂



#### `feign.ReflectiveFeign.FeignInvocationHandler`  

动态代理Feign的处理 

```JAVA
private final Target target;
private final Map<Method, MethodHandler> dispatch;
```

- `feign.SynchronousMethodHandler`用户定义Feign接口里面方法的执行抽象

  ```JAVA
  public Object invoke(Object[] argv) throws Throwable {
    //通过方法元信息构造请求模板(方法解析)
    RequestTemplate template = buildTemplateFromArgs.create(argv);
    Options options = findOptions(argv);
    Retryer retryer = this.retryer.clone();
    //异常重试逻辑
    while (true) {
      try {
        //执行请求逻辑并解码返回值
        return executeAndDecode(template, options);
      } catch (RetryableException e) {
        try {
          retryer.continueOrPropagate(e);
        } catch (RetryableException th) {
          Throwable cause = th.getCause();
          if (propagationPolicy == UNWRAP && cause != null) {
            throw cause;
          } else {
            throw th;
          }
        }
        if (logLevel != Logger.Level.NONE) {
          logger.logRetry(metadata.configKey(), logLevel);
        }
        continue;
      }
    }
  }
  Object executeAndDecode(RequestTemplate template, Options options) throws Throwable {
    //执行RequestInterceptor 并创建http描述调用请求
    Request request = targetRequest(template);

    if (logLevel != Logger.Level.NONE) {
      logger.logRequest(metadata.configKey(), logLevel, request);
    }

    Response response;
    long start = System.nanoTime();
    try {
      response = client.execute(request, options);
      // ensure the request is set. TODO: remove in Feign 12
      response = response.toBuilder()
        .request(request)
        .requestTemplate(template)
        .build();
    } catch (IOException e) {
      if (logLevel != Logger.Level.NONE) {
        logger.logIOException(metadata.configKey(), logLevel, e, elapsedTime(start));
      }
      throw errorExecuting(request, e);
    }
    long elapsedTime = TimeUnit.NANOSECONDS.toMillis(System.nanoTime() - start);

  }if (decoder != null)
    return decoder.decode(response, metadata.returnType());

  //异步处理
  CompletableFuture<Object> resultFuture = new CompletableFuture<>();
  asyncResponseHandler.handleResponse(resultFuture, metadata.configKey(), response,
                                      metadata.returnType(),
                                      elapsedTime);
    try {
      if (!resultFuture.isDone())
        throw new IllegalStateException("Response handling not done");

      return resultFuture.join();
    } catch (CompletionException e) {
      Throwable cause = e.getCause();
      if (cause != null)
        throw cause;
      throw e;
    }
  }
  ```




#### `feign.InvocationHandlerFacry.MethodHandler`

 代理的接口处理逻辑 实现类`feign.SynchronousMethodHandler`

`feign.ReflectiveFeign.ParseHandlersByName#apply`

  ```java
public Map<String, MethodHandler> apply(Target target) {
      List<MethodMetadata> metadata = contract.parseAndValidateMetadata(target.type());
      Map<String, MethodHandler> result = new LinkedHashMap<String, MethodHandler>();
  //MethodMetadata 方法描述元信息
      for (MethodMetadata md : metadata) {
       
        BuildTemplateByResolvingArgs buildTemplate;
        if (!md.formParams().isEmpty() && md.template().bodyTemplate() == null) {
          buildTemplate =
              new BuildFormEncodedTemplateFromArgs(md, encoder, queryMapEncoder, target);
        } else if (md.bodyIndex() != null || md.alwaysEncodeBody()) {
          buildTemplate = new BuildEncodedTemplateFromArgs(md, encoder, queryMapEncoder, target);
        } else {
          buildTemplate = new BuildTemplateByResolvingArgs(md, queryMapEncoder, target);
        }
        if (md.isIgnored()) {
          result.put(md.configKey(), args -> {
            throw new IllegalStateException(md.configKey() + " is not a method handled by feign");
          });
        } else {
          //通过RequestTemplate.Factory 构造MethodHandler具体的类型SynchronousMethodHandler
          result.put(md.configKey(),
              factory.create(target, md, buildTemplate, options, decoder, errorDecoder));
        }
      }
      return result;
    }
  }
  ```



##### `feign.RequestTemplate.Factory`

 Factory for creating RequestTemplate；  实现类`feign.ReflectiveFeign.BuildTemplateByResolvingArgs`

在该类里面的create方法，会解析接口的元信息构造HTTP请求模板对象

```JAVA
public RequestTemplate create(Object[] argv) {
      RequestTemplate mutable = RequestTemplate.from(metadata.template());
     ....
       //模板方法，实现类里面可以对encode进行自定义扩展
      RequestTemplate template = resolve(argv, mutable, varBuilder);
      if (metadata.queryMapIndex() != null) {
        // add query map parameters after initial resolve so that they take
        // precedence over any predefined values
        Object value = argv[metadata.queryMapIndex()];
        Map<String, Object> queryMap = toQueryMap(value);
        template = addQueryMapQueryParameters(queryMap, template);
      }
     ...
      return template;
    }

```




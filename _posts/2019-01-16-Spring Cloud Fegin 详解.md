---
layout:     post                    # 使用的布局（不需要改）
title:     Spring Cloud Fegin 详解(一)               # 标题 
subtitle:   Spring Cloud Fegin 工作原理
date:       2019-01-14              # 时间
author:     BY                      # 作者
catalog: true                       # 是否归档
tags:                               #标签
    - Spring Cloud
---



# Spring Cloud Fegin 详解

## 1). Fegin 的基础功能

FeginClient 注解@Target(ElementType.TYPE)修饰，表示FeginClient注解的作用目标在接口上。FeginClient注解对应的属性：

* name ： 指定FeginClient 的名称，如果项目使用了Ribbon ， name属性会作为微服务的名称，用于服务发现。
* url：url一般用于调试，可以手动指定@FeginClient 调用地址
* decode404 ： 当发生404错误时，如果会调用decoder解码，否则抛出FeginException
* configuration：Fegin配置类，可以自定义Fegin的Encoder ，Decoder ，LogLevel ，Contract
* fallback：定义容错的处理类，当调用远程接口失败或超时，回调用对应接口的容错逻辑，fallback指定的类必须实现@FeginClient 标识的接口。
* fallbackFactory ： 工厂类，用于生成fallback实例，通过这个属性我们可以实现每个接口通用的容错逻辑，减少代码冗余
* path ： 定义当前FeginClient的统一前缀。



```java
if (!StringUtils.hasText(this.url)) {
   String url;
   if (!this.name.startsWith("http")) {
      url = "http://" + this.name;
   }
   else {
      url = this.name;
   }
   url += cleanPath();
   return loadBalance(builder, context, new HardCodedTarget<>(this.type,
         this.name, url));
}
```



Spring Cloud Fegin  支持对请求和响应进行GZIP压缩，提高通信效率。压缩配置：

```yaml
fegin:
  compression:
    request:
      enabled: true
      mime-types: text/xml,application/xml,application/json
      min-request-size: 2048
    response:
      enabled: true
```



需要注意的是：由于开启GZIP 压缩之后，Fegin之间的调用通过二进制协议进行传输，返回值需要修改为ResponseEntity<byte []> 才可以正常显示。

其他的一些配置及基本功能这里就不介绍了。 

## 2).Spring Cloud Fegin 工作原理

我们的切入点还是Spring启动类，再我们使用Feign时会在启动类上加上@EnableFeignClients注解

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@Documented
@Import(FeignClientsRegistrar.class)
public @interface EnableFeignClients {
	// 指定需要扫的包
	String[] basePackages() default {};
    // 默认的配置类
    Class<?>[] defaultConfiguration() default {};
    // 被@FeignClient 标注的类集合
    Class<?>[] clients() default {};

}
```



在SpringBoot 启动时，在ApplicationContext#refresh方法中会调用invokeBeanFactoryPostProcessors方法将调用invokeBeanDefinitionRegistryPostProcessors() 会将所有的configClass（配置类）@Import引入的ImportBeanDefinitionRegistrar子类加载，并执行registerBeanDefinitions方法。（具体过程，读者可在FeignClientsRegistrar#registerBeanDefinitions打断点查看调用栈）。

具体看看FeignClientsRegistrar：

```java
class FeignClientsRegistrar implements ImportBeanDefinitionRegistrar,
		ResourceLoaderAware, EnvironmentAware {
        
            ......
                // 注册 FeignClient
                public void registerFeignClients(AnnotationMetadata metadata,
			BeanDefinitionRegistry registry) {
		ClassPathScanningCandidateComponentProvider scanner = getScanner();
		scanner.setResourceLoader(this.resourceLoader);

		Set<String> basePackages;

		Map<String, Object> attrs = metadata
				.getAnnotationAttributes(EnableFeignClients.class.getName());
           //  FeignClient 过滤器 
		AnnotationTypeFilter annotationTypeFilter = new AnnotationTypeFilter(
				FeignClient.class);
          // 判断@EnableFeignClients 是否配置了 clients  为空 ，使用  
		final Class<?>[] clients = attrs == null ? null
				: (Class<?>[]) attrs.get("clients");
           //  clients  为空 ，使用 FeignClient 过滤器  
		if (clients == null || clients.length == 0) {
			scanner.addIncludeFilter(annotationTypeFilter);
			basePackages = getBasePackages(metadata);
		}
		else {
            // 不为空  使用  要clients包含的 过滤器 
			final Set<String> clientClasses = new HashSet<>();
			basePackages = new HashSet<>();
			for (Class<?> clazz : clients) {
				basePackages.add(ClassUtils.getPackageName(clazz));
				clientClasses.add(clazz.getCanonicalName());
			}
			AbstractClassTestingTypeFilter filter = new AbstractClassTestingTypeFilter() {
				@Override
				protected boolean match(ClassMetadata metadata) {
					String cleaned = metadata.getClassName().replaceAll("\\$", ".");
					return clientClasses.contains(cleaned);
				}
			};
			scanner.addIncludeFilter(
					new AllTypeFilter(Arrays.asList(filter, annotationTypeFilter)));
		}
			//  扫包 
		for (String basePackage : basePackages) {
			Set<BeanDefinition> candidateComponents = scanner
					.findCandidateComponents(basePackage);
			for (BeanDefinition candidateComponent : candidateComponents) {
				if (candidateComponent instanceof AnnotatedBeanDefinition) {
					// 检查是否为一个接口 
					AnnotatedBeanDefinition beanDefinition = (AnnotatedBeanDefinition) candidateComponent;
					AnnotationMetadata annotationMetadata = beanDefinition.getMetadata();
					Assert.isTrue(annotationMetadata.isInterface(),
							"@FeignClient can only be specified on an interface");

					Map<String, Object> attributes = annotationMetadata
							.getAnnotationAttributes(
									FeignClient.class.getCanonicalName());

					String name = getClientName(attributes);
                    // 注册 FeignClientSpecification 
					registerClientConfiguration(registry, name,
							attributes.get("configuration"));
					// 注册  FeignClient  
					registerFeignClient(registry, annotationMetadata, attributes);
				}
			}
		}
	}
    // 注册  FeignClient  
	private void registerFeignClient(BeanDefinitionRegistry registry,
			AnnotationMetadata annotationMetadata, Map<String, Object> attributes) {
		String className = annotationMetadata.getClassName();
         // 获取FeignClientFactoryBean 的bean定义类 
		BeanDefinitionBuilder definition = BeanDefinitionBuilder
				.genericBeanDefinition(FeignClientFactoryBean.class);
		validate(attributes);
        // 将FeignClient 上的信息写入 bean定义类 
		definition.addPropertyValue("url", getUrl(attributes));
		definition.addPropertyValue("path", getPath(attributes));
		String name = getName(attributes);
		definition.addPropertyValue("name", name);
		definition.addPropertyValue("type", className);
		definition.addPropertyValue("decode404", attributes.get("decode404"));
		definition.addPropertyValue("fallback", attributes.get("fallback"));
		definition.addPropertyValue("fallbackFactory", attributes.get("fallbackFactory"));
        // 通过 Type   自动注入  
		definition.setAutowireMode(AbstractBeanDefinition.AUTOWIRE_BY_TYPE);
		// 配置别名 
		String alias = name + "FeignClient";
		AbstractBeanDefinition beanDefinition = definition.getBeanDefinition();

		boolean primary = (Boolean)attributes.get("primary"); // has a default, won't be null

		beanDefinition.setPrimary(primary);

		String qualifier = getQualifier(attributes);
		if (StringUtils.hasText(qualifier)) {
			alias = qualifier;
		}

		BeanDefinitionHolder holder = new BeanDefinitionHolder(beanDefinition, className,
				new String[] { alias });
        // 注册 
		BeanDefinitionReaderUtils.registerBeanDefinition(holder, registry);
	}
     // 注册 FeignClientSpecification   FeignClientSpecification 属性中 name 为 @FeignClient中的name  configuration 为 @FeignClient 中的  configuration 
      private void registerClientConfiguration(BeanDefinitionRegistry registry, Object name,
			Object configuration) {
		BeanDefinitionBuilder builder = BeanDefinitionBuilder
				.genericBeanDefinition(FeignClientSpecification.class);
		builder.addConstructorArgValue(name);
		builder.addConstructorArgValue(configuration);
		registry.registerBeanDefinition(
				name + "." + FeignClientSpecification.class.getSimpleName(),
				builder.getBeanDefinition());
      }   
            .....
        
}
```



从上面的代码可以看出 ，Spring Cloud 为 @EnableFeignClients 中指定的包中所有被@FeignClient 标注的类制定了一个FeignClientFactoryBean 和 FeignClientSpecification（为每个@FeignClient 标注的类指定配置）。



FeignClientFactoryBean 是一个FactoryBean（这里不介绍FactoryBean），Spring 注入的时候会调用 FactoryBean的getObject方法 ， 那么关键就在FeignClientFactoryBean的getObject方法  ：

```java
@Override
	public Object getObject() throws Exception {
		FeignContext context = applicationContext.getBean(FeignContext.class);
        // 装配好 Feign.Builder （日志，编码器，解码器等）
		Feign.Builder builder = feign(context);
		// 没有url 属性 
		if (!StringUtils.hasText(this.url)) {
			String url;
            // 配置好 url 
			if (!this.name.startsWith("http")) {
				url = "http://" + this.name;
			}
			else {
				url = this.name;
			}
			url += cleanPath();
            //  创建Object 
			return loadBalance(builder, context, new HardCodedTarget<>(this.type,
					this.name, url));
		}
		if (StringUtils.hasText(this.url) && !this.url.startsWith("http")) {
			this.url = "http://" + this.url;
		}
		String url = this.url + cleanPath();
        // 获取 Client
		Client client = getOptional(context, Client.class);
		if (client != null) {
			if (client instanceof LoadBalancerFeignClient) {
				// not load balancing because we have a url,
				// but ribbon is on the classpath, so unwrap
				client = ((LoadBalancerFeignClient)client).getDelegate();
			}
			builder.client(client);
		}
		Targeter targeter = get(context, Targeter.class);
		return targeter.target(this, builder, context, new HardCodedTarget<>(
				this.type, this.name, url));
	}



protected <T> T loadBalance(Feign.Builder builder, FeignContext context,
			HardCodedTarget<T> target) {
     // 获取 Client  有ribbon 时为LoadBalancerFeignClient 
		Client client = getOptional(context, Client.class);
		if (client != null) {
			builder.client(client);
            // 获取  Targeter   有 Hystrix 时为  HystrixTargeter 
			Targeter targeter = get(context, Targeter.class);
			return targeter.target(this, builder, context, target);
		}

		throw new IllegalStateException(
				"No Feign Client for loadBalancing defined. Did you forget to include spring-cloud-starter-netflix-ribbon?");
	}
```



上面都是将一些配置准备好，接下来看Targeter 的target 方法 ，直接看HystrixTargeter：

```java
@Override
	public <T> T target(FeignClientFactoryBean factory, Feign.Builder feign, FeignContext context,
						Target.HardCodedTarget<T> target) {
		if (!(feign instanceof feign.hystrix.HystrixFeign.Builder)) {
			return feign.target(target);
		}
		feign.hystrix.HystrixFeign.Builder builder = (feign.hystrix.HystrixFeign.Builder) feign;
		SetterFactory setterFactory = getOptional(factory.getName(), context,
			SetterFactory.class);
		if (setterFactory != null) {
			builder.setterFactory(setterFactory);
		}
		Class<?> fallback = factory.getFallback();
		if (fallback != void.class) {
			return targetWithFallback(factory.getName(), context, target, builder, fallback);
		}
		Class<?> fallbackFactory = factory.getFallbackFactory();
		if (fallbackFactory != void.class) {
			return targetWithFallbackFactory(factory.getName(), context, target, builder, fallbackFactory);
		}

		return feign.target(target);
    }
```

将feign转为 feign.hystrix.HystrixFeign.Builder 设置好fallback 或fallbackFactory 调用 feign.hystrix.HystrixFeign.Builder的target方法 

```java
  public <T> T target(Target<T> target, T fallback) {
      return build(fallback != null ? new FallbackFactory.Default<T>(fallback) : null)
          .newInstance(target);
    }


Feign build(final FallbackFactory<?> nullableFallbackFactory) {
      super.invocationHandlerFactory(new InvocationHandlerFactory() {
        @Override public InvocationHandler create(Target target,
            Map<Method, MethodHandler> dispatch) {
          return new HystrixInvocationHandler(target, dispatch, setterFactory, nullableFallbackFactory);
        }
      });
      super.contract(new HystrixDelegatingContract(contract));
      return super.build();
    }
```

设置 HystrixInvocationHandler   和 HystrixDelegatingContract。 调用Feign#newInstance方法，其实现在ReflectiveFeign 

```java
@Override
  public <T> T newInstance(Target<T> target) {
    Map<String, MethodHandler> nameToHandler = targetToHandlersByName.apply(target);
    Map<Method, MethodHandler> methodToHandler = new LinkedHashMap<Method, MethodHandler>();
    List<DefaultMethodHandler> defaultMethodHandlers = new LinkedList<DefaultMethodHandler>();

    for (Method method : target.type().getMethods()) {
      if (method.getDeclaringClass() == Object.class) {
        continue;
      } else if(Util.isDefault(method)) {
        DefaultMethodHandler handler = new DefaultMethodHandler(method);
        defaultMethodHandlers.add(handler);
        methodToHandler.put(method, handler);
      } else {
        methodToHandler.put(method, nameToHandler.get(Feign.configKey(target.type(), method)));
      }
    }
    InvocationHandler handler = factory.create(target, methodToHandler);
    T proxy = (T) Proxy.newProxyInstance(target.type().getClassLoader(), new Class<?>[]{target.type()}, handler);

    for(DefaultMethodHandler defaultMethodHandler : defaultMethodHandlers) {
      defaultMethodHandler.bindTo(proxy);
    }
    return proxy;
  }
```

这里知道了，使用了JDK的动态代理生成了代理类 ， 具体的执行逻辑看InvocationHandler 接口，接下来看factory.create(target, methodToHandler)  ， 在feign.hystrix.HystrixFeign.Builder的target 方法设置了InvocationHandlerFactory factory  为  HystrixInvocationHandler

```java
new InvocationHandlerFactory() {
  @Override public InvocationHandler create(Target target,
      Map<Method, MethodHandler> dispatch) {
    return new HystrixInvocationHandler(target, dispatch, setterFactory, nullableFallbackFactory);
  }
```

进入HystrixInvocationHandler 看其invoke 方法 ，就是具体方法的执行逻辑

```java
 @Override
  public Object invoke(final Object proxy, final Method method, final Object[] args)
      throws Throwable {
    // early exit if the invoked method is from java.lang.Object
    // code is the same as ReflectiveFeign.FeignInvocationHandler
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

    HystrixCommand<Object> hystrixCommand = new HystrixCommand<Object>(setterMethodMap.get(method)) {
      @Override
      protected Object run() throws Exception {
        try {
          return HystrixInvocationHandler.this.dispatch.get(method).invoke(args);
        } catch (Exception e) {
          throw e;
        } catch (Throwable t) {
          throw (Error) t;
        }
      }

      @Override
      protected Object getFallback() {
        if (fallbackFactory == null) {
          return super.getFallback();
        }
        try {
          Object fallback = fallbackFactory.create(getExecutionException());
          Object result = fallbackMethodMap.get(method).invoke(fallback, args);
          if (isReturnsHystrixCommand(method)) {
            return ((HystrixCommand) result).execute();
          } else if (isReturnsObservable(method)) {
            // Create a cold Observable
            return ((Observable) result).toBlocking().first();
          } else if (isReturnsSingle(method)) {
            // Create a cold Observable as a Single
            return ((Single) result).toObservable().toBlocking().first();
          } else if (isReturnsCompletable(method)) {
            ((Completable) result).await();
            return null;
          } else {
            return result;
          }
        } catch (IllegalAccessException e) {
          // shouldn't happen as method is public due to being an interface
          throw new AssertionError(e);
        } catch (InvocationTargetException e) {
          // Exceptions on fallback are tossed by Hystrix
          throw new AssertionError(e.getCause());
        }
      }
    };

    if (isReturnsHystrixCommand(method)) {
      return hystrixCommand;
    } else if (isReturnsObservable(method)) {
      // Create a cold Observable
      return hystrixCommand.toObservable();
    } else if (isReturnsSingle(method)) {
      // Create a cold Observable as a Single
      return hystrixCommand.toObservable().toSingle();
    } else if (isReturnsCompletable(method)) {
      return hystrixCommand.toObservable().toCompletable();
    }
    return hystrixCommand.execute();
  }
```



可以看出其主要逻辑是创建HystrixCommand ，使其具有 Hystrix特性。具体业务在HystrixInvocationHandler.this.dispatch.get(method).invoke(args);  也就是MethodHandler的invoke方法 

```java
  @Override
  public Object invoke(Object[] argv) throws Throwable {
    RequestTemplate template = buildTemplateFromArgs.create(argv);
    Retryer retryer = this.retryer.clone();
    while (true) {
      try {
        return executeAndDecode(template);
      } catch (RetryableException e) {
        retryer.continueOrPropagate(e);
        if (logLevel != Logger.Level.NONE) {
          logger.logRetry(metadata.configKey(), logLevel);
        }
        continue;
      }
    }
  }

  Object executeAndDecode(RequestTemplate template) throws Throwable {
    Request request = targetRequest(template);

    if (logLevel != Logger.Level.NONE) {
      logger.logRequest(metadata.configKey(), logLevel, request);
    }

    Response response;
    long start = System.nanoTime();
    try {
      response = client.execute(request, options);
      // ensure the request is set. TODO: remove in Feign 10
      response.toBuilder().request(request).build();
    } catch (IOException e) {
      if (logLevel != Logger.Level.NONE) {
        logger.logIOException(metadata.configKey(), logLevel, e, elapsedTime(start));
      }
      throw errorExecuting(request, e);
    }
    long elapsedTime = TimeUnit.NANOSECONDS.toMillis(System.nanoTime() - start);

    boolean shouldClose = true;
    try {
      if (logLevel != Logger.Level.NONE) {
        response =
            logger.logAndRebufferResponse(metadata.configKey(), logLevel, response, elapsedTime);
        // ensure the request is set. TODO: remove in Feign 10
        response.toBuilder().request(request).build();
      }
      if (Response.class == metadata.returnType()) {
        if (response.body() == null) {
          return response;
        }
        if (response.body().length() == null ||
                response.body().length() > MAX_RESPONSE_BUFFER_SIZE) {
          shouldClose = false;
          return response;
        }
        // Ensure the response body is disconnected
        byte[] bodyData = Util.toByteArray(response.body().asInputStream());
        return response.toBuilder().body(bodyData).build();
      }
      if (response.status() >= 200 && response.status() < 300) {
        if (void.class == metadata.returnType()) {
          return null;
        } else {
          return decode(response);
        }
      } else if (decode404 && response.status() == 404 && void.class != metadata.returnType()) {
        return decode(response);
      } else {
        throw errorDecoder.decode(metadata.configKey(), response);
      }
    } catch (IOException e) {
      if (logLevel != Logger.Level.NONE) {
        logger.logIOException(metadata.configKey(), logLevel, e, elapsedTime);
      }
      throw errorReading(request, response, e);
    } finally {
      if (shouldClose) {
        ensureClosed(response.body());
      }
    }
  }
```

由上代码看的到，通过RequestTemplate 发送HTTP请求给下游服务 ， 此时的RequestTemplate 是具有负载均衡特性的 （上面提到过Client是LoadBalancerFeignClient）。



总结：主要是扫描@EnableFeignClients注解指定的包，为被@FeignClient标注的接口通过JDK动态代理生成代理类，在调用时，使用RequestTemplate 发送请求。可通过配置是否开启负载均衡和服务熔断 。下一节，结束Feign和Ribbon和Hystrix整合。
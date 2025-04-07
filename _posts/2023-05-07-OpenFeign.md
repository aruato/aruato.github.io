---
layout:     post
title:      OpenFeign源码笔记
date:       2023-05-07
author:     zjh
header-img: img/post-bg-coffee.jpeg
catalog: true
tags:
    - openfeign
    - springcloud
---

> 两年前的笔记了，以后有时间画个图吧

# 配置类注解入口（解析配置类的逻辑）
openFeign的入口是我们在配置类上添加的注解@EnableFeignClients，在解析当前配置类的时候，判断到当前类上有一个@Import注解就会进入以下逻辑
```java
@Import(FeignClientsRegistrar.class)
public @interface EnableFeignClients {}
```
该注解所在配置类在解析的时候就会向容器中导入一个类，这个类实现了ImportBeanDefinitionRegistrar接口，重写了registerBeanDefinitions()方法
```java
class FeignClientsRegistrar
		implements ImportBeanDefinitionRegistrar, ResourceLoaderAware, EnvironmentAware {}
```
因此，SpringBoot容器在刷新的时候，每一轮配置类解析完并生成beanDefinition注册到beanFactory之后就会去调用其registerBeanDefinitions()方法，该方法实现如下：
```java
// 注意这里参数传入的metadata是当前配置类的所有注解信息（Spring通过ASM拿到的）
@Override
public void registerBeanDefinitions(AnnotationMetadata metadata, BeanDefinitionRegistry registry) {
    // 1. 如果@FeignClients注解中有指定defaultConfiguration，那么就创建一个FeignClientSpecification.class的beanDefinition，并将defaultConfiguration指定的类和全类名设置到生成的bean中
    // 2. 也就是说，我们可以给每一个服务对应的OpenFeign子容器指定单独的配置（一般也不会指定）
    registerDefaultConfiguration(metadata, registry);
    // 3. 注册@FeignClient标注的接口到SpringBoot容器中（重要）
    registerFeignClients(metadata, registry);
}
```
（1）先看第一个方法：registerDefaultConfiguration()
```java
private void registerDefaultConfiguration(AnnotationMetadata metadata, BeanDefinitionRegistry registry) {
    // 拿到当前配置类上@EnableFeignClients注解的属性
    Map<String, Object> defaultAttrs = metadata
				.getAnnotationAttributes(EnableFeignClients.class.getName(), true);

    // 如果有指定defaultConfiguration（一般我们也不会去指定吧）
    if (defaultAttrs != null && defaultAttrs.containsKey("defaultConfiguration")) {
        // name = default. + 指定类的全类名
        String name = "default." + metadata.getClassName();
        // 传入的参数包括beanFactory、name以及defaultConfiguration属性指定的类
        registerClientConfiguration(registry, name,
					defaultAttrs.get("defaultConfiguration"));
    }
}

private void registerClientConfiguration(BeanDefinitionRegistry registry, Object name, Object configuration) {
    // 创建一个生成FeignClientSpecification.class的beanDefinition的builder
    BeanDefinitionBuilder builder = BeanDefinitionBuilder
        .genericBeanDefinition(FeignClientSpecification.class);
    // 添加构造函数的参数，这样在创建FeignClientSpecification这个bean的时候，就会给其属性赋值
    builder.addConstructorArgValue(name);
    builder.addConstructorArgValue(configuration);
    // 注册beanDefinition到beanFactory中
    registry.registerBeanDefinition(
        name + "." + FeignClientSpecification.class.getSimpleName(),
        builder.getBeanDefinition());
}
```
一般我们不会单独的为FeignClient指定一个配置类，最主要的还是下面注册FeignClient的逻辑

（2）再来看看最重要的注册FeignClient的方法：registerFeignClients()
```java
// 注意这里参数传入的metadata是当前配置类的所有注解信息（Spring通过ASM拿到的）
public void registerFeignClients(AnnotationMetadata metadata,
			BeanDefinitionRegistry registry) {

    // 创建一个集合用于保存找到的标注了@FeignClient注解的接口的beanDeifnition
    LinkedHashSet<BeanDefinition> candidateComponents = new LinkedHashSet<>();
    // 1. 拿到@EnableFeignClients注解的属性
    Map<String, Object> attrs = metadata
				.getAnnotationAttributes(EnableFeignClients.class.getName());
    
    // 2 如果注解属性不为空，拿到注解手动指定的FeignClient（因此可以看到，我们可以手动指定我们的FeignClient，如果指定了就不会走扫描了）
    final Class<?>[] clients = attrs == null ? null
				: (Class<?>[]) attrs.get("clients");
    
    // 3. 如果没有手动指定FeignClient，那么就去扫描（下面2.1的内容）
    if (clients == null || clients.length == 0) {
        // 创建一个ClassPathbeanDefinitionScanner，用于扫描指定包路径下的.class文件
        ClassPathScanningCandidateComponentProvider scanner = getScanner();
        
        // scanner中设置ResourceLoader，ResourceLoader就是当前SpringBoot容器
        // 解析配置类的@Import时，如果导入的是ImportBeanDefinitionRegistrar，那么会直接实例化（不然怎么调用当前方法，不会放到容器中），同时将当前SpringBoot容器设置给当前类的resourceLoader
        scanner.setResourceLoader(this.resourceLoader);
        
        // 给scanner添加一个过滤器，用于过滤标注了@FeignClient注解的接口
        scanner.addIncludeFilter(new AnnotationTypeFilter(FeignClient.class));
        
        // 获取当前@EnableFeignClients注解中指定的扫描包路径，如果没有指定默认使用当前配置类所在的包路径
        Set<String> basePackages = getBasePackages(metadata);
        
        // 扫描包路径下的.class文件，通过ASM判断是否存在@FeignClient注解
        for (String basePackage : basePackages) {
            candidateComponents.addAll(scanner.findCandidateComponents(basePackage));
        }
    }
    else {
        // 4. 如果在@EnableFeignClients中指定了FeignClient，不扫描，直接注册指定的FeignClient
        for (Class<?> clazz : clients) {
            candidateComponents.add(new AnnotatedGenericBeanDefinition(clazz));
        }
    }

    // 下面（2.2的内容）
    for (BeanDefinition candidateComponent : candidateComponents) {
        if (candidateComponent instanceof AnnotatedBeanDefinition) {
            // verify annotated class is an interface
        AnnotatedBeanDefinition beanDefinition = (AnnotatedBeanDefinition) candidateComponent;
        AnnotationMetadata annotationMetadata = beanDefinition.getMetadata();
        
        // FeignClient只能为接口
        Assert.isTrue(annotationMetadata.isInterface(),
						"@FeignClient can only be specified on an interface");

        // 获得@FeignClient注解的元信息
        Map<String, Object> attributes = annotationMetadata
						.getAnnotationAttributes(FeignClient.class.getCanonicalName());

        // 拿到指定的服务名
        String name = getClientName(attributes);
        registerClientConfiguration(registry, name,
						attributes.get("configuration"));

        registerFeignClient(registry, annotationMetadata, attributes);
        }
    }
}
```
（2.1）来看看如何扫描FeignClient（其实这是Spring的逻辑）
```java
public Set<BeanDefinition> findCandidateComponents(String basePackage) {
    if (this.componentsIndex != null && indexSupportsIncludeFilters()) {
        return addCandidateComponentsFromIndex(this.componentsIndex, basePackage);
    }
    else {
        return scanCandidateComponents(basePackage);
    }
}

private Set<BeanDefinition> scanCandidateComponents(String basePackage) {
        Set<BeanDefinition> candidates = new LinkedHashSet<>();
        try {
            // 1. 拼接扫描路径：classpath*:com/zjh/**/*.class
            String packageSearchPath = ResourcePatternResolver.CLASSPATH_ALL_URL_PREFIX +
                resolveBasePackage(basePackage) + '/' + this.resourcePattern;
            // 2. 拿到路径下所有的.class文件（注意：这里不是类加载，而是文件流）
            Resource[] resources = getResourcePatternResolver().getResources(packageSearchPath);
            // 3. 遍历每一个.class文件
            for (Resource resource : resources) {
                if (resource.isReadable()) {
                    try {
                        // 3. ASM获取类元信息，包括类上有什么注解
                        MetadataReader metadataReader = getMetadataReaderFactory().getMetadataReader(resource);
                        // 4. 判断当前类上是否有@FeignClient注解
                        if (isCandidateComponent(metadataReader)) {
                            // 5. 如果有就创建当前FeignClient的beanDefinition
                            ScannedGenericBeanDefinition sbd = new ScannedGenericBeanDefinition(metadataReader);
                            sbd.setSource(resource);
                            // 5.1 判断当前类是否独立：即是否是顶层类或者静态内部类（因为我们的普通内部类也会是一个单独的.class文件，这里会过滤掉）
                            // 5.2 判断当前类是否是注解：如果是注解类也要过滤掉
                            if (isCandidateComponent(sbd)) {
                                // 6. 将当前FeignClient添加到候选集合中
                                candidates.add(sbd);
                            }
                        }
                    } catch(Exception ex) {}
                }
            }
        }
    // 返回找到的FeignClient的beanDefinition
    return candidates;
}
```
（2.2）在找到FeignClient之后，就要开始注册bean了
```java
// 遍历找到的FeignClient
for (BeanDefinition candidateComponent : candidateComponents) {
    if (candidateComponent instanceof AnnotatedBeanDefinition) {
        AnnotatedBeanDefinition beanDefinition = (AnnotatedBeanDefinition) candidateComponent;
        // 拿到当前FeignClient的注解元信息
        AnnotationMetadata annotationMetadata = beanDefinition.getMetadata();
        
        // 1. 验证找到的FeignClient是否是接口，如果不是就直接抛异常
        Assert.isTrue(annotationMetadata.isInterface(),
                "@FeignClient can only be specified on an interface");

        // 2. 拿到当前FeignClient接口上的@FeignClient注解属性
        Map<String, Object> attributes = annotationMetadata
                .getAnnotationAttributes(FeignClient.class.getCanonicalName());

        // 3. 拿到@FeignClient注解指定的服务名（name、value、serverId都可以）
        String name = getClientName(attributes);
        
        // 4. 拿到@FeignClient注解指定的配置文件，创建一个FeignClientSpecification.class的bean并将配置文件类保存在其中，再注册到SpringBoot容器中（和之前@FeignClients的逻辑一样）
        registerClientConfiguration(registry, name, attributes.get("configuration"));
        
        // 5. 注册当前FeignClient到SpringBoot容器中（重要）
        registerFeignClient(registry, annotationMetadata, attributes);
    }
}

// 注册当前FeignClient到SpringBoot容器中
private void registerFeignClient(BeanDefinitionRegistry registry,
        AnnotationMetadata annotationMetadata, Map<String, Object> attributes) {
    // 1. 拿到当前FeignClient的全类名并加载得到类
    String className = annotationMetadata.getClassName();
    Class clazz = ClassUtils.resolveClassName(className, null);
    
    ConfigurableBeanFactory beanFactory = registry instanceof ConfigurableBeanFactory
        ? (ConfigurableBeanFactory) registry : null;
    
    // 2. 拿到contextId，如果没有指定就调用getName()：拿到name、value
    String contextId = getContextId(beanFactory, attributes);
    
    // 3. 拿到当前@FeignClient注解中指定的服务名
    String name = getName(attributes);
    
    // 3. 创建一个FeignClientFactoryBean（其getObject()方法会创建当前FeignClient接口的代理对象）
    FeignClientFactoryBean factoryBean = new FeignClientFactoryBean();
    // 设置beanFactory
    factoryBean.setBeanFactory(beanFactory);
    
    // 设置服务名
    factoryBean.setName(name);
    // 设置contextId，作为服务名，之后拿OpenFeign容器的时候就会通过contextId去拿
    factoryBean.setContextId(contextId);
    
    // 设置当前FeignClient接口
    factoryBean.setType(clazz);
    
    // 4 创建一个BeanDefinitionBuilder，用于创建FeignClient接口的beanDefinition（注意：这里不是创建FeignClientFactoryBean的beanDefinition，并没有将FeignClientFactoryBean注册到容器中）
    BeanDefinitionBuilder definition = BeanDefinitionBuilder
        // lambda表达式是一个回调，会在创建FeignClient接口bean的时候，createBeanInstance()的第一步就调用（也就是说在回调里面调用getObject()）
        .genericBeanDefinition(clazz, () -> {
            // 4.1 设置@FeignClient注解中指定的URL（如果指定了就不会创建）
            factoryBean.setUrl(getUrl(beanFactory, attributes));
            // 4.2 设置@FeignClient注解中指定的path
            factoryBean.setPath(getPath(beanFactory, attributes));
            factoryBean.setDecode404(Boolean
            .parseBoolean(String.valueOf(attributes.get("decode404"))));
            // 拿到当前@FeignClient注解中指定的fallback（sentinel yyds）
            Object fallback = attributes.get("fallback");
            if (fallback != null) {
            factoryBean.setFallback(fallback instanceof Class
                ? (Class<?>) fallback
                : ClassUtils.resolveClassName(fallback.toString(), null));
            }
            Object fallbackFactory = attributes.get("fallbackFactory");
            if (fallbackFactory != null) {
            factoryBean.setFallbackFactory(fallbackFactory instanceof Class
                ? (Class<?>) fallbackFactory
                : ClassUtils.resolveClassName(fallbackFactory.toString(),
                null));
            }
            // 4.3 最重要的，调用getObject()方法
            return factoryBean.getObject();
        });
    // 设置按照类型自动装配
    definition.setAutowireMode(AbstractBeanDefinition.AUTOWIRE_BY_TYPE);
    // 设置懒加载（也就是说，只会在Service中依赖注入的时候才会创建（但是Service不是懒加载的啊！））
    definition.setLazyInit(true);
    validate(attributes);

    AbstractBeanDefinition beanDefinition = definition.getBeanDefinition();
    beanDefinition.setAttribute(FactoryBean.OBJECT_TYPE_ATTRIBUTE, className);
    beanDefinition.setAttribute("feignClientsRegistrarFactoryBean", factoryBean);

    // has a default, won't be null
    boolean primary = (Boolean) attributes.get("primary");

    beanDefinition.setPrimary(primary);

    String[] qualifiers = getQualifiers(attributes);
    if (ObjectUtils.isEmpty(qualifiers)) {
        qualifiers = new String[] { contextId + "FeignClient" };
    }

    BeanDefinitionHolder holder = new BeanDefinitionHolder(beanDefinition, className, qualifiers);
    // 5. 将当前@FeignClient的接口的beanDefinition注册到SpringBoot的beanFactory中
    BeanDefinitionReaderUtils.registerBeanDefinition(holder, registry);
}
```
到这里，整个registerBeanDefinitions的方法就执行完了，主要就是做了两件事：
1. 扫描指定路径或者配置类所在路径的所有标注了@FeignClient注解的类
2. 将@FeignClient注解标注的接口的beanDefinition注册到SpringBoot容器中，同时设置回调，在调用createBeanInstance()创建FeignClient实例的时候会触发回调，回调中会调用到FeignClientFactoryBean的getObject()方法

# 自动配置类（实例化非懒加载单实例bean时的逻辑）
OpenFeign包下的重要的自动配置类有两个，FeignRibbonClientAutoConfiguration和FeignAutoConfiguration
前者负责配置httpClient、LoadBalancerFeignClient和整合Ribbon，后者负责Feign的一些组件相关配置
FeignRibbonClientAutoConfiguration上有标注@AutoConfigureBefore(FeignAutoConfiguration.class)，因此FeignRibbonClientAutoConfiguration自动配置类先解析，因此先来看这个自动配置类
## FeignRibbonClientAutoConfiguration
自动配置类上的@Import注解向容器中导入了四个配置类，这四个配置类待会说，先看当前配置类的内容
```java
@ConditionalOnClass({ ILoadBalancer.class, Feign.class })
@ConditionalOnProperty(value = "spring.cloud.loadbalancer.ribbon.enabled",
		matchIfMissing = true)
@Configuration(proxyBeanMethods = false)
@AutoConfigureBefore(FeignAutoConfiguration.class) // 在FeignAutoConfiguration之前解析
@EnableConfigurationProperties({ FeignHttpClientProperties.class })
// @Import导入了四个配置类（注意排列的顺序）
// Order is important here, last should be the default, first should be optional
@Import({ HttpClientFeignLoadBalancedConfiguration.class,
		OkHttpFeignLoadBalancedConfiguration.class,
		HttpClient5FeignLoadBalancedConfiguration.class,
		DefaultFeignLoadBalancedConfiguration.class })
public class FeignRibbonClientAutoConfiguration {
    
    // 配置一个CachingSpringLoadBalancerFactory，配置的时候将Ribbon配置的SpringClientFactory添加了进去
    @Bean
    @Primary
    @ConditionalOnMissingBean
    @ConditionalOnMissingClass("org.springframework.retry.support.RetryTemplate")
    public CachingSpringLoadBalancerFactory cachingLBClientFactory(
            SpringClientFactory factory) {
        return new CachingSpringLoadBalancerFactory(factory);
    }

    // 如果我们没有配置Request.Options，就帮我们配置一个默认的Request.Options
    // 默认配置的Options中，连接超时时10s，请求超时时60s，同时允许请求重定向
    // 因此，如果我们要自己配置的话，可以自己在配置类中配置Request.Options指定超时时间（这样是全局配置）
    @Bean
    @ConditionalOnMissingBean
    public Request.Options feignRequestOptions() {
        // 默认配置的Options中，连接超时时10s，请求超时时60s，同时允许请求重定向
        return LoadBalancerFeignClient.DEFAULT_OPTIONS;
    }
    
}
```
再看FeignRibbonClientAutoConfiguration自动配置类上的@Import，向容器中导入了四个配置类，这四个配置类中都是向容器中导入一个LoadBalancerFeignClient
点击任意一个进去，例如HttpClientFeignLoadBalancedConfiguration，然后就又会看到一个@Import，这个主要是配置对应的HttpClient，下面的@Bean就是配置LoadBalancerFeignClient
要注意的是：@Bean方法还有一个条件注解@ConditionalOnMissingBean(Client.class)，这里面的Client是LoadBalancerFeignClient所实现的接口，如果容器中已经存在LoadBalancerFeignClient，则不会再配置
由于四个配置类都会配置LoadBalancerFeignClient，因此其顺序就非常重要了！这也是为什么源码中还有一行注释说顺序很重要的原因！如果我们即引入了ApacheHttpClient又引入了OKHttpClient，最终生效的只会是ApacheHttpClient（除非手动feign.httpclient.enabled=false指定不使用ApacheHttpClient，同时feign.okhttp.enabled默认为false，还需要手动设置为true）
```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnClass(ApacheHttpClient.class)
@ConditionalOnProperty(value = "feign.httpclient.enabled", matchIfMissing = true)
@Conditional(HttpClient5DisabledConditions.class) 
@Import(HttpClientFeignConfiguration.class) // 配置HttpClient
class HttpClientFeignLoadBalancedConfiguration {

    // 配置LoadBalancerFeignClient
	@Bean
	@ConditionalOnMissingBean(Client.class)
	public Client feignClient(CachingSpringLoadBalancerFactory cachingFactory,
			SpringClientFactory clientFactory, HttpClient httpClient) {
		ApacheHttpClient delegate = new ApacheHttpClient(httpClient);
		// 可以看到，LoadBalancerFeignClient中有httpClient、Feign的Factory和ribbon的Factory
		return new LoadBalancerFeignClient(delegate, cachingFactory, clientFactory);
	}

}

@Configuration(proxyBeanMethods = false)
@ConditionalOnClass(OkHttpClient.class)
@ConditionalOnProperty("feign.okhttp.enabled")
@Import(OkHttpFeignConfiguration.class)
class OkHttpFeignLoadBalancedConfiguration {

    @Bean
    @ConditionalOnMissingBean(Client.class)
    public Client feignClient(CachingSpringLoadBalancerFactory cachingFactory,
                              SpringClientFactory clientFactory, okhttp3.OkHttpClient okHttpClient) {
        OkHttpClient delegate = new OkHttpClient(okHttpClient);
        return new LoadBalancerFeignClient(delegate, cachingFactory, clientFactory);
    }

}
```
## FeignAutoConfiguration
org.springframework.cloud.openfeign.FeignAutoConfiguration
```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnClass(Feign.class)
// 开启配置属性
@EnableConfigurationProperties({ FeignClientProperties.class,
		FeignHttpClientProperties.class, FeignEncoderProperties.class })
@Import(DefaultGzipDecoderConfiguration.class)
public class FeignAutoConfiguration {

    // 拿到容器中注入的FeignClientSpecification（如果没在@EnableFeignClients和@FeignClient添加指定配置，FeignClientSpecification中的configuration = null（因此实际没什么意义））
    @Autowired(required = false)
    private List<FeignClientSpecification> configurations = new ArrayList<>();

    @Bean
    public HasFeatures feignFeature() {
        return HasFeatures.namedFeature("Feign", Feign.class);
    }

    // 创建一个FeignContext（和ribbon中的SpringClientFactory作用相同，为每一个服务都创建一个服务容器，实现服务配置隔离）
    @Bean
    public FeignContext feignContext() {
        // 构造方法会做一些初始化，内容如下：
        // super(FeignClientsConfiguration.class, "feign", "feign.client.name")
        // 1. 会指定FeignContext的defaultConfigType属性为FeignClientsConfiguration（会把这个类传入到服务容器中解析）
        // 2. 会指定propertySourceName = feign
        // 3. 会指定propertyName = feign.client.name（之后会以feign.client.name为key，服务名为value设置到服务容器的Environment中）
        FeignContext context = new FeignContext();
        // 设置FeignClientSpecification
        context.setConfigurations(this.configurations);
        return context;
    }

    @Configuration(proxyBeanMethods = false)
    @Conditional(DefaultFeignTargeterConditions.class)
    protected static class DefaultFeignTargeterConfiguration {

        // 配置一个Targeter，其target方法会创建代理对象
        // 注意：由于OpenFeign默认导入了hystrix的包，因此如果不手动的把feign.hystrix.enabled=false的话，默认会创建HystrixTargeter
        @Bean
        @ConditionalOnMissingBean
        public Targeter feignTargeter() {
            return new DefaultTargeter();
        }

    }

    @Configuration(proxyBeanMethods = false)
    @Conditional(FeignCircuitBreakerDisabledConditions.class)
    @ConditionalOnClass(name = "feign.hystrix.HystrixFeign")
    @ConditionalOnProperty(value = "feign.hystrix.enabled", havingValue = "true",
            matchIfMissing = true)
    protected static class HystrixFeignTargeterConfiguration {

        @Bean
        @ConditionalOnMissingBean
        public Targeter feignTargeter() {
            return new HystrixTargeter();
        }

    }
    // ...
}
```
到此整个配置类也解析完了，注意是配置类解析完了，bean还没有创建，接下来就是开始创建Bean
# FeignClientBean创建时的回调
之前在解析@FeignClients所在的配置类时，@FeignClients注解导入了一个ImportBeanDefinitionRegistrar（实际上会直接实例化并调用方法，不会放到容器中），其registerBeanDefinitions()方法中为每一个@FeignClient接口创建了一个beanDefinition并注册到容器中，同时还设置了一个InstanceSupplier，即创建bean实例的回调方法
Springboot刷新的过程中，就会创建bean，在执行到doCreateBean().CreateBeanInstance()时，第一步就会判断是否存在InstanceSupplier，如果存在就直接执行回调方法创建bean并返回，回调方法如下：
```java
FeignClientFactoryBean factoryBean = new FeignClientFactoryBean();

// 4 创建一个BeanDefinitionBuilder，用于创建FeignClient接口的beanDefinition（注意：这里不是创建FeignClientFactoryBean的beanDefinition，并没有将FeignClientFactoryBean注册到容器中）
BeanDefinitionBuilder definition = BeanDefinitionBuilder
    // lambda表达式是一个回调，会在创建FeignClient接口bean的时候，createBeanInstance()的第一步就调用（也就是说在回调里面调用getObject()）
    .genericBeanDefinition(clazz, () -> {
    // 4.1 设置@FeignClient注解中指定的URL（如果指定了就不会创建）
    factoryBean.setUrl(getUrl(beanFactory, attributes));
    // 4.2 设置@FeignClient注解中指定的path
    factoryBean.setPath(getPath(beanFactory, attributes));
    factoryBean.setDecode404(Boolean.parseBoolean(String.valueOf(attributes.get("decode404"))));
    // 拿到当前@FeignClient注解中指定的fallback（sentinel yyds）
    Object fallback = attributes.get("fallback");
    if (fallback != null) {
    factoryBean.setFallback(fallback instanceof Class
        ? (Class<?>) fallback : ClassUtils.resolveClassName(fallback.toString(), null));
    }
    Object fallbackFactory = attributes.get("fallbackFactory");
    if (fallbackFactory != null) {
        factoryBean.setFallbackFactory(fallbackFactory instanceof Class
        ? (Class<?>) fallbackFactory : ClassUtils.resolveClassName(fallbackFactory.toString(),null));
    }
    // 4.3 最重要的，调用getObject()方法
    return factoryBean.getObject();
});
```
# getObject()
注意此时的FeignClientFactoryBean#getObject()方法的调用是在创建FeignClient这个SpringBoot容器Bean的时候，也就是容器正在刷新创建非懒加载单实例Bean的时候
```java
// FeignClientFactoryBean.java
@Override
public Object getObject() {
    return getTarget();
}

<T> T getTarget() {
    // 从SpringBoot容器中拿到FeignContext类型的bean（FeignAutoConfiguration配置的）
    // 此时去beanFactory拿的时候，如果FeignContext还没有实例化就会提前触发实例化（实例化会制定FeignContext的defaultConfigType属性为FeignClientsConfiguration（Fegin服务容器的配置类），会注入到Feign的服务容器中）
    // （1）会讲实例化的细节
    FeignContext context = beanFactory.getBean(FeignContext.class)

    // 创建一个Builder，用于为接口创建JDK动态代理对象
    // （2）（3）会讲实例化和初始化细节
    Feign.Builder builder = feign(context);

    // 如果没有在@FeignClient指定url，就会创建具备负载均衡（Ribbon）的代理对象，利用注册中心获取服务节点地址
    if (!StringUtils.hasText(url)) {
        
        // 开始拼接url，name = 服务名
        if (!name.startsWith("http")) {
            // 添加http://
            url = "http://" + name; 
        }
        else {
            url = name;
        }

        // 拿到我们在@FeignClient中指定的path，拼接在后面
        url += cleanPath();
        
        // 到这里，URL = http://name/path
        
        // 为FeignClient接口创建代理对象（4）单独讲
        return (T) loadBalance(builder, context,
            new HardCodedTarget<>(type, name, url));
    }
    // 后面是手动在@FeignClient指定URL的逻辑，我们一般会使用注册中心
}
```
（1）先来看看FeignContext实例化的细节
```java
// 创建一个FeignContext（和ribbon中的SpringClientFactory作用相同，为每一个服务都创建一个服务容器，实现服务配置隔离）
@Bean
public FeignContext feignContext() {
    // 构造方法会做一些初始化，内容如下：
    // super(FeignClientsConfiguration.class, "feign", "feign.client.name")
    // 1. 会指定FeignContext的defaultConfigType属性为FeignClientsConfiguration（Fegin服务容器的配置类）
    // 2. 会指定propertySourceName = feign
    // 3. 会指定propertyName = feign.client.name（之后会以feign.client.name为key，服务名为value设置到服务容器的Environment中）
    FeignContext context = new FeignContext();
    
    // 设置FeignClientSpecification，如果没在@EnableFeignClients和@FeignClient添加指定配置，FeignClientSpecification中的configuration = null（因此实际没什么意义）
    context.setConfigurations(this.configurations);
    return context;
}
```
其次，FeignContext继承了NamedContextFactory，而NamedContextFactory又实现了ApplicationContextAware，因此会在FeignContext初始化的时候调用其setApplicationContext将SpringBoot容器注入到FeignContext中

（2）再来看看Feign.Builder实例化的细节，该builder会在刷新OpenFeign子容器的时候创建
```java
public Builder() {
    // 1. 默认日志级别为NONE
    this.logLevel = Level.NONE;
    // 契约会被覆盖成SpringMVCContract（创建服务容器时，注册到服务容器的FeignClientsConfiguration配置类中有配置）
    this.contract = new Default();
    this.client = new feign.Client.Default((SSLSocketFactory)null, (HostnameVerifier)null);
    // retryer会被覆盖成Retryer.NEVER_RETRY（创建服务容器时，注册到服务容器的FeignClientsConfiguration配置类中有配置）
    this.retryer = new feign.Retryer.Default();
    this.logger = new NoOpLogger();
    // 编解码器会被覆盖（创建服务容器时，注册到服务容器的FeignClientsConfiguration配置类中有配置）
    this.encoder = new feign.codec.Encoder.Default();
    this.decoder = new feign.codec.Decoder.Default();
    this.queryMapEncoder = new FieldQueryMapEncoder();
    this.errorDecoder = new feign.codec.ErrorDecoder.Default();
    // 默认会使用10s连接超时，60s请求超时并允许请求转发
    this.options = new Options();
    this.invocationHandlerFactory = new feign.InvocationHandlerFactory.Default();
    this.closeAfterDecode = true;
    this.propagationPolicy = ExceptionPropagationPolicy.NONE;
    this.forceDecoding = false;
    this.capabilities = new ArrayList();
}
```

（3）再来看看Feign.Builder初始化的细节
```java
protected Feign.Builder feign(FeignContext context) {
    // 1. 从context中拿到FeignLoggerFactory（注意，这个context不是容器，而是FeignContext），这里就会为当前服务创建OpenFeign的子容器
    // get(context, xxx.class)方法很重要，先跳到下一大节
    FeignLoggerFactory loggerFactory = get(context, FeignLoggerFactory.class);
    
    // 创建一个Logger，上一步服务容器在创建FeignLoggerFactory的时候就往其中注入了一个Logger
    // 如果Logger = null，就会new Slf4jLogger(type)并返回（type是之前创建FeignClientFactoryBean的时候设置进来的，即@FeignClient接口）
    Logger logger = loggerFactory.create(type);

    // 2. 从服务容器中拿到Feign.Builder，并为Builder设置各种属性
    // @formatter:off
    Feign.Builder builder = get(context, Feign.Builder.class)
            // required values（先把一些必须的属性设置好，如果我们有自己的配置，下面的configureFeign中会覆盖）
            
            // 给Feign.Builder中设置logger
            .logger(logger) 
        
            // 从服务容器中拿到Encoder设置到Feign.Builder中
            .encoder(get(context, Encoder.class))
        
            // 从服务容器中拿到Decoder设置到Feign.Builder中
            .decoder(get(context, Decoder.class))
        
            // 从服务容器中拿到Contract设置到Feign.Builder中
            .contract(get(context, Contract.class));
    // @formatter:on

    // 3. 配置builder会拿到我们在父容器（先从服务容器中拿，没拿到再去父容器）中自己的配置并注入到builder中（下方会有解释）
    // 包括Logger.Level、Retryer、ErrorDecoder、Request.Options、RequestInterceptor、QueryMapEncoder和ExceptionPropagationPolicy注入到Builder中
    configureFeign(context, builder);
    
    // 如果我们自己有配置FeignBuilderCustomizer并重写customize()方法，在这里会对builder再进行配置
    applyBuildCustomizers(context, builder);

    // 返回builder
    return builder;
}

protected void configureFeign(FeignContext context, Feign.Builder builder) {
    // 这里beanFactory != null，因此会从SpringBoot容器中拿到FeignClientProperties，即Feign在yaml中的配置
    FeignClientProperties properties = beanFactory != null
        ? beanFactory.getBean(FeignClientProperties.class)
        : applicationContext.getBean(FeignClientProperties.class);

    // 拿到服务容器中配置的FeignClientConfigurer
    FeignClientConfigurer feignClientConfigurer = getOptional(context, FeignClientConfigurer.class);
    // 参数默认为true，设置FeignClientFactoryBean的inheritParentContext属性 = true，待会找组件的时候就会去父容器也就是SpringBoot容器中找
    setInheritParentContext(feignClientConfigurer.inheritParentConfiguration());

    // 成立
    if (properties != null && inheritParentContext) {
        // 默认为true
        if (properties.isDefaultToProperties()) {
            // 从服务容器（及其父容器）中拿到我们配置的Logger.Level、RequestInterceptor和Request.Options等信息，设置到builder中
            configureUsingConfiguration(context, builder);
            // config = null，而且我没有找到任何地方给这个config赋了值，因此方法会直接返回
            configureUsingProperties(
                properties.getConfig().get(properties.getDefaultConfig()),
                builder);
            // 同上
            configureUsingProperties(properties.getConfig().get(contextId), builder);
        }
        else {
            configureUsingProperties(
                properties.getConfig().get(properties.getDefaultConfig()),
                builder);
            configureUsingProperties(properties.getConfig().get(contextId), builder);
            configureUsingConfiguration(context, builder);
        }
    }
    else {
        configureUsingConfiguration(context, builder);
    }
}
```

（4）进入loadBalance()
```java
// FeignClientFactoryBean.java
// target为type（FeignClient接口类） + name（服务名） + URL（http://name/path）的封装类
protected <T> T loadBalance(Feign.Builder builder, FeignContext context,
        HardCodedTarget<T> target) {
    
    // 从服务容器中拿到LoadBalancerFeignClient，包含三个属性
    // delegate = HTTPClient
    // lbClientFactory = CachingSpringLoadBalancerFactory
    // clientFacotry = SpringClientFactory（Ribbon的组件）
    Client client = getOptional(context, Client.class);
    
    if (client != null) {
        // 将client设置到builder中
        builder.client(client);
        
        // 从SpringBoot容器中拿到Targeter，feign默认导入了Hystrix包，在手动配置关闭hystrix的情况下，拿到的是DefaultTargeter
        Targeter targeter = get(context, Targeter.class);
        
        // 调用DefaultTargeter.tartget()
        return targeter.target(this, builder, context, target);
    }

    throw new IllegalStateException(
            "No Feign Client for loadBalancing defined. Did you forget to include spring-cloud-starter-netflix-ribbon or spring-cloud-starter-loadbalancer?");
}

// DefaultTargeter implements Targeter
@Override
public <T> T target(FeignClientFactoryBean factory, Feign.Builder feign,
    FeignContext context, Target.HardCodedTarget<T> target) {
    // 其实可以看到，虽然传入了这么多参数，只会用到两个，一个builder，一个target为type（FeignClient接口类） + name（服务名） + URL（http://name/path）的封装类
    // 其它参数只会在HystrixTargeter用到
    return feign.target(target);
}

// Feign.builder
public <T> T target(Target<T> target) {
    // 下面（1）会讲build()方法，其实就是对builder中的信息进行封装，最终保存ReflectiveFeign中并返回
    // 下面（2）会讲newInstance方法，创建代理对象
    return this.build().newInstance(target);
}

// （1）先进入Feign.builder#build()方法
public Feign build() {
    // capabilities为空，因此都直接返回原来的对象
    Client client = (Client)Capability.enrich(this.client, this.capabilities);
    Retryer retryer = (Retryer)Capability.enrich(this.retryer, this.capabilities);
    List<RequestInterceptor> requestInterceptors = (List)this.requestInterceptors.stream().map((ri) -> {
            return (RequestInterceptor)Capability.enrich(ri, this.capabilities);
        }).collect(Collectors.toList());
    Logger logger = (Logger)Capability.enrich(this.logger, this.capabilities);
    Contract contract = (Contract)Capability.enrich(this.contract, this.capabilities);
    Options options = (Options)Capability.enrich(this.options, this.capabilities);
    Encoder encoder = (Encoder)Capability.enrich(this.encoder, this.capabilities);
    Decoder decoder = (Decoder)Capability.enrich(this.decoder, this.capabilities);
    InvocationHandlerFactory invocationHandlerFactory = (InvocationHandlerFactory)Capability.enrich(this.invocationHandlerFactory, this.capabilities);
    QueryMapEncoder queryMapEncoder = (QueryMapEncoder)Capability.enrich(this.queryMapEncoder, this.capabilities);
    
    // 讲以上信息进行封装
    Factory synchronousMethodHandlerFactory = new Factory(client, retryer, requestInterceptors, logger, this.logLevel, this.decode404, this.closeAfterDecode, this.propagationPolicy, this.forceDecoding);
    ParseHandlersByName handlersByName = new ParseHandlersByName(contract, options, encoder, decoder, queryMapEncoder, this.errorDecoder, synchronousMethodHandlerFactory);
    
    // 创建一个ReflectiveFeign
    return new ReflectiveFeign(handlersByName, invocationHandlerFactory, queryMapEncoder);
}

// （2）再进入newInstance()方法
@Override
public <T> T newInstance(Target<T> target) {
    // 以springmvc的格式解析当前feignClient接口得到所有方法及其注解信息，再将当前builder中的信息和解析到的信息进行封装得到一个个的MethodHandler
    Map<String, MethodHandler> nameToHandler = targetToHandlersByName.apply(target);
    // 保存普通方法和其MehtodHandler
    Map<Method, MethodHandler> methodToHandler = new LinkedHashMap<Method, MethodHandler>();
    // 保存接口中默认方法及其MethodHandler
    List<DefaultMethodHandler> defaultMethodHandlers = new LinkedList<DefaultMethodHandler>();

    // 遍历接口中的每一个方法，将方法和该方法的MethodHandler对应起来保存到methodToHandler中
    for (Method method : target.type().getMethods()) {
        if (method.getDeclaringClass() == Object.class) {
            continue;
        } else if (Util.isDefault(method)) {
            DefaultMethodHandler handler = new DefaultMethodHandler(method);
            defaultMethodHandlers.add(handler);
            methodToHandler.put(method, handler);
        } else {
            methodToHandler.put(method, nameToHandler.get(Feign.configKey(target.type(), method)));
        }
    }
    // factory = InvocationHandlerFactory，这里将target和methodToHandler封装成一个InvocationHandler（分别对应tartget和dispatch属性）
    InvocationHandler handler = factory.create(target, methodToHandler);
    
    // JDK动态代理为接口创建代理对象
    T proxy = (T) Proxy.newProxyInstance(target.type().getClassLoader(),
        new Class<?>[] {target.type()}, handler);

    // 添加DefaultMethodHandler到代理对象中，我们一般也不会在FeignClient接口中定义默认方法
    for (DefaultMethodHandler defaultMethodHandler : defaultMethodHandlers) {
        defaultMethodHandler.bindTo(proxy);
    }
    // 返回代理对象
    return proxy;
}
```
此时，整个getObject()方法就调用结束了，再往后就是等待容器全部刷新完，刷新的过程中还会去创建Ribbon的各种组件，之后容器刷新结束，项目就启动了，就只需要等待外界调用controller即可，controller中我们就会调用feignClient代理对象的方法，就会进入到InvocationHandler中的invoke()方法

# FeignContext.getInstance()
在getObject()方法的执行过程中，会多次执行get(context, xxx.class)，这个方法就是创建、刷新服务容器以及从服务容器中拿bean的逻辑
```java
protected <T> T get(FeignContext context, Class<T> type) {
    // 调用FeignContext.getInstance()方法，拿到指定type类型的bean
    // 注意这里传入了contextId = 服务名 = 创建FeignClientFactoryBean时注入进来的从@FeignClient中拿到的服务名
    T instance = context.getInstance(contextId, type);
    if (instance == null) {
        throw new IllegalStateException(
                "No bean found of type " + type + " for " + contextId);
    }
    return instance;
}

// 注意这里已经进入到FeignContext bean中的getInstance()方法了
public <T> T getInstance(String name, Class<T> type) {
    // 可以看到，先通过服务名拿到一个容器（注意这个容器不是SpringBoot容器，而是SpringMVC使用过的容器）
    AnnotationConfigApplicationContext context = getContext(name);
    try {
        // 通过类型去服务容器中找到bean
        return context.getBean(type);
    }
    catch (NoSuchBeanDefinitionException e) {
        // ignore
    }
    return null;
}

protected AnnotationConfigApplicationContext getContext(String name) {
    if (!this.contexts.containsKey(name)) {
        synchronized (this.contexts) { // 双重检测锁
            if (!this.contexts.containsKey(name)) {
                // 如果通过name服务名没有拿到容器，就调用createContext(name)创建一个服务容器
                this.contexts.put(name, createContext(name)); 
            }
        }
    }
    // 可以看到，这里通过name服务名去当前FeignContext中拿到容器，如果没有上面还会调用createContext创建
    return this.contexts.get(name);
}
```
可以看到，getContext()中的逻辑就是先从this.contexts中根据服务名拿到服务容器，拿到就直接返回，如果没拿到就为当前服务创建一个容器
那这个this.contexts又是个什么呢？我们来看一下FeignContext的属性（父类中）
```java
public abstract class NamedContextFactory<C extends NamedContextFactory.Specification>
		implements DisposableBean, ApplicationContextAware {

    // 配置FeignContext时，new FeignContext()无参构造方法中赋值 = feign
    private final String propertySourceName;

    // 配置FeignContext时，new FeignContext()无参构造方法中赋值 = feign.client.name
    private final String propertyName;

    // 以服务名为key，服务所对应的容器为value的map（也就是说，一个服务一个服务容器）
    private Map<String, AnnotationConfigApplicationContext> contexts = new ConcurrentHashMap<>();

    // 配置FeignContext时，new完FeignContext之后，调用setConfigurations()注入进来的FeignClientSpecification
    private Map<String, C> configurations = new ConcurrentHashMap<>();

    // 通过实现ApplicationContextAware接口，在实例化FeignContext时注入进来的SpringBoot容器（看名字也能猜到肯定会作为服务容器的父容器）
    private ApplicationContext parent;

    // 配置FeignContext时，new FeignContext()无参构造方法中赋值 = FeignClientsConfiguration（服务容器的配置类）
    private Class<?> defaultConfigType;
}
```
contexts.get(name)就是调用map.get()，因此**最重要的就是createContext(name)**
```java
protected AnnotationConfigApplicationContext createContext(String name) {
    // 创建一个容器（注意：使用的无参构造函数，现在没有刷新）
    AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext();
    // 拿到configurations即FeignClientSpecification，将FeignClientSpecification中的Configuration注册到服务容器中
    if (this.configurations.containsKey(name)) {
        for (Class<?> configuration : this.configurations.get(name)
                .getConfiguration()) {
            context.register(configuration);
        }
    }
    // 将所有名字以default.开头的FeignClientSpecification中的Configuration注册到服务容器中
    // Ribbon会在这里注入。。。忘记了，以后填坑
    for (Map.Entry<String, C> entry : this.configurations.entrySet()) {
        if (entry.getKey().startsWith("default.")) {
            for (Class<?> configuration : entry.getValue().getConfiguration()) {
                context.register(configuration);
            }
        }
    }
    // 注册defaultConfigType -> FeignClientsConfiguration到服务容器中
    context.register(PropertyPlaceholderAutoConfiguration.class,
            this.defaultConfigType);
    // 将key = propertyName = feign.client.name，value = nam = 服务名注册到服务容器的Environment中
    context.getEnvironment().getPropertySources().addFirst(new MapPropertySource(
            this.propertySourceName,
            Collections.<String, Object>singletonMap(this.propertyName, name)));
    // 设置SpringBoot容器为服务容器的父容器
    if (this.parent != null) {
        // Uses Environment from parent as well as beans
        context.setParent(this.parent);
        // jdk11 issue
        // https://github.com/spring-cloud/spring-cloud-netflix/issues/3101
        context.setClassLoader(this.parent.getClassLoader());
    }
    context.setDisplayName(generateDisplayName(name));
    // 刷新容器（容器刷新的过程中就会解析刚才注册到容器中的配置类，例如FeignClientsConfiguration，将配置类中配置的bean实例化并保存在服务容器中）
    context.refresh();
    // 返回容器
    return context;
}
```
服务容器刷新的过程中就会解析FeignClientsConfiguration，我们来看配置类中帮我们配置了哪些bean
```java
@Configuration(proxyBeanMethods = false)
public class FeignClientsConfiguration {

    @Autowired
    private ObjectFactory<HttpMessageConverters> messageConverters;

    @Autowired(required = false)
    private List<AnnotatedParameterProcessor> parameterProcessors = new ArrayList<>();

    @Autowired(required = false)
    private List<FeignFormatterRegistrar> feignFormatterRegistrars = new ArrayList<>();

    @Autowired(required = false)
    private Logger logger;

    @Autowired(required = false)
    private SpringDataWebProperties springDataWebProperties;

    @Autowired(required = false)
    private FeignClientProperties feignClientProperties;

    @Autowired(required = false)
    private FeignEncoderProperties encoderProperties;

    // 如果服务容器（及其父容器）中没有Decoder就创建一个默认的Decoder（因此我们如果手动配置了，就不会再创建）
    @Bean
    @ConditionalOnMissingBean
    public Decoder feignDecoder() {
        return new OptionalDecoder(
                new ResponseEntityDecoder(new SpringDecoder(this.messageConverters)));
    }

    // 同上
    @Bean
    @ConditionalOnMissingBean
    @ConditionalOnMissingClass("org.springframework.data.domain.Pageable")
    public Encoder feignEncoder(ObjectProvider<AbstractFormWriter> formWriterProvider) {
        return springEncoder(formWriterProvider, encoderProperties);
    }

    private Encoder springEncoder(ObjectProvider<AbstractFormWriter> formWriterProvider,
                                  FeignEncoderProperties encoderProperties) {
        AbstractFormWriter formWriter = formWriterProvider.getIfAvailable();

        if (formWriter != null) {
            return new SpringEncoder(new SpringPojoFormEncoder(formWriter),
                    this.messageConverters, encoderProperties);
        } else {
            return new SpringEncoder(new SpringFormEncoder(), this.messageConverters,
                    encoderProperties);
        }
    }

    // 如果服务容器（及其父容器）中没有配置Contract，就帮我们默认配置一个SpringMvcContract（这也是为什么默认是SpringMVC契约配置的原因）
    @Bean
    @ConditionalOnMissingBean
    public Contract feignContract(ConversionService feignConversionService) {
        boolean decodeSlash = feignClientProperties == null
                || feignClientProperties.isDecodeSlash();
        return new SpringMvcContract(this.parameterProcessors, feignConversionService,
                decodeSlash);
    }

    @Bean
    public FormattingConversionService feignConversionService() {
        FormattingConversionService conversionService = new DefaultFormattingConversionService();
        for (FeignFormatterRegistrar feignFormatterRegistrar : this.feignFormatterRegistrars) {
            feignFormatterRegistrar.registerFormatters(conversionService);
        }
        return conversionService;
    }

    // 如果服务容器（及其父容器）中没有配置Retryer，就帮我们配置一个Retryer的实现NEVER_RETRY，也就是默认不失败重试（幂等）
    @Bean
    @ConditionalOnMissingBean
    public Retryer feignRetryer() {
        return Retryer.NEVER_RETRY;
    }

    // 如果服务容器（及其父容器）中没有配置Feign.Builder，就帮我们创建一个Feign.Builder同时指定重试策略
    @Bean
    @Scope("prototype")
    @ConditionalOnMissingBean
    public Feign.Builder feignBuilder(Retryer retryer) {
        return Feign.builder().retryer(retryer);
    }

    // 如果服务容器（及其父容器）中没有配置FeignLoggerFactory，就帮我们配置一个DefaultFeignLoggerFactory
    @Bean
    @ConditionalOnMissingBean(FeignLoggerFactory.class)
    public FeignLoggerFactory feignLoggerFactory() {
        return new DefaultFeignLoggerFactory(this.logger);
    }

    // 如果服务容器（及其父容器）中没有配置FeignClientConfigurer，就帮我们配置一个FeignClientConfigurer
    @Bean
    @ConditionalOnMissingBean(FeignClientConfigurer.class)
    public FeignClientConfigurer feignClientConfigurer() {
        return new FeignClientConfigurer() {
        };
    }

    
    // ... hystrix的相关配置
}
```

# invoke()
> 先看methodHandler.invoke()
```java
@Override
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
  // 如果是equals等方法就直接调用原始方法
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

  // 根据当前调用的method，拿到methodHandler，再调用methodHandler的invoke()方法，并传入参数
  return dispatch.get(method).invoke(args);
}

// MehtodHandler.invoke()
@Override
public Object invoke(Object[] argv) throws Throwable {
    // 传入参数得到一个RequestTemplate对象，这个方法就会根据参数给URL或者请求体赋值
    RequestTemplate template = buildTemplateFromArgs.create(argv);
    // 拿到options，要么是默认配置的，要么就是我们配置的
    Options options = findOptions(argv);
    // 拿到retryer，默认是不重试的
    Retryer retryer = this.retryer.clone();
    
    // while true 来实现失败重试（幂等）
    while (true) {
        try {
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

// methodHandler.executeAndDecode()
Object executeAndDecode(RequestTemplate template, Options options) throws Throwable {
    // 1. 遍历每一个RequestInterceptor，调用其apply(template)
    // 2. Request.create(this.method, this.url(), this.headers(), this.body, this) 根据HTTP请求的格式，创建一个Request（这里请求行中的HTTP协议版本默认写死为1.1）
    Request request = targetRequest(template);

    // 打印日志，里面会根据我们配置的Level有选择的打印具体日志（就是一些if判断）
    if (logLevel != Logger.Level.NONE) {
        logger.logRequest(metadata.configKey(), logLevel, request);
    }

    Response response;
    long start = System.nanoTime();
    try {
        // 调用LoadBalancerFeignClient.execute()
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


    if (decoder != null)
        return decoder.decode(response, metadata.returnType());

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
> 再看进入到LoadBalancerFeignClient.execute()
```java
@Override
public Response execute(Request request, Request.Options options) throws IOException {
    try {
        // 拿到URI
        URI asUri = URI.create(request.url());
        // 拿到服务名
        String clientName = asUri.getHost();
        // 将URI中的服务名删除 -> http:///path/xxx
        URI uriWithoutHost = cleanUrl(request.url(), clientName);
        
        // 又封装
        FeignLoadBalancer.RibbonRequest ribbonRequest = new FeignLoadBalancer.RibbonRequest(
                this.delegate, request, uriWithoutHost);

        // 拿到请求的配置信息，这里传入了OpenFeign的options
        // 1. 如果options = DEFAULT_OPTIONS（也就是我们不去配置，使用默认配置的情况下），会去ribbon中服务容器中拿IClientConfig
        // 2. 如果配置了options，就直接使用我们配置的options，包括连接超时、请求超时以及是否允许请求转发等（这也是openFeign的配置会覆盖Ribbon配置的原因）
        IClientConfig requestConfig = getClientConfig(options, clientName);
        
        // lbClient(clientName) -> this.lbClientFactory.create(clientName) -> CachingSpringLoadBalancerFactory.create(clientName) -> 下面（1）会讲
        // executeWithLoadBalancer -> 1.通过ribbon拿到一个服务节点信息 2.通过httpClient调用 -> 下面（2）会讲
        return lbClient(clientName)
                .executeWithLoadBalancer(ribbonRequest, requestConfig).toResponse();
    }
    catch (ClientException e) {
        IOException io = findIOException(e);
        if (io != null) {
            throw io;
        }
        throw new RuntimeException(e);
    }
}
```
（1）先来看获取FeignLoadBalancer的过程：即lbClient(clientName)，最终是调用到了FeignRibbonClientAutoConfiguration中自动配置的CachingSpringLoadBalancerFactory
```java
public FeignLoadBalancer create(String clientName) {
    // 先根据服务名取缓存中拿，拿到就返回，没拿到就创建
    FeignLoadBalancer client = this.cache.get(clientName);
    if (client != null) {
        return client;
    }
    
    // 从ribbon的服务容器中拿到IClientConfig
    IClientConfig config = this.factory.getClientConfig(clientName);
    
    // 从ribbon的服务容器中拿到ILoadBalancer -> 即ZoneAwareLoadBalancer
    ILoadBalancer lb = this.factory.getLoadBalancer(clientName);

    // 从ribbon的服务容器中拿到serverIntrospector
    ServerIntrospector serverIntrospector = this.factory.getInstance(clientName,
            ServerIntrospector.class);
    
    // ribbon不会创建loadBalancedRetryFactory（LoadBalancer才会），这里直接创建一个FeignLoadBalancer
    client = this.loadBalancedRetryFactory != null
            ? new RetryableFeignLoadBalancer(lb, config, serverIntrospector,
                    this.loadBalancedRetryFactory)
            : new FeignLoadBalancer(lb, config, serverIntrospector);
    
    // 将FeignLoadBalancer放到缓存中（因此这里缓存的目的主要是解决拿几个从ribbon容器中找bean的时间）
    this.cache.put(clientName, client);
    return client;
}
```
（2）再看executeWithLoadBalancer
这里的代码很多，主要是看submit()中的selectServer()方法和下方lambda表达式中根据server替换的execute()方法
```java
public T executeWithLoadBalancer(final S request, final IClientConfig requestConfig) throws ClientException {
        LoadBalancerCommand<T> command = buildLoadBalancerCommand(request, requestConfig);

    try {
        // 1. submit中会调用selectServer()拿到一个服务节点（server），然后再以服务节点为参数调用下面的call方法
        return command.submit( 
            new ServerOperation<T>() {
                @Override
                public Observable<T> call(Server server) {
                    // 2. 拿到server中的ip和port，设置到原来的URL中得到最终的URL
                    URI finalUri = reconstructURIWithServer(server, request.getUri());
                    // 3. 将最终的URL设置到request中
                    S requestForServer = (S) request.replaceUri(finalUri);
                    
                    try {
                        // 4. 执行FeignLoadBalancer.execute()方法
                        return Observable.just(AbstractLoadBalancerAwareClient.this.execute(requestForServer, requestConfig));
                    } 
                    catch (Exception e) {
                        return Observable.error(e);
                    }
                }
            })
            .toBlocking()
            .single();
    } catch (Exception e) {
        Throwable t = e.getCause();
        if (t instanceof ClientException) {
            throw (ClientException) t;
        } else {
            throw new ClientException(e);
        }
    }
    
}

// 1. selectServer() -> FeignLoadBalancer.getServerFromLoadBalancer()
public Server getServerFromLoadBalancer(@Nullable URI original, @Nullable Object loadBalancerKey) throws ClientException {
    
    // 拿到负载均衡器，即Ribbon的ZoneAwareLoadBalancer（之前创建FeignLoadBalancer时从Ribbon服务容器中拿到的）
    ILoadBalancer lb = getLoadBalancer();
    
    if (lb != null){
        // 后面就是Ribbon的逻辑了，注意，这里传入的loadBalancerKey = null，不是服务的名字，ZoneAwareLoadBalancer是在Ribbon的服务容器里面的，本身就已经对应了一个服务
        Server svc = lb.chooseServer(loadBalancerKey);
        host = svc.getHost();
        if (host == null){ // 如果没拿到就抛异常
            throw new ClientException(ClientException.ErrorType.GENERAL,
                "Invalid Server for :" + svc);
        }
        return svc;
    } else {
        // ...
    }
}

// 4. 执行FeignLoadBalancer.execute()方法
@Override
public RibbonResponse execute(RibbonRequest request, IClientConfig configOverride) throws IOException {
    Request.Options options;
    
    // configOverride就是之前在LoadBalancerFeignClient中执行execute()方法时创建的IClientConfig
    // 如果我们没有配置Options，那就用ribbon的IClientConfig，否则就是用我们自己的配置的Options
    if (configOverride != null) {
        RibbonProperties override = RibbonProperties.from(configOverride);
        // 从configOverride中拿出参数添加到options中（emmmm。。。如果我们配置了Options为啥不直接用了，替换过来又替换过去。。。。）
        options = new Request.Options(override.connectTimeout(connectTimeout),
        TimeUnit.MILLISECONDS, override.readTimeout(readTimeout),
        TimeUnit.MILLISECONDS, override.isFollowRedirects(followRedirects));
    }
    else {
        options = new Request.Options(connectTimeout, TimeUnit.MILLISECONDS,
            readTimeout, TimeUnit.MILLISECONDS, followRedirects);
    }
    
    // 这里的request = 之前创建的RibbonRequest，创建的时候往里面设置了HTTPClient
    Response response = request.client().execute(request.toRequest(), options);
    return new RibbonResponse(request.getUri(), response);
}
```
# ApacheHttpClient.execute()
这里我们单独看看ApacheHttpClient是如何与Feign整合并最终调用的
```java
@Override
public Response execute(Request request, Request.Options options) throws IOException {
    HttpUriRequest httpUriRequest;
    try {
        // 将OpenFeign的reqeust转换为ApacheHttpClient的Request
        httpUriRequest = toHttpUriRequest(request, options);
    } catch (URISyntaxException e) {
    throw new IOException("URL '" + request.url() + "' couldn't be parsed into a URI", e);
    }
    // HTTP的内容此时已经准备好了，只需要按照HTTP的规范通过socket将请求行、请求头、请求体发送过去即可
    HttpResponse httpResponse = client.execute(httpUriRequest);
    return toFeignResponse(httpResponse, request);
}

（1）toHttpUriRequest()，看看是如何从Feign的Request转换为ApacheHttpClient的Request的 
HttpUriRequest toHttpUriRequest(Request request, Request.Options options) throws URISyntaxException {
    // 创建一个RequestBuilder，里面会指定Request的请求类型，并且默认字符集为utf8
    // 1. 设置method
    RequestBuilder requestBuilder = RequestBuilder.create(request.httpMethod().name());

    // 创建RequestConfig并设置本次请求的连接超时和请求超时（从Options中拿）
    RequestConfig requestConfig =
        (client instanceof Configurable ? RequestConfig.copy(((Configurable) client).getConfig()) : RequestConfig.custom())
        .setConnectTimeout(options.connectTimeoutMillis())
        .setSocketTimeout(options.readTimeoutMillis())
        .build();
    
    // 将连接超时和请求超时设置到requestBuilder中
    requestBuilder.setConfig(requestConfig);

    // 拿到URI（包含了请求参数  ?param=xxx）
    URI uri = new URIBuilder(request.url()).build();

    // 2. 设置URI（不包含请求参数）
    requestBuilder.setUri(uri.getScheme() + "://" + uri.getAuthority() + uri.getRawPath());

    // 3. 拿到请求参数并设置到builder中
    List<NameValuePair> queryParams = URLEncodedUtils.parse(uri, requestBuilder.getCharset());
    for (NameValuePair queryParam : queryParams) {
        requestBuilder.addParameter(queryParam);
    }

    // 4. 拿到请求头
    boolean hasAcceptHeader = false;
    for (Map.Entry<String, Collection<String>> headerEntry : request.headers().entrySet()) {
        String headerName = headerEntry.getKey();
        
        // 判断是否有指定Accept，如果没有后面就会设置通配符（我全都要）
        if (headerName.equalsIgnoreCase(ACCEPT_HEADER_NAME)) {
            hasAcceptHeader = true;
        }

        // 判断是否有指定content-length，如果有指定直接跳过（也就是说我们指定是没用的，ApacheHttpClient会自己去设置）
        if (headerName.equalsIgnoreCase(Util.CONTENT_LENGTH)) {
            // The 'Content-Length' header is always set by the Apache client and it
            // doesn't like us to set it as well.
            continue;
        }

        // 将指定的请求头信息设置到builder中
        for (String headerValue : headerEntry.getValue()) {
            requestBuilder.addHeader(headerName, headerValue);
        }
    }
    
    // 如果没有指定Accept，默认添加通配符，表示接受任何类型的响应
    if (!hasAcceptHeader) {
        requestBuilder.addHeader(ACCEPT_HEADER_NAME, "*/*");
    }

    // 5. 设置请求体
    if (request.body() != null) {
        HttpEntity entity = null;
        if (request.charset() != null) {
            ContentType contentType = getContentType(request);
            String content = new String(request.body(), request.charset());
            entity = new StringEntity(content, contentType);
        } else {
            entity = new ByteArrayEntity(request.body());
        }
        requestBuilder.setEntity(entity);
    } else {
        requestBuilder.setEntity(new ByteArrayEntity(new byte[0]));
    }

    // 构建ApacheRequest
    return requestBuilder.build();
}
```
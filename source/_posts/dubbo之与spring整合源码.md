---
title: dubbo之与spring整合源码
date: 2022-01-10 15:40:12
tags:
    - dubbo
categories:
    - dubbo
---

# dubbo之与spring整合源码

## 问题
1. dubbo怎么启动服务来监听的？
2. dubbo怎么生成代理类？
3. dubbo怎么注册服务的？

## 内容
带着上面的问题，我们通过下面要描述的内容来解释

1. dubbo里面的properties文件解析
2. dubbo里面的@Service解析
3. dubbo里面的@Reference解析

## 前言
这里描述下前言，dubbo和spring整合源码，这里会有大量设计到spring知识点的内容，如有不懂，可自行学习，如需要介绍的，也会说明。主题还是在dubbo上，请知晓。


<!-- more -->
## 入口
我们通过dubbo源码的方式来知晓dubbo入口，我们也进一步来理解dubbo源码，下面是dubbo-example例子下面的代码
```java
public class Application {
    public static void main(String[] args) throws Exception {
        AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext(ProviderConfiguration.class);
        context.start();
        System.in.read();
    }

    /**
     *
     * <pre>
     *  1. 解析下配置
     *  2. 扫描@Service注解
     *      * 暴露服务，启动netty
     *  3. 扫描@Reference注解
     * </pre>
     */
    @Configuration
    @EnableDubbo(scanBasePackages = "org.apache.dubbo.demo.provider")
    @PropertySource("classpath:/spring/dubbo-provider.properties") // 这个就是读取的配置
    static class ProviderConfiguration {
        @Bean
        public RegistryConfig registryConfig() {
            RegistryConfig registryConfig = new RegistryConfig();
            registryConfig.setAddress("zookeeper://127.0.0.1:2181");
            return registryConfig;
        }
    }
}
```
通过上面的代码，我们可知，这里最主要的就是@PropertySource和@EnableDubbo，@PropertySource其实就是将dubbo-provider.properties配置文件的内容加载到spring容器中，Environment这个类就能获取对应的属性，一般就是key、value的键值对，也能通过@Value的方式来获取对应值，而@EnableDubbo其实就是对应启动dubbo相关，可想而知，这个是dubbo里面的重中之重，那我们这就开始我们的探索吧~



## @EnableDubbo
流程如下
![EnableDubbo流程](/images/dubbo-4-1.png)

```java
//...省略
@EnableDubboConfig
@DubboComponentScan
public @interface EnableDubbo {
    //...省略
}
```
见上面，@EnableDubbo其实有两个比较重要的注解，一个是@EnableDubboConfig，一个是@DubboComponentScan，这两个类也就是我们的入口

### @EnableDubboConfig
```java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Inherited
@Documented
@Import(DubboConfigConfigurationRegistrar.class)
public @interface EnableDubboConfig {

    /**
     * It indicates whether binding to multiple Spring Beans.
     *
     * @return the default value is <code>false</code>
     * @revised 2.5.9
     */
    boolean multiple() default true;

}
```
这里有一个注解@Import，导入`DubboConfigConfigurationRegistrar`类，作为spring的bean

#### DubboConfigConfigurationRegistrar

整体流程
![DubboConfigConfigurationRegistrar](/images/dubbo-4-2.png)

这个方法就是注入一些bean到spring容器里
```java
/**
 * 流程如下
 * `@EnableDubbo -> @EnableDubboConfig -> @import(DubboConfigConfigurationRegistrar)`
 */
public class DubboConfigConfigurationRegistrar implements ImportBeanDefinitionRegistrar {

    @Override
    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {

        AnnotationAttributes attributes = AnnotationAttributes.fromMap(
                importingClassMetadata.getAnnotationAttributes(EnableDubboConfig.class.getName())); // 获取@EnableDubboConfig注解的属性

        boolean multiple = attributes.getBoolean("multiple"); // 默认为true

        // Single Config Bindings
        /**
         * beanName：dubboConfigConfiguration.Single
         * beanClass：DubboConfigConfiguration.Single.class
         */
        registerBeans(registry, DubboConfigConfiguration.Single.class); // 注册bean，beanName：

        /**
         * 默认为true
         */
        if (multiple) { // Since 2.6.6 https://github.com/apache/dubbo/issues/3193
            /**
             * beanName：dubboConfigConfiguration.Multiple
             * beanClass：DubboConfigConfiguration.Multiple.class
             */
            registerBeans(registry, DubboConfigConfiguration.Multiple.class);
        }

        // Since 2.7.6
        registerCommonBeans(registry); // 注册commonBean
    }
}
```
```java
/**
* 注册bean
* annotatedClasses就是DubboConfigConfiguration.Single.class、DubboConfigConfiguration.Multiple.class。还有可能是其他注解，registry就是spring容器，将
* annotatedClasses注册到spring容器里
*/
public static void registerBeans(BeanDefinitionRegistry registry, Class<?>... annotatedClasses) {

    if (ObjectUtils.isEmpty(annotatedClasses)) {
        return;
    }

    Set<Class<?>> classesToRegister = new LinkedHashSet<Class<?>>(asList(annotatedClasses));

    // Remove all annotated-classes that have been registered
    Iterator<Class<?>> iterator = classesToRegister.iterator();

    while (iterator.hasNext()) {
        Class<?> annotatedClass = iterator.next();
        /**
        * bean容器里是否存在
        */
        if (isPresentBean(registry, annotatedClass)) {
            iterator.remove();
        }
    }

    AnnotatedBeanDefinitionReader reader = new AnnotatedBeanDefinitionReader(registry);

    if (logger.isDebugEnabled()) {
        logger.debug(registry.getClass().getSimpleName() + " will register annotated classes : " + asList(annotatedClasses) + " .");
    }

    reader.register(classesToRegister.toArray(EMPTY_CLASS_ARRAY)); // 注册bean

}
```

```java
/**
* 注册公共的bean到spring容器，用于后续加载
* 比较重要
*/
static void registerCommonBeans(BeanDefinitionRegistry registry) {
    /**
        * 这里是处理@Reference注解
        */
    // Since 2.5.7 Register @Reference Annotation Bean Processor as an infrastructure Bean
    registerInfrastructureBean(registry, ReferenceAnnotationBeanPostProcessor.BEAN_NAME,
            ReferenceAnnotationBeanPostProcessor.class);

    /**
        * 设置别名，使用AbstractConfig的id属性
        */
    // Since 2.7.4 [Feature] https://github.com/apache/dubbo/issues/5093
    registerInfrastructureBean(registry, DubboConfigAliasPostProcessor.BEAN_NAME,
            DubboConfigAliasPostProcessor.class);

    /**
        * dubbo启动完成，然后暴露一些其他可扩展，这里可以再看下
        */
    // Since 2.7.5 Register DubboLifecycleComponentApplicationListener as an infrastructure Bean
    registerInfrastructureBean(registry, DubboLifecycleComponentApplicationListener.BEAN_NAME,
            DubboLifecycleComponentApplicationListener.class);

    /**
        * dubbo监听服务，监听spring容器启动完成，这里就会暴露服务，启动netty
        */
    // Since 2.7.4 Register DubboBootstrapApplicationListener as an infrastructure Bean
    registerInfrastructureBean(registry, DubboBootstrapApplicationListener.BEAN_NAME,
            DubboBootstrapApplicationListener.class);

    /**
        * 属性填充，设置AbstractConfig 子类配置类的id和name属性
        */
    // Since 2.7.6 Register DubboConfigDefaultPropertyValueBeanPostProcessor as an infrastructure Bean
    registerInfrastructureBean(registry, DubboConfigDefaultPropertyValueBeanPostProcessor.BEAN_NAME,
            DubboConfigDefaultPropertyValueBeanPostProcessor.class);
}
```
注册`DubboConfigConfiguration.Single.class`、`DubboConfigConfiguration.Multiple.class`，然后还有注册一些公共的后置处理器和监听器，这些类都是比较重要的，其中我们可以看到的是`ReferenceAnnotationBeanPostProcessor`，这个就是处理@Reference注解的，其他可见上面注释


```java
public class DubboConfigConfiguration {

    /**
     * Single Dubbo {@link AbstractConfig Config} Bean Binding
     */
    @EnableConfigurationBeanBindings({
            @EnableConfigurationBeanBinding(prefix = "dubbo.application", type = ApplicationConfig.class),
            @EnableConfigurationBeanBinding(prefix = "dubbo.module", type = ModuleConfig.class),
            @EnableConfigurationBeanBinding(prefix = "dubbo.registry", type = RegistryConfig.class),
            @EnableConfigurationBeanBinding(prefix = "dubbo.protocol", type = ProtocolConfig.class),
            @EnableConfigurationBeanBinding(prefix = "dubbo.monitor", type = MonitorConfig.class),
            @EnableConfigurationBeanBinding(prefix = "dubbo.provider", type = ProviderConfig.class),
            @EnableConfigurationBeanBinding(prefix = "dubbo.consumer", type = ConsumerConfig.class),
            @EnableConfigurationBeanBinding(prefix = "dubbo.config-center", type = ConfigCenterBean.class),
            @EnableConfigurationBeanBinding(prefix = "dubbo.metadata-report", type = MetadataReportConfig.class),
            @EnableConfigurationBeanBinding(prefix = "dubbo.metrics", type = MetricsConfig.class),
            @EnableConfigurationBeanBinding(prefix = "dubbo.ssl", type = SslConfig.class)
    })
    public static class Single {

    }

    /**
     * Multiple Dubbo {@link AbstractConfig Config} Bean Binding
     */
    @EnableConfigurationBeanBindings({
            @EnableConfigurationBeanBinding(prefix = "dubbo.applications", type = ApplicationConfig.class, multiple = true),
            @EnableConfigurationBeanBinding(prefix = "dubbo.modules", type = ModuleConfig.class, multiple = true),
            @EnableConfigurationBeanBinding(prefix = "dubbo.registries", type = RegistryConfig.class, multiple = true),
            @EnableConfigurationBeanBinding(prefix = "dubbo.protocols", type = ProtocolConfig.class, multiple = true),
            @EnableConfigurationBeanBinding(prefix = "dubbo.monitors", type = MonitorConfig.class, multiple = true),
            @EnableConfigurationBeanBinding(prefix = "dubbo.providers", type = ProviderConfig.class, multiple = true),
            @EnableConfigurationBeanBinding(prefix = "dubbo.consumers", type = ConsumerConfig.class, multiple = true),
            @EnableConfigurationBeanBinding(prefix = "dubbo.config-centers", type = ConfigCenterBean.class, multiple = true),
            @EnableConfigurationBeanBinding(prefix = "dubbo.metadata-reports", type = MetadataReportConfig.class, multiple = true),
            @EnableConfigurationBeanBinding(prefix = "dubbo.metricses", type = MetricsConfig.class, multiple = true)
    })
    public static class Multiple {

    }
}

@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(ConfigurationBeanBindingsRegister.class)
public @interface EnableConfigurationBeanBindings {

    /**
     * @return the array of {@link EnableConfigurationBeanBinding EnableConfigurationBeanBindings}
     */
    EnableConfigurationBeanBinding[] value();
}
```


##### ConfigurationBeanBindingsRegister
此类就是对用户配置的properties解析，然后对应配置参数，生成对应的ApplicationConfig bean、ModuleConfig bean等对象，然后这里生产的bean其实是不完全的，因为属性还是空的，什么时候会填充呢？其实生成这些bean的时候其实会绑定一个后置处理器，那就是ConfigurationBeanBindingPostProcessor
```java
/**
 * 流程
 * `@EnableDubbo -> @EnableDubboConfig -> @import(DubboConfigConfigurationRegistrar) -> 将DubboConfigConfiguration.Single.class、DubboConfigConfiguration.Multiple.class注入到spring bean里，然后会加载上面的注解@EnableConfigurationBeanBindings -> @import(ConfigurationBeanBindingsRegister)`
 */
public class ConfigurationBeanBindingsRegister implements ImportBeanDefinitionRegistrar, EnvironmentAware {

    private ConfigurableEnvironment environment;

    /**
     * 这里会进入两次，一次是DubboConfigConfiguration.Single.class的属性，一次是DubboConfigConfiguration.Multiple.class的属性
     * @param importingClassMetadata
     * @param registry
     */
    @Override
    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {

        AnnotationAttributes attributes = AnnotationAttributes.fromMap(
                importingClassMetadata.getAnnotationAttributes(EnableConfigurationBeanBindings.class.getName()));

        AnnotationAttributes[] annotationAttributes = attributes.getAnnotationArray("value");

        ConfigurationBeanBindingRegistrar registrar = new ConfigurationBeanBindingRegistrar();

        registrar.setEnvironment(environment);

        for (AnnotationAttributes element : annotationAttributes) {
            registrar.registerConfigurationBeanDefinitions(element, registry);
        }
    }

    @Override
    public void setEnvironment(Environment environment) {
        Assert.isInstanceOf(ConfigurableEnvironment.class, environment);
        this.environment = (ConfigurableEnvironment) environment;
    }
}
```
```java
/**
 * `@EnableDubbo -> @EnableDubboConfig -> @import(DubboConfigConfigurationRegistrar) -> @EnableConfigurationBeanBindings -> @EnableConfigurationBeanBinding
 * -> @import(ConfigurationBeanBindingRegistrar)`
 *
 * The {@link ImportBeanDefinitionRegistrar} implementation for {@link EnableConfigurationBeanBinding @EnableConfigurationBinding}
 *
 * @author <a href="mailto:mercyblitz@gmail.com">Mercy</a>
 * @since 1.0.3
 */
public class ConfigurationBeanBindingRegistrar implements ImportBeanDefinitionRegistrar, EnvironmentAware {

    final static Class ENABLE_CONFIGURATION_BINDING_CLASS = EnableConfigurationBeanBinding.class;

    private final static String ENABLE_CONFIGURATION_BINDING_CLASS_NAME = ENABLE_CONFIGURATION_BINDING_CLASS.getName();

    private final Log log = LogFactory.getLog(getClass());

    private ConfigurableEnvironment environment;

    /**
     * 这个方法其实没有走，@EnableDubbo 注释是没有调用到这里的
     * @param metadata
     * @param registry
     */
    @Override
    public void registerBeanDefinitions(AnnotationMetadata metadata, BeanDefinitionRegistry registry) {

        Map<String, Object> attributes = metadata.getAnnotationAttributes(ENABLE_CONFIGURATION_BINDING_CLASS_NAME);

        registerConfigurationBeanDefinitions(attributes, registry);
    }

    /**
     * `@EnableDubbo` 流程走到这里，会调用这里，但是不是通过ConfigurationBeanBindingRegistrar.registerBeanDefinitions调用过来，是通过ConfigurationBeanBindingsRegister#registerBeanDefinitions调用过来，手动创建ConfigurationBeanBindingRegistrar，然后直接调用registerConfigurationBeanDefinitions方法
     * 那这里还有个问题，其中的environment那是怎么来的？这里也是赋值得来的
     * ConfigurationBeanBindingRegistrar registrar = new ConfigurationBeanBindingRegistrar();
     * registrar.setEnvironment(environment)
     * @param attributes
     * @param registry
     */
    public void registerConfigurationBeanDefinitions(Map<String, Object> attributes, BeanDefinitionRegistry registry) {

        /**
         * dubbo.application
         */
        String prefix = getRequiredAttribute(attributes, "prefix");

        prefix = environment.resolvePlaceholders(prefix);

        /**
         * ApplicationConfig.class
         */
        Class<?> configClass = getRequiredAttribute(attributes, "type");

        /**
         * false
         */
        boolean multiple = getAttribute(attributes, "multiple", valueOf(DEFAULT_MULTIPLE));
        // true
        boolean ignoreUnknownFields = getAttribute(attributes, "ignoreUnknownFields", valueOf(DEFAULT_IGNORE_UNKNOWN_FIELDS));
        // true
        boolean ignoreInvalidFields = getAttribute(attributes, "ignoreInvalidFields", valueOf(DEFAULT_IGNORE_INVALID_FIELDS));

        registerConfigurationBeans(prefix, configClass, multiple, ignoreUnknownFields, ignoreInvalidFields, registry);
    }


    private void registerConfigurationBeans(String prefix, Class<?> configClass, boolean multiple,
                                            boolean ignoreUnknownFields, boolean ignoreInvalidFields,
                                            BeanDefinitionRegistry registry) {

        /**
         * 这里就是获取子属性
         * 以dubbo.application为属性来说，这里返回的就是
         * name -> dubbo-demo-annotation-provider
         */
        Map<String, Object> configurationProperties = PropertySourcesUtils.getSubProperties(environment.getPropertySources(), environment, prefix);

        if (CollectionUtils.isEmpty(configurationProperties)) {
            if (log.isDebugEnabled()) {
                log.debug("There is no property for binding to configuration class [" + configClass.getName()
                        + "] within prefix [" + prefix + "]");
            }
            return;
        }

        /**
         * 这里是兼容的，如果是单个配置就是spring生成的beanName，如果是多个配置，那就是取的是用户自己配置的
         * dubbo.application.name=dubbo-demo-annotation-provider该配置就是
         * beanName：org.apache.dubbo.config.ApplicationConfig#0
         *
         * dubbo.protocols.p1.name=dubbo
         * dubbo.protocols.p1.port=20880
         * dubbo.protocols.p1.host=0.0.0.0
         *
         * dubbo.protocols.p2.name=dubbo
         * dubbo.protocols.p2.port=20881
         * dubbo.protocols.p2.host=0.0.0.0
         * beanName：p1 | p2
         */
        Set<String> beanNames = multiple ? resolveMultipleBeanNames(configurationProperties) :
                singleton(resolveSingleBeanName(configurationProperties, configClass, registry));

        for (String beanName : beanNames) {
            /**
             * 这里就是根据beanName，class来注册到spring容器里，这里是有一个问题的，这里的属性其实是没有填充的
             * 那什么时候填充呢？通过后置处理器来填充ConfigurationBeanBindingPostProcessor
             */
            registerConfigurationBean(beanName, configClass, multiple, ignoreUnknownFields, ignoreInvalidFields,
                    configurationProperties, registry);
        }
        /**
         * 这里增加一个后置处理器来绑定，这里注册就是ConfigurationBeanBindingPostProcessor，这里虽然会多次调用，但是只是会注册一次
         */
        registerConfigurationBindingBeanPostProcessor(registry);
    }

    private void registerConfigurationBean(String beanName, Class<?> configClass, boolean multiple,
                                           boolean ignoreUnknownFields, boolean ignoreInvalidFields,
                                           Map<String, Object> configurationProperties,
                                           BeanDefinitionRegistry registry) {

        BeanDefinitionBuilder builder = rootBeanDefinition(configClass);

        AbstractBeanDefinition beanDefinition = builder.getBeanDefinition();
        /**
         * 这里绑定真正的属性的时候就会起到作用了
         * 将改bd的source属性设置为EnableConfigurationBeanBinding.class
         */
        setSource(beanDefinition);

        /**
         * 获取全部属性
         * 只有multiple为true的时候，才会去解析，为false的时候直接就返回configurationProperties
         */
        Map<String, Object> subProperties = resolveSubProperties(multiple, beanName, configurationProperties);

        /**
         * 设置BeanDefinition的属性
         */
        initBeanMetadataAttributes(beanDefinition, subProperties, ignoreUnknownFields, ignoreInvalidFields);

        registry.registerBeanDefinition(beanName, beanDefinition);

        if (log.isInfoEnabled()) {
            log.info("The configuration bean definition [name : " + beanName + ", content : " + beanDefinition
                    + "] has been registered.");
        }
    }
}
```

##### ConfigurationBeanBindingPostProcessor
这是一个后置处理器，里面就是属性绑定，这里有个逻辑，判断是bd source属性为EnableConfigurationBeanBinding.class，并不会全部处理
```java
public class ConfigurationBeanBindingPostProcessor implements BeanPostProcessor, BeanFactoryAware {


    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {

        BeanDefinition beanDefinition = getNullableBeanDefinition(beanName);

        if (isConfigurationBean(bean, beanDefinition)) {
            bindConfigurationBean(bean, beanDefinition);
            customize(beanName, bean); // TODO 通用配置，底层调用没有进行什么逻辑
        }

        return bean;
    }

    private void bindConfigurationBean(Object configurationBean, BeanDefinition beanDefinition) {

        /**
         * name -> dubbo-demo-annotation-provider
         */
        Map<String, Object> configurationProperties = getConfigurationProperties(beanDefinition);
        /**
         * true
         */
        boolean ignoreUnknownFields = getIgnoreUnknownFields(beanDefinition);
        /**
         * true
         */
        boolean ignoreInvalidFields = getIgnoreInvalidFields(beanDefinition);

        getConfigurationBeanBinder().bind(configurationProperties, ignoreUnknownFields, ignoreInvalidFields, configurationBean);

        if (log.isInfoEnabled()) {
            log.info("The configuration bean [" + configurationBean + "] have been binding by the " +
                    "configuration properties [" + configurationProperties + "]");
        }
    }


}

```

##### DefaultConfigurationBeanBinder
这个是采用的spring的databinder技术，properties属性是key-value里面，DataBinder技术就是key和属性比较，如果匹配，就会赋值
```properties
dubbo.application.name=dubbo-demo-annotation-provider  
```
ApplicationConfig配置，其中name属性就是dubbo-demo-annotation-provider


```properties
dubbo.protocols.p1.name=dubbo
dubbo.protocols.p1.port=20880
dubbo.protocols.p1.host=0.0.0.0

dubbo.protocols.p2.name=dubbo
dubbo.protocols.p2.port=20881
dubbo.protocols.p2.host=0.0.0.0

```
ProtocolConfig配置，这里是两个bean对象，beanName分别是p1和p2，对应name、port、host里面值

#### 总结
<font color='red'><b>@EnableDubboConfig就是对properties属性解析，然后根据属性来生成对象bean，然后注册一些后置处理器和监听器，用于后续使用</b></font>



### @DubboComponentScan

流程
![DubboComponentScan整体流程](/images/dubbo-4-3.png)

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(DubboComponentScanRegistrar.class)
public @interface DubboComponentScan {
    /**
     * Alias for the {@link #basePackages()} attribute. Allows for more concise annotation
     * declarations e.g.: {@code @DubboComponentScan("org.my.pkg")} instead of
     * {@code @DubboComponentScan(basePackages="org.my.pkg")}.
     *
     * @return the base packages to scan
     */
    String[] value() default {};

    /**
     * Base packages to scan for annotated @Service classes. {@link #value()} is an
     * alias for (and mutually exclusive with) this attribute.
     * <p>
     * Use {@link #basePackageClasses()} for a type-safe alternative to String-based
     * package names.
     *
     * @return the base packages to scan
     */
    String[] basePackages() default {};

    /**
     * Type-safe alternative to {@link #basePackages()} for specifying the packages to
     * scan for annotated @Service classes. The package of each class specified will be
     * scanned.
     *
     * @return classes from the base packages to scan
     */
    Class<?>[] basePackageClasses() default {};
}
```

#### DubboComponentScanRegistrar
这里就是注册ServiceAnnotationBeanPostProcessor，然后将配置的路径以构造函数放入对应属性
```java
/**
 *流程如下
 *`@EnableDubbo -> @DubboComponentScan -> @import(DubboComponentScanRegistrar)`
 */
public class DubboComponentScanRegistrar implements ImportBeanDefinitionRegistrar {

    @Override
    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {

        /**
         * 路径：org.apache.dubbo.demo.provider
         */
        Set<String> packagesToScan = getPackagesToScan(importingClassMetadata);

        /**
         * 注册ServiceAnnotationBeanPostProcessor.class
         * 设置packagesToScan路径
         */
        registerServiceAnnotationBeanPostProcessor(packagesToScan, registry);

        // @since 2.7.6 Register the common beans
        registerCommonBeans(registry);
    }

    /**
     * Registers {@link ServiceAnnotationBeanPostProcessor}
     *
     * @param packagesToScan packages to scan without resolving placeholders
     * @param registry       {@link BeanDefinitionRegistry}
     * @since 2.5.8
     */
    private void registerServiceAnnotationBeanPostProcessor(Set<String> packagesToScan, BeanDefinitionRegistry registry) {

        BeanDefinitionBuilder builder = rootBeanDefinition(ServiceAnnotationBeanPostProcessor.class);
        builder.addConstructorArgValue(packagesToScan); // 构造函数
        builder.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
        AbstractBeanDefinition beanDefinition = builder.getBeanDefinition();
        BeanDefinitionReaderUtils.registerWithGeneratedName(beanDefinition, registry);

    }
}
```

##### ServiceAnnotationBeanPostProcessor
`ServiceAnnotationBeanPostProcessor`这个其实是`BeanFactoryPostProcessor`，名字有意义，这个是注入bean的。
这个作用有两个
1. 就是去扫描类上面加了@Service注解，然后将其扫描成spring bean。
    1. 这个扫描器不仅扫描出来一个bd，还会生成额外的ServiceBean类，ServiceBean也会放入到spring中
2. 注入`DubboBootstrapApplicationListener`，这个类很重要的，这个类是做什么的呢？这个是监听器，但是是spring监听器，监听的是spring容器启动完成，然后开始导出dubbo服务

```java
public class ServiceAnnotationBeanPostProcessor implements BeanDefinitionRegistryPostProcessor, EnvironmentAware,
        ResourceLoaderAware, BeanClassLoaderAware {

            /**
     * 这里是什么时机调用呢？这里的名字其实有歧义，如果不仔细看的话，这是一个后置处理器，但是这个ServiceAnnotationBeanPostProcessor这个类其实是BeanFactoryPostProcessor，bean工厂的后置处理器，这个是注册bean的，那BeanPostProcessor是用作什么的，那是加工bean(依赖)
     * 比BeanPostProcessor还要早，但是会比@Import类后
     *
     * @param registry
     * @throws BeansException
     */
    @Override
    public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException {

        // @since 2.7.5
        registerBeans(registry, DubboBootstrapApplicationListener.class);

        Set<String> resolvedPackagesToScan = resolvePackagesToScan(packagesToScan);

        if (!CollectionUtils.isEmpty(resolvedPackagesToScan)) {
            registerServiceBeans(resolvedPackagesToScan, registry);
        } else {
            if (logger.isWarnEnabled()) {
                logger.warn("packagesToScan is empty , ServiceBean registry will be ignored!");
            }
        }

    }

    /**
     * Registers Beans whose classes was annotated {@link Service}
     *
     * @param packagesToScan The base packages to scan
     * @param registry       {@link BeanDefinitionRegistry}
     */
    private void registerServiceBeans(Set<String> packagesToScan, BeanDefinitionRegistry registry) {

        /**
         * 扫描器，这块继承的ClassPathBeanDefinitionScanner，没有修改什么，设置useDefaultFilters=false(意义在不扫描spring的注解)，只是扫描dubbo的注解
         */
        DubboClassPathBeanDefinitionScanner scanner =
                new DubboClassPathBeanDefinitionScanner(registry, environment, resourceLoader);

        BeanNameGenerator beanNameGenerator = resolveBeanNameGenerator(registry);

        scanner.setBeanNameGenerator(beanNameGenerator);

        scanner.addIncludeFilter(new AnnotationTypeFilter(Service.class)); // 增加扫描类型

        /**
         * Add the compatibility for legacy Dubbo's @Service
         *
         * The issue : https://github.com/apache/dubbo/issues/4330
         * @since 2.7.3
         */
        scanner.addIncludeFilter(new AnnotationTypeFilter(com.alibaba.dubbo.config.annotation.Service.class));

        for (String packageToScan : packagesToScan) { // 目录

            // Registers @Service Bean first
            scanner.scan(packageToScan);

            /**
             * 扫描器，扫描出org.apache.dubbo.config.annotation.Service和com.alibaba.dubbo.config.annotation.Service类
             * 作为spring容器的bean
             */
            // Finds all BeanDefinitionHolders of @Service whether @ComponentScan scans or not.
            Set<BeanDefinitionHolder> beanDefinitionHolders =
                    findServiceBeanDefinitionHolders(scanner, packageToScan, registry, beanNameGenerator);

            if (!CollectionUtils.isEmpty(beanDefinitionHolders)) {

                /**
                 * 这里需要注意下，这里不仅是注册的DemoServiceImpl，也注册了ServiceBean
                 * 这里ServiceBean的意义在哪？
                 *  1. @Service 注解这么多配置，需要放在哪
                 *  2. @Service 也是一个服务
                 *  3. 跟原有的接口也有关系，ref关联DemoServiceImpl
                 *  4. 那这里应该有很多ServiceBean，一个被@Service注释的类，都会存在，这里就是ByName，class是一致的，但是beanName是不一样的，每个beanName就是
                 *  ServiceAnnotationBeanPostProcessor#generateServiceBeanName生成
                 */
                for (BeanDefinitionHolder beanDefinitionHolder : beanDefinitionHolders) {
                    registerServiceBean(beanDefinitionHolder, registry, scanner);
                }

                if (logger.isInfoEnabled()) {
                    logger.info(beanDefinitionHolders.size() + " annotated Dubbo's @Service Components { " +
                            beanDefinitionHolders +
                            " } were scanned under package[" + packageToScan + "]");
                }

            } else {

                if (logger.isWarnEnabled()) {
                    logger.warn("No Spring Bean annotating Dubbo's @Service was found under package["
                            + packageToScan + "]");
                }

            }

        }
    }
}
```

###### ServiceBean
这里的ServiceBean存在的意义，存储@Service注解里面的内容，ServiceBean就是表示一个Dubbo服务。如果要判断本地是否有dubbo service服务，那需要判断spring里面有ServiceBean
ServiceBean的beanName：ServiceBean:interfaceClassName:version:group
ServiceBean里面有很多属性，例如
1. ref，表示服务的具体实现类 
2. interface，表示服务的接⼝ 
3. parameters，表示服务的参数（@Service注解中所配置的信息） 
4. application，表示服务所属的应⽤ 
5. protocols，表示服务所使⽤的协议 
6. registries，表示服务所要注册的注册中⼼

```java
/**
     * Registers {@link ServiceBean} from new annotated {@link Service} {@link BeanDefinition}
     *
     * @param beanDefinitionHolder
     * @param registry
     * @param scanner
     * @see ServiceBean
     * @see BeanDefinition
     */
    private void registerServiceBean(BeanDefinitionHolder beanDefinitionHolder, BeanDefinitionRegistry registry,
                                     DubboClassPathBeanDefinitionScanner scanner) {
        /**
         * 这里@Service扫描出来的类
         * org.apache.dubbo.demo.provider.DemoServiceImpl
         */
        Class<?> beanClass = resolveClass(beanDefinitionHolder);
        // @Service 注解类
        Annotation service = findServiceAnnotation(beanClass);

        /**
         * The {@link AnnotationAttributes} of @Service annotation
         */
        /**
         * 获取@Service注解的属性
         *
         * "actives" -> {Integer@1955} 0
         * "proxy" -> ""
         * "document" -> ""
         * "export" -> {Boolean@1961} true
         * "stub" -> ""
         * "local" -> ""
         * "executes" -> {Integer@1955} 0
         * "layer" -> ""
         * "sent" -> {Boolean@1970} false
         * "mock" -> ""
         * "async" -> {Boolean@1970} false
         * "retries" -> {Integer@1975} 2
         * "token" -> ""
         * "listener" -> {String[0]@1979} []
         * "tag" -> ""
         * "registry" -> {String[0]@1983} []
         * "delay" -> {Integer@1955} 0
         * "owner" -> ""
         * "monitor" -> ""
         * "cluster" -> ""
         * "timeout" -> {Integer@1955} 0
         * "weight" -> {Integer@1955} 0
         * "module" -> ""
         * "dynamic" -> {Boolean@1961} true
         * "deprecated" -> {Boolean@1970} false
         * "connections" -> {Integer@1955} 0
         * "onconnect" -> ""
         * "ondisconnect" -> ""
         * "callbacks" -> {Integer@2003} 1
         * "loadbalance" -> "random"
         * "validation" -> ""
         * "application" -> ""
         * "interfaceClass" -> {Class@2011} "void"
         * "accesslog" -> ""
         * "interfaceName" -> ""
         * "group" -> ""
         * "cache" -> ""
         * "register" -> {Boolean@1961} true
         * "provider" -> ""
         * "parameters" -> {String[0]@2024} []
         * "path" -> ""
         * "protocol" -> {String[0]@2028} []
         * "filter" -> {String[0]@2030} []
         * "version" -> ""
         * "methods" -> {Method[0]@2034}
         */
        AnnotationAttributes serviceAnnotationAttributes = getAnnotationAttributes(service, false, false);

        Class<?> interfaceClass = resolveServiceInterfaceClass(serviceAnnotationAttributes, beanClass);
        /**
         * demoServiceImpl
         * 这个有什么作用？
         * 其实是通过这个beanName找到对应的bd，然后通过ref设置其中的属性
         */
        String annotatedServiceBeanName = beanDefinitionHolder.getBeanName();

        /**
         * 构建ServiceBean
         * 这里会将ServiceBean里面操作
         * ref：<demoServiceImpl>
         * interface：org.apache.dubbo.demo.DemoService
         * parameters：null
         */
        AbstractBeanDefinition serviceBeanDefinition =
                buildServiceBeanDefinition(service, serviceAnnotationAttributes, interfaceClass, annotatedServiceBeanName);

        /**
         * ServiceBean:org.apache.dubbo.demo.DemoService
         */
        String beanName = generateServiceBeanName(serviceAnnotationAttributes, interfaceClass);

        if (scanner.checkCandidate(beanName, serviceBeanDefinition)) { // check duplicated candidate bean
            /**
             * 定义org.apache.dubbo.config.annotation.Service的类，定义后也能是spring容器里面的一个类，就能使用@Autowired注入到其他类上
             * 该类也是spring容器里面的
             */
            registry.registerBeanDefinition(beanName, serviceBeanDefinition);

            if (logger.isInfoEnabled()) {
                logger.info("The BeanDefinition[" + serviceBeanDefinition +
                        "] of ServiceBean has been registered with name : " + beanName);
            }

        } else {

            if (logger.isWarnEnabled()) {
                logger.warn("The Duplicated BeanDefinition[" + serviceBeanDefinition +
                        "] of ServiceBean[ bean name : " + beanName +
                        "] was be found , Did @DubboComponentScan scan to same package in many times?");
            }

        }

    }
```
#### 总结
1. 这个类就是扫描@Service类，然后将该对象注入到spring容器，并且会生成对应的ServiceBean放入到spring容器


## @Reference处理
`ReferenceAnnotationBeanPostProcessor`就是处理@Reference，实际我们看的时候，其实需要先去看他的父类`AbstractAnnotationBeanPostProcessor`，这个类实现MergedBeanDefinitionPostProcessor，是一个BeanPostProcessor。他的入口是postProcessMergedBeanDefinition
流程
![ReferenceAnnotationBeanPostProcessor流程](/images/dubbo-4-4.png)

### AbstractAnnotationBeanPostProcessor
这个类是@Reference的核心，ReferenceAnnotationBeanPostProcessor继承AbstractAnnotationBeanPostProcessor，其实都是先处理AbstractAnnotationBeanPostProcessor，然后再调用到ReferenceAnnotationBeanPostProcessor。

再将ReferenceAnnotationBeanPostProcessor注入spring容器里，会将AbstractAnnotationBeanPostProcessor里面annotationTypes就@Reference注解

查询属性或者方法上找到注入点，然后就会调用到ReferenceAnnotationBeanPostProcessor的doGetInjectedBean方法得到对象，然后注入到属性对象里面

```java

/**
 * 这里是和AutowiredAnnotationBeanPostProcessor类似，里面的postProcessPropertyValues和postProcessMergedBeanDefinition方法也是差不多的
 * 所有的bean都会走这里的，只要是spring容器里面的bean，但是所有的bean走到这里，只处理@Reference属性注入，而AutowiredAnnotationBeanPostProcessor是只处理@Autowired属性
 * 这里需要好好理解下
 */
public abstract class AbstractAnnotationBeanPostProcessor extends
        InstantiationAwareBeanPostProcessorAdapter implements MergedBeanDefinitionPostProcessor, PriorityOrdered,
        BeanFactoryAware, BeanClassLoaderAware, EnvironmentAware, DisposableBean {

    private final static int CACHE_SIZE = Integer.getInteger("", 32);

    private final Log logger = LogFactory.getLog(getClass());

    /**
     * 这里的类型是什么时候赋值呢？是在ReferenceAnnotationBeanPostProcessor构造函数
     * Reference.class
     * com.alibaba.dubbo.config.annotation.Reference.class
     */
    private final Class<? extends Annotation>[] annotationTypes;

    private final ConcurrentMap<String, AbstractAnnotationBeanPostProcessor.AnnotatedInjectionMetadata> injectionMetadataCache =
            new ConcurrentHashMap<String, AbstractAnnotationBeanPostProcessor.AnnotatedInjectionMetadata>(CACHE_SIZE);

    private final ConcurrentMap<String, Object> injectedObjectsCache = new ConcurrentHashMap<String, Object>(CACHE_SIZE);

    private ConfigurableListableBeanFactory beanFactory;

    private Environment environment;

    private ClassLoader classLoader;

    /**
     * make sure higher priority than {@link AutowiredAnnotationBeanPostProcessor}
     */
    private int order = Ordered.LOWEST_PRECEDENCE - 3;

    /**
     * @param annotationTypes the multiple types of {@link Annotation annotations}
     */
    public AbstractAnnotationBeanPostProcessor(Class<? extends Annotation>... annotationTypes) {
        Assert.notEmpty(annotationTypes, "The argument of annotations' types must not empty");
        this.annotationTypes = annotationTypes;
    }

    private static <T> Collection<T> combine(Collection<? extends T>... elements) {
        List<T> allElements = new ArrayList<T>();
        for (Collection<? extends T> e : elements) {
            allElements.addAll(e);
        }
        return allElements;
    }

    /**
     * Annotation type
     *
     * @return non-null
     * @deprecated 2.7.3, uses {@link #getAnnotationTypes()}
     */
    @Deprecated
    public final Class<? extends Annotation> getAnnotationType() {
        return annotationTypes[0];
    }

    protected final Class<? extends Annotation>[] getAnnotationTypes() {
        return annotationTypes;
    }

    @Override
    public void setBeanFactory(BeanFactory beanFactory) throws BeansException {
        Assert.isInstanceOf(ConfigurableListableBeanFactory.class, beanFactory,
                "AnnotationInjectedBeanPostProcessor requires a ConfigurableListableBeanFactory");
        this.beanFactory = (ConfigurableListableBeanFactory) beanFactory;
    }

    /**
     * 后置处理器第二步处理这里
     * 填充属性
     */
    @Override
    public PropertyValues postProcessPropertyValues(
            PropertyValues pvs, PropertyDescriptor[] pds, Object bean, String beanName) throws BeanCreationException {

        /**
         * 寻找注入点
         * 找一下是否属性有@Reference标注的field
         */
        InjectionMetadata metadata = findInjectionMetadata(beanName, bean.getClass(), pvs);
        try {
            metadata.inject(bean, beanName, pvs);
        } catch (BeanCreationException ex) {
            throw ex;
        } catch (Throwable ex) {
            throw new BeanCreationException(beanName, "Injection of @" + getAnnotationType().getSimpleName()
                    + " dependencies is failed", ex);
        }
        return pvs;
    }


    /**
     * Finds {@link InjectionMetadata.InjectedElement} Metadata from annotated fields
     *
     * @param beanClass The {@link Class} of Bean
     * @return non-null {@link List}
     */
    private List<AbstractAnnotationBeanPostProcessor.AnnotatedFieldElement> findFieldAnnotationMetadata(final Class<?> beanClass) {

        final List<AbstractAnnotationBeanPostProcessor.AnnotatedFieldElement> elements = new LinkedList<AbstractAnnotationBeanPostProcessor.AnnotatedFieldElement>();

        /**
         * 找到beanClass里面属性是否有Reference.class和com.alibaba.dubbo.config.annotation.Reference.class属性的字段
         */
        ReflectionUtils.doWithFields(beanClass, new ReflectionUtils.FieldCallback() {
            @Override
            public void doWith(Field field) throws IllegalArgumentException, IllegalAccessException {

                for (Class<? extends Annotation> annotationType : getAnnotationTypes()) {

                    AnnotationAttributes attributes = getAnnotationAttributes(field, annotationType, getEnvironment(), true, true);

                    if (attributes != null) {

                        if (Modifier.isStatic(field.getModifiers())) {
                            if (logger.isWarnEnabled()) {
                                logger.warn("@" + annotationType.getName() + " is not supported on static fields: " + field);
                            }
                            return;
                        }

                        elements.add(new AnnotatedFieldElement(field, attributes));
                    }
                }
            }
        });

        return elements;

    }

    /**
     * Finds {@link InjectionMetadata.InjectedElement} Metadata from annotated methods
     *
     * @param beanClass The {@link Class} of Bean
     * @return non-null {@link List}
     */
    private List<AbstractAnnotationBeanPostProcessor.AnnotatedMethodElement> findAnnotatedMethodMetadata(final Class<?> beanClass) {

        final List<AbstractAnnotationBeanPostProcessor.AnnotatedMethodElement> elements = new LinkedList<AbstractAnnotationBeanPostProcessor.AnnotatedMethodElement>();

        ReflectionUtils.doWithMethods(beanClass, new ReflectionUtils.MethodCallback() {
            @Override
            public void doWith(Method method) throws IllegalArgumentException, IllegalAccessException {

                Method bridgedMethod = findBridgedMethod(method);

                if (!isVisibilityBridgeMethodPair(method, bridgedMethod)) {
                    return;
                }


                for (Class<? extends Annotation> annotationType : getAnnotationTypes()) {

                    AnnotationAttributes attributes = getAnnotationAttributes(bridgedMethod, annotationType, getEnvironment(), true, true);

                    if (attributes != null && method.equals(ClassUtils.getMostSpecificMethod(method, beanClass))) {
                        if (Modifier.isStatic(method.getModifiers())) {
                            if (logger.isWarnEnabled()) {
                                logger.warn("@" + annotationType.getName() + " annotation is not supported on static methods: " + method);
                            }
                            return;
                        }
                        if (method.getParameterTypes().length == 0) {
                            if (logger.isWarnEnabled()) {
                                logger.warn("@" + annotationType.getName() + " annotation should only be used on methods with parameters: " +
                                        method);
                            }
                        }
                        PropertyDescriptor pd = BeanUtils.findPropertyForMethod(bridgedMethod, beanClass);
                        elements.add(new AnnotatedMethodElement(method, pd, attributes));
                    }
                }
            }
        });

        return elements;
    }


    private AbstractAnnotationBeanPostProcessor.AnnotatedInjectionMetadata buildAnnotatedMetadata(final Class<?> beanClass) {
        /**
         * 属性
         */
        Collection<AbstractAnnotationBeanPostProcessor.AnnotatedFieldElement> fieldElements = findFieldAnnotationMetadata(beanClass);
        /**
         * 方法
         */
        Collection<AbstractAnnotationBeanPostProcessor.AnnotatedMethodElement> methodElements = findAnnotatedMethodMetadata(beanClass);
        return new AbstractAnnotationBeanPostProcessor.AnnotatedInjectionMetadata(beanClass, fieldElements, methodElements);
    }

    private InjectionMetadata findInjectionMetadata(String beanName, Class<?> clazz, PropertyValues pvs) {
        // Fall back to class name as cache key, for backwards compatibility with custom callers.
        String cacheKey = (StringUtils.hasLength(beanName) ? beanName : clazz.getName());
        // Quick check on the concurrent map first, with minimal locking.
        AbstractAnnotationBeanPostProcessor.AnnotatedInjectionMetadata metadata = this.injectionMetadataCache.get(cacheKey);
        if (InjectionMetadata.needsRefresh(metadata, clazz)) {
            synchronized (this.injectionMetadataCache) {
                metadata = this.injectionMetadataCache.get(cacheKey);
                if (InjectionMetadata.needsRefresh(metadata, clazz)) {
                    if (metadata != null) {
                        metadata.clear(pvs);
                    }
                    try {
                        metadata = buildAnnotatedMetadata(clazz); // 构建注入元数据
                        this.injectionMetadataCache.put(cacheKey, metadata);
                    } catch (NoClassDefFoundError err) {
                        throw new IllegalStateException("Failed to introspect object class [" + clazz.getName() +
                                "] for annotation metadata: could not find class that it depends on", err);
                    }
                }
            }
        }
        return metadata;
    }

    /**
     * spring的bean都会走到这里，只不过这里只是处理dubbo的@Reference注解的属性注入
     * 后置处理器先是处理这里，在这里校验配置的属性
     */
    @Override
    public void postProcessMergedBeanDefinition(RootBeanDefinition beanDefinition, Class<?> beanType, String beanName) {
        if (beanType != null) {
            InjectionMetadata metadata = findInjectionMetadata(beanName, beanType, null);
            metadata.checkConfigMembers(beanDefinition);
        }
    }

    @Override
    public int getOrder() {
        return order;
    }

    public void setOrder(int order) {
        this.order = order;
    }

    @Override
    public void destroy() throws Exception {

        for (Object object : injectedObjectsCache.values()) {
            if (logger.isInfoEnabled()) {
                logger.info(object + " was destroying!");
            }

            if (object instanceof DisposableBean) {
                ((DisposableBean) object).destroy();
            }
        }

        injectionMetadataCache.clear();
        injectedObjectsCache.clear();

        if (logger.isInfoEnabled()) {
            logger.info(getClass() + " was destroying!");
        }

    }

    @Override
    public void setBeanClassLoader(ClassLoader classLoader) {
        this.classLoader = classLoader;
    }

    @Override
    public void setEnvironment(Environment environment) {
        this.environment = environment;
    }

    protected Environment getEnvironment() {
        return environment;
    }

    protected ClassLoader getClassLoader() {
        return classLoader;
    }

    protected ConfigurableListableBeanFactory getBeanFactory() {
        return beanFactory;
    }

    /**
     * Gets all injected-objects.
     *
     * @return non-null {@link Collection}
     */
    protected Collection<Object> getInjectedObjects() {
        return this.injectedObjectsCache.values();
    }

    /**
     * Get injected-object from specified {@link AnnotationAttributes annotation attributes} and Bean Class
     *
     * @param attributes      {@link AnnotationAttributes the annotation attributes}
     * @param bean            Current bean that will be injected
     * @param beanName        Current bean name that will be injected
     * @param injectedType    the type of injected-object
     * @param injectedElement {@link InjectionMetadata.InjectedElement}
     * @return An injected object
     * @throws Exception If getting is failed
     */
    protected Object getInjectedObject(AnnotationAttributes attributes, Object bean, String beanName, Class<?> injectedType,
                                       InjectionMetadata.InjectedElement injectedElement) throws Exception {
        // ServiceBean:org.apache.dubbo.demo.DemoService#source=private org.apache.dubbo.demo.DemoService org.apache.dubbo.demo.consumer.comp.DemoServiceComponent.demoService#attributes={}
        String cacheKey = buildInjectedObjectCacheKey(attributes, bean, beanName, injectedType, injectedElement);

        Object injectedObject = injectedObjectsCache.get(cacheKey);

        if (injectedObject == null) {
            injectedObject = doGetInjectedBean(attributes, bean, beanName, injectedType, injectedElement);
            // Customized inject-object if necessary
            injectedObjectsCache.putIfAbsent(cacheKey, injectedObject);
        }

        return injectedObject;

    }

    /**
     * Subclass must implement this method to get injected-object. The context objects could help this method if
     * necessary :
     * <ul>
     * <li>{@link #getBeanFactory() BeanFactory}</li>
     * <li>{@link #getClassLoader() ClassLoader}</li>
     * <li>{@link #getEnvironment() Environment}</li>
     * </ul>
     *
     * @param attributes      {@link AnnotationAttributes the annotation attributes}
     * @param bean            Current bean that will be injected
     * @param beanName        Current bean name that will be injected
     * @param injectedType    the type of injected-object
     * @param injectedElement {@link InjectionMetadata.InjectedElement}
     * @return The injected object
     * @throws Exception If resolving an injected object is failed.
     */
    protected abstract Object doGetInjectedBean(AnnotationAttributes attributes, Object bean, String beanName, Class<?> injectedType,
                                                InjectionMetadata.InjectedElement injectedElement) throws Exception;

    /**
     * Build a cache key for injected-object. The context objects could help this method if
     * necessary :
     * <ul>
     * <li>{@link #getBeanFactory() BeanFactory}</li>
     * <li>{@link #getClassLoader() ClassLoader}</li>
     * <li>{@link #getEnvironment() Environment}</li>
     * </ul>
     *
     * @param attributes      {@link AnnotationAttributes the annotation attributes}
     * @param bean            Current bean that will be injected
     * @param beanName        Current bean name that will be injected
     * @param injectedType    the type of injected-object
     * @param injectedElement {@link InjectionMetadata.InjectedElement}
     * @return Bean cache key
     */
    protected abstract String buildInjectedObjectCacheKey(AnnotationAttributes attributes, Object bean, String beanName,
                                                          Class<?> injectedType,
                                                          InjectionMetadata.InjectedElement injectedElement);

    /**
     * Get {@link Map} in injected field.
     *
     * @return non-null ready-only {@link Map}
     */
    protected Map<InjectionMetadata.InjectedElement, Object> getInjectedFieldObjectsMap() {

        Map<InjectionMetadata.InjectedElement, Object> injectedElementBeanMap =
                new LinkedHashMap<InjectionMetadata.InjectedElement, Object>();

        for (AbstractAnnotationBeanPostProcessor.AnnotatedInjectionMetadata metadata : injectionMetadataCache.values()) {

            Collection<AbstractAnnotationBeanPostProcessor.AnnotatedFieldElement> fieldElements = metadata.getFieldElements();

            for (AbstractAnnotationBeanPostProcessor.AnnotatedFieldElement fieldElement : fieldElements) {

                injectedElementBeanMap.put(fieldElement, fieldElement.bean);

            }

        }

        return Collections.unmodifiableMap(injectedElementBeanMap);

    }

    /**
     * Get {@link Map} in injected method.
     *
     * @return non-null {@link Map}
     */
    protected Map<InjectionMetadata.InjectedElement, Object> getInjectedMethodObjectsMap() {

        Map<InjectionMetadata.InjectedElement, Object> injectedElementBeanMap =
                new LinkedHashMap<InjectionMetadata.InjectedElement, Object>();

        for (AbstractAnnotationBeanPostProcessor.AnnotatedInjectionMetadata metadata : injectionMetadataCache.values()) {

            Collection<AbstractAnnotationBeanPostProcessor.AnnotatedMethodElement> methodElements = metadata.getMethodElements();

            for (AbstractAnnotationBeanPostProcessor.AnnotatedMethodElement methodElement : methodElements) {

                injectedElementBeanMap.put(methodElement, methodElement.object);

            }

        }

        return Collections.unmodifiableMap(injectedElementBeanMap);

    }

    /**
     * {@link Annotation Annotated} {@link InjectionMetadata} implementation
     */
    private class AnnotatedInjectionMetadata extends InjectionMetadata {

        private final Collection<AbstractAnnotationBeanPostProcessor.AnnotatedFieldElement> fieldElements;

        private final Collection<AbstractAnnotationBeanPostProcessor.AnnotatedMethodElement> methodElements;

        public AnnotatedInjectionMetadata(Class<?> targetClass, Collection<AbstractAnnotationBeanPostProcessor.AnnotatedFieldElement> fieldElements,
                                          Collection<AbstractAnnotationBeanPostProcessor.AnnotatedMethodElement> methodElements) {
            super(targetClass, combine(fieldElements, methodElements));
            this.fieldElements = fieldElements;
            this.methodElements = methodElements;
        }

        public Collection<AbstractAnnotationBeanPostProcessor.AnnotatedFieldElement> getFieldElements() {
            return fieldElements;
        }

        public Collection<AbstractAnnotationBeanPostProcessor.AnnotatedMethodElement> getMethodElements() {
            return methodElements;
        }
    }

    /**
     * {@link Annotation Annotated} {@link Method} {@link InjectionMetadata.InjectedElement}
     */
    private class AnnotatedMethodElement extends InjectionMetadata.InjectedElement {

        private final Method method;

        private final AnnotationAttributes attributes;

        private volatile Object object;

        protected AnnotatedMethodElement(Method method, PropertyDescriptor pd, AnnotationAttributes attributes) {
            super(method, pd);
            this.method = method;
            this.attributes = attributes;
        }

        /**
         * 处理方法中包含注解的属性
         */
        @Override
        protected void inject(Object bean, String beanName, PropertyValues pvs) throws Throwable {

            Class<?> injectedType = pd.getPropertyType();

            Object injectedObject = getInjectedObject(attributes, bean, beanName, injectedType, this);

            ReflectionUtils.makeAccessible(method);

            method.invoke(bean, injectedObject);

        }

    }

    /**
     * {@link Annotation Annotated} {@link Field} {@link InjectionMetadata.InjectedElement}
     */
    public class AnnotatedFieldElement extends InjectionMetadata.InjectedElement {

        private final Field field;

        private final AnnotationAttributes attributes;

        private volatile Object bean;

        protected AnnotatedFieldElement(Field field, AnnotationAttributes attributes) {
            super(field, null);
            this.field = field;
            this.attributes = attributes;
        }

        /**
         * 处理属性中包含注解
         */
        @Override
        protected void inject(Object bean, String beanName, PropertyValues pvs) throws Throwable {
            // org.apache.dubbo.demo.DemoService
            Class<?> injectedType = field.getType();

            Object injectedObject = getInjectedObject(attributes, bean, beanName, injectedType, this);

            ReflectionUtils.makeAccessible(field);

            field.set(bean, injectedObject);

        }

    }
}
```

### ReferenceAnnotationBeanPostProcessor
doGetInjectedBean，这个得到的对象肯定是一个代理对象，有代理逻辑，处理对应的请求
流程：
![getOrCreateProxy](/images/dubbo-4-4-续.png)

处理流程如下：
1. 获取referencedBeanName，其实就是ServiceBean的beanName
2. 获取ReferenceBeanName，这个生成规则是@Reference+"("+注解属性遍历+")"+" "+接口名，如果没有注解属性，直接就是@Reference+" "+接口名
3. 构建ReferenceBean，跟之前的ServiceBean构建类似，也是把属性填充
4. 注册ReferenceBean，如果是本地服务，将其ServiceBean的ref的bean增加一个别名ReferenceBeanName，否则不是本地服务，直接根据ReferenceBeanName注册bean（怎么判断是否是一个本地服务，根据@Reference注解生成ServiceBean的beanName格式的beanName，然后去spring容器里面判断，如果存在则是本地服务）
5. 创建代理对象，如果这里是本地的服务，是ServiceBean的ref生成的代理类，是由ReferencedBeanInvocationHandler生成的代理类，如果不是本地服务，那就是RefereceBean.get()对象

```java
/**
 * 这里是处理@Reference的，这里是处理属性依赖的，想比较@Service是更加复杂的
 * 这里其实和@Autowired比较类似，处理逻辑也是和AutowiredAnnotationBeanPostProcessor比较类似了
 */
public class ReferenceAnnotationBeanPostProcessor extends AbstractAnnotationBeanPostProcessor implements
        ApplicationContextAware, ApplicationListener<ServiceBeanExportedEvent> {
        
        
        /**
     * To support the legacy annotation that is @com.alibaba.dubbo.config.annotation.Reference since 2.7.3
     */
    public ReferenceAnnotationBeanPostProcessor() {
        super(Reference.class, com.alibaba.dubbo.config.annotation.Reference.class);
    }
        
        
        @Override
    protected Object doGetInjectedBean(AnnotationAttributes attributes, Object bean, String beanName, Class<?> injectedType,
                                       InjectionMetadata.InjectedElement injectedElement) throws Exception {
        /**
         * The name of bean that annotated Dubbo's {@link Service @Service} in local Spring {@link ApplicationContext}
         */
        /**
         * referencedBeanName其实就是serviceBeanName
         * ServiceBean:org.apache.dubbo.demo.DemoService
         */
        String referencedBeanName = buildReferencedBeanName(attributes, injectedType);

        /**
         * The name of bean that is declared by {@link Reference @Reference} annotation injection
         */
        /**
         * 这里其实是referenceBeanName：@Reference org.apache.dubbo.demo.DemoService
         */
        String referenceBeanName = getReferenceBeanName(attributes, injectedType);

        /**
         * 获取ReferenceBean，这里其实和ServiceBean类似，都是处理@Reference的属性
         */
        ReferenceBean referenceBean = buildReferenceBeanIfAbsent(referenceBeanName, attributes, injectedType);

        /**
         * 是否本地服务，怎么判断自己的容器里面是否存在dubbo @Service的内容，要判断服务类么？这里肯定是不行的，因为@Autowired也是可以获取到的，只能通过ServiceBean的方式来获取，就是独一无二的
         */
        boolean localServiceBean = isLocalServiceBean(referencedBeanName, referenceBean, attributes);

        /**
         * 这里同样的注册ReferenceBean
         */
        registerReferenceBean(referencedBeanName, referenceBean, attributes, localServiceBean, injectedType);

        cacheInjectedReferenceBean(referenceBean, injectedElement);

        /**
         * 代理类
         */
        return getOrCreateProxy(referencedBeanName, referenceBean, localServiceBean, injectedType);
    }

    /**
     * Get or Create a proxy of {@link ReferenceBean} for the specified the type of Dubbo service interface
     *
     * @param referencedBeanName   The name of bean that annotated Dubbo's {@link Service @Service} in the Spring {@link ApplicationContext}
     * @param referenceBean        the instance of {@link ReferenceBean}
     * @param localServiceBean     Is Local Service bean or not
     * @param serviceInterfaceType the type of Dubbo service interface
     * @return non-null
     * @since 2.7.4
     */
    private Object getOrCreateProxy(String referencedBeanName, ReferenceBean referenceBean, boolean localServiceBean,
                                    Class<?> serviceInterfaceType) {
        if (localServiceBean) { // If the local @Service Bean exists, build a proxy of Service
            return newProxyInstance(getClassLoader(), new Class[]{serviceInterfaceType},
                    newReferencedBeanInvocationHandler(referencedBeanName));
        } else {
            exportServiceBeanIfNecessary(referencedBeanName); // If the referenced ServiceBean exits, export it immediately
            return referenceBean.get();
        }
    }

    private class ReferencedBeanInvocationHandler implements InvocationHandler {

        private final String referencedBeanName;

        private Object bean;

        private ReferencedBeanInvocationHandler(String referencedBeanName) {
            this.referencedBeanName = referencedBeanName;
        }

        @Override
        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
            Object result = null;
            try {
                if (bean == null) {
                    init();
                }
                result = method.invoke(bean, args);
            } catch (InvocationTargetException e) {
                // re-throws the actual Exception.
                throw e.getTargetException();
            }
            return result;
        }

        private void init() {
            ServiceBean serviceBean = applicationContext.getBean(referencedBeanName, ServiceBean.class);
            this.bean = serviceBean.getRef();
        }
    }
}
```
<br>

<font color='red'><b>这里说明下，因为@Reference生成的ReferenceBean对象，但是我们其实要调用的是一个代理对象，见下面，其实demoService和demoService1是一个对象，那我们要兼容@AutoWired对象，ReferenceBean这里其实采用的是FactoryBean，通过get就会获取代理对象</b></font>

```java
@Component("demoServiceComponent")
public class DemoServiceComponent implements DemoService {
    @Reference
    private DemoService demoService;

    @Autowired
    private DemoService demoService1;

    @Override
    public String sayHello(String name) {
        return demoService.sayHello(name);
    }

    @Override
    public CompletableFuture<String> sayHelloAsync(String name) {
        return null;
    }
}
```

### 总结
这个就是处理@Reference，生成ReferenceBean，生成代理对象。

## 写在最后
dubbo和spring整合就写在这里，关于@Service服务导出，@Reference服务调用，后续还会有文章介绍
---
title: dubbo之导出服务详解
date: 2022-01-18 20:16:07
tags:
    - dubbo
categories:
    - dubbo
---

# dubbo之导出服务详解
前言：分析的dubbo源码是2.7.7版本

# 入口
我们要看服务导出，入口在哪？`DubboBootstrapApplicationListener`监听spring启动完成，具体是在这里启动的`dubboBootstrap.start`

```java
public class DubboBootstrapApplicationListener extends OneTimeExecutionApplicationContextEventListener
        implements Ordered {

    /**
     * The bean name of {@link DubboBootstrapApplicationListener}
     *
     * @since 2.7.6
     */
    public static final String BEAN_NAME = "dubboBootstrapApplicationListener";

    private final DubboBootstrap dubboBootstrap;

    public DubboBootstrapApplicationListener() {
        this.dubboBootstrap = DubboBootstrap.getInstance();
    }

    @Override
    public void onApplicationContextEvent(ApplicationContextEvent event) {
        if (event instanceof ContextRefreshedEvent) {
            onContextRefreshedEvent((ContextRefreshedEvent) event);
        } else if (event instanceof ContextClosedEvent) {
            onContextClosedEvent((ContextClosedEvent) event);
        }
    }

    private void onContextRefreshedEvent(ContextRefreshedEvent event) {
        dubboBootstrap.start();
    }

    private void onContextClosedEvent(ContextClosedEvent event) {
        dubboBootstrap.stop();
    }

    @Override
    public int getOrder() {
        return LOWEST_PRECEDENCE;
    }
}
```
<!-- more -->


# 分析
在我们正式分析`dubboBootstrap.start`源码之前，我们先分析下dubbo服务导出应该做什么事情，先去思考然后去看源码会有思路
```text
1. 启动服务netty或者tomcat
2. 将服务注册到zk注册中心上面

在这两步之前其实还有
0. 读取配置，读取最新最全的配置，大致逻辑我们清楚了，是否是这样的，我们可以看下
```

# dubboBootstrap.start
流程图
![dubboBootstrap.start](/images/dubbo-5-1.png)

那咱们最主要的就是`initialize、exportServices、referServices`，其实从方法上面看就能猜到一些意思，<font color='red'><b>`initialize`初始化，`exportServices`导出服务，`referServices`引入服务</b></font>

这篇我们主要是介绍`initialize、exportServices`，`referServices`下篇来介绍

<font color='red'><b>其实还有两个比较重要，就是ConfigManager配置管理器，Environment环境</b></font>，其中`ConfigManager`就是把`ProtocolConfig、RegistryConfig等`放进到里面，`Environment`是dubbo包下，这个就是读取配置，见图，配置其实可以有很多地方读取到

## initialize
初始化流程图如下
![initialize](/images/dubbo-5-2.png)

```java
    private void initialize() {
        /**
         *  方法刚进入时，configManager里面的configsCache只有下面5个
         * "registry" -> {HashMap@2796}  size = 2
         * "protocol" -> {HashMap@2798}  size = 2
         * "application" -> {HashMap@2800}  size = 1
         * "provider" -> {HashMap@2802}  size = 1
         * "service" -> {HashMap@2804}  size = 1
         */
        if (!initialized.compareAndSet(false, true)) {
            return;
        }
        /**
         * 加载一下dubbo-admin里面的应用和全部配置，加载到Environment（这个是dubbo里面的org.apache.dubbo.common.config.Environment）下面属性
         * private final InmemoryConfiguration externalConfiguration;
         * private final InmemoryConfiguration appExternalConfiguration;
         */
        ApplicationModel.initFrameworkExts();

        /**
         * 开启配置中心 & 更新配置
         *
         */
        startConfigCenter();
        /**
         * 如果需要的话，用注册中心作为配置中心，然后调用startConfigCenter方法，开启配置中心，然后重新的更新配置
         */
        useRegistryAsConfigCenterIfNecessary();
        /**
         * 上面两个方法后configManager里面的configsCache就会多出来
         * key = config-center
         * value = config-center-zookeeper-2181 -> {ConfigCenterConfig@3549} "<dubbo:config-center check="true" group="dubbo" configFile="dubbo.properties" highestPriority="false" timeout="3000" address="zookeeper://127.0.0.1:2181" protocol="zookeeper" port="2181" />"
         */



        /**
         * 加载一下远端的配置，这个远端的配置是注册中心的配置
         * 1. externalConfigurationMap
         * 2. appExternalConfigurationMap
         *
         * 然后把dubbo.registries.、dubbo.protocols.两个配置分别加入到RegistryConfig 和 ProtocolConfig里面
         *
         */
        loadRemoteConfigs();

        /**
         * 这个方法完成后configManager里面的configsCache就会多出来，如果之前存在，就校验参数，更新配置
         *
         * key = provider
         * value = providerConfig -> {ProviderConfig@3555} "<dubbo:provider />"
         *
         * key = consumer
         * value = ConsumerConfig#default -> {ConsumerConfig@3561} "<dubbo:consumer />"
         *
         * key = monitor
         * value = MonitorConfig#default -> {MonitorConfig@3567} "<dubbo:monitor />"
         *
         * key = metrics
         * value = MetricsConfig#default -> {MetricsConfig@3573} "<dubbo:metrics />"
         *
         * key = module
         * value = ModuleConfig#default -> {ModuleConfig@3579} "<dubbo:module />"
         *
         * key = ssl
         * value = SslConfig#default -> {SslConfig@3585} "<dubbo:ssl />"
         */
        checkGlobalConfigs();

        /**
         * 该方法是初始化元数据中心，这块是后续新增的功能
         * 将metadataService设置为InMemoryWritableMetadataService
         * 将metadataServiceExporter设置成new ConfigurableMetadataServiceExporter(metadataService)
         */
        initMetadataService();

        /**
         * 将DubboBootstrap对象注册为监听器，表示配置都是加载完成，也创建了对应的对象
         *
         * 调用initialize方法是在容器事件发布后，此时各个配置对象都已经创建完毕，在initialize中使用没有什么问题，但是客户端调用该方法的时候，是在对ReferenceBean
         * 对象执行bean后处理器，换句话说spring此时还在对ReferenceBean对象创建的过程中，那么dubbo如何保证客户端启动的时候initialize中需要的配置对象都创建完毕了
         * 呢？大家可以看ReferenceBean里面提到一个方法doGetInjectedBean，该方法创建ReferenceBean对象，但创建其实委托给ReferenceBeanBuilder，在
         * new ReferenceBean()后，会执行ReferenceBeanBuilder的方法prepareDubboConfigBeans
         */
        initEventListener();

        if (logger.isInfoEnabled()) {
            logger.info(NAME + " has been initialized!");
        }
        /**
         * 执行完后，configManager里面的configsCache有11个内容
         * "registry" -> {HashMap@3351}  size = 2
         * "protocol" -> {HashMap@3353}  size = 2
         * "application" -> {HashMap@3355}  size = 1
         * "provider" -> {HashMap@3357}  size = 1
         * "service" -> {HashMap@3359}  size = 1
         * "module" -> {HashMap@3361}  size = 1
         * "monitor" -> {HashMap@3363}  size = 1
         * "metrics" -> {HashMap@3365}  size = 1
         * "ssl" -> {HashMap@3367}  size = 1
         * "consumer" -> {HashMap@3369}  size = 1
         * "config-center" -> {HashMap@3371}  size = 1
         */
    }
```
总结下，就是将ConfigManager配置管理器的配置补充完成，配置是最新的，最全的。这些配置从哪里读取呢？那就是Environment环境里面

### AbstractConfig#refresh
流程图
![initialize](/images/dubbo-5-2-1.png)

读取的就是Environment环境里面配置，加载顺序和逻辑如上

## exportServices
流程图
![exportServices](/images/dubbo-5-3.png)

### ServiceConfig#export
流程图
![exportServices](/images/dubbo-5-3-1.png)

```java
/**
     * 真正的导出服务
     */
    public synchronized void export() {
        if (!shouldExport()) {
            return;
        }

        if (bootstrap == null) { // 这个里面是非常重要的信息，各种配置信息
            bootstrap = DubboBootstrap.getInstance();
            bootstrap.init();
        }
        /**
         * 这个是获取ServiceBean 里面最新 & 最全的配置
         */
        checkAndUpdateSubConfigs();

        /**
         * ServiceBean信息
         *
         * init serviceMetadata
         */
        serviceMetadata.setVersion(version);
        serviceMetadata.setGroup(group);
        serviceMetadata.setDefaultGroup(group);
        serviceMetadata.setServiceType(getInterfaceClass());
        serviceMetadata.setServiceInterfaceName(getInterface());
        serviceMetadata.setTarget(getRef());

        if (shouldDelay()) { // 可延迟导出
            DELAY_EXPORT_EXECUTOR.schedule(this::doExport, getDelay(), TimeUnit.MILLISECONDS);
        } else {
            doExport();
        }
        /**
         * 服务导出成功后，发送一个ServiceConfigExportedEvent事件
          */
        exported();
    }
```

#### checkAndUpdateSubConfigs
流程图
![checkAndUpdateSubConfigs](/images/dubbo-5-3-2.png)

<font color='red'><b>这里就是获取ServiceBean最新最全的配置，配置更新，配置更新也会刷新ConfigManager</b></font>，当然也会检查参数

`this.refresh()`方法其实就是AbstractConfig#refresh，这个就是刷新ServiceBean的属性，当然此时已经有一些属性了，那就是@Service上的值。
除了这些地方，也可以从别的地方读取
1. -D的方式，源码的方式SystemConfiguration
2. jvm的方式，源码的方式EnvironmentConfiguration
3. dubbo-admin里的应用配置，源码的方式InmemoryConfiguration（属性store不同）
4. dubbo-admin里的全局配置，源码的方式InmemoryConfiguration（属性store不同）
5. 当前值
6. dubbo.properties，resources目录下的该文件里的内容，注意这个和spring里面的properties是不同的，是dubbo自己加载的


> dubbo.properties⽂件、配置中⼼、-D 中配置的方式
> dubbo.service.{interface-name}[.{method-name}].timeout=3000

```java
/**
     * 补全 & 更新配置
     */
    private void checkAndUpdateSubConfigs() {
        /**
         * 补全配置，根据ProviderConfig配置来设置，这个就是<dubbo:provider></dubbo:provider>配置下面的属性
         * 优先级是很高的，设置下面属性，属性为空就跳过，
         *
         * 优先级：
         * application
         * 	1. providerConfig
         * module
         * 	1. providerConfig
         * registries
         * 	1. application
         * 	2. module
         * 	3. providerConfig
         * monitor
         * 	1. application
         * 	2. module
         * 	3. providerConfig
         * protocols
         * 	1. providerConfig
         * configCenter
         * 	1. providerConfig
         * registryIds
         * 	1. providerConfig
         * protocolIds
         * 	1. providerConfig
         *
         * Use default configs defined explicitly with global scope
          */
        completeCompoundConfigs();
        /**
         * 设置provider属性，没有的话就从ConfigManager里拿
         */
        checkDefault();
        /**
         * ServiceBean里如果没有配置protocol，就从ConfigManager里拿，设置ServiceBean里的protocols属性
         */
        checkProtocol();
        // init some null configuration.
        List<ConfigInitializer> configInitializers = ExtensionLoader.getExtensionLoader(ConfigInitializer.class)
                .getActivateExtension(URL.valueOf("configInitializer://"), (String[]) null);
        configInitializers.forEach(e -> e.initServiceConfig(this));

        // if protocol is not injvm checkRegistry
        if (!isOnlyInJvm()) { // true
            checkRegistry();
        }
        /**
         * 更新下ServiceBean 属性
         */
        this.refresh();

        if (StringUtils.isEmpty(interfaceName)) {
            throw new IllegalStateException("<dubbo:service interface=\"\" /> interface not allow null!");
        }

        if (ref instanceof GenericService) {
            interfaceClass = GenericService.class;
            if (StringUtils.isEmpty(generic)) {
                generic = Boolean.TRUE.toString();
            }
        } else {
            try {
                interfaceClass = Class.forName(interfaceName, true, Thread.currentThread()
                        .getContextClassLoader());
            } catch (ClassNotFoundException e) {
                throw new IllegalStateException(e.getMessage(), e);
            }
            checkInterfaceAndMethods(interfaceClass, getMethods()); // 校验接口和方法，getMethods()参数是空的
            checkRef(); // 校验ref类
            generic = Boolean.FALSE.toString(); // 泛型为false
        }
        if (local != null) {
            if ("true".equals(local)) {
                local = interfaceName + "Local";
            }
            Class<?> localClass;
            try {
                localClass = ClassUtils.forNameWithThreadContextClassLoader(local);
            } catch (ClassNotFoundException e) {
                throw new IllegalStateException(e.getMessage(), e);
            }
            if (!interfaceClass.isAssignableFrom(localClass)) {
                throw new IllegalStateException("The local implementation class " + localClass.getName() + " not implement interface " + interfaceName);
            }
        }
        if (stub != null) {
            if ("true".equals(stub)) {
                stub = interfaceName + "Stub";
            }
            Class<?> stubClass;
            try {
                stubClass = ClassUtils.forNameWithThreadContextClassLoader(stub);
            } catch (ClassNotFoundException e) {
                throw new IllegalStateException(e.getMessage(), e);
            }
            if (!interfaceClass.isAssignableFrom(stubClass)) {
                throw new IllegalStateException("The stub implementation class " + stubClass.getName() + " not implement interface " + interfaceName);
            }
        }
        checkStubAndLocal(interfaceClass);
        ConfigValidationUtils.checkMock(interfaceClass, this);
        ConfigValidationUtils.validateServiceConfig(this);
        postProcessConfig();
    }

```
其中读取配置，也是走的AbstractConfig#refresh，同样也是走的上面的配置地方


#### doExport
<b>上面获取ServiceBean里面最新最全的配置后</b>，真正的来导出服务，导出服务是还是比较难的。
具体的逻辑如下：
1. 构建URL
2. 启动服务，如果是dubbo，就是启动NettyServer，如果是http，就是启动tomcat、jetty
3. 将注册信息注册到zk上
4. dubbo支持动态修改配置，需要监听器listener来监听服务配置，动态修改配置，重新导出服务



```java
/**
     * 这里重要的是构建
     * ServiceRepository
     *
     * 然后设置
     */
    @SuppressWarnings({"unchecked", "rawtypes"})
    private void doExportUrls() {
        /**
         * 这个ServiceRepository是单例的，通过spi获取的
         *
         * ServiceRepository里面的services属性
         * "org.apache.dubbo.rpc.service.EchoService" -> {ServiceDescriptor@3713}
         * "org.apache.dubbo.rpc.service.GenericService" -> {ServiceDescriptor@3715}
         * "org.apache.dubbo.monitor.MetricsService" -> {ServiceDescriptor@3717}
         * "org.apache.dubbo.monitor.MonitorService" -> {ServiceDescriptor@3719}
         */
        ServiceRepository repository = ApplicationModel.getServiceRepository();
        /**
         * ServiceDescriptor
         *  serviceName : org.apache.dubbo.demo.DemoService
         *  serviceInterfaceClass : org.apache.dubbo.demo.DemoService
         *  methods:
         *      "sayHello" -> {ArrayList@3584}  size = 1
         *      "sayHelloAsync" -> {ArrayList@3586}  size = 1
         *  descToMethods:
         *      "sayHello" -> {HashMap@3591}  size = 1
         *      "sayHelloAsync" -> {HashMap@3592}  size = 1
         */
        ServiceDescriptor serviceDescriptor = repository.registerService(getInterfaceClass());
        /**
         * ServiceRepository设置providers、providersWithoutGroup属性
         *
         * providers
         * jianghe-group/org.apache.dubbo.demo.DemoService:1.0.0 -> {ProviderModel@3597}
         *
         * providersWithoutGroup
         * org.apache.dubbo.demo.DemoService:1.0.0 -> {ProviderModel@3597}
         *
         * ProviderModel这里是由serviceMetadata组成的
         */
        repository.registerProvider(
                getUniqueServiceName(),
                ref,
                serviceDescriptor,
                this,
                serviceMetadata
        );
        // registry://127.0.0.1:2181/org.apache.dubbo.registry.RegistryService?application=dubbo-demo-annotation-provider&dubbo=2.0.2&pid=18733&registry=zookeeper&timestamp=1642583719781
        List<URL> registryURLs = ConfigValidationUtils.loadRegistries(this, true);
        /**
         * 0 = {ProtocolConfig@3637} "<dubbo:protocol name="dubbo" host="0.0.0.0" port="20880" />"
         * 1 = {ProtocolConfig@3649} "<dubbo:protocol name="dubbo" host="0.0.0.0" port="20881" />"
         */
        for (ProtocolConfig protocolConfig : protocols) {
            /**
             * jianghe-group/org.apache.dubbo.demo.DemoService:1.0.0
              */
            String pathKey = URL.buildKey(getContextPath(protocolConfig)
                    .map(p -> p + "/" + path)
                    .orElse(path), group, version);
            /**
             * 在这里会在repository的services属性增加
             * org.apache.dubbo.demo.DemoService -> {ServiceDescriptor@3565}
             * 如果接口设置了group、version，这里另外会增加一个
             * jianghe-group/org.apache.dubbo.demo.DemoService:1.0.0 -> {ServiceDescriptor@3565}
             *
             * In case user specified path, register service one more time to map it to path.
              */
            repository.registerService(pathKey, interfaceClass);
            /**
             * 这里设置是@Service的信息
             */
            // TODO, uncomment this line once service key is unified
            serviceMetadata.setServiceKey(pathKey);
            doExportUrlsFor1Protocol(protocolConfig, registryURLs);
        }

```

##### doExportUrlsFor1Protocol
这个地方就是<b>构建url，`URL url = new URL(name, host, port, getContextPath(protocolConfig).map(p -> p + "/" + path).orElse(path), map);`</b>，获取到对应的URL，然后来导出服务<b>`Exporter<?> exporter = PROTOCOL.export(wrapperInvoker);`</b>

```java
/**
     * 一个协议导出服务
     * @param protocolConfig
     * @param registryURLs
     */
    private void doExportUrlsFor1Protocol(ProtocolConfig protocolConfig, List<URL> registryURLs) {
        String name = protocolConfig.getName();
        if (StringUtils.isEmpty(name)) { // 默认值
            name = DUBBO;
        }

        Map<String, String> map = new HashMap<String, String>();
        map.put(SIDE_KEY, PROVIDER_SIDE); // provider

        ServiceConfig.appendRuntimeParameters(map);
        /**
         * 往map里增加属性，条件是：AbstractConfig子类里面get方法上有@Parameter，就可以设置
         */
        AbstractConfig.appendParameters(map, getMetrics());
        AbstractConfig.appendParameters(map, getApplication());
        AbstractConfig.appendParameters(map, getModule());
        // remove 'default.' prefix for configs from ProviderConfig
        // appendParameters(map, provider, Constants.DEFAULT_KEY);
        AbstractConfig.appendParameters(map, provider);
        AbstractConfig.appendParameters(map, protocolConfig);
        /**
         * 在这执行之前，以下是map具体内容，timeout其实还是2000
         * <pre>
         *     "side" -> "provider"
         *     "application" -> "dubbo-demo-annotation-provider"
         *     "release" -> ""
         *     "deprecated" -> "false"
         *     "dubbo" -> "2.0.2"
         *     "loadbalance" -> "leastactive"
         *     "pid" -> "6652"
         *     "dynamic" -> "true"
         *     "version" -> "1"
         *     "timeout" -> "2000"
         *     "timestamp" -> "1643014266614"
         * </pre>
         *
         * 经过下面方法，以下是map具体内容，timeout就变成为1000，而这个1000是注解@Service自己的属性
         *
         * <pro>
         *      "side" -> "provider"
         *      "release" -> ""
         *      "deprecated" -> "false"
         *      "dubbo" -> "2.0.2"
         *      "loadbalance" -> "leastactive"
         *      "pid" -> "6652"
         *      "interface" -> "org.apache.dubbo.demo.DemoService"
         *      "version" -> "1.0.0"
         *      "timeout" -> "1000"
         *      "generic" -> "false"
         *      "application" -> "dubbo-demo-annotation-provider"
         *      "dynamic" -> "true"
         *      "timestamp" -> "1643014266614"
         *      "group" -> "jianghe-group"
         * </pro>
         */
        AbstractConfig.appendParameters(map, this);
        MetadataReportConfig metadataReportConfig = getMetadataReportConfig();
        if (metadataReportConfig != null && metadataReportConfig.isValid()) {
            map.putIfAbsent(METADATA_KEY, REMOTE_METADATA_STORAGE_TYPE);
        }
        if (CollectionUtils.isNotEmpty(getMethods())) { // @Service 里面可配置method属性，配置后这里就会解析
            for (MethodConfig method : getMethods()) {
                AbstractConfig.appendParameters(map, method, method.getName());
                String retryKey = method.getName() + ".retry";
                if (map.containsKey(retryKey)) {
                    String retryValue = map.remove(retryKey);
                    if ("false".equals(retryValue)) {
                        map.put(method.getName() + ".retries", "0");
                    }
                }
                List<ArgumentConfig> arguments = method.getArguments();
                if (CollectionUtils.isNotEmpty(arguments)) {
                    for (ArgumentConfig argument : arguments) {
                        // convert argument type
                        if (argument.getType() != null && argument.getType().length() > 0) {
                            Method[] methods = interfaceClass.getMethods();
                            // visit all methods
                            if (methods.length > 0) {
                                for (int i = 0; i < methods.length; i++) {
                                    String methodName = methods[i].getName();
                                    // target the method, and get its signature
                                    if (methodName.equals(method.getName())) {
                                        Class<?>[] argtypes = methods[i].getParameterTypes();
                                        // one callback in the method
                                        if (argument.getIndex() != -1) {
                                            if (argtypes[argument.getIndex()].getName().equals(argument.getType())) {
                                                AbstractConfig.appendParameters(map, argument, method.getName() + "." + argument.getIndex());
                                            } else {
                                                throw new IllegalArgumentException("Argument config error : the index attribute and type attribute not match :index :" + argument.getIndex() + ", type:" + argument.getType());
                                            }
                                        } else {
                                            // multiple callbacks in the method
                                            for (int j = 0; j < argtypes.length; j++) {
                                                Class<?> argclazz = argtypes[j];
                                                if (argclazz.getName().equals(argument.getType())) {
                                                    AbstractConfig.appendParameters(map, argument, method.getName() + "." + j);
                                                    if (argument.getIndex() != -1 && argument.getIndex() != j) {
                                                        throw new IllegalArgumentException("Argument config error : the index attribute and type attribute not match :index :" + argument.getIndex() + ", type:" + argument.getType());
                                                    }
                                                }
                                            }
                                        }
                                    }
                                }
                            }
                        } else if (argument.getIndex() != -1) {
                            AbstractConfig.appendParameters(map, argument, method.getName() + "." + argument.getIndex());
                        } else {
                            throw new IllegalArgumentException("Argument config must set index or type attribute.eg: <dubbo:argument index='0' .../> or <dubbo:argument type=xxx .../>");
                        }

                    }
                }
            } // end of methods for
        }

        if (ProtocolUtils.isGeneric(generic)) { // 是否是泛化调用
            map.put(GENERIC_KEY, generic);
            map.put(METHODS_KEY, ANY_VALUE);
        } else {
            String revision = Version.getVersion(interfaceClass, version);
            if (revision != null && revision.length() > 0) {
                map.put(REVISION_KEY, revision);
            }
            // 方法参数 methods -> sayHello,sayHelloAsync
            String[] methods = Wrapper.getWrapper(interfaceClass).getMethodNames();
            if (methods.length == 0) {
                logger.warn("No method found in service interface " + interfaceClass.getName());
                map.put(METHODS_KEY, ANY_VALUE);
            } else {
                map.put(METHODS_KEY, StringUtils.join(new HashSet<String>(Arrays.asList(methods)), ","));
            }
        }

        /**
         * Here the token value configured by the provider is used to assign the value to ServiceConfig#token
         */
        if(ConfigUtils.isEmpty(token) && provider != null) {
            token = provider.getToken();
        }

        if (!ConfigUtils.isEmpty(token)) {
            if (ConfigUtils.isDefault(token)) {
                map.put(TOKEN_KEY, UUID.randomUUID().toString());
            } else {
                map.put(TOKEN_KEY, token);
            }
        }
        //init serviceMetadata attachments
        serviceMetadata.getAttachments().putAll(map);

        // export service
        String host = findConfigedHosts(protocolConfig, registryURLs, map);
        Integer port = findConfigedPorts(protocolConfig, name, map);
        /**
         * 构建URL
         * dubbo://172.16.116.200:20880/org.apache.dubbo.demo.DemoService?anyhost=true&application=dubbo-demo-annotation-provider&bind.ip=172.16.116.200&bind.port=20880&deprecated=false&dubbo=2.0.2&dynamic=true&generic=false&interface=org.apache.dubbo.demo.DemoService&loadbalance=leastactive&methods=sayHello,sayHelloAsync&pid=18733&release=&side=provider&timeout=3000&timestamp=1642583971341&version=1
          */
        URL url = new URL(name, host, port, getContextPath(protocolConfig).map(p -> p + "/" + path).orElse(path), map);

        // 你可以扩展Configurator，来扩展URL参数
        // You can customize Configurator to append extra parameters
        if (ExtensionLoader.getExtensionLoader(ConfiguratorFactory.class)
                .hasExtension(url.getProtocol())) {
            url = ExtensionLoader.getExtensionLoader(ConfiguratorFactory.class)
                    .getExtension(url.getProtocol()).getConfigurator(url).configure(url);
        }

        String scope = url.getParameter(SCOPE_KEY);
        // don't export when none is configured
        if (!SCOPE_NONE.equalsIgnoreCase(scope)) { // 不导出

            // export to local if the config is not remote (export to remote only when config is remote)
            if (!SCOPE_REMOTE.equalsIgnoreCase(scope)) { // 本地导出
                exportLocal(url);
            }
            // export to remote if the config is not local (export to local only when config is local)
            if (!SCOPE_LOCAL.equalsIgnoreCase(scope)) { // 远程导出
                if (CollectionUtils.isNotEmpty(registryURLs)) {
                    for (URL registryURL : registryURLs) {
                        //if protocol is only injvm ,not register
                        if (LOCAL_PROTOCOL.equalsIgnoreCase(url.getProtocol())) {
                            continue;
                        }
                        url = url.addParameterIfAbsent(DYNAMIC_KEY, registryURL.getParameter(DYNAMIC_KEY));
                        URL monitorUrl = ConfigValidationUtils.loadMonitor(this, registryURL);
                        if (monitorUrl != null) {
                            url = url.addParameterAndEncoded(MONITOR_KEY, monitorUrl.toFullString());
                        }
                        if (logger.isInfoEnabled()) {
                            if (url.getParameter(REGISTER_KEY, true)) {
                                logger.info("Register dubbo service " + interfaceClass.getName() + " url " + url + " to registry " + registryURL);
                            } else {
                                logger.info("Export dubbo service " + interfaceClass.getName() + " to url " + url);
                            }
                        }

                        // For providers, this is used to enable custom proxy to generate invoker
                        String proxy = url.getParameter(PROXY_KEY);
                        if (StringUtils.isNotEmpty(proxy)) {
                            registryURL = registryURL.addParameter(PROXY_KEY, proxy);
                        }
                        /**
                         * url : dubbo://127.0.0.1:20880/org.apache.dubbo.demo.DemoService?anyhost=true&application=dubbo-demo-annotation-provider&bind.ip=127.0.0.1&bind.port=20880&deprecated=false&dubbo=2.0.2&dynamic=true&generic=false&interface=org.apache.dubbo.demo.DemoService&loadbalance=leastactive&methods=sayHello,sayHelloAsync&pid=2208&release=&side=provider&timeout=3000&timestamp=1642848259301&version=1
                         *
                         *
                         * registryURL.addParameterAndEncoded(EXPORT_KEY, url.toFullString()) 是下面的url，注意这里是有一个export
                         * registry://127.0.0.1:2181/org.apache.dubbo.registry.RegistryService?application=dubbo-demo-annotation-provider&dubbo=2.0.2&export=dubbo%3A%2F%2F127.0.0.1%3A20880%2Forg.apache.dubbo.demo.DemoService%3Fanyhost%3Dtrue%26application%3Ddubbo-demo-annotation-provider%26bind.ip%3D127.0.0.1%26bind.port%3D20880%26deprecated%3Dfalse%26dubbo%3D2.0.2%26dynamic%3Dtrue%26generic%3Dfalse%26interface%3Dorg.apache.dubbo.demo.DemoService%26loadbalance%3Dleastactive%26methods%3DsayHello%2CsayHelloAsync%26pid%3D2230%26release%3D%26side%3Dprovider%26timeout%3D3000%26timestamp%3D1642848479145%26version%3D1&pid=2230&registry=zookeeper&timestamp=1642848476400
                         *
                         * invoker里的url :  registry://127.0.0.1:2181/org.apache.dubbo.registry.RegistryService?application=dubbo-demo-annotation-provider&dubbo=2.0.2&export=dubbo%3A%2F%2F127.0.0.1%3A20880%2Forg.apache.dubbo.demo.DemoService%3Fanyhost%3Dtrue%26application%3Ddubbo-demo-annotation-provider%26bind.ip%3D127.0.0.1%26bind.port%3D20880%26deprecated%3Dfalse%26dubbo%3D2.0.2%26dynamic%3Dtrue%26generic%3Dfalse%26interface%3Dorg.apache.dubbo.demo.DemoService%26loadbalance%3Dleastactive%26methods%3DsayHello%2CsayHelloAsync%26pid%3D2208%26release%3D%26side%3Dprovider%26timeout%3D3000%26timestamp%3D1642848259301%26version%3D1&pid=2208&registry=zookeeper&timestamp=1642848255194
                         *
                         * URL转Invoker 具体的方法是AbstractProxyInvoker构造方法，生成的是一个可执行的对象
                         * PROXY_FACTORY.getInvoker执行的调用链：JavassistProxyFactory#getInvoker（默认）
                         */
                        Invoker<?> invoker = PROXY_FACTORY.getInvoker(ref, (Class) interfaceClass, registryURL.addParameterAndEncoded(EXPORT_KEY, url.toFullString()));
                        DelegateProviderMetaDataInvoker wrapperInvoker = new DelegateProviderMetaDataInvoker(invoker, this);


                        /**
                         * 真正的导出服务
                         * 
                         * 这里是获取Protocol的Adaptive类，会根据wrapperInvoker的getUrl属性来获取对应的bean，这里获取的就是invoker的url
                         * 那这里就是RegistryProtocol
                         */
                        Exporter<?> exporter = PROTOCOL.export(wrapperInvoker);
                        exporters.add(exporter); // exporter在这里添加后，为unexport使用
                    }
                } else {
                    if (logger.isInfoEnabled()) {
                        logger.info("Export dubbo service " + interfaceClass.getName() + " to url " + url);
                    }
                    Invoker<?> invoker = PROXY_FACTORY.getInvoker(ref, (Class) interfaceClass, url);
                    DelegateProviderMetaDataInvoker wrapperInvoker = new DelegateProviderMetaDataInvoker(invoker, this);
                    Exporter<?> exporter = PROTOCOL.export(wrapperInvoker);
                    exporters.add(exporter);
                }
                /**
                 * @since 2.7.0
                 * ServiceData Store
                 *
                 * InMemoryWritableMetadataService
                 */
                WritableMetadataService metadataService = WritableMetadataService.getExtension(url.getParameter(METADATA_KEY, DEFAULT_METADATA_STORAGE_TYPE));
                if (metadataService != null) {
                    metadataService.publishServiceDefinition(url);
                }
            }
        }
        this.urls.add(url);
    }
```

##### RegistryProtocol#export

这里就是导出服务的地方，这里是<b>`RegistryProtocol`类</b>
<b>1. `OverrideListener`就是监听配置</b>
<b>2. `doLocalExport(originInvoker, providerUrl)`就是启动服务</b>
<b>3. `register(registryUrl, registeredProviderUrl);`就是注册中心注册</b>

```java
/**
     * 导出
     * @param originInvoker
     * @param <T>
     * @return
     * @throws RpcException
     */
    @Override
    public <T> Exporter<T> export(final Invoker<T> originInvoker) throws RpcException {
        /**
         * 这个url其实是包含key "export"，而export是具体服务
         * zookeeper://127.0.0.1:2181/org.apache.dubbo.registry.RegistryService?application=dubbo-demo-annotation-provider&dubbo=2.0.2&export=dubbo%3A%2F%2F127.0.0.1%3A20880%2Forg.apache.dubbo.demo.DemoService%3Fanyhost%3Dtrue%26application%3Ddubbo-demo-annotation-provider%26bind.ip%3D127.0.0.1%26bind.port%3D20880%26deprecated%3Dfalse%26dubbo%3D2.0.2%26dynamic%3Dtrue%26generic%3Dfalse%26interface%3Dorg.apache.dubbo.demo.DemoService%26loadbalance%3Dleastactive%26methods%3DsayHello%2CsayHelloAsync%26pid%3D2230%26release%3D%26side%3Dprovider%26timeout%3D3000%26timestamp%3D1642848479145%26version%3D1&pid=2230&timestamp=1642848476400
         */
        URL registryUrl = getRegistryUrl(originInvoker);
        // url to export locally
        /**
         * dubbo://127.0.0.1:20880/org.apache.dubbo.demo.DemoService?anyhost=true&application=dubbo-demo-annotation-provider&bind.ip=127.0.0.1&bind.port=20880&deprecated=false&dubbo=2.0.2&dynamic=true&generic=false&interface=org.apache.dubbo.demo.DemoService&loadbalance=leastactive&methods=sayHello,sayHelloAsync&pid=2230&release=&side=provider&timeout=3000&timestamp=1642848479145&version=1
         */
        URL providerUrl = getProviderUrl(originInvoker);

        // Subscribe the override data
        // FIXME When the provider subscribes, it will affect the scene : a certain JVM exposes the service and call
        //  the same service. Because the subscribed is cached key with the name of the service, it causes the
        //  subscription information to cover.
        /**
         *
         * 订阅URL，增加下面参数category=configurators&check=false
         *
         * 转换的URL:
         * provider://172.16.116.200:20880/org.apache.dubbo.demo.DemoService?anyhost=true&application=dubbo-demo-annotation-provider&bind.ip=172.16.116.200&bind.port=20880&category=configurators&check=false&deprecated=false&dubbo=2.0.2&dynamic=true&generic=false&group=jianghe-group&interface=org.apache.dubbo.demo.DemoService&loadbalance=leastactive&methods=sayHello,sayHelloAsync&pid=7941&release=&revision=1.0.0&side=provider&timeout=1000&timestamp=1643023876985&version=1.0.0
         */
        final URL overrideSubscribeUrl = getSubscribedOverrideUrl(providerUrl);
        /**
         * 新增覆盖监听器
         */
        final OverrideListener overrideSubscribeListener = new OverrideListener(overrideSubscribeUrl, originInvoker);
        overrideListeners.put(overrideSubscribeUrl, overrideSubscribeListener); // cache
        /**
         * 在配置下重写url? 什么配置需要重写url ? 这里overrideSubscribeListener也是用在这里，这个就是dubbo-admin里面的配置管理的动态配置
         * 动态配置分为应用配置和服务配置，应用配置是整个应用配置，而服务配置只是对应某一个服务的配置
         *
         * dubbo://172.16.116.200:20880/org.apache.dubbo.demo.DemoService?anyhost=true&application=dubbo-demo-annotation-provider&bind.ip=172.16.116.200&bind.port=20880&deprecated=false&dubbo=2.0.2&dynamic=true&generic=false&group=jianghe-group&interface=org.apache.dubbo.demo.DemoService&loadbalance=leastactive&methods=sayHello,sayHelloAsync&pid=7941&release=&revision=1.0.0&side=provider&timeout=1000&timestamp=1643023876985&version=1.0.0
         */
        providerUrl = overrideUrlWithConfig(providerUrl, overrideSubscribeListener);
        /**
         * export invoker
         *
         * 这里就是DubboProtocol 来导出服务，然后启动NettyServer
         */
        final ExporterChangeableWrapper<T> exporter = doLocalExport(originInvoker, providerUrl);

        // url to registry
        final Registry registry = getRegistry(originInvoker);
        /**
         * 这个是最终版本URL，去除一些不必要的参数
         */
        final URL registeredProviderUrl = getUrlToRegistry(providerUrl, registryUrl); //dubbo://172.16.116.200:20880/org.apache.dubbo.demo.DemoService?anyhost=true&application=dubbo-demo-annotation-provider&deprecated=false&dubbo=2.0.2&dynamic=true&generic=false&group=jianghe-group&interface=org.apache.dubbo.demo.DemoService&loadbalance=leastactive&methods=sayHello,sayHelloAsync&pid=8128&release=&revision=1.0.0&side=provider&timeout=1000&timestamp=1643025299490&version=1.0.0

        // decide if we need to delay publish
        boolean register = providerUrl.getParameter(REGISTER_KEY, true);
        if (register) { // 默认true
            /**
             * 注册服务到zk上面，这里就是ZookeeperRegistry（其中dynamic属性判断是否是临时节点，默认是）
             * org.apache.dubbo.registry.ListenerRegistryWrapper#register(org.apache.dubbo.common.URL)
             *      org.apache.dubbo.registry.support.FailbackRegistry#register(org.apache.dubbo.common.URL)
             *             org.apache.dubbo.registry.zookeeper.ZookeeperRegistry#doRegister(org.apache.dubbo.common.URL)
             */
            register(registryUrl, registeredProviderUrl);
        }

        /**
         * register stated url on provider model
          */
        registerStatedUrl(registryUrl, registeredProviderUrl, register);

        /**
         * Deprecated! Subscribe to override rules in 2.6.x or before.
         *
         * 这个是监听老的dubbo-admin里面的配置管理，只是做下兼容
         *
         * 订阅，同样也是zk上面的操作，这里也是ZookeeperRegistry
         * org.apache.dubbo.registry.ListenerRegistryWrapper#subscribe(org.apache.dubbo.common.URL)
         *      org.apache.dubbo.registry.support.FailbackRegistry#subscribe(org.apache.dubbo.common.URL)
         *             org.apache.dubbo.registry.zookeeper.ZookeeperRegistry#doSubscribe(org.apache.dubbo.common.URL)
          */
        registry.subscribe(overrideSubscribeUrl, overrideSubscribeListener);

        exporter.setRegisterUrl(registeredProviderUrl);
        exporter.setSubscribeUrl(overrideSubscribeUrl);

        notifyExport(exporter);
        /**
         * Ensure that a new exporter instance is returned every time export
         *
         * DestroyableExporter -> ExporterChangeableWrapper -> ProtocolFilterWrapper -> ProtocolListenerWrapper -> DubboExporter
         * 这里返回是Exporter对象？逻辑在于unexport
         */
        return new DestroyableExporter<>(exporter);
    }
```
其中`doLocalExport`就是导出服务，`register`往zk注册


流程图
![doExport](/images/dubbo-5-3-3.png)

#### 启动服务

流程图
![openServer](/images/dubbo-5-3-4.png)

#### 服务调用
启动服务后，里面handler加载顺序，调用顺序
<font color='red'><b>NettyServerHandler->NettyServer->MultiMessageHandler->HeartbeatHandler->AllChannelHandler->DecodeHandler->HeaderExchangeHandler->ExchangeHandlerAdapter</b></font>

1. NettyServerHandler
与NettyServer直接绑定的请求处理器，负责从Netty接受请求，channelRead方法获取到请求，调用到下一层的Handler(NettyServer)的received，此时对象还是Object

```java
@Override
public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
    NettyChannel channel = NettyChannel.getOrAddChannel(ctx.channel(), url, handler);
    handler.received(channel, msg); // handler就是NettyServer
}
```

2. NettyServer
NettyServer的父类AbstractPeer中存在received方法，该方法没有做什么，直接走到下一层

```java
@Override
    public void received(Channel ch, Object msg) throws RemotingException {
        if (closed) {
            return;
        }
        handler.received(ch, msg);
    }
```

3. MultiMessageHandler
MultiMessageHandler会判断message是MultiMessage（就是多个Invocation合并成一个），如果是的话，会循环该message，传给下一层Handler（HeartbeatHandler，这个msg对象是确定的），如果不是的话，直接传给下一层Handler（HeartbeatHandler）

```java
@SuppressWarnings("unchecked")
@Override
public void received(Channel channel, Object message) throws RemotingException {
    if (message instanceof MultiMessage) {
        MultiMessage list = (MultiMessage) message;
        for (Object obj : list) {
            handler.received(channel, obj);
        }
    } else {
        handler.received(channel, message);
    }
}
```

4. HeartbeatHandler
判断msg如果是心跳请求，返回一个Response对象，如果是心跳反应，不做什么，都不是的话，直接返回下一层Handler（AllChannelHandler）

```java
@Override
    public void received(Channel channel, Object message) throws RemotingException {
        setReadTimestamp(channel);
        if (isHeartbeatRequest(message)) {
            Request req = (Request) message;
            if (req.isTwoWay()) {
                Response res = new Response(req.getId(), req.getVersion());
                res.setEvent(HEARTBEAT_EVENT);
                channel.send(res);
                if (logger.isInfoEnabled()) {
                    int heartbeat = channel.getUrl().getParameter(Constants.HEARTBEAT_KEY, 0);
                    if (logger.isDebugEnabled()) {
                        logger.debug("Received heartbeat from remote channel " + channel.getRemoteAddress()
                                + ", cause: The channel has no data-transmission exceeds a heartbeat period"
                                + (heartbeat > 0 ? ": " + heartbeat + "ms" : ""));
                    }
                }
            }
            return;
        }
        if (isHeartbeatResponse(message)) {
            if (logger.isDebugEnabled()) {
                logger.debug("Receive heartbeat response in thread " + Thread.currentThread().getName());
            }
            return;
        }
        handler.received(channel, message);
    }

    private boolean isHeartbeatRequest(Object message) {
        return message instanceof Request && ((Request) message).isHeartbeat();
    }

    private boolean isHeartbeatResponse(Object message) {
        return message instanceof Response && ((Response) message).isHeartbeat();
    }
```

5. AllChannelHandler

<font color='red'><b>这个是相当重要的handler，逻辑将msg封装成一个ChannelEventRunnable对象，然后把对象放进到线程池里，异步来处理msg，在ChannelEventRunnable里的run会调用下一层DecodeHandler，那这里要说明下，其实是把Netty里的IOThreads换成线程池处理，这里就释放了IOThreads的压力。</b></font>

```java
@Override
public void received(Channel channel, Object message) throws RemotingException {
    ExecutorService executor = getPreferredExecutorService(message);
    try {
        executor.execute(new ChannelEventRunnable(channel, handler, ChannelState.RECEIVED, message));
    } catch (Throwable t) {
        if(message instanceof Request && t instanceof RejectedExecutionException){
            sendFeedback(channel, (Request) message, t);
            return;
        }
        throw new ExecutionException(message, channel, getClass() + " error when process received event .", t);
    }
}

public ExecutorService getPreferredExecutorService(Object msg) {
        if (msg instanceof Response) {
            Response response = (Response) msg;
            DefaultFuture responseFuture = DefaultFuture.getFuture(response.getId());
            // a typical scenario is the response returned after timeout, the timeout response may has completed the future
            if (responseFuture == null) {
                return getSharedExecutorService();
            } else {
                ExecutorService executor = responseFuture.getExecutor();
                if (executor == null || executor.isShutdown()) {
                    executor = getSharedExecutorService();
                }
                return executor;
            }
        } else {
            return getSharedExecutorService();
        }
    }
```

```java
@Override
public class ChannelEventRunnable implements Runnable {
    public void run() {
        if (state == ChannelState.RECEIVED) {
            try {
                handler.received(channel, message);
            } catch (Exception e) {
                logger.warn("ChannelEventRunnable handle " + state + " operation error, channel is " + channel
                        + ", message is " + message, e);
            }
        } else {
            switch (state) {
            case CONNECTED:
                try {
                    handler.connected(channel);
                } catch (Exception e) {
                    logger.warn("ChannelEventRunnable handle " + state + " operation error, channel is " + channel, e);
                }
                break;
            case DISCONNECTED:
                try {
                    handler.disconnected(channel);
                } catch (Exception e) {
                    logger.warn("ChannelEventRunnable handle " + state + " operation error, channel is " + channel, e);
                }
                break;
            case SENT:
                try {
                    handler.sent(channel, message);
                } catch (Exception e) {
                    logger.warn("ChannelEventRunnable handle " + state + " operation error, channel is " + channel
                            + ", message is " + message, e);
                }
                break;
            case CAUGHT:
                try {
                    handler.caught(channel, exception);
                } catch (Exception e) {
                    logger.warn("ChannelEventRunnable handle " + state + " operation error, channel is " + channel
                            + ", message is: " + message + ", exception is " + exception, e);
                }
                break;
            default:
                logger.warn("unknown state: " + state + ", message is " + message);
            }
        }

    }
}
```

再次说明下，其实这里是dubbo的线程模型，详细的可看：https://dubbo.apache.org/zh/docs/advanced/thread-model/
![线程模型](/images/dubbo-protocol.jpg)
Dispatcher
* `all`所有消息都派发到线程池，包括请求，响应，连接事件，断开事件，心跳等
* `direct`所有消息都不派发到线程池，全部在 IO 线程上直接执行
* `message`只有请求响应消息派发到线程池，其它连接断开事件，心跳等消息，直接在 IO 线程上执行
* `execution`只有请求消息派发到线程池，不含响应，响应和其它连接断开事件，心跳等消息，直接在 IO 线程上执行
* `connection`在 IO 线程上，将连接断开事件放入队列，有序逐个执行，其它消息派发到线程池

6. DecodeHandler
通过received来接受信息，会对msg解码，调用下一层Handler(HeaderExchangeHandler)

```java
@Override
public void received(Channel channel, Object message) throws RemotingException {
    if (message instanceof Decodeable) {
        decode(message);
    }

    if (message instanceof Request) {
        decode(((Request) message).getData());
    }

    if (message instanceof Response) {
        decode(((Response) message).getResult());
    }

    handler.received(channel, message);
}

private void decode(Object message) {
        if (message instanceof Decodeable) {
            try {
                ((Decodeable) message).decode();
                if (log.isDebugEnabled()) {
                    log.debug("Decode decodeable message " + message.getClass().getName());
                }
            } catch (Throwable e) {
                if (log.isWarnEnabled()) {
                    log.warn("Call Decodeable.decode failed: " + e.getMessage(), e);
                }
            } // ~ end of catch
        } // ~ end of if
    } // ~ end of method decode
```

7. HeaderExchangeHandler
通过received来调用，会判断msg类型
    1. 如果msg = Request，request是TwoWay，会调用下一层handler（ExchangeHandlerAdapter）的reply，返回对象
    1. 如果msg = Request，request不是TwoWay，直接调用下一层Handler（ExchangeHandlerAdapter）的received不返回对象

```java
@Override
public void received(Channel channel, Object message) throws RemotingException {
    final ExchangeChannel exchangeChannel = HeaderExchangeChannel.getOrAddChannel(channel);
    if (message instanceof Request) {
        // handle request.
        Request request = (Request) message;
        if (request.isEvent()) {
            handlerEvent(channel, request);
        } else {
            if (request.isTwoWay()) {
                handleRequest(exchangeChannel, request);
            } else {
                handler.received(exchangeChannel, request.getData());
            }
        }
    } else if (message instanceof Response) {
        handleResponse(channel, (Response) message);
    } else if (message instanceof String) {
        if (isClientSide(channel)) {
            Exception e = new Exception("Dubbo client can not supported string message: " + message + " in channel: " + channel + ", url: " + channel.getUrl());
            logger.error(e.getMessage(), e);
        } else {
            String echo = handler.telnet(channel, (String) message);
            if (echo != null && echo.length() > 0) {
                channel.send(echo);
            }
        }
    } else {
        handler.received(exchangeChannel, message);
    }
}

void handleRequest(final ExchangeChannel channel, Request req) throws RemotingException {
        Response res = new Response(req.getId(), req.getVersion());
        if (req.isBroken()) {
            Object data = req.getData();

            String msg;
            if (data == null) {
                msg = null;
            } else if (data instanceof Throwable) {
                msg = StringUtils.toString((Throwable) data);
            } else {
                msg = data.toString();
            }
            res.setErrorMessage("Fail to decode request due to: " + msg);
            res.setStatus(Response.BAD_REQUEST);

            channel.send(res);
            return;
        }
        // find handler by message class.
        Object msg = req.getData();
        try {
            CompletionStage<Object> future = handler.reply(channel, msg);
            future.whenComplete((appResult, t) -> {
                try {
                    if (t == null) {
                        res.setStatus(Response.OK);
                        res.setResult(appResult);
                    } else {
                        res.setStatus(Response.SERVICE_ERROR);
                        res.setErrorMessage(StringUtils.toString(t));
                    }
                    channel.send(res);
                } catch (RemotingException e) {
                    logger.warn("Send result to consumer failed, channel is " + channel + ", msg is " + e);
                }
            });
        } catch (Throwable e) {
            res.setStatus(Response.SERVICE_ERROR);
            res.setErrorMessage(StringUtils.toString(e));
            channel.send(res);
        }
    }
```
8. ExchangeHandlerAdapter
<font color='red'><b>这个处理器是真正处理器，reply是返回对象的，真正返回对象的invoker.invoke执行的结果，而是received是不返回对象的，其中请求过来是Invocation对象，然后根据Invocation对象拿到serviceKey，然后再从exporterMap里面获取DubboExporter，然后真正获取Invoker，拿到Invoker后就会调用Invoker.invoke方法真正调用结果</b></font>

```java
private ExchangeHandler requestHandler = new ExchangeHandlerAdapter() {

        @Override
        public CompletableFuture<Object> reply(ExchangeChannel channel, Object message) throws RemotingException {

            if (!(message instanceof Invocation)) {
                throw new RemotingException(channel, "Unsupported request: "
                        + (message == null ? null : (message.getClass().getName() + ": " + message))
                        + ", channel: consumer: " + channel.getRemoteAddress() + " --> provider: " + channel.getLocalAddress());
            }

            Invocation inv = (Invocation) message;
            Invoker<?> invoker = getInvoker(channel, inv);
            // need to consider backward-compatibility if it's a callback
            if (Boolean.TRUE.toString().equals(inv.getObjectAttachments().get(IS_CALLBACK_SERVICE_INVOKE))) {
                String methodsStr = invoker.getUrl().getParameters().get("methods");
                boolean hasMethod = false;
                if (methodsStr == null || !methodsStr.contains(",")) {
                    hasMethod = inv.getMethodName().equals(methodsStr);
                } else {
                    String[] methods = methodsStr.split(",");
                    for (String method : methods) {
                        if (inv.getMethodName().equals(method)) {
                            hasMethod = true;
                            break;
                        }
                    }
                }
                if (!hasMethod) {
                    logger.warn(new IllegalStateException("The methodName " + inv.getMethodName()
                            + " not found in callback service interface ,invoke will be ignored."
                            + " please update the api interface. url is:"
                            + invoker.getUrl()) + " ,invocation is :" + inv);
                    return null;
                }
            }
            RpcContext.getContext().setRemoteAddress(channel.getRemoteAddress());
            Result result = invoker.invoke(inv);
            return result.thenApply(Function.identity());
        }

        @Override
        public void received(Channel channel, Object message) throws RemotingException {
            if (message instanceof Invocation) {
                reply((ExchangeChannel) channel, message);

            } else {
                super.received(channel, message);
            }
        }

        @Override
        public void connected(Channel channel) throws RemotingException {
            invoke(channel, ON_CONNECT_KEY);
        }

        @Override
        public void disconnected(Channel channel) throws RemotingException {
            if (logger.isDebugEnabled()) {
                logger.debug("disconnected from " + channel.getRemoteAddress() + ",url:" + channel.getUrl());
            }
            invoke(channel, ON_DISCONNECT_KEY);
        }

        private void invoke(Channel channel, String methodKey) {
            Invocation invocation = createInvocation(channel, channel.getUrl(), methodKey);
            if (invocation != null) {
                try {
                    received(channel, invocation);
                } catch (Throwable t) {
                    logger.warn("Failed to invoke event method " + invocation.getMethodName() + "(), cause: " + t.getMessage(), t);
                }
            }
        }

        /**
         * FIXME channel.getUrl() always binds to a fixed service, and this service is random.
         * we can choose to use a common service to carry onConnect event if there's no easy way to get the specific
         * service this connection is binding to.
         * @param channel
         * @param url
         * @param methodKey
         * @return
         */
        private Invocation createInvocation(Channel channel, URL url, String methodKey) {
            String method = url.getParameter(methodKey);
            if (method == null || method.length() == 0) {
                return null;
            }

            RpcInvocation invocation = new RpcInvocation(method, url.getParameter(INTERFACE_KEY), new Class<?>[0], new Object[0]);
            invocation.setAttachment(PATH_KEY, url.getPath());
            invocation.setAttachment(GROUP_KEY, url.getParameter(GROUP_KEY));
            invocation.setAttachment(INTERFACE_KEY, url.getParameter(INTERFACE_KEY));
            invocation.setAttachment(VERSION_KEY, url.getParameter(VERSION_KEY));
            if (url.getParameter(STUB_EVENT_KEY, false)) {
                invocation.setAttachment(STUB_EVENT_KEY, Boolean.TRUE.toString());
            }

            return invocation;
        }
    };
```
```java
Invoker<?> getInvoker(Channel channel, Invocation inv) throws RemotingException {
        boolean isCallBackServiceInvoke = false;
        boolean isStubServiceInvoke = false;
        int port = channel.getLocalAddress().getPort();
        String path = (String) inv.getObjectAttachments().get(PATH_KEY);

        // if it's callback service on client side
        isStubServiceInvoke = Boolean.TRUE.toString().equals(inv.getObjectAttachments().get(STUB_EVENT_KEY));
        if (isStubServiceInvoke) {
            port = channel.getRemoteAddress().getPort();
        }

        //callback
        isCallBackServiceInvoke = isClientSide(channel) && !isStubServiceInvoke;
        if (isCallBackServiceInvoke) {
            path += "." + inv.getObjectAttachments().get(CALLBACK_SERVICE_KEY);
            inv.getObjectAttachments().put(IS_CALLBACK_SERVICE_INVOKE, Boolean.TRUE.toString());
        }

        String serviceKey = serviceKey(
                port,
                path,
                (String) inv.getObjectAttachments().get(VERSION_KEY),
                (String) inv.getObjectAttachments().get(GROUP_KEY)
        );
        DubboExporter<?> exporter = (DubboExporter<?>) exporterMap.get(serviceKey);

        if (exporter == null) {
            throw new RemotingException(channel, "Not found exported service: " + serviceKey + " in " + exporterMap.keySet() + ", may be version or group mismatch " +
                    ", channel: consumer: " + channel.getRemoteAddress() + " --> provider: " + channel.getLocalAddress() + ", message:" + getInvocationWithoutData(inv));
        }

        return exporter.getInvoker();
    }
```

## Invoker
下面的invoker链就是在服务export导出的时候构建出来的

<font color='red'><b>ProtocolFilterWrapper->RegistryProtocol$InvokerDelegate->DelegateProviderMetaDataInvoker->AbstractProxyInvoker</b></font>

1. ProtocolFilterWrapper
其中有8个filters
0 = {EchoFilter@3367} 
1 = {ClassLoaderFilter@3368} 
2 = {GenericFilter@3369} 
3 = {ContextFilter@3370} 
4 = {TraceFilter@3371} 
5 = {TimeoutFilter@3372} 
6 = {MonitorFilter@3373} 
7 = {ExceptionFilter@3374} 
执行8个filters是倒叙执行，其中在这里就新建Invoker，把上一个的Invoker做参数，第一个参数当时是RegistryProtocol$InvokerDelegate，然后再调用的时候会执行Invoke里的invoke方法，里面就会执行filter.invoke，就会执行这些过滤器了

```java
private static <T> Invoker<T> buildInvokerChain(final Invoker<T> invoker, String key, String group) {
        Invoker<T> last = invoker;
        List<Filter> filters = ExtensionLoader.getExtensionLoader(Filter.class).getActivateExtension(invoker.getUrl(), key, group);

        if (!filters.isEmpty()) {
            for (int i = filters.size() - 1; i >= 0; i--) { // 这里有8个filter
                final Filter filter = filters.get(i);
                final Invoker<T> next = last;
                last = new Invoker<T>() {

                    @Override
                    public Class<T> getInterface() {
                        return invoker.getInterface();
                    }

                    @Override
                    public URL getUrl() {
                        return invoker.getUrl();
                    }

                    @Override
                    public boolean isAvailable() {
                        return invoker.isAvailable();
                    }

                    @Override
                    public Result invoke(Invocation invocation) throws RpcException {
                        Result asyncResult;
                        try {
                            asyncResult = filter.invoke(next, invocation);
                        } catch (Exception e) {
                            if (filter instanceof ListenableFilter) {
                                ListenableFilter listenableFilter = ((ListenableFilter) filter);
                                try {
                                    Filter.Listener listener = listenableFilter.listener(invocation);
                                    if (listener != null) {
                                        listener.onError(e, invoker, invocation);
                                    }
                                } finally {
                                    listenableFilter.removeListener(invocation);
                                }
                            } else if (filter instanceof Filter.Listener) {
                                Filter.Listener listener = (Filter.Listener) filter;
                                listener.onError(e, invoker, invocation);
                            }
                            throw e;
                        } finally {

                        }
                        return asyncResult.whenCompleteWithContext((r, t) -> {
                            if (filter instanceof ListenableFilter) {
                                ListenableFilter listenableFilter = ((ListenableFilter) filter);
                                Filter.Listener listener = listenableFilter.listener(invocation);
                                try {
                                    if (listener != null) {
                                        if (t == null) {
                                            listener.onResponse(r, invoker, invocation);
                                        } else {
                                            listener.onError(t, invoker, invocation);
                                        }
                                    }
                                } finally {
                                    listenableFilter.removeListener(invocation);
                                }
                            } else if (filter instanceof Filter.Listener) {
                                Filter.Listener listener = (Filter.Listener) filter;
                                if (t == null) {
                                    listener.onResponse(r, invoker, invocation);
                                } else {
                                    listener.onError(t, invoker, invocation);
                                }
                            }
                        });
                    }

                    @Override
                    public void destroy() {
                        invoker.destroy();
                    }

                    @Override
                    public String toString() {
                        return invoker.toString();
                    }
                };
            }
        }

        return last;
    }

@Override
    public <T> Exporter<T> export(Invoker<T> invoker) throws RpcException {
        if (UrlUtils.isRegistry(invoker.getUrl())) {
            return protocol.export(invoker);
        }
        return protocol.export(buildInvokerChain(invoker, SERVICE_FILTER_KEY, CommonConstants.PROVIDER));
    }

```
2. RegistryProtocol$InvokerDelegate
包含服务对应的providerUrl，和下面的Invoker

3. DelegateProviderMetaDataInvoker
包含ServiceConfig对象和AbstractProxyInvoker对象

4. AbstractProxyInvoker
里面是通过ref, Class<T> 接口class, URL url（dubbo://127.0.0.1:20880/org.apache.dubbo.demo.DemoService?anyhost=true&application=dubbo-demo-annotation-provider&bind.ip=127.0.0.1&bind.port=20880&deprecated=false&dubbo=2.0.2&dynamic=true&generic=false&interface=org.apache.dubbo.demo.DemoService&loadbalance=leastactive&methods=sayHello,sayHelloAsync&pid=2230&release=&side=provider&timeout=3000&timestamp=1642848479145&version=1）组成的代理类


## Exporter
<font color='red'><b>这是一个Dubbo链，这个地方用在哪里？其实是在服务导出失败后的操作，DestroyableExporter->ExporterChangeableWrapper->ListenerExporterWrapper->DubboExporter</b></font>
1. DestroyableExporter

最重要的用unexport
```java
 private static class DestroyableExporter<T> implements Exporter<T> {

    private Exporter<T> exporter;

    public DestroyableExporter(Exporter<T> exporter) {
        this.exporter = exporter;
    }

    @Override
    public Invoker<T> getInvoker() {
        return exporter.getInvoker();
    }

    @Override
    public void unexport() {
        exporter.unexport();
    }
}
```

2. ExporterChangeableWrapper
在unexport之前，把服务URL从zk上面移除，移除动态配置监听

```java
/**
     * exporter proxy, establish the corresponding relationship between the returned exporter and the exporter
     * exported by the protocol, and can modify the relationship at the time of override.
     *
     * @param <T>
     */
    private class ExporterChangeableWrapper<T> implements Exporter<T> {

        private final ExecutorService executor = newSingleThreadExecutor(new NamedThreadFactory("Exporter-Unexport", true));

        private final Invoker<T> originInvoker;
        private Exporter<T> exporter;
        private URL subscribeUrl;
        private URL registerUrl;

        public ExporterChangeableWrapper(Exporter<T> exporter, Invoker<T> originInvoker) {
            this.exporter = exporter;
            this.originInvoker = originInvoker;
        }

        public Invoker<T> getOriginInvoker() {
            return originInvoker;
        }

        @Override
        public Invoker<T> getInvoker() {
            return exporter.getInvoker();
        }

        public void setExporter(Exporter<T> exporter) {
            this.exporter = exporter;
        }

        @Override
        public void unexport() {
            String key = getCacheKey(this.originInvoker);
            bounds.remove(key);

            Registry registry = RegistryProtocol.this.getRegistry(originInvoker);
            try {
                registry.unregister(registerUrl); // 移除注册
            } catch (Throwable t) {
                logger.warn(t.getMessage(), t);
            }
            try {
                NotifyListener listener = RegistryProtocol.this.overrideListeners.remove(subscribeUrl);
                registry.unsubscribe(subscribeUrl, listener); // 移除监听
                ExtensionLoader.getExtensionLoader(GovernanceRuleRepository.class).getDefaultExtension()
                        .removeListener(subscribeUrl.getServiceKey() + CONFIGURATORS_SUFFIX,
                                serviceConfigurationListeners.get(subscribeUrl.getServiceKey()));
            } catch (Throwable t) {
                logger.warn(t.getMessage(), t);
            }

            executor.submit(() -> {
                try {
                    int timeout = ConfigurationUtils.getServerShutdownTimeout();
                    if (timeout > 0) {
                        logger.info("Waiting " + timeout + "ms for registry to notify all consumers before unexport. " +
                                "Usually, this is called when you use dubbo API");
                        Thread.sleep(timeout);
                    }
                    exporter.unexport();
                } catch (Throwable t) {
                    logger.warn(t.getMessage(), t);
                }
            });
        }

        public void setSubscribeUrl(URL subscribeUrl) {
            this.subscribeUrl = subscribeUrl;
        }

        public void setRegisterUrl(URL registerUrl) {
            this.registerUrl = registerUrl;
        }

        public URL getRegisterUrl() {
            return registerUrl;
        }
    }
```

3. ListenerExporterWrapper
这个类负责去除导出监听

```java
{

    private static final Logger logger = LoggerFactory.getLogger(ListenerExporterWrapper.class);

    private final Exporter<T> exporter;

    private final List<ExporterListener> listeners;

    public ListenerExporterWrapper(Exporter<T> exporter, List<ExporterListener> listeners) {
        if (exporter == null) {
            throw new IllegalArgumentException("exporter == null");
        }
        this.exporter = exporter;
        this.listeners = listeners;
        if (CollectionUtils.isNotEmpty(listeners)) {
            RuntimeException exception = null;
            for (ExporterListener listener : listeners) {
                if (listener != null) {
                    try {
                        listener.exported(this);
                    } catch (RuntimeException t) {
                        logger.error(t.getMessage(), t);
                        exception = t;
                    }
                }
            }
            if (exception != null) {
                throw exception;
            }
        }
    }

    @Override
    public Invoker<T> getInvoker() {
        return exporter.getInvoker();
    }

    @Override
    public void unexport() {
        try {
            exporter.unexport();
        } finally {
            if (CollectionUtils.isNotEmpty(listeners)) {
                RuntimeException exception = null;
                for (ExporterListener listener : listeners) {
                    if (listener != null) {
                        try {
                            listener.unexported(this);
                        } catch (RuntimeException t) {
                            logger.error(t.getMessage(), t);
                            exception = t;
                        }
                    }
                }
                if (exception != null) {
                    throw exception;
                }
            }
        }
    }

}
```

4. DubboExporter
这个类就是unexport里就是调用Invoker里的destroy

```java
public class DubboExporter<T> extends AbstractExporter<T> {

    private final String key;

    private final Map<String, Exporter<?>> exporterMap;

    public DubboExporter(Invoker<T> invoker, String key, Map<String, Exporter<?>> exporterMap) {
        super(invoker);
        this.key = key;
        this.exporterMap = exporterMap;
    }

    @Override
    public void unexport() {
        super.unexport();
        exporterMap.remove(key);
    }

}
```
```java
public abstract class AbstractExporter<T> implements Exporter<T> {

    protected final Logger logger = LoggerFactory.getLogger(getClass());

    private final Invoker<T> invoker;

    private volatile boolean unexported = false;

    public AbstractExporter(Invoker<T> invoker) {
        if (invoker == null) {
            throw new IllegalStateException("service invoker == null");
        }
        if (invoker.getInterface() == null) {
            throw new IllegalStateException("service type == null");
        }
        if (invoker.getUrl() == null) {
            throw new IllegalStateException("service url == null");
        }
        this.invoker = invoker;
    }

    @Override
    public Invoker<T> getInvoker() {
        return invoker;
    }

    @Override
    public void unexport() {
        if (unexported) {
            return;
        }
        unexported = true;
        getInvoker().destroy();
    }

    @Override
    public String toString() {
        return getInvoker().toString();
    }

}
```

## 服务监听
服务导出的时候会向dubbo-admin里的动态配置的数据订阅。这个会有新老版本兼容的问题。
> 在dubbo2.7之前，只能支持某个服务的动态配置
> 在dubbo2.7之后，可以支持某个服务和应用的动态配置

在dubbo2.7之前，监听的路径的是：
/dubbo/org.apache.dubbo.demo.DemoService/configurators/override://0.0.0.0/org.apache.dubbo.demo.DemoService?category=configurators&compatible_config=true&dynamic=false&enabled=true&timeout=5000，其中监听的节点路径，不是节点内容


在dubbo2.7之后，监听的路径的是：
1. 服务监听：
/dubbo/config/dubbo/org.apache.dubbo.demo.DemoService:1.0.0:jianghe-group.configurators
监听的是节点内容：
```json
configVersion: v2.7
configs:
- addresses:
  - 0.0.0.0
  enabled: false
  parameters:
    timeout: 5000
  side: consumer
enabled: true
key: org.apache.dubbo.demo.DemoService
scope: service
```
> 注意，这里如果修改服务监听的配置，那老版本的值`/dubbo/org.apache.dubbo.demo.DemoService/configurators/override://0.0.0.0/org.apache.dubbo.demo.DemoService?category=configurators&compatible_config=true&dynamic=false&enabled=true&timeout=5000`也会随着修改

2. 应用监听：
/dubbo/config/dubbo/dubbo-demo-annotation-provider.configurators
监听的是节点内容：
```json
configVersion: v2.7
configs:
- addresses:
  - 0.0.0.0
  enabled: false
  parameters:
    timeout: 3000
  side: consumer
enabled: true
key: dubbo-demo-annotation-provider
scope: application
```


<font color='red'><b>贴一下比较重要的地方，下面服务如果匹配成功的话，就会更新提供者的URL，然后重新的reExport</b></font>
```java
@Override
public URL configure(URL url) {
    // If override url is not enabled or is invalid, just return.
    if (!configuratorUrl.getParameter(ENABLED_KEY, true) || configuratorUrl.getHost() == null || url == null || url.getHost() == null) {
        return url;
    }
    /*
        * This if branch is created since 2.7.0.
        */
    String apiVersion = configuratorUrl.getParameter(CONFIG_VERSION_KEY);// v2.7
    if (StringUtils.isNotEmpty(apiVersion)) {
        String currentSide = url.getParameter(SIDE_KEY); // provider
        String configuratorSide = configuratorUrl.getParameter(SIDE_KEY); // consumer
        if (currentSide.equals(configuratorSide) && CONSUMER.equals(configuratorSide) && 0 == configuratorUrl.getPort()) {
            url = configureIfMatch(NetUtils.getLocalHost(), url);
        } else if (currentSide.equals(configuratorSide) && PROVIDER.equals(configuratorSide) && url.getPort() == configuratorUrl.getPort()) { // 提供端匹配必须是端口要匹配
            url = configureIfMatch(url.getHost(), url);
        }
    }
    /*
        * This else branch is deprecated and is left only to keep compatibility with versions before 2.7.0
        */
    else {
        url = configureDeprecated(url);
    }
    return url;
}
```
以下是动态修改配置调用的堆栈，然后重新导出服务reExport
```java
export:285, DubboProtocol (org.apache.dubbo.rpc.protocol.dubbo)
export:62, ProtocolListenerWrapper (org.apache.dubbo.rpc.protocol)
export:153, ProtocolFilterWrapper (org.apache.dubbo.rpc.protocol)
export:-1, Protocol$Adaptive (org.apache.dubbo.rpc)
reExport:345, RegistryProtocol (org.apache.dubbo.registry.integration)
doOverrideIfNecessary:693, RegistryProtocol$OverrideListener (org.apache.dubbo.registry.integration)
notify:667, RegistryProtocol$OverrideListener (org.apache.dubbo.registry.integration)
notify:426, AbstractRegistry (org.apache.dubbo.registry.support)
doNotify:373, FailbackRegistry (org.apache.dubbo.registry.support)
notify:364, FailbackRegistry (org.apache.dubbo.registry.support)
doSubscribe:180, ZookeeperRegistry (org.apache.dubbo.registry.zookeeper)
doRetry:44, FailedSubscribedTask (org.apache.dubbo.registry.retry)
run:124, AbstractRetryTask (org.apache.dubbo.registry.retry)
expire:648, HashedWheelTimer$HashedWheelTimeout (org.apache.dubbo.common.timer)
expireTimeouts:727, HashedWheelTimer$HashedWheelBucket (org.apache.dubbo.common.timer)
run:449, HashedWheelTimer$Worker (org.apache.dubbo.common.timer)
run:748, Thread (java.lang)
```





# 总结
我们的猜想其实和服务导出的逻辑差不多，那我们这时候再总结下
```text
1. 确定服务的参数
2. 获取最新、最全的配置参数
3. 构建URL
4. 启动服务
5. 注册服务到注册中心
6. 动态配置监听
7. 注册dubbo服务导出完成事件
```
其中也分析服务调用的handler和invoke调用的内容，更加的对服务导出了解
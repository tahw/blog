---
title: dubbo之引入服务详解
date: 2022-02-04 20:29:01
tags:
    - dubbo
categories:
    - dubbo
---

# dubbo之引入服务详解
前言：分析的dubbo源码是2.7.7版本

# 入口
流程图
![入口](/images/dubbo-6-1.png)

```java
public synchronized T get() {
    if (destroyed) {
        throw new IllegalStateException("The invoker of ReferenceConfig(" + url + ") has already destroyed!");
    }
    if (ref == null) {
        init(); // 初始化
    }
    return ref;
}
```
<!-- more -->

# 分析
```text
dubbo引入要做什么？一看到这个其实是有点懵的，dubbo服务导出的时候逻辑还有点清晰，那引入呢？我们这块只能先分析下调用逻辑

消费者端：
private A a;
a.a();

当调用一个接口方法时逻辑是其实这里的a是一个代理对象，那下面的逻辑就是

代理对象.方法(){
    // 分析下方法里面应该做什么

    1. 构建一下invocation
    2. 调用的时候就是invoker.invoke(invocation);
    3. 启动下nettyClient;
    4. 发送下请求

    // 上面四步基本都是知道的，我们接着再分析下
    0. 获取下providers的URL（监听）
    1.1 mock
    1.2 路由（监听）
    1.3 负载
    2.1 容错
}

由上面的逻辑我们总结下调用思路
    0. 读取下providers的URL（读取要缓存下，并且要监听）
    1. 构建下Invocation
    2. MockClusterInvoker.invoke
        FailbackClusterInvoker.invoke（容错）
            RegistryDirectory.list（这里的服务目录RegistryDirectory可以理解成消费端的缓存）
            0. 缓存providers URL(监听)
            1. 路由（监听）
            2. 负载
            4. 选取一个真正的Invoker，然后调用Invoker.invoke方法
    3. 启动下nettyClient
    4. 发生请求

在上面调用总结下，我们总结下引入要做的事情
1. 其实就是构建Invoker链
2. 构建RegistryDirectory，里面包含List<Invoker>，其中的Invoker就是DubboInvoker
3. 监听逻辑

```

# ReferenceConfig#init
流程图
![init](/images/dubbo-6-2.png)
```java
/**
* 初始化，这里初始化后，ref就赋值了
**/
public synchronized void init() {
        if (initialized) {
            return;
        }

        if (bootstrap == null) { // 初始化配置
            bootstrap = DubboBootstrap.getInstance();
            bootstrap.init();
        }

        checkAndUpdateSubConfigs(); // 获取最新最全的ReferenceConfig配置

        checkStubAndLocal(interfaceClass);
        ConfigValidationUtils.checkMock(interfaceClass, this);

        Map<String, String> map = new HashMap<String, String>();
        map.put(SIDE_KEY, CONSUMER_SIDE);

        ReferenceConfigBase.appendRuntimeParameters(map);
        if (!ProtocolUtils.isGeneric(generic)) {
            String revision = Version.getVersion(interfaceClass, version);
            if (revision != null && revision.length() > 0) {
                map.put(REVISION_KEY, revision);
            }

            String[] methods = Wrapper.getWrapper(interfaceClass).getMethodNames();
            if (methods.length == 0) {
                logger.warn("No method found in service interface " + interfaceClass.getName());
                map.put(METHODS_KEY, ANY_VALUE);
            } else {
                map.put(METHODS_KEY, StringUtils.join(new HashSet<String>(Arrays.asList(methods)), COMMA_SEPARATOR));
            }
        }
        map.put(INTERFACE_KEY, interfaceName);
        AbstractConfig.appendParameters(map, getMetrics());
        AbstractConfig.appendParameters(map, getApplication());
        AbstractConfig.appendParameters(map, getModule());
        // remove 'default.' prefix for configs from ConsumerConfig
        // appendParameters(map, consumer, Constants.DEFAULT_KEY);
        AbstractConfig.appendParameters(map, consumer);
        AbstractConfig.appendParameters(map, this);
        MetadataReportConfig metadataReportConfig = getMetadataReportConfig();
        if (metadataReportConfig != null && metadataReportConfig.isValid()) {
            map.putIfAbsent(METADATA_KEY, REMOTE_METADATA_STORAGE_TYPE);
        }
        Map<String, AsyncMethodInfo> attributes = null;
        if (CollectionUtils.isNotEmpty(getMethods())) {
            attributes = new HashMap<>();
            for (MethodConfig methodConfig : getMethods()) {
                AbstractConfig.appendParameters(map, methodConfig, methodConfig.getName());
                String retryKey = methodConfig.getName() + ".retry";
                if (map.containsKey(retryKey)) {
                    String retryValue = map.remove(retryKey);
                    if ("false".equals(retryValue)) {
                        map.put(methodConfig.getName() + ".retries", "0");
                    }
                }
                AsyncMethodInfo asyncMethodInfo = AbstractConfig.convertMethodConfig2AsyncInfo(methodConfig);
                if (asyncMethodInfo != null) {
//                    consumerModel.getMethodModel(methodConfig.getName()).addAttribute(ASYNC_KEY, asyncMethodInfo);
                    attributes.put(methodConfig.getName(), asyncMethodInfo);
                }
            }
        }

        String hostToRegistry = ConfigUtils.getSystemProperty(DUBBO_IP_TO_REGISTRY);
        if (StringUtils.isEmpty(hostToRegistry)) {
            hostToRegistry = NetUtils.getLocalHost();
        } else if (isInvalidLocalHost(hostToRegistry)) {
            throw new IllegalArgumentException("Specified invalid registry ip from property:" + DUBBO_IP_TO_REGISTRY + ", value:" + hostToRegistry);
        }
        map.put(REGISTER_IP_KEY, hostToRegistry);

        serviceMetadata.getAttachments().putAll(map);

        /**
         * "init" -> "false"
         * "side" -> "consumer"
         * "register.ip" -> "127.0.0.1"
         * "release" -> ""
         * "methods" -> "sayHello,sayHelloAsync"
         * "dubbo" -> "2.0.2"
         * "pid" -> "15300"
         * "interface" -> "org.apache.dubbo.demo.DemoService"
         * "version" -> "1.0.0"
         * "timeout" -> "3000"
         * "revision" -> "1.0.0"
         * "application" -> "dubbo-demo-annotation-consumer"
         * "sticky" -> "false"
         * "timestamp" -> "1644129114622"
         * "group" -> "jianghe-group"
         */
        ref = createProxy(map);

        serviceMetadata.setTarget(ref);
        serviceMetadata.addAttribute(PROXY_CLASS_REF, ref);
        ConsumerModel consumerModel = repository.lookupReferredService(serviceMetadata.getServiceKey());
        consumerModel.setProxyObject(ref);
        consumerModel.init(attributes);

        initialized = true;

        // dispatch a ReferenceConfigInitializedEvent since 2.7.4
        dispatch(new ReferenceConfigInitializedEvent(this, invoker));
    }
```
上面初始化ReferenceBean参数很大部分和ServiceBean类似，都是获取最新最全的配置，然后将信息放入到map里面，这块就不细说了，详细可以看ServiceBean配置，重要的是`createProxy`，这个就是创建代理对象，然后设置到`ConsumerModel`，`ConsumerModel`是在调用的时候使用的，后续调用的时候再介绍。
ReferenceBean初始化完成后就发布`ReferenceConfigInitializedEvent`成功事件

## ReferenceConfig#createProxy
```java
@SuppressWarnings({"unchecked", "rawtypes", "deprecation"})
    private T createProxy(Map<String, String> map) {
        if (shouldJvmRefer(map)) { // false
            URL url = new URL(LOCAL_PROTOCOL, LOCALHOST_VALUE, 0, interfaceClass.getName()).addParameters(map);
            invoker = REF_PROTOCOL.refer(interfaceClass, url);
            if (logger.isInfoEnabled()) {
                logger.info("Using injvm service " + interfaceClass.getName());
            }
        } else {
            urls.clear();
            if (url != null && url.length() > 0) { // user specified URL, could be peer-to-peer address, or register center's address.
                String[] us = SEMICOLON_SPLIT_PATTERN.split(url);
                if (us != null && us.length > 0) {
                    for (String u : us) {
                        URL url = URL.valueOf(u);
                        if (StringUtils.isEmpty(url.getPath())) {
                            url = url.setPath(interfaceName);
                        }
                        if (UrlUtils.isRegistry(url)) {
                            urls.add(url.addParameterAndEncoded(REFER_KEY, StringUtils.toQueryString(map)));
                        } else {
                            urls.add(ClusterUtils.mergeUrl(url, map));
                        }
                    }
                }
            } else { // assemble URL from register center's configuration
                // if protocols not injvm checkRegistry
                if (!LOCAL_PROTOCOL.equalsIgnoreCase(getProtocol())) { // true
                    checkRegistry(); // 检查协议
                    /**
                     * 这里的this就是ReferenceConfig，将ReferenceConfig属性转换为URL
                     * 
                     * registry://127.0.0.1:2181/org.apache.dubbo.registry.RegistryService?application=dubbo-demo-annotation-consumer&dubbo=2.0.2&pid=12982&registry=zookeeper&timestamp=1644053669416
                     */
                    List<URL> us = ConfigValidationUtils.loadRegistries(this, false);
                    if (CollectionUtils.isNotEmpty(us)) {
                        for (URL u : us) {
                            URL monitorUrl = ConfigValidationUtils.loadMonitor(this, u);
                            if (monitorUrl != null) {
                                map.put(MONITOR_KEY, URL.encode(monitorUrl.toFullString()));
                            }
                            /**
                             * 里面会包含refer关键字
                             * registry://127.0.0.1:2181/org.apache.dubbo.registry.RegistryService?application=dubbo-demo-annotation-consumer&dubbo=2.0.2&pid=12982&refer=application%3Ddubbo-demo-annotation-consumer%26dubbo%3D2.0.2%26group%3Djianghe-group%26init%3Dfalse%26interface%3Dorg.apache.dubbo.demo.DemoService%26methods%3DsayHello%2CsayHelloAsync%26pid%3D12982%26register.ip%3D127.0.0.1%26revision%3D1.0.0%26side%3Dconsumer%26sticky%3Dfalse%26timestamp%3D1644053460153%26version%3D1.0.0&registry=zookeeper&timestamp=1644053669416
                             */
                            urls.add(u.addParameterAndEncoded(REFER_KEY, StringUtils.toQueryString(map)));
                        }
                    }
                    if (urls.isEmpty()) {
                        throw new IllegalStateException("No such any registry to reference " + interfaceName + " on the consumer " + NetUtils.getLocalHost() + " use dubbo version " + Version.getVersion() + ", please config <dubbo:registry address=\"...\" /> to your spring config.");
                    }
                }
            }
            /**
            * 这里的注册中心如果是一个走的逻辑不一致，如果是多个走的逻辑也不一致。
            **/
            if (urls.size() == 1) {
                /**
                 * 动态目录
                 */
                invoker = REF_PROTOCOL.refer(interfaceClass, urls.get(0));
            } else {
                /**
                 * 静态目录
                 */
                List<Invoker<?>> invokers = new ArrayList<Invoker<?>>();
                URL registryURL = null;
                for (URL url : urls) {
                    invokers.add(REF_PROTOCOL.refer(interfaceClass, url));
                    if (UrlUtils.isRegistry(url)) {
                        registryURL = url; // use last registry url
                    }
                }
                if (registryURL != null) { // registry url is available
                    // for multi-subscription scenario, use 'zone-aware' policy by default
                    URL u = registryURL.addParameterIfAbsent(CLUSTER_KEY, ZoneAwareCluster.NAME);
                    // The invoker wrap relation would be like: ZoneAwareClusterInvoker(StaticDirectory) -> FailoverClusterInvoker(RegistryDirectory, routing happens here) -> Invoker
                    invoker = CLUSTER.join(new StaticDirectory(u, invokers));
                } else { // not a registry url, must be direct invoke.
                    invoker = CLUSTER.join(new StaticDirectory(invokers));
                }
            }
        }

        if (shouldCheck() && !invoker.isAvailable()) {
            invoker.destroy();
            throw new IllegalStateException("Failed to check the status of the service "
                    + interfaceName
                    + ". No provider available for the service "
                    + (group == null ? "" : group + "/")
                    + interfaceName +
                    (version == null ? "" : ":" + version)
                    + " from the url "
                    + invoker.getUrl()
                    + " to the consumer "
                    + NetUtils.getLocalHost() + " use dubbo version " + Version.getVersion());
        }
        if (logger.isInfoEnabled()) {
            logger.info("Refer dubbo service " + interfaceClass.getName() + " from url " + invoker.getUrl());
        }
        /**
         * @since 2.7.0
         * ServiceData Store
         */
        String metadata = map.get(METADATA_KEY);
        WritableMetadataService metadataService = WritableMetadataService.getExtension(metadata == null ? DEFAULT_METADATA_STORAGE_TYPE : metadata);
        if (metadataService != null) {
            URL consumerURL = new URL(CONSUMER_PROTOCOL, map.remove(REGISTER_IP_KEY), 0, map.get(INTERFACE_KEY), map);
            metadataService.publishServiceDefinition(consumerURL);
        }
        /**
         * 创建javassist代理类，实际调用的是InvokerInvocationHandler
         */
        // create service proxy
        return (T) PROXY_FACTORY.getProxy(invoker, ProtocolUtils.isGeneric(generic));
    }
```
上面的逻辑才算真正的到核心里面
1. `List<URL> us = ConfigValidationUtils.loadRegistries(this, false);`生成注册中心的URL
2. <b>根据URL然后生成引用的Invoker，`invoker = REF_PROTOCOL.refer(interfaceClass, urls.get(0))`</b>（这个一定要知道是怎么回事）
3. Invoker生成后，经过`PROXY_FACTORY.getProxy(invoker, ProtocolUtils.isGeneric(generic))`生成代理类
4. <font color='red'><b>生成的代理类对应的Handler就是InvokerInvocationHandler，那这里就是调用的入口</b></font>

```java
public abstract class AbstractProxyFactory implements ProxyFactory {
    private static final Class<?>[] INTERNAL_INTERFACES = new Class<?>[]{
            EchoService.class, Destroyable.class
    };

    @Override
    public <T> T getProxy(Invoker<T> invoker) throws RpcException {
        return getProxy(invoker, false);
    }

    @Override
    public <T> T getProxy(Invoker<T> invoker, boolean generic) throws RpcException {
        Set<Class<?>> interfaces = new HashSet<>();

        String config = invoker.getUrl().getParameter(INTERFACES);
        if (config != null && config.length() > 0) {
            String[] types = COMMA_SPLIT_PATTERN.split(config);
            for (String type : types) {
                // TODO can we load successfully for a different classloader?.
                interfaces.add(ReflectUtils.forName(type));
            }
        }

        if (generic) {
            if (!GenericService.class.isAssignableFrom(invoker.getInterface())) {
                interfaces.add(com.alibaba.dubbo.rpc.service.GenericService.class);
            }

            try {
                // find the real interface from url
                String realInterface = invoker.getUrl().getParameter(Constants.INTERFACE);
                interfaces.add(ReflectUtils.forName(realInterface));
            } catch (Throwable e) {
                // ignore
            }
        }

        interfaces.add(invoker.getInterface());
        interfaces.addAll(Arrays.asList(INTERNAL_INTERFACES));

        return getProxy(invoker, interfaces.toArray(new Class<?>[0]));
    }

    public abstract <T> T getProxy(Invoker<T> invoker, Class<?>[] types);

}

public class JavassistProxyFactory extends AbstractProxyFactory {

    @Override
    @SuppressWarnings("unchecked")
    public <T> T getProxy(Invoker<T> invoker, Class<?>[] interfaces) {
        return (T) Proxy.getProxy(interfaces).newInstance(new InvokerInvocationHandler(invoker));
    }
}
```

## REF_PROTOCOL.refer(interfaceClass, url)
这个方法就是生成Invoker的，具体流程见下面，详细可以看流程图

流程图
![createProxy](/images/dubbo-6-3.png)
<p></p>

1. interfaceClass是要引入服务的接口，url是注册中心的url(registry://)
2. 根据spi底层会调用到RegistryProtocol，但是注意这里是有Wrapper，ProtocolFilterWrapper、ProtocolListenerWrapper这两个Wrapper很熟悉，服务导出的时候使用的，经过两个Wrapper类后，最终调用到RegistryProtocol#refer方法
```java
@Override
    @SuppressWarnings("unchecked")
    public <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException {
        /**
         * 把registry://换成zookeeper://
         */
        url = getRegistryUrl(url);
        /**
         * registryFactory：Registry$Adaptive
         * 这是通过setRegistryFactory方法，通过spi的方式来获取设置的，在哪里获取的时候呢？在ReferenceConfig类中REF_PROTOCOL.refer来获取的
         * 这里也能更加理解SPI的ioc，由于这里设置是根据接口和name来获取对应对象，RegistryFactory的name就是registryFactory，这里是获取不到对象的，但是如果这个接口有实现类，但是没有对应的实现类，就会返回Adaptive类，详细可借鉴类来阅读org.apache.dubbo.remoting.zookeeper.ZookeeperTransporter$Adaptive
         *
         * 那这里获取的Registry调用链
         *  org.apache.dubbo.registry.RegistryFactoryWrapper#getRegistry(org.apache.dubbo.common.URL)
         *      org.apache.dubbo.registry.support.AbstractRegistryFactory#getRegistry(org.apache.dubbo.common.URL)
         *          org.apache.dubbo.registry.support.AbstractRegistryFactory#createRegistry(org.apache.dubbo.common.URL)
         *              org.apache.dubbo.registry.zookeeper.ZookeeperRegistryFactory#createRegistry(org.apache.dubbo.common.URL)
         *                  org.apache.dubbo.registry.zookeeper.ZookeeperRegistry#ZookeeperRegistry(org.apache.dubbo.common.URL, org.apache.dubbo.remoting.zookeeper.ZookeeperTransporter)
         *                      org.apache.dubbo.registry.support.FailbackRegistry#FailbackRegistry(org.apache.dubbo.common.URL)
         *                          org.apache.dubbo.registry.support.AbstractRegistry#AbstractRegistry(org.apache.dubbo.common.URL)
         *
         * Registry其实是：
         *  ListenerRegistryWrapper -> ZookeeperRegistry
         *  <b>
         *      注意，这里的路径这么深其实就是为了构造ZookeeperRegistry，这里其实就是为了创建consumers下的节点，ZookeeperRegistry里面也是包含创建好的zkClient
         *  </b>
         *
         *
         * url：zookeeper://127.0.0.1:2181/org.apache.dubbo.registry.RegistryService?application=dubbo-demo-annotation-consumer&dubbo=2.0.2&pid=15549&refer=application%3Ddubbo-demo-annotation-consumer%26dubbo%3D2.0.2%26group%3Djianghe-group%26init%3Dfalse%26interface%3Dorg.apache.dubbo.demo.DemoService%26methods%3DsayHello%2CsayHelloAsync%26pid%3D15549%26register.ip%3D127.0.0.1%26revision%3D1.0.0%26side%3Dconsumer%26sticky%3Dfalse%26timeout%3D3000%26timestamp%3D1644131247489%26version%3D1.0.0&timestamp=1644131249527
         */
        Registry registry = registryFactory.getRegistry(url);
        if (RegistryService.class.equals(type)) { // false，老版本兼容解决问题
            /**
             * proxyFactory：ProxyFactory$Adaptive，
             * 这是通过setProxyFactory方法，spi的方式来设置的
             */
            return proxyFactory.getInvoker((T) registry, type, url);
        }

        /**
         * "init" -> "false"
         * "side" -> "consumer"
         * "register.ip" -> "127.0.0.1"
         * "methods" -> "sayHello,sayHelloAsync"
         * "dubbo" -> "2.0.2"
         * "pid" -> "15300"
         * "interface" -> "org.apache.dubbo.demo.DemoService"
         * "version" -> "1.0.0"
         * "timeout" -> "3000"
         * "revision" -> "1.0.0"
         * "application" -> "dubbo-demo-annotation-consumer"
         * "sticky" -> "false"
         * "group" -> "jianghe-group"
         * "timestamp" -> "1644129114622"
         */
        // group="a,b" or group="*"
        Map<String, String> qs = StringUtils.parseQueryString(url.getParameterAndDecoded(REFER_KEY));
        /**
         * jianghe-group
         */
        String group = qs.get(GROUP_KEY);
        if (group != null && group.length() > 0) {
            if ((COMMA_SPLIT_PATTERN.split(group)).length > 1 || "*".equals(group)) { // 多group
                return doRefer(getMergeableCluster(), registry, type, url);
            }
        }
        return doRefer(cluster, registry, type, url);
    }
```
3. `Registry registry = registryFactory.getRegistry(url)`工厂模式，底层最终构建ZooKeeperRegistry，底层肯定操作zookeeper
4. 然后调用RegistryProtocol.doRefer 真正引入
```java
/**
     * 真正的引用
     * @param cluster
     * @param registry
     * @param type
     * @param url
     * @param <T>
     * @return
     */
    private <T> Invoker<T> doRefer(Cluster cluster, Registry registry, Class<T> type, URL url) {
        /**
         * 构造服务目录
         *
         * zookeeper://127.0.0.1:2181/org.apache.dubbo.registry.RegistryService?application=dubbo-demo-annotation-consumer&dubbo=2.0.2&pid=15300&refer=application%3Ddubbo-demo-annotation-consumer%26dubbo%3D2.0.2%26group%3Djianghe-group%26init%3Dfalse%26interface%3Dorg.apache.dubbo.demo.DemoService%26methods%3DsayHello%2CsayHelloAsync%26pid%3D15300%26register.ip%3D127.0.0.1%26revision%3D1.0.0%26side%3Dconsumer%26sticky%3Dfalse%26timeout%3D3000%26timestamp%3D1644129114622%26version%3D1.0.0&timestamp=1644129134554
         */
        RegistryDirectory<T> directory = new RegistryDirectory<T>(type, url); // 构造服务目录
        directory.setRegistry(registry); // ListenerRegistryWrapper -> ZookeeperRegistry
        directory.setProtocol(protocol); // Protocol$Adaptive
        // all attributes of REFER_KEY
        Map<String, String> parameters = new HashMap<String, String>(directory.getConsumerUrl().getParameters());
        /**
         * subscribeUrl
         *  consumer://127.0.0.1/org.apache.dubbo.demo.DemoService?application=dubbo-demo-annotation-consumer&dubbo=2.0.2&group=jianghe-group&init=false&interface=org.apache.dubbo.demo.DemoService&methods=sayHello,sayHelloAsync&pid=15812&revision=1.0.0&side=consumer&sticky=false&timeout=3000&timestamp=1644134813635&version=1.0.0
         */
        URL subscribeUrl = new URL(CONSUMER_PROTOCOL, parameters.remove(REGISTER_IP_KEY), 0, type.getName(), parameters);
        /**
         * 这里判断的逻辑还是register是否为true && 接口不是*
         */
        if (directory.isShouldRegister()) { // true
            /**
             * 简化URL
             */
            directory.setRegisteredConsumerUrl(subscribeUrl);
            /**
             * zookeeper 注册服务
             *  ListenerRegistryWrapper#register -> FailbackRegistry#register(ZookeeperRegistry的) -> ZookeeperRegistry#doRegister(org.apache.dubbo.common.URL)
             */
            registry.register(directory.getRegisteredConsumerUrl());
        }
        /**
         * 构建路由链，路由链是调用的时候起作用的，这里就是调用的时候M->N
         * 新版本标签路由目录：/dubbo/config/dubbo/dubbo-demo-annotation-provider.tag-router
         * 新版本条件路由目录：/dubbo/config/dubbo/org.apache.dubbo.demo.DemoService:1.0.0:jianghe-group.condition-router
         */
        directory.buildRouterChain(subscribeUrl); // 路由链
        /**
         * 订阅（最终会调用到）
         *
         * toSubscribeUrl(subscribeUrl)增加了参数category=providers,configurators,routers
         *
         * consumer://127.0.0.1/org.apache.dubbo.demo.DemoService?application=dubbo-demo-annotation-consumer&category=providers,configurators,routers&dubbo=2.0.2&group=jianghe-group&init=false&interface=org.apache.dubbo.demo.DemoService&methods=sayHello,sayHelloAsync&pid=15812&revision=1.0.0&side=consumer&sticky=false&timeout=3000&timestamp=1644134813635&version=1.0.0
         *
         * 当前应用对应的动态配置目录：/dubbo/config/dubbo/dubbo-demo-annotation-consumer.configurators
         * 当前服务对应的动态配置目录：/dubbo/config/dubbo/org.apache.dubbo.demo.DemoService:1.0.0:jianghe-group.configurators
         * 当前引入服务提供者的目录：/dubbo/org.apache.dubbo.demo.DemoService/providers
         * 当前引入服务动态配置目录（兼容老版本，这里只有针对服务的，没有应用的）：/dubbo/org.apache.dubbo.demo.DemoService/configurators
         * 当前引入服务路由目录（兼容老版本，这里只有针对服务的，没有应用的）：/dubbo/org.apache.dubbo.demo.DemoService/routers
         */
        directory.subscribe(toSubscribeUrl(subscribeUrl)); // 订阅

        /**
         * 构建Invoker链
         *
         * 将RegistryDirectory作为属性，RegistryDirectory里面有包含List<DubboInvoker>，最终返回一个Invoker
         * MockClusterInvoker ->
         *  AbstractCluster$Interceptor
         *      FailoverClusterInvoker ->
         *          RegistryDirectory ->
         *              List<RegistryDirectory$InvokerDelegate>
         */
        Invoker<T> invoker = cluster.join(directory);
        List<RegistryProtocolListener> listeners = findRegistryProtocolListeners(url);
        if (CollectionUtils.isEmpty(listeners)) { // 空的
            return invoker;
        }

        RegistryInvokerWrapper<T> registryInvokerWrapper = new RegistryInvokerWrapper<>(directory, cluster, invoker, subscribeUrl);
        for (RegistryProtocolListener listener : listeners) {
            listener.onRefer(this, registryInvokerWrapper);
        }
        return registryInvokerWrapper;
    }
```
5. 这里构建了一个服务目录RegistryDirectory（路由，负载，动态配置）
6. `registry.register(directory.getRegisteredConsumerUrl())`这个就很清晰，这个就是创建consumers消费者
7. `directory.buildRouterChain(subscribeUrl)`建立路由链
8. `directory.subscribe(toSubscribeUrl(subscribeUrl))`订阅服务，这里其实就是动态配置（<b>完成监听器的订阅后，会自动触发一次去获取动态配置</b>）
9. <font color='red'><b>然后调用cluster.join(directory)根据服务目录构建Invoker，通过SPI方式返回Invoker</b></font>
```text
MockClusterInvoker ->
    AbstractCluster$Interceptor
        FailoverClusterInvoker ->
            RegistryDirectory ->
                List<RegistryDirectory$InvokerDelegate>
```


### RegistryDirectory.buildRouterChain
buildRouterChain最终调用构建RouterChain(URL url)，这里的url就是：consumer://127.0.0.1/org.apache.dubbo.demo.DemoService?application=dubbo-demo-annotation-consumer&category=providers,configurators,routers&dubbo=2.0.2&group=jianghe-group&init=false&interface=org.apache.dubbo.demo.DemoService&methods=sayHello,sayHelloAsync&pid=15812&revision=1.0.0&side=consumer&sticky=false&timeout=3000&timestamp=1644134813635&version=1.0.0

```java
private RouterChain(URL url) {
    /**
        * 0 = {MockRouterFactory@4528}
        * 1 = {TagRouterFactory@4529} 标签路由
        * 2 = {AppRouterFactory@4530}  应用条件路由
        * 3 = {ServiceRouterFactory@4531} 服务条件路由
        */
    List<RouterFactory> extensionFactories = ExtensionLoader.getExtensionLoader(RouterFactory.class)
            .getActivateExtension(url, "router");
    /**
        * 工厂模式，RouterFactory根据url生成各个Router
        * MockInvokersSelector 这个可以再看下，不是很清晰
        * 注意，这里生成的Router是真是可用的
        *  1. 其中AppRouter和ServiceRouter已经是完全可以用的，里面也有从配置中心读取的内容，监听
        *  2. TagRouter还不可用，为什么？标签路由是根据提供者名称来配置，目前是无法获取到提供者的名称的，哪里能获取，/providers下面是可以获取到的
        *      那TagRouter什么时候起作用？RegistryDirectory#refreshInvoker方法里面
        *
        * 0 = {MockInvokersSelector@3917}
        * 1 = {TagRouter@3918}
        * 2 = {AppRouter@3919}
        * 3 = {ServiceRouter@3920}
        */
    List<Router> routers = extensionFactories.stream()
            .map(factory -> factory.getRouter(url))
            .collect(Collectors.toList());

    /**
        * 排序
        */
    initWithRouters(routers);
}
```
RouterChain是泛型，泛型是URL，每个URL榜单一个路由链，其中URL直接跟routers属性绑定 

builtinRouters：
 1. {MockInvokersSelector@3917} ，Mock路由
 2. {TagRouter@3918} ，标签路由
 3. {AppRouter@3919} ，应用条件路由
 4. {ServiceRouter@3920} ，服务条件路由

routers（有顺序的，根据优先级排序）：
 1. MockInvokersSelector，优先级是 = Integer.MIN_VALUE 
 2. TagRouter，优先级 = 100
 3. ServiceRouter（这是新版本监听服务路由），优先级 = 140 
    监听的路径：/dubbo/config/dubbo/org.apache.dubbo.demo.DemoService:1.0.0:jianghe-group.condition-router
 4. AppRouter（这是新版本监听应用路由），优先级 = 150  
    监听的路径：/dubbo/config/dubbo/dubbo-demo-annotation-consumer.condition-router



<font color='red'><b>注意，这里生成的Router是真是可用的</b></font>
<b>
1. MockInvokersSelector 这个可以再看下，不是很清晰
2. 其中AppRouter和ServiceRouter已经是完全可以用的，里面也有从配置中心读取的内容，监听
3. TagRouter还不可用，为什么？标签路由是根据提供者名称来配置，目前是无法获取到提供者的名称的，哪里能获取，提供者/providers下面URL是可以获取到的
4. 那TagRouter什么时候起作用？RegistryDirectory#refreshInvoker方法里面
</b>

### RegistryDirectory.subscribe
订阅这块比较复杂，单列出来说，有新老版本，这里的url就是consumer://127.0.0.1/org.apache.dubbo.demo.DemoService?application=dubbo-demo-annotation-consumer&category=providers,configurators,routers&dubbo=2.0.2&group=jianghe-group&init=false&interface=org.apache.dubbo.demo.DemoService&methods=sayHello,sayHelloAsync&pid=13273&revision=1.0.0&side=consumer&sticky=false&timestamp=1644054153748&version=1.0.0
流程图
![服务目录订阅](/images/dubbo-6-3-1.png)

```java
public void subscribe(URL url) {
    /**
        * consumer://127.0.0.1/org.apache.dubbo.demo.DemoService?application=dubbo-demo-annotation-consumer&category=providers,configurators,routers&dubbo=2.0.2&group=jianghe-group&init=false&interface=org.apache.dubbo.demo.DemoService&methods=sayHello,sayHelloAsync&pid=13273&revision=1.0.0&side=consumer&sticky=false&timestamp=1644054153748&version=1.0.0
        */
    setConsumerUrl(url);
    /**
        * 新版本的应用配置
        */
    CONSUMER_CONFIGURATION_LISTENER.addNotifyListener(this);
    /**
        * 新版本的服务配置监听
        */
    serviceConfigurationListener = new ReferenceConfigurationListener(this, url);
    /**
        * 老版本的服务配置监听
        */
    registry.subscribe(url, this);
}
```
1. ConsumerConfigurationListener这个是新版本的应用动态配置监听
    监听路径：/dubbo/config/dubbo/dubbo-demo-consumer-application.configurators
2. ReferenceConfigurationListener这个是新版本的服务动态配置监听
    监听路径：/dubbo/config/dubbo/org.apache.dubbo.demo.DemoService:1.0.0:jianghe-group.configurators
3. `registry.subscribe(url, this);`这个是老版本的动态配置监听，这里会包含下面三个目录，其中`/dubbo/org.apache.dubbo.demo.DemoService/providers`这不算是老的配置，这里为什么会放在一起，只是目录前缀一致
    1. 当前引入服务提供者的目录：/dubbo/org.apache.dubbo.demo.DemoService/providers
    2. 当前引入服务动态配置目录（兼容老版本，这里只有针对服务的，没有应用的）：/dubbo/org.apache.dubbo.demo.DemoService/configurators
    3. 当前引入服务路由目录（兼容老版本，这里只有针对服务的，没有应用的）：/dubbo/org.apache.dubbo.demo.DemoService/routers
4. 然后最终调用ZookeeperRegistry.doSubscribe
```java
/**
     * @param url
     * @param listener 这里的监听器就是RegistryDirectory
     */
    @Override
    public void doSubscribe(final URL url, final NotifyListener listener) {
        try {
            /**
             * 判断接口是否为*
             */
            if (ANY_VALUE.equals(url.getServiceInterface())) { // false
                String root = toRootPath();
                ConcurrentMap<NotifyListener, ChildListener> listeners = zkListeners.computeIfAbsent(url, k -> new ConcurrentHashMap<>());
                ChildListener zkListener = listeners.computeIfAbsent(listener, k -> (parentPath, currentChilds) -> {
                    for (String child : currentChilds) {
                        child = URL.decode(child);
                        if (!anyServices.contains(child)) {
                            anyServices.add(child);
                            subscribe(url.setPath(child).addParameters(INTERFACE_KEY, child,
                                    Constants.CHECK_KEY, String.valueOf(false)), k);
                        }
                    }
                });
                zkClient.create(root, false);
                List<String> services = zkClient.addChildListener(root, zkListener);
                if (CollectionUtils.isNotEmpty(services)) {
                    for (String service : services) {
                        service = URL.decode(service);
                        anyServices.add(service);
                        subscribe(url.setPath(service).addParameters(INTERFACE_KEY, service,
                                Constants.CHECK_KEY, String.valueOf(false)), listener);
                    }
                }
            } else {
                List<URL> urls = new ArrayList<>();
                /**
                 * 0 = "/dubbo/org.apache.dubbo.demo.DemoService/providers"
                 * 1 = "/dubbo/org.apache.dubbo.demo.DemoService/configurators"
                 * 2 = "/dubbo/org.apache.dubbo.demo.DemoService/routers"
                 */
                for (String path : toCategoriesPath(url)) { //监听服务
                    ConcurrentMap<NotifyListener, ChildListener> listeners = zkListeners.computeIfAbsent(url, k -> new ConcurrentHashMap<>());
                    ChildListener zkListener = listeners.computeIfAbsent(listener, k -> (parentPath, currentChilds) -> ZookeeperRegistry.this.notify(url, k, toUrlsWithEmpty(url, parentPath, currentChilds)));
                    zkClient.create(path, false); // 持久化节点
                    /**
                     * 这里获取的就是子目录的路径值
                     * path：/dubbo/org.apache.dubbo.demo.DemoService/providers
                     * children(这里是子目录，不是节点内容)：
                     *  1. dubbo%3A%2F%2F172.16.116.200%3A21881%2Forg.apache.dubbo.demo.DemoService%3Fanyhost%3Dtrue%26application%3Ddubbo-demo-annotation-provider%26deprecated%3Dfalse%26dubbo%3D2.0.2%26dynamic%3Dtrue%26generic%3Dfalse%26group%3Djianghe-group%26interface%3Dorg.apache.dubbo.demo.DemoService%26loadbalance%3Dleastactive%26methods%3DsayHello%2CsayHelloAsync%26pid%3D2456%26release%3D%26revision%3D1.0.0%26side%3Dprovider%26timeout%3D1000%26timestamp%3D1644460848263%26version%3D1.0.0
                     *  2. dubbo%3A%2F%2F172.16.116.200%3A21880%2Forg.apache.dubbo.demo.DemoService%3Fanyhost%3Dtrue%26application%3Ddubbo-demo-annotation-provider%26deprecated%3Dfalse%26dubbo%3D2.0.2%26dynamic%3Dtrue%26generic%3Dfalse%26group%3Djianghe-group%26interface%3Dorg.apache.dubbo.demo.DemoService%26loadbalance%3Dleastactive%26methods%3DsayHello%2CsayHelloAsync%26pid%3D2456%26release%3D%26revision%3D1.0.0%26side%3Dprovider%26timeout%3D1000%26timestamp%3D1644460847650%26version%3D1.0.0
                     */
                    List<String> children = zkClient.addChildListener(path, zkListener); // 监听
                    if (children != null) {
                        urls.addAll(toUrlsWithEmpty(url, path, children));
                    }
                }
                /**
                 * 主动去监听，主动去触发
                 *
                 * url:
                 *  consumer://127.0.0.1/org.apache.dubbo.demo.DemoService?application=dubbo-demo-annotation-consumer&category=providers,configurators,routers&dubbo=2.0.2&group=jianghe-group&init=false&interface=org.apache.dubbo.demo.DemoService&methods=sayHello,sayHelloAsync&pid=13534&revision=1.0.0&side=consumer&sticky=false&timestamp=1644055748778&version=1.0.0
                 *
                 * urls:
                 *  0 = {URL@4912} "dubbo://127.0.0.1:21880/org.apache.dubbo.demo.DemoService?anyhost=true&application=dubbo-demo-annotation-provider&deprecated=false&dubbo=2.0.2&dynamic=true&generic=false&group=jianghe-group&interface=org.apache.dubbo.demo.DemoService&loadbalance=leastactive&methods=sayHello,sayHelloAsync&pid=12965&release=&revision=1.0.0&side=provider&timeout=5000&timestamp=1644052494096&version=1.0.0"
                 * 1 = {URL@4913} "dubbo://127.0.0.1:21881/org.apache.dubbo.demo.DemoService?anyhost=true&application=dubbo-demo-annotation-provider&deprecated=false&dubbo=2.0.2&dynamic=true&generic=false&group=jianghe-group&interface=org.apache.dubbo.demo.DemoService&loadbalance=leastactive&methods=sayHello,sayHelloAsync&pid=12965&release=&revision=1.0.0&side=provider&timeout=1000&timestamp=1644052494777&version=1.0.0"
                 * 2 = {URL@4917} "empty://127.0.0.1/org.apache.dubbo.demo.DemoService?application=dubbo-demo-annotation-consumer&category=configurators&dubbo=2.0.2&group=jianghe-group&init=false&interface=org.apache.dubbo.demo.DemoService&methods=sayHello,sayHelloAsync&pid=13534&revision=1.0.0&side=consumer&sticky=false&timestamp=1644055748778&version=1.0.0"
                 * 3 = {URL@4918} "empty://127.0.0.1/org.apache.dubbo.demo.DemoService?application=dubbo-demo-annotation-consumer&category=routers&dubbo=2.0.2&group=jianghe-group&init=false&interface=org.apache.dubbo.demo.DemoService&methods=sayHello,sayHelloAsync&pid=13534&revision=1.0.0&side=consumer&sticky=false&timestamp=1644055748778&version=1.0.0"
                 */
                notify(url, listener, urls);
            }
        } catch (Throwable e) {
            throw new RpcException("Failed to subscribe " + url + " to zookeeper " + getUrl() + ", cause: " + e.getMessage(), e);
        }
    }
```
5. 根据三个类目路径，获取到信息，主动调用notify(url, listener, urls);
    1. /dubbo/org.apache.dubbo.demo.DemoService/providers
    2. /dubbo/org.apache.dubbo.demo.DemoService/configurators
    3. /dubbo/org.apache.dubbo.demo.DemoService/routers
6. 然后调用到AbstractRegistry#notify，这里的listener是RegistryDirectory，最终会调用到RegistryDirectory#notify方法，其中categoryList就是节点内容，例如/providers那就是该接口的提供者列表
```java
/**
     * Notify changes from the Provider side.
     *
     * @param url      consumer side url
     * @param listener listener
     * @param urls     provider latest urls
     */
    protected void notify(URL url, NotifyListener listener, List<URL> urls) {
        if (url == null) {
            throw new IllegalArgumentException("notify url == null");
        }
        if (listener == null) {
            throw new IllegalArgumentException("notify listener == null");
        }
        if ((CollectionUtils.isEmpty(urls))
                && !ANY_VALUE.equals(url.getServiceInterface())) {
            logger.warn("Ignore empty notify urls for subscribe url " + url);
            return;
        }
        if (logger.isInfoEnabled()) {
            logger.info("Notify urls for subscribe url " + url + ", urls: " + urls);
        }
        // keep every provider's category.
        Map<String, List<URL>> result = new HashMap<>();
        for (URL u : urls) {
            if (UrlUtils.isMatch(url, u)) {
                String category = u.getParameter(CATEGORY_KEY, DEFAULT_CATEGORY);
                List<URL> categoryList = result.computeIfAbsent(category, k -> new ArrayList<>());
                categoryList.add(u);
            }
        }
        if (result.size() == 0) {
            return;
        }
        Map<String, List<URL>> categoryNotified = notified.computeIfAbsent(url, u -> new ConcurrentHashMap<>());
        /**
         * 其中的result有三个值
         * key = routers
         * value = empty://127.0.0.1/org.apache.dubbo.demo.DemoService?application=dubbo-demo-annotation-consumer&category=routers&dubbo=2.0.2&group=jianghe-group&init=false&interface=org.apache.dubbo.demo.DemoService&methods=sayHello,sayHelloAsync&pid=18535&revision=1.0.0&side=consumer&sticky=false&timeout=3000&timestamp=1644215337200&version=1.0.0
         *
         * key = configurators
         * value = empty://127.0.0.1/org.apache.dubbo.demo.DemoService?application=dubbo-demo-annotation-consumer&category=configurators&dubbo=2.0.2&group=jianghe-group&init=false&interface=org.apache.dubbo.demo.DemoService&methods=sayHello,sayHelloAsync&pid=18535&revision=1.0.0&side=consumer&sticky=false&timeout=3000&timestamp=1644215337200&version=1.0.0
         *
         * key = providers
         * value =
         *  dubbo://127.0.0.1:21881/org.apache.dubbo.demo.DemoService?anyhost=true&application=dubbo-demo-annotation-provider&deprecated=false&dubbo=2.0.2&dynamic=true&generic=false&group=jianghe-group&interface=org.apache.dubbo.demo.DemoService&loadbalance=leastactive&methods=sayHello,sayHelloAsync&pid=17908&release=&revision=1.0.0&side=provider&timeout=1000&timestamp=1644205565914&version=1.0.0
         *  dubbo://127.0.0.1:21880/org.apache.dubbo.demo.DemoService?anyhost=true&application=dubbo-demo-annotation-provider&deprecated=false&dubbo=2.0.2&dynamic=true&generic=false&group=jianghe-group&interface=org.apache.dubbo.demo.DemoService&loadbalance=leastactive&methods=sayHello,sayHelloAsync&pid=17908&release=&revision=1.0.0&side=provider&timeout=5000&timestamp=1644205564950&version=1.0.0
         */
        for (Map.Entry<String, List<URL>> entry : result.entrySet()) {
            String category = entry.getKey();
            List<URL> categoryList = entry.getValue();
            categoryNotified.put(category, categoryList);
            /**
             * 这里就是调用RegistryDirectory.notify接口，真正监听器的notify方法
             */
            listener.notify(categoryList);
            // We will update our cache file after each notification.
            // When our Registry has a subscribe failure due to network jitter, we can return at least the existing cache URL.
            saveProperties(url); // 持久化
        }
    }
```
7. 这个就是RegistryDirectory#notify，
    1. 如果获取到/configurators内容，就会放到configurators配置里，获取内容，监听，然后会调用refreshOverrideAndInvoker里的overrideDirectoryUrl方法重写URL
    2. 如果获取到/routers内容，就会将入到RouterChain里面，调用的时候使用路由链
    3. 如果获取到/providers，就会调用refreshOverrideAndInvoker里的refreshInvoker获取Invoker
```java
/**
     * 这里是真正唤醒的方法
     * @param urls The list of registered information , is always not empty. The meaning is the same as the return value of {@link org.apache.dubbo.registry.RegistryService#lookup(URL)}.
     */
    @Override
    public synchronized void notify(List<URL> urls) {
        Map<String, List<URL>> categoryUrls = urls.stream()
                .filter(Objects::nonNull)
                .filter(this::isValidCategory)
                .filter(this::isNotCompatibleFor26x)
                .collect(Collectors.groupingBy(this::judgeCategory));

        /**
         * 老版本的动态配置
         */
        List<URL> configuratorURLs = categoryUrls.getOrDefault(CONFIGURATORS_CATEGORY, Collections.emptyList());
        this.configurators = Configurator.toConfigurators(configuratorURLs).orElse(this.configurators);

        /**
         * 老版本路由的动态配置
         */
        List<URL> routerURLs = categoryUrls.getOrDefault(ROUTERS_CATEGORY, Collections.emptyList());
        toRouters(routerURLs).ifPresent(this::addRouters);

        /**
         * 这里就是查询提供者的信息
         * 这里的提供者为什么在这里？其实是跟目录有关系，/dubbo/org.apache.dubbo.demo.DemoService/providers
         * 跟上面的路径只有后面/providers 不一样，其他都是这样，所以提供者就在这里
         */
        // providers
        List<URL> providerURLs = categoryUrls.getOrDefault(PROVIDERS_CATEGORY, Collections.emptyList());
        /**
         * 3.x added for extend URL address
         */
        ExtensionLoader<AddressListener> addressListenerExtensionLoader = ExtensionLoader.getExtensionLoader(AddressListener.class);
        List<AddressListener> supportedListeners = addressListenerExtensionLoader.getActivateExtension(getUrl(), (String[]) null);
        if (supportedListeners != null && !supportedListeners.isEmpty()) { // 扩展类，扩展URL
            for (AddressListener addressListener : supportedListeners) {
                providerURLs = addressListener.notify(providerURLs, getConsumerUrl(),this);
            }
        }
        /**
         * 只有categoryUrls中的key是providers，下面的providerURLs才会有值
         * 0 = {URL@4147} "dubbo://127.0.0.1:21881/org.apache.dubbo.demo.DemoService?anyhost=true&application=dubbo-demo-annotation-provider&deprecated=false&dubbo=2.0.2&dynamic=true&generic=false&group=jianghe-group&interface=org.apache.dubbo.demo.DemoService&loadbalance=leastactive&methods=sayHello,sayHelloAsync&pid=17908&release=&revision=1.0.0&side=provider&timeout=1000&timestamp=1644205565914&version=1.0.0"
         * 1 = {URL@4148} "dubbo://127.0.0.1:21880/org.apache.dubbo.demo.DemoService?anyhost=true&application=dubbo-demo-annotation-provider&deprecated=false&dubbo=2.0.2&dynamic=true&generic=false&group=jianghe-group&interface=org.apache.dubbo.demo.DemoService&loadbalance=leastactive&methods=sayHello,sayHelloAsync&pid=17908&release=&revision=1.0.0&side=provider&timeout=5000&timestamp=1644205564950&version=1.0.0"
         */
        refreshOverrideAndInvoker(providerURLs);
    }
```
8. 配置更新这块不细说了，有时间可以再看，我们详细看下refreshInvoker(urls)，这个的urls就是
```text
0 = {URL@4147} "dubbo://127.0.0.1:21881/org.apache.dubbo.demo.DemoService?anyhost=true&application=dubbo-demo-annotation-provider&deprecated=false&dubbo=2.0.2&dynamic=true&generic=false&group=jianghe-group&interface=org.apache.dubbo.demo.DemoService&loadbalance=leastactive&methods=sayHello,sayHelloAsync&pid=17908&release=&revision=1.0.0&side=provider&timeout=1000&timestamp=1644205565914&version=1.0.0"
1 = {URL@4148} "dubbo://127.0.0.1:21880/org.apache.dubbo.demo.DemoService?anyhost=true&application=dubbo-demo-annotation-provider&deprecated=false&dubbo=2.0.2&dynamic=true&generic=false&group=jianghe-group&interface=org.apache.dubbo.demo.DemoService&loadbalance=leastactive&methods=sayHello,sayHelloAsync&pid=17908&release=&revision=1.0.0&side=provider&timeout=5000&timestamp=1644205564950&version=1.0.0"
```
```java
/**
     * Convert the invokerURL list to the Invoker Map. The rules of the conversion are as follows:
     * <ol>
     * <li> If URL has been converted to invoker, it is no longer re-referenced and obtained directly from the cache,
     * and notice that any parameter changes in the URL will be re-referenced.</li>
     * <li>If the incoming invoker list is not empty, it means that it is the latest invoker list.</li>
     * <li>If the list of incoming invokerUrl is empty, It means that the rule is only a override rule or a route
     * rule, which needs to be re-contrasted to decide whether to re-reference.</li>
     * </ol>
     *
     * @param invokerUrls this parameter can't be null
     */
    // TODO: 2017/8/31 FIXME The thread pool should be used to refresh the address, otherwise the task may be accumulated.
    private void refreshInvoker(List<URL> invokerUrls) {
        Assert.notNull(invokerUrls, "invokerUrls should not be null");

        if (invokerUrls.size() == 1
                && invokerUrls.get(0) != null
                && EMPTY_PROTOCOL.equals(invokerUrls.get(0).getProtocol())) {
            this.forbidden = true; // Forbid to access
            this.invokers = Collections.emptyList();
            routerChain.setInvokers(this.invokers);
            destroyAllInvokers(); // Close all invokers
        } else { // 这里例子是2个，就走复杂逻辑
            this.forbidden = false; // Allow to access
            Map<String, Invoker<T>> oldUrlInvokerMap = this.urlInvokerMap; // local reference
            if (invokerUrls == Collections.<URL>emptyList()) {
                invokerUrls = new ArrayList<>();
            }
            if (invokerUrls.isEmpty() && this.cachedInvokerUrls != null) {
                invokerUrls.addAll(this.cachedInvokerUrls);
            } else {
                this.cachedInvokerUrls = new HashSet<>();
                this.cachedInvokerUrls.addAll(invokerUrls);//Cached invoker urls, convenient for comparison
            }
            if (invokerUrls.isEmpty()) {
                return;
            }
            /**
             * 根据提供者的url来转换Invoker
             * 从这里构造的Invoker里面是包含Filter链的，主要是在protocol.refer(serviceType, url)里面构建的
             *
             */
            Map<String, Invoker<T>> newUrlInvokerMap = toInvokers(invokerUrls);// Translate url list to Invoker map

            /**
             * If the calculation is wrong, it is not processed.
             *
             * 1. The protocol configured by the client is inconsistent with the protocol of the server.
             *    eg: consumer protocol = dubbo, provider only has other protocol services(rest).
             * 2. The registration center is not robust and pushes illegal specification data.
             *
             */
            if (CollectionUtils.isEmptyMap(newUrlInvokerMap)) {
                logger.error(new IllegalStateException("urls to invokers error .invokerUrls.size :" + invokerUrls.size() + ", invoker.size :0. urls :" + invokerUrls
                        .toString()));
                return;
            }

            List<Invoker<T>> newInvokers = Collections.unmodifiableList(new ArrayList<>(newUrlInvokerMap.values()));

            // pre-route and build cache, notice that route cache should build on original Invoker list.
            // toMergeMethodInvokerMap() will wrap some invokers having different groups, those wrapped invokers not should be routed.
            routerChain.setInvokers(newInvokers); // 这里是TagRoute调用
            this.invokers = multiGroup ? toMergeInvokerList(newInvokers) : newInvokers;
            this.urlInvokerMap = newUrlInvokerMap;

            try {
                destroyUnusedInvokers(oldUrlInvokerMap, newUrlInvokerMap); // Close the unused Invoker
            } catch (Exception e) {
                logger.warn("destroyUnusedInvokers error. ", e);
            }
        }
    }
```
9. <font color='red'><b>然后调用toInvokers(invokerUrls)根据URL转换成Invoker，得到结果List<Invoker>放入到RegistryDirectory里面</b></font>
```java
/**
     * Turn urls into invokers, and if url has been refer, will not re-reference.
     *
     * @param urls
     * @return invokers
     */
    private Map<String, Invoker<T>> toInvokers(List<URL> urls) {
        Map<String, Invoker<T>> newUrlInvokerMap = new HashMap<>();
        if (urls == null || urls.isEmpty()) {
            return newUrlInvokerMap;
        }
        Set<String> keys = new HashSet<>();
        String queryProtocols = this.queryMap.get(PROTOCOL_KEY); // ReferenceBean里面有配置protocol
        // 遍历当前所有的提供者服务
        for (URL providerUrl : urls) {
            // If protocol is configured at the reference side, only the matching protocol is selected
            if (queryProtocols != null && queryProtocols.length() > 0) {
                boolean accept = false;
                String[] acceptProtocols = queryProtocols.split(",");
                for (String acceptProtocol : acceptProtocols) {
                    if (providerUrl.getProtocol().equals(acceptProtocol)) {
                        accept = true;
                        break;
                    }
                }
                if (!accept) {
                    continue;
                }
            }
            if (EMPTY_PROTOCOL.equals(providerUrl.getProtocol())) {
                continue;
            }
            if (!ExtensionLoader.getExtensionLoader(Protocol.class).hasExtension(providerUrl.getProtocol())) { // 扩展
                logger.error(new IllegalStateException("Unsupported protocol " + providerUrl.getProtocol() +
                        " in notified url: " + providerUrl + " from registry " + getUrl().getAddress() +
                        " to consumer " + NetUtils.getLocalHost() + ", supported protocol: " +
                        ExtensionLoader.getExtensionLoader(Protocol.class).getSupportedExtensions()));
                continue;
            }
            /**
             * 根据当前提供者的url来获取最终的url
             */
            URL url = mergeUrl(providerUrl);

            String key = url.toFullString(); // The parameter urls are sorted
            if (keys.contains(key)) { // Repeated url
                continue;
            }
            keys.add(key);
            // Cache key is url that does not merge with consumer side parameters, regardless of how the consumer combines parameters, if the server url changes, then refer again
            Map<String, Invoker<T>> localUrlInvokerMap = this.urlInvokerMap; // local reference
            Invoker<T> invoker = localUrlInvokerMap == null ? null : localUrlInvokerMap.get(key);
            if (invoker == null) { // Not in the cache, refer again
                try {
                    boolean enabled = true;
                    if (url.hasParameter(DISABLED_KEY)) {
                        enabled = !url.getParameter(DISABLED_KEY, false);
                    } else {
                        enabled = url.getParameter(ENABLED_KEY, true);
                    }
                    if (enabled) {
                        /**
                         * 这里就会调用DubboProtocol.refer，里面当然肯定会有Wrapper类，非常重要
                         * RegistryDirectory$InvokerDelegate ->
                         *  ListenerInvokerWrapper ->
                         *      Invoker$0(ConsumerContextFilter)
                         *      Invoker$1(FutureFilter)
                         *      Invoker$2(MonitorFilter) ->
                         *                               AsyncToSyncInvoker ->
                         *                                  DubboInvoker
                         *
                         */
                        invoker = new InvokerDelegate<>(protocol.refer(serviceType, url), url, providerUrl);
                    }
                } catch (Throwable t) {
                    logger.error("Failed to refer invoker for interface:" + serviceType + ",url:(" + url + ")" + t.getMessage(), t);
                }
                if (invoker != null) { // Put new invoker in cache
                    newUrlInvokerMap.put(key, invoker);
                }
            } else {
                newUrlInvokerMap.put(key, invoker);
            }
        }
        keys.clear();
        return newUrlInvokerMap;
    }
```
10. 调用mergeUrl(providerUrl)来获取最终的URL，这里providerUrl是提供者url
```java
/**
     * Merge url parameters. the order is: override > -D >Consumer > Provider
     *
     * @param providerUrl
     * @return
     */
    private URL mergeUrl(URL providerUrl) {
        /**
         * ReferenceBean里面的属性
         */
        providerUrl = ClusterUtils.mergeUrl(providerUrl, queryMap); // Merge the consumer side parameters
        /**
         * 动态配置
         */
        providerUrl = overrideWithConfigurator(providerUrl);
        // check=false
        providerUrl = providerUrl.addParameter(Constants.CHECK_KEY, String.valueOf(false)); // Do not check whether the connection is successful or not, always create Invoker!

        // The combination of directoryUrl and override is at the end of notify, which can't be handled here
        this.overrideDirectoryUrl = this.overrideDirectoryUrl.addParametersIfAbsent(providerUrl.getParameters()); // Merge the provider side parameters

        if ((providerUrl.getPath() == null || providerUrl.getPath()
                .length() == 0) && DUBBO_PROTOCOL.equals(providerUrl.getProtocol())) { // Compatible version 1.0
            //fix by tony.chenl DUBBO-44
            String path = directoryUrl.getParameter(INTERFACE_KEY);
            if (path != null) {
                int i = path.indexOf('/');
                if (i >= 0) {
                    path = path.substring(i + 1);
                }
                i = path.lastIndexOf(':');
                if (i >= 0) {
                    path = path.substring(0, i);
                }
                providerUrl = providerUrl.setPath(path);
            }
        }
        return providerUrl;
    }
```
```java
private URL overrideWithConfigurator(URL providerUrl) {
        /**
         * 老版本动态配置
         */
        // override url with configurator from "override://" URL for dubbo 2.6 and before
        providerUrl = overrideWithConfigurators(this.configurators, providerUrl);

        /**
         * 新版本应用动态配置
         */
        // override url with configurator from configurator from "app-name.configurators"
        providerUrl = overrideWithConfigurators(CONSUMER_CONFIGURATION_LISTENER.getConfigurators(), providerUrl);

        /**
         * 新版本服务动态配置
         */
        // override url with configurator from configurators from "service-name.configurators"
        if (serviceConfigurationListener != null) {
            providerUrl = overrideWithConfigurators(serviceConfigurationListener.getConfigurators(), providerUrl);
        }

        return providerUrl;
    }
```

11.`new InvokerDelegate<>(protocol.refer(serviceType, url), url, providerUrl)`其中`protocol.refer(serviceType, url)`会获取到Invoker，然后将结果作为属性创建InvokerDelegate，直接返回
12. <font color='red'><b>protocol.refer(serviceType, url)这里的protocol是DubboProtocol，这里同样会经过Wrapper类，ProtocolFilterWrapper、ProtocolListenerWrapper调用后，最终会调用DubboProtocol.refer，逻辑和服务导出有点类似，这里直接出结果，不仔细描述，可以跟着代码走一下</b></font>
13. 那最终得到的Invoker是什么？

```java
RegistryDirectory$InvokerDelegate ->
    ListenerInvokerWrapper ->
        ProtocolFilterWrapper$Invoker(Invoker$0(ConsumerContextFilter) -> Invoker$1(FutureFilter) -> Invoker$2(MonitorFilter)) ->
           AsyncToSyncInvoker ->
                DubboInvoker
```

# 总结

1. ReferenceBean.get()获取代理类
2. 如果调用方法，直接调用到InvokerInvocationHandler
3. 其中的Invoker其实是下面的
```text
MockClusterInvoker ->
    AbstractCluster$Interceptor
        FailoverClusterInvoker ->
            RegistryDirectory ->
                路由
                List<RegistryDirectory$InvokerDelegate>
                负载
                    RegistryDirectory$InvokerDelegate
                        ListenerInvokerWrapper ->
                            ProtocolFilterWrapper$Invoker(Invoker$0(ConsumerContextFilter) -> Invoker$1(FutureFilter) -> Invoker$2(MonitorFilter)) ->
                                AsyncToSyncInvoker ->
                                    DubboInvoker


1. MockClusterInvoker：完成mock功能
2. FailoverClusterInvoker：完成集群容错功能，重试功能
3. ProtocolFilterWrapper$Invoker：完成对filter调用
4. DubboInvoker：通过dubbo协议发送数据
```
4. NettyClient连接，发送请求返回（这块调用的时候仔细看下）

# 附录

见源码：https://github.com/tahw/dubbo
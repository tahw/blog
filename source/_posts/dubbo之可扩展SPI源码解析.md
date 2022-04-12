---
title: dubbo之可扩展SPI源码解析
date: 2021-12-21 18:51:43
tags:
    - dubbo
categories:
    - dubbo
---

# dubbo之可扩展SPI源码解析
&nbsp;&nbsp;&nbsp;&nbsp;在dubbo中，很多的扩展都是spi来加载的，比如Procotol、LoadBalance，而dubbo的扩展点加载是基于jdk标准的spi扩展机制增强而来的，dubbo解决jdk有的一些问题
* [x] jdk标准的spi会一次性实例化所有的实现，不能按需加载
* [x] 扩展点如果加载失败，不会友好的提示异常信息
* [x] 可以对扩展点实现ioc和aop，ioc通过setter()方法来注入，而aop可以通过wrapper类来增强

## 官方地址
https://dubbo.apache.org/zh/docsv2.7/dev/source/adaptive-extension/

## SPI应用

这里是引用github dubbo master的代码，版本是2.7.7-SNAPSHOT
见：https://github.com/tahw/dubbo

### 引用
新增模块，要使用dubbo，需引入下面pom
```xml
<dependency>
    <groupId>org.apache.dubbo</groupId>
    <artifactId>dubbo-common</artifactId>
    <version>2.7.7-SNAPSHOT</version>
</dependency>
<dependency>
    <groupId>org.apache.dubbo</groupId>
    <artifactId>dubbo-rpc-api</artifactId>
    <version>2.7.7-SNAPSHOT</version>
    <scope>compile</scope>
</dependency>
<dependency>
    <groupId>org.apache.dubbo</groupId>
    <artifactId>dubbo-rpc-dubbo</artifactId>
    <version>2.7.7-SNAPSHOT</version>
</dependency>
<dependency>
    <groupId>org.apache.dubbo</groupId>
    <artifactId>dubbo-rpc-http</artifactId>
    <version>2.7.7-SNAPSHOT</version>
</dependency>
```

<!-- more -->

### SPI&AOP
dubbo aop相对来说是比较简单，没有像spring那么复杂，aop其实就是代码增强，dubbo aop其实就是使用Wrapper来实现，先看下应用

```java
package com.apache.dubbo.api;

import org.apache.dubbo.common.extension.SPI;

/*
* 该接口需要被@SPI注释，接口里面的值是默认值
*/
@SPI("default")
public interface HelloService {

    public void hello();
}
```

对应spi文件路径：META-INF/dubbo/com.apache.dubbo.api.HelloService
```text
default,default1=com.apache.dubbo.service.DefaultHelloService
java=com.apache.dubbo.service.JavaHelloService
com.apache.dubbo.wrapper.HelloServiceWrapper
com.apache.dubbo.wrapper.HelloServiceWrapper2
```
> 普通的实现类key可以配置多个，多个用逗号来分隔
注意其中的wrapper是不需要配置key的，里面可以配置多个Wrapper类

#### 例子
```java
package com.apache.dubbo.service;

import com.apache.dubbo.api.HelloService;

public class DefaultHelloService implements HelloService {

    @Override
    public void hello() {
        System.out.println("default");
    }
}
```
```java
package com.apache.dubbo.service;

import com.apache.dubbo.api.HelloService;

public class JavaHelloService implements HelloService {

    @Override
    public void hello() {
        System.out.println("hello world");
    }
}
```


```java
package com.apache.dubbo.wrapper;

import com.apache.dubbo.api.HelloService;

public class HelloServiceWrapper implements HelloService {

    private HelloService helloService;

    public HelloServiceWrapper(HelloService helloService) {
        this.helloService = helloService;
    }

    @Override
    public void hello() {
        System.out.println("wrapper before");
        helloService.hello();
    }
}
```
```java
package com.apache.dubbo.wrapper;

import com.apache.dubbo.api.HelloService;

public class HelloServiceWrapper2 implements HelloService {

    private HelloService helloService;

    public HelloServiceWrapper2(HelloService helloService) {
        this.helloService = helloService;
    }

    @Override
    public void hello() {
        System.out.println("wrapper before2");
        helloService.hello();
    }
}
```
<font color='red'><b>一般把这里的类命名后缀名为wrapper，就是表示aop增强类，这里的wrapper类也是需要实现spi接口，然后还需要一个带有接口参数的构造函数，这个其实就是适配器模式，这样dubbo会根据构造函数设置接口类，然后Wrapper也可以加一些其他的逻辑</b></font>

```java
package com.apache.dubbo;

import com.apache.dubbo.api.HelloService;
import org.apache.dubbo.common.extension.ExtensionLoader;


public class TestSpi {

    public static void main(String[] args) {
        // 获取spi扩展类，dubbo固定模板
        ExtensionLoader<HelloService> extensionLoader = ExtensionLoader.getExtensionLoader(HelloService.class);
        HelloService helloService = extensionLoader.getExtension("java");
        helloService.hello();
    }
}
/*
* 输出结果
* wrapper before
* wrapper2 before
* hello world
*/
```
<font color='red'><b>注意这里是通过ExtensionLoader来获取扩展接口，extensionLoader.getExtension("java")中的java是通过配置文件的key来对应的。其中的wrapper是无序的，底层是ConcurrentHashSet存储的。后续详见源码解析</b></font>

### SPI&IOC
dubbo ioc相对来说是比较复杂的，先看下应用来理解，后续还是会用源码来解释

```java
package com.apache.dubbo.api;

import org.apache.dubbo.common.extension.SPI;

@SPI("a")
public interface A {

    public B getB();

}
```
```java
package com.apache.dubbo.api;

import org.apache.dubbo.common.URL;
import org.apache.dubbo.common.extension.Adaptive;
import org.apache.dubbo.common.extension.SPI;

@SPI
public interface B {

    @Adaptive
    public String b(URL url);

    public String bOther();
}

```

文件路径：META-INF/dubbo/com.apache.dubbo.api.A

```text
a=com.apache.dubbo.service.AImpl
```
文件路径：META-INF/dubbo/com.apache.dubbo.api.B
```text
b=com.apache.dubbo.service.BImpl
bOther=com.apache.dubbo.service.BOtherImpl
```

#### 例子
```java
package com.apache.dubbo.service;

import com.apache.dubbo.api.A;
import com.apache.dubbo.api.B;

public class AImpl implements A {

    private B b;

    public void setB(B b) {
        this.b = b;
    }

    @Override
    public B getB() {
        return b;
    }
}
```

```java
package com.apache.dubbo.service;

import com.apache.dubbo.api.B;
import org.apache.dubbo.common.URL;

public class BImpl implements B {
    @Override
    public String b(URL url) {
        return "bImpl";
    }

    @Override
    public String bOther() {
        return null;
    }
}
```

```java
package com.apache.dubbo.service;

import com.apache.dubbo.api.B;
import org.apache.dubbo.common.URL;
import org.apache.dubbo.common.extension.Adaptive;

@Adaptive
public class BOtherImpl implements B {
    @Override
    public String b(URL url) {
        return "bOtherImpl";
    }

    @Override
    public String bOther() {
        return null;
    }
}
```

<font color='red'><b>注意dubbo ioc是通过Adaptive来实现的。在AImpl里面注入B，在AImpl里面需要添加setter()方法，dubbo ioc，就是通过setter方法来赋值，还有一点是需要注意的，这里要注入的B，其实就是下面两种方式
1. B是@Adaptive修饰类
2. B里面有@Adaptive修饰方法，dubbo根据@Adaptive方法来生成代理类，代理类需要下面两个条件
    1. 一个是@Adaptive修饰方法
    2. 方法参数需要有URL参数
</b></font>

#### @Adaptive
@Adaptive是作用于依赖注入IOC来说，这里介绍下@Adaptive，@Adaptive可以注释在类上或者方法上，那注释不同的地方其实含义是不一样的，<font color='red'><b>当Adaptive注释在类上，Dubbo不会为该类生成代理类。注释在方法上，Dubbo则会为该方法生成代理逻辑。</b></font>Adaptive注释在类上很少，分别是 AdaptiveCompiler 和 AdaptiveExtensionFactory。其他情况都是由dubbo来生成代理类

##### IOC场景一
> 存在@Adaptive实现类注入不随参数的改变而改变

```java
package com.apache.dubbo;

import com.apache.dubbo.api.A;
import com.apache.dubbo.api.B;
import com.apache.dubbo.api.HelloService;
import org.apache.dubbo.common.URL;
import org.apache.dubbo.common.extension.ExtensionLoader;


public class TestSpiIoc {

    public static void main(String[] args) {
        ExtensionLoader<A> extensionLoader = ExtensionLoader.getExtensionLoader(A.class);
        A a = extensionLoader.getExtension("true");
        URL url = new URL("x","127.0.0.1",8080);
        url = url.addParameter("b","b");
        System.out.println(a.getB().b(url));
    }
}

# 输出结果是bOtherImpl，BOtherImpl类上有@Adaptive
```
<b>注意输出还是：bOtherImpl，只有一个实现类中类上面有@Adaptive注解，这里注入的就是该类，不会伴随参数的改变而改变</b>

##### IOC场景二
> 只存在一个实现类，而且被@Adaptive修饰，<font color='red'><b>这种场景是注入不成功的，没有一个普通的实现类，是注入不成功的</b></font>

如果这种场景，如果我想获取扩展类，是采用getAdaptiveExtension获取对象
```java
package com.apache.dubbo;

import com.apache.dubbo.api.B;
import org.apache.dubbo.common.URL;
import org.apache.dubbo.common.extension.ExtensionLoader;


public class TestSpiAdaptive {

    public static void main(String[] args) {
        B b = ExtensionLoader.getExtensionLoader(B.class).getAdaptiveExtension();
        System.out.println(b.b(new URL()));
    }
}

# 这里能返回B的@Adaptive 注解的类
```

##### IOC场景三
> 不存在@Adaptive类，方法上面有@@Adaptive注解，生成代理类，然后URL传参的时候就获取对应的实现类
```java
package com.apache.dubbo;

import com.apache.dubbo.api.A;
import com.apache.dubbo.api.B;
import com.apache.dubbo.api.HelloService;
import org.apache.dubbo.common.URL;
import org.apache.dubbo.common.extension.ExtensionLoader;


public class TestSpiIoc {

    public static void main(String[] args) {
        ExtensionLoader<A> extensionLoader = ExtensionLoader.getExtensionLoader(A.class);
        A a = extensionLoader.getExtension("true");
        URL url = new URL("x","127.0.0.1",8080);
        url = url.addParameter("b","b");
        System.out.println(a.getB().b(url));
    }
}
# 这里的B就是BImpl
```
<font color='red'><b>下面就是通过dubbo生成的代理类，bOther()方法没有通过@Adaptive注解修饰，直接会抛异常，这里生成的方法没有调用是不会报错的。类名：
接口名$Adaptive，然后看下b()方法，里面从url获取参数b，如果URL没有传参数，ioc注入是不会错的，只是代理类，但是调用方法，如果找不到的话，就会报错的。</b></font>

```java
package com.apache.dubbo.api;

import org.apache.dubbo.common.extension.ExtensionLoader;

public class B$Adaptive implements com.apache.dubbo.api.B {
    public java.lang.String bOther() {
        throw new UnsupportedOperationException("The method public abstract java.lang.String com.apache.dubbo.api.B.bOther() of interface com.apache.dubbo.api.B is not adaptive method!");
    }

    public java.lang.String b(org.apache.dubbo.common.URL arg0) {
        if (arg0 == null) throw new IllegalArgumentException("url == null");
        org.apache.dubbo.common.URL url = arg0;
        String extName = url.getParameter("b"); // ? 这里为什么会是b，带着疑问我们往下看
        if (extName == null)
            throw new IllegalStateException("Failed to get extension (com.apache.dubbo.api.B) name from url (" + url.toString() + ") use keys([b])");
        com.apache.dubbo.api.B extension = (com.apache.dubbo.api.B) ExtensionLoader.getExtensionLoader(com.apache.dubbo.api.B.class).getExtension(extName);
        return extension.b(arg0);
    }
}
```

## 源码解析
clone官网的dubbo源码，创建测试模块，通过源码的方式来学习，还是非常清晰的
地址：https://github.com/tahw/dubbo

### 介绍ExtensionLoader
spi操作基本方法都是在ExtensionLoader里面的，这里介绍一下它的属性

```java
public class ExtensionLoader<T> {

    private static final Logger logger = LoggerFactory.getLogger(ExtensionLoader.class);

    /*
    * 加载的目录
    */
    private static final String SERVICES_DIRECTORY = "META-INF/services/";

    private static final String DUBBO_DIRECTORY = "META-INF/dubbo/";

    private static final String DUBBO_INTERNAL_DIRECTORY = DUBBO_DIRECTORY + "internal/";

    /*
    * 配置文件key的分隔符
    */
    private static final Pattern NAME_SEPARATOR = Pattern.compile("\\s*[,]+\\s*");

    /*
    * 这里是一个ConcurrentHashMap，并发安全的，缓存ExtensionLoader
    */
    private static final ConcurrentMap<Class<?>, ExtensionLoader<?>> EXTENSION_LOADERS = new ConcurrentHashMap<>(64);

    /*
    * 这里是一个ConcurrentHashMap，并发安全的，缓存扩展实例
    * 不包含Wrapper和Adaptive类
    */
    private static final ConcurrentMap<Class<?>, Object> EXTENSION_INSTANCES = new ConcurrentHashMap<>(64);
    /*
    * 看起来很简单，这是全局最重要的属性了，接口类型
    */
    private final Class<?> type;

    /*
    * 扩展工厂，生成代理类，这里默认就是AdaptiveExtensionFactory
    */
    private final ExtensionFactory objectFactory;
    /*
    * 这里是一个ConcurrentHashMap，并发安全的，缓存类和key
    * A a
    * A a1
    * 不包含Wrapper和Adaptive类
    */  
    private final ConcurrentMap<Class<?>, String> cachedNames = new ConcurrentHashMap<>();

    /*
    * 这里是一个ConcurrentHashMap，并发安全的，cachedNames中的key和value反过来
    * 不包含Wrapper和Adaptive类
    * 
    */  
    private final Holder<Map<String, Class<?>>> cachedClasses = new Holder<>();
    /*
    * 这里是一个ConcurrentHashMap，并发安全的，被@Activate修饰的类，如果多个key，取第一个
    * TODO 这个是做什么的？后续看看的
    */  
    private final Map<String, Object> cachedActivates = new ConcurrentHashMap<>();

    /*
    * 这里是一个ConcurrentHashMap，并发安全的，缓存实例，这个是Holder，实际是ExtensionLoader实例
    */  
    private final ConcurrentMap<String, Holder<Object>> cachedInstances = new ConcurrentHashMap<>();
    /*
    * 这里是一个当前的Holder，实际是ExtensionLoader实例
    */  
    private final Holder<Object> cachedAdaptiveInstance = new Holder<>();
    /*
    * 这里是一个volatile类，是缓存的Adaptive类，Adaptive只有一个
    */  
    private volatile Class<?> cachedAdaptiveClass = null;
    /*
    * spi默认的名称
    */
    private String cachedDefaultName;

    private volatile Throwable createAdaptiveInstanceError;

    /*
    * 这里是set，但是实际是ConcurrentHashSet，这里是没有顺序的，这里存储的是AOP的Wrapper类
    */
    private Set<Class<?>> cachedWrapperClasses;
    /*
    * 异常信息，加载文件中一行一行记录异常
    * 如果找不到name对应的实现类，就会报出异常，通过exceptions来返回
    */  
    private Map<String, IllegalStateException> exceptions = new ConcurrentHashMap<>();

    private static LoadingStrategy DUBBO_INTERNAL_STRATEGY =  () -> DUBBO_INTERNAL_DIRECTORY;
    private static LoadingStrategy DUBBO_STRATEGY = () -> DUBBO_DIRECTORY;
    private static LoadingStrategy SERVICES_STRATEGY = () -> SERVICES_DIRECTORY;

    /*
    * 加载的类目录顺序的
    */  
    private static LoadingStrategy[] strategies = new LoadingStrategy[] { DUBBO_INTERNAL_STRATEGY, DUBBO_STRATEGY, SERVICES_STRATEGY };

}

```

### 获取ExtensionLoader
![dubbo整体结构](/images/dubbo-3-1.png)

```java
/**
* type是贯穿全局的，就是要操作的那个接口，这里要注意下，ExtensionLoader是根据type生成出来，不同的type的ExtensionLoader是不一样的，一样的type只会留一份
* @param type
* @param <T>
* @return
*/
@SuppressWarnings("unchecked")
public static <T> ExtensionLoader<T> getExtensionLoader(Class<T> type) {
    if (type == null) {
        throw new IllegalArgumentException("Extension type == null");
    }
    if (!type.isInterface()) {
        throw new IllegalArgumentException("Extension type (" + type + ") is not an interface!");
    }
    if (!withExtensionAnnotation(type)) {
        throw new IllegalArgumentException("Extension type (" + type +
                ") is not an extension, because it is NOT annotated with @" + SPI.class.getSimpleName() + "!");
    }
    return (ExtensionLoader<T>) EXTENSION_LOADERS.computeIfAbsent(type, k -> new ExtensionLoader<T>(type));
}
```

### ExtendsionLoader的getExtension

```java
/**
    * Find the extension with the given name. If the specified name is not found, then {@link IllegalStateException}
    * will be thrown.
    *
    * 参数name如果为true的话，就是默认spi扩展类，默认的名称不能有两个，只能是一个名称
    */
@SuppressWarnings("unchecked")
public T getExtension(String name) {
    if (StringUtils.isEmpty(name)) {
        throw new IllegalArgumentException("Extension name == null");
    }
    if ("true".equals(name)) {
        return getDefaultExtension();
    }
    final Holder<Object> holder = getOrCreateHolder(name);
    Object instance = holder.get();
    /**
        * <pre>看到这里是否会有疑问？为什么存在Holder，就是包装了一下，没有什么其他的作用，为什么要有这个Holder？</pre>
        * 下面的是dcl
        *
        * 这里说明下
        * <ul>
        *     <li>一个spi的接口的name同一时间只能由一个线程去加载</li>
        *     <li>同时加载的时候，其实还没加载出来对应的实现类，还没办法成为dcl的锁，这里就借助Holder来成为锁</li>
        * </ul>
        */
    if (instance == null) {
        synchronized (holder) {
            instance = holder.get();
            if (instance == null) {
                instance = createExtension(name);
                holder.set(instance);
            }
        }
    }
    return (T) instance;
}

/**
* 这里其实就是创建Holder类，然后放入到ConcurrentHashMap里面，这里根据name获取到Holder（这个方法是在ExtensionLoader下面），
* 所以是spi 接口下面的name，不会有重复name问题
* @param name
* @return
*/
private Holder<Object> getOrCreateHolder(String name) {
    Holder<Object> holder = cachedInstances.get(name);
    if (holder == null) {
        cachedInstances.putIfAbsent(name, new Holder<>());
        holder = cachedInstances.get(name);
    }
    return holder;
}
```
### ExtendsionLoader的createExtension
这里就是实际获取对象的方法了，这里是比较重要的，需要注意看下下面的方法
1. getExtensionClasses()获取所有的扩展类
2. injectExtension(instance) ioc注入
3. injectExtension((T) wrapperClass.getConstructor(type).newInstance(instance));

```java
@SuppressWarnings("unchecked")
private T createExtension(String name) {
    /**
        * getExtensionClasses() 就是获取当前spi接口里的所有的实现
        */
    Class<?> clazz = getExtensionClasses().get(name);
    if (clazz == null) {
        throw findException(name);
    }
    try {
        /**
        * 直接通过反射实例化
        */
        T instance = (T) EXTENSION_INSTANCES.get(clazz);
        if (instance == null) {
            EXTENSION_INSTANCES.putIfAbsent(clazz, clazz.newInstance());
            instance = (T) EXTENSION_INSTANCES.get(clazz);
        }
        /**
        * dubbo ioc
        * 这里是相对复杂的
        */
        injectExtension(instance);
        Set<Class<?>> wrapperClasses = cachedWrapperClasses;
        /**
        * dubbo aop
        * cachedWrapperClasses是一个ConcurrentHashSet，是无序的，这里读取的配置Wrapper类是没有先后顺序的
        *
        * 什么是Wrapper类，其实就是适配器模式
        * ```java
        * @SPI("default")
        * public interface HelloService {
        *
        *     public void hello();
        * }
        *
        * public class HelloServiceWrapper implements HelloService {
        *
        *     private HelloService helloService;
        *
        *     public HelloServiceWrapper(HelloService helloService) {
        *         this.helloService = helloService;
        *     }
        *
        *     @Override
        *     public void hello() {
        *         helloService.hello();
        *     }
        * }
        * ```
        */
        if (CollectionUtils.isNotEmpty(wrapperClasses)) {
            for (Class<?> wrapperClass : wrapperClasses) {
                instance = injectExtension((T) wrapperClass.getConstructor(type).newInstance(instance));
            }
        }
        /**
        * TODO 初始化扩展，有空再去深入一下，这块可以暂时不用看
        */
        initExtension(instance);
        return instance;
    } catch (Throwable t) {
        throw new IllegalStateException("Extension instance (name: " + name + ", class: " +
                type + ") couldn't be instantiated: " + t.getMessage(), t);
    }
}

```

#### getExtensionClasses()

这里返回的是spi的实现类，但是这里是不包含@Adaptive和Wrapper类的

![getExtensionClasses](/images/dubbo-3-3.png)

```java
/**
* dcl验证
* 注意：这里返回的是spi的实现类，但是这里是不包含@Adaptive和Wrapper类的
* @return
*/
private Map<String, Class<?>> getExtensionClasses() {
    Map<String, Class<?>> classes = cachedClasses.get();
    if (classes == null) {
        synchronized (cachedClasses) {
            classes = cachedClasses.get();
            if (classes == null) {
                classes = loadExtensionClasses();
                cachedClasses.set(classes);
            }
        }
    }
    return classes;
}
```

```java
这次是安全的加载，每次找的时候都是通过type来找，这个其实还是很快的
/*
* synchronized in getExtensionClasses
* 
*/
private Map<String, Class<?>> loadExtensionClasses() {
    cacheDefaultExtensionName();

    Map<String, Class<?>> extensionClasses = new HashMap<>();

    /**
        * 1. META-INF/dubbo/internal/
        * 2. META-INF/dubbo
        * 3. META-INF/services/
        */
    for (LoadingStrategy strategy : strategies) {
        loadDirectory(extensionClasses, strategy.directory(), type.getName(), strategy.preferExtensionClassLoader(), strategy.excludedPackages());
        loadDirectory(extensionClasses, strategy.directory(), type.getName().replace("org.apache", "com.alibaba"), strategy.preferExtensionClassLoader(), strategy.excludedPackages());
    }
    return extensionClasses;
}
```
```java
/*
* 缓存默认的扩展名，通过这里可以知道，默认的名称可以配置多个，只要匹配规则就行
* extract and cache default extension name if exists
*/
private void cacheDefaultExtensionName() {
    final SPI defaultAnnotation = type.getAnnotation(SPI.class);
    if (defaultAnnotation == null) {
        return;
    }
    String value = defaultAnnotation.value();
    if ((value = value.trim()).length() > 0) {
        String[] names = NAME_SEPARATOR.split(value);
        if (names.length > 1) {
            throw new IllegalStateException("More than 1 default extension name on extension " + type.getName()
                    + ": " + Arrays.toString(names));
        }
        if (names.length == 1) {
            cachedDefaultName = names[0];
        }
    }
}
```
```java
/**
* 加载目录
* @param extensionClasses 类型是HashMap
* @param dir 目录
* @param type 接口名称
* @param extensionLoaderClassLoaderFirst false
* @param excludedPackages null
*/
private void loadDirectory(Map<String, Class<?>> extensionClasses, String dir, String type,
                            boolean extensionLoaderClassLoaderFirst, String... excludedPackages) {
    String fileName = dir + type; // 目录文件
    try {
        Enumeration<java.net.URL> urls = null;
        ClassLoader classLoader = findClassLoader(); // 类加载器

        // try to load from ExtensionLoader's ClassLoader first
        if (extensionLoaderClassLoaderFirst) {
            ClassLoader extensionLoaderClassLoader = ExtensionLoader.class.getClassLoader();
            if (ClassLoader.getSystemClassLoader() != extensionLoaderClassLoader) {
                urls = extensionLoaderClassLoader.getResources(fileName);
            }
        }

        if(urls == null || !urls.hasMoreElements()) {
            if (classLoader != null) {
                urls = classLoader.getResources(fileName);
            } else {
                urls = ClassLoader.getSystemResources(fileName);
            }
        }

        if (urls != null) {
            while (urls.hasMoreElements()) {
                java.net.URL resourceURL = urls.nextElement();
                // 真正加载
                loadResource(extensionClasses, classLoader, resourceURL, excludedPackages);
            }
        }
    } catch (Throwable t) {
        logger.error("Exception occurred when loading extension class (interface: " +
                type + ", description file: " + fileName + ").", t);
    }
}
```

```java
private static ClassLoader findClassLoader() {
    return ClassUtils.getClassLoader(ExtensionLoader.class);
}

/**
* 这里其实有点注意的，这里传入的都是ExtensionLoader.class，但是传入的不知道是哪一个ExtensionLoader，不一样的类型其实加载的ClassLoader是不一样的，应用类还是系统类
* get class loader
*
* @param clazz
* @return class loader
*/
public static ClassLoader getClassLoader(Class<?> clazz) {
    ClassLoader cl = null;
    try {
        cl = Thread.currentThread().getContextClassLoader();
    } catch (Throwable ex) {
        // Cannot access thread context ClassLoader - falling back to system class loader...
    }
    if (cl == null) {
        // No thread context class loader -> use class loader of this class.
        cl = clazz.getClassLoader();
        if (cl == null) {
            // getClassLoader() returning null indicates the bootstrap ClassLoader
            try {
                cl = ClassLoader.getSystemClassLoader();
            } catch (Throwable ex) {
                // Cannot access system ClassLoader - oh well, maybe the caller can live with null...
            }
        }
    }

    return cl;
}
```
```java
/**
* 真正加载类的方法
* @param extensionClasses
* @param classLoader
* @param resourceURL
* @param excludedPackages
*/
private void loadResource(Map<String, Class<?>> extensionClasses, ClassLoader classLoader,
                            java.net.URL resourceURL, String... excludedPackages) {
    try {
        try (BufferedReader reader = new BufferedReader(new InputStreamReader(resourceURL.openStream(), StandardCharsets.UTF_8))) {
            String line;
            while ((line = reader.readLine()) != null) {
                final int ci = line.indexOf('#');
                if (ci >= 0) {
                    line = line.substring(0, ci);
                }
                line = line.trim();
                if (line.length() > 0) {
                    try {
                        String name = null;
                        int i = line.indexOf('=');
                        if (i > 0) {
                            name = line.substring(0, i).trim();
                            line = line.substring(i + 1).trim();
                        }
                        if (line.length() > 0 && !isExcluded(line, excludedPackages)) {
                            loadClass(extensionClasses, resourceURL, Class.forName(line, true, classLoader), name);
                        }
                    } catch (Throwable t) {
                        IllegalStateException e = new IllegalStateException("Failed to load extension class (interface: " + type + ", class line: " + line + ") in " + resourceURL + ", cause: " + t.getMessage(), t);
                        exceptions.put(line, e);
                    }
                }
            }
        }
    } catch (Throwable t) {
        logger.error("Exception occurred when loading extension class (interface: " +
                type + ", class file: " + resourceURL + ") in " + resourceURL, t);
    }
}
```


#### loadClass
真正加载类的地方，这个地方需要认真看下，这里不仅加载出来实现类，也会加载出来Adaptive类，也会加载出来Wrapper类，还会加载出来Activate类
```java
/**
* 真正加载类
* @param extensionClasses
* @param resourceURL
* @param clazz
* @param name
* @throws NoSuchMethodException
*/
private void loadClass(Map<String, Class<?>> extensionClasses, java.net.URL resourceURL, Class<?> clazz, String name) throws NoSuchMethodException {
    /**
        * 判断该类是否是继承关系
        * 父类.isAssignableFrom(子类)
        * 子类 instanceOf 父类
        */
    if (!type.isAssignableFrom(clazz)) {
        throw new IllegalStateException("Error occurred when loading extension class (interface: " +
                type + ", class line: " + clazz.getName() + "), class "
                + clazz.getName() + " is not subtype of interface.");
    }
    if (clazz.isAnnotationPresent(Adaptive.class)) { // 该类是否有@Adaptive注解
        cacheAdaptiveClass(clazz);
    } else if (isWrapperClass(clazz)) { // 是否是包装类
        cacheWrapperClass(clazz);
    } else {
        clazz.getConstructor(); // 判断是否有无参的构造方法，没有就会抛异常
        if (StringUtils.isEmpty(name)) {
            name = findAnnotationName(clazz);
            if (name.length() == 0) {
                throw new IllegalStateException("No such extension name for the class " + clazz.getName() + " in the config " + resourceURL);
            }
        }

        String[] names = NAME_SEPARATOR.split(name);
        if (ArrayUtils.isNotEmpty(names)) { // 配置文件的key也可以配置多个，以逗号分隔
            cacheActivateClass(clazz, names[0]);
            for (String n : names) {
                cacheName(clazz, n);
                saveInExtensionClass(extensionClasses, clazz, n);
            }
        }
    }
}
```

##### AdaptiveClass
```java
这里只能有一个
/**
* cache Adaptive class which is annotated with <code>Adaptive</code>
*/
private void cacheAdaptiveClass(Class<?> clazz) {
    if (cachedAdaptiveClass == null) {
        cachedAdaptiveClass = clazz;
    } else if (!cachedAdaptiveClass.equals(clazz)) {
        throw new IllegalStateException("More than 1 adaptive class found: "
                + cachedAdaptiveClass.getName()
                + ", " + clazz.getName());
    }
}
```

##### WrapperClass
怎么判断是一个Wrapper类？并不是根据名称来判断，看下下面的源码，还是很巧妙的
```java
/**
    * test if clazz is a wrapper class
    * <p>
    * which has Constructor with given class type as its only argument
    */
private boolean isWrapperClass(Class<?> clazz) {
    try {
        clazz.getConstructor(type);
        return true;
    } catch (NoSuchMethodException e) {
        return false;
    }
}
```
判断完成后，就会缓存WrapperClass，这里看下源码，是ConcurrentHashSet，安全没有顺序的集合
```java
/**
* cache wrapper class
* <p>
* like: ProtocolFilterWrapper, ProtocolListenerWrapper
*/
private void cacheWrapperClass(Class<?> clazz) {
    if (cachedWrapperClasses == null) {
        cachedWrapperClasses = new ConcurrentHashSet<>();
    }
    cachedWrapperClasses.add(clazz);
}
```

##### ActivateClass
这个是缓存类上面有@Activate注解类，具体是干嘛的，后续再补充了......
后续补充......，最近看了下，这个注解是非常的有作用，表示一个扩展是否被激活（使用），可以放在接口上或者方法上，dubbo用它在spi扩展类定义上，表示这个扩展激活的时机和条件，这个类在服务导出的时候会重点去介绍

```java
URL url = new URL("x","",1111);
url = url.addParameter("a","a");
String key = "a";
// 调用的方式
List<HelloActivateService> activateExtension = ExtensionLoader.getExtensionLoader(HelloActivateService.class).getActivateExtension(url, key, CommonConstants.PROVIDER);
```

```java
/**
* cache Activate class which is annotated with <code>Activate</code>
* <p>
* for compatibility, also cache class with old alibaba Activate annotation
*/
private void cacheActivateClass(Class<?> clazz, String name) {
    Activate activate = clazz.getAnnotation(Activate.class);
    if (activate != null) {
        cachedActivates.put(name, activate);
    } else {
        // support com.alibaba.dubbo.common.extension.Activate
        com.alibaba.dubbo.common.extension.Activate oldActivate = clazz.getAnnotation(com.alibaba.dubbo.common.extension.Activate.class);
        if (oldActivate != null) {
            cachedActivates.put(name, oldActivate);
        }
    }
}
```

##### ImplClass
<font color='red'><b>除了WrapperClass和AdaptiveClass外，其他实现，就是基本实现类了，这里会先判断一下是否有默认的构造方法，没有的话就会抛出异常，然后就会更新到extensionClasses里，这里是不包含AdaptiveClass和WrapperClass的</b></font>

```java
/**
* put clazz in extensionClasses
*/
private void saveInExtensionClass(Map<String, Class<?>> extensionClasses, Class<?> clazz, String name) {
    Class<?> c = extensionClasses.get(name);
    if (c == null) {
        extensionClasses.put(name, clazz);
    } else if (c != clazz) {
        String duplicateMsg = "Duplicate extension " + type.getName() + " name " + name + " on " + c.getName() + " and " + clazz.getName();
        logger.error(duplicateMsg);
        throw new IllegalStateException(duplicateMsg);
    }
}
```

#### injectExtension()
dubbo ioc & dubbo aop都会调用这个方法来实现的，这个方法我们先好好看一下

![injectExtension](/images/dubbo-3-4.png)

```java
/**
    * dubbo ioc
    * objectFactory = (type == ExtensionFactory.class ? null : ExtensionLoader.getExtensionLoader(ExtensionFactory.class).getAdaptiveExtension());
    * @param instance
    * @return
    */
private T injectExtension(T instance) { // instance 就是要传的接口

    if (objectFactory == null) {
        return instance;
    }

    try {
        for (Method method : instance.getClass().getMethods()) {
            if (!isSetter(method)) { // 这里其实还有Object方法
                continue;
            }
            /**
                * Check {@link DisableInject} to see if we need auto injection for this property
                */
            if (method.getAnnotation(DisableInject.class) != null) { // 方法被@DisableInject注解修饰就不会生成方法
                continue;
            }
            Class<?> pt = method.getParameterTypes()[0]; // 第一个参数类型
            if (ReflectUtils.isPrimitives(pt)) {
                continue;
            }

            try {
                String property = getSetterProperty(method);
                Object object = objectFactory.getExtension(pt, property); // B.class，b
                if (object != null) {
                    method.invoke(instance, object);
                }
            } catch (Exception e) {
                logger.error("Failed to inject via method " + method.getName()
                        + " of interface " + type.getName() + ": " + e.getMessage(), e);
            }

        }
    } catch (Exception e) {
        logger.error(e.getMessage(), e);
    }
    return instance;
}
```

objectFactory.getExtension中的objectFactory其实就是AdaptiveExtensionFactory，这块等会儿看下源码
```java
/**
 * AdaptiveExtensionFactory
 */
@Adaptive
public class AdaptiveExtensionFactory implements ExtensionFactory {

    /**
     * 这里的factories只有一个SpiExtensionFactory
     */
    private final List<ExtensionFactory> factories;

    public AdaptiveExtensionFactory() {
        ExtensionLoader<ExtensionFactory> loader = ExtensionLoader.getExtensionLoader(ExtensionFactory.class);
        List<ExtensionFactory> list = new ArrayList<ExtensionFactory>();
        for (String name : loader.getSupportedExtensions()) {
            list.add(loader.getExtension(name));
        }
        factories = Collections.unmodifiableList(list);
    }

    /**
     * 
     * @param type object type.
     * @param name object name.
     * @param <T>
     * @return
     */
    @Override
    public <T> T getExtension(Class<T> type, String name) {
        for (ExtensionFactory factory : factories) {
            T extension = factory.getExtension(type, name);
            if (extension != null) {
                return extension;
            }
        }
        return null;
    }
}
```
其中的factories就是ExtensionFactory接口的实现类，这里就是SpiExtensionFactory，如果目前有结合spring使用，需要引入dubbo-config-spring，这里的实现类还有SpringExtensionFactory
```java
public Set<String> getSupportedExtensions() {
    Map<String, Class<?>> clazzes = getExtensionClasses();
    return Collections.unmodifiableSet(new TreeSet<>(clazzes.keySet()));
}
```

这里的SpringExtensionFactory就是根据ByName去找，找不到根据ByType去找
```java
/**
 * SpringExtensionFactory
 */
public class SpringExtensionFactory implements ExtensionFactory {
    private static final Logger logger = LoggerFactory.getLogger(SpringExtensionFactory.class);

    private static final Set<ApplicationContext> CONTEXTS = new ConcurrentHashSet<ApplicationContext>();

    public static void addApplicationContext(ApplicationContext context) {
        CONTEXTS.add(context);
        if (context instanceof ConfigurableApplicationContext) {
            ((ConfigurableApplicationContext) context).registerShutdownHook();
        }
    }

    public static void removeApplicationContext(ApplicationContext context) {
        CONTEXTS.remove(context);
    }

    public static Set<ApplicationContext> getContexts() {
        return CONTEXTS;
    }

    // currently for test purpose
    public static void clearContexts() {
        CONTEXTS.clear();
    }

    @Override
    @SuppressWarnings("unchecked")
    public <T> T getExtension(Class<T> type, String name) {

        //SPI should be get from SpiExtensionFactory
        if (type.isInterface() && type.isAnnotationPresent(SPI.class)) {
            return null;
        }

        for (ApplicationContext context : CONTEXTS) {
            T bean = BeanFactoryUtils.getOptionalBean(context, name, type);
            if (bean != null) {
                return bean;
            }
        }

        logger.warn("No spring extension (bean) named:" + name + ", try to find an extension (bean) of type " + type.getName());

        return null;
    }
}
```
##### SpiExtensionFactory
```java
/**
 * SpiExtensionFactory
 */
public class SpiExtensionFactory implements ExtensionFactory {

    @Override
    public <T> T getExtension(Class<T> type, String name) {
        if (type.isInterface() && type.isAnnotationPresent(SPI.class)) {
            ExtensionLoader<T> loader = ExtensionLoader.getExtensionLoader(type);
            /**
             * 这里有个知识点需要注意，只有一个实现类，如果被@Adaptive注释，则条件为false，最后返回null，ioc不会成功
             */
            if (!loader.getSupportedExtensions().isEmpty()) {
                return loader.getAdaptiveExtension();
            }
        }
        return null;
    }
}
```

```java
@SuppressWarnings("unchecked")
public T getAdaptiveExtension() {
    Object instance = cachedAdaptiveInstance.get();
    if (instance == null) {
        if (createAdaptiveInstanceError != null) {
            throw new IllegalStateException("Failed to create adaptive instance: " +
                    createAdaptiveInstanceError.toString(),
                    createAdaptiveInstanceError);
        }

        synchronized (cachedAdaptiveInstance) {
            instance = cachedAdaptiveInstance.get();
            if (instance == null) {
                try {
                    instance = createAdaptiveExtension();
                    cachedAdaptiveInstance.set(instance);
                } catch (Throwable t) {
                    createAdaptiveInstanceError = t;
                    throw new IllegalStateException("Failed to create adaptive instance: " + t.toString(), t);
                }
            }
        }
    }

    return (T) instance;
}
```

```java
@SuppressWarnings("unchecked")
private T createAdaptiveExtension() {
    try {
        return injectExtension((T) getAdaptiveExtensionClass().newInstance());
    } catch (Exception e) {
        throw new IllegalStateException("Can't create adaptive extension " + type + ", cause: " + e.getMessage(), e);
    }
}
```

这里获取对应Adaptive类，如果有@Adaptive注解类，就直接返回该类，如果没有的话，就直接通过代码生成器生成代码
```java
/**
* 这里虽然看起来比较简单，也是有一个小知识点的，需要理解
* <ul>
*     <li>如果一个spi接口实现类，类上面有@Adaptive，就直接会缓存保存到cachedAdaptiveClass，获取的时候直接返回，<b>还有一点是只有一个实现类被@Adaptive修饰</b></font></li>
*     <li>如果一个spi接口实现类类注解没有@Adaptive，cachedAdaptiveClass就会空，然后执行createAdaptiveExtensionClass方法，这里还有一点是想要dubbo来生成，接口方法需要加入@Adaptive修饰，否则会抛异常</li>
* </ul>
* @return
*/
private Class<?> getAdaptiveExtensionClass() {
    getExtensionClasses();
    if (cachedAdaptiveClass != null) {
        return cachedAdaptiveClass;
    }
    return cachedAdaptiveClass = createAdaptiveExtensionClass();
}
```

###### 生成Adaptive类
```java
/**
* 其中Compiler默认就是使用javassist，也可以使用jdk的方式
*/
private Class<?> createAdaptiveExtensionClass() {
    String code = new AdaptiveClassCodeGenerator(type, cachedDefaultName).generate();
    ClassLoader classLoader = findClassLoader();
    org.apache.dubbo.common.compiler.Compiler compiler = ExtensionLoader.getExtensionLoader(org.apache.dubbo.common.compiler.Compiler.class).getAdaptiveExtension();
    return compiler.compile(code, classLoader);
}
```

```java
// AdaptiveClassCodeGenerator生成代码
/**
    * generate method declaration
    */
private String generateMethod(Method method) {
    String methodReturnType = method.getReturnType().getCanonicalName();
    String methodName = method.getName();
    String methodContent = generateMethodContent(method);
    String methodArgs = generateMethodArguments(method);
    String methodThrows = generateMethodThrows(method);
    return String.format(CODE_METHOD_DECLARATION, methodReturnType, methodName, methodArgs, methodThrows, methodContent);
}
```
那生成就是下面这个类，由接口类加上$Adaptive，发现源码是下面，通过参数的调用，通过在URL里面来获取，然后再找到对应的实现类，然后调用结果返回
```java
package com.apache.dubbo.api;

import org.apache.dubbo.common.extension.ExtensionLoader;

public class B$Adaptive implements com.apache.dubbo.api.B {
    public java.lang.String bOther() {
        throw new UnsupportedOperationException("The method public abstract java.lang.String com.apache.dubbo.api.B.bOther() of interface com.apache.dubbo.api.B is not adaptive method!");
    }

    public java.lang.String b(org.apache.dubbo.common.URL arg0) {
        if (arg0 == null) throw new IllegalArgumentException("url == null");
        org.apache.dubbo.common.URL url = arg0;
        String extName = url.getParameter("b");
        if (extName == null)
            throw new IllegalStateException("Failed to get extension (com.apache.dubbo.api.B) name from url (" + url.toString() + ") use keys([b])");
        com.apache.dubbo.api.B extension = (com.apache.dubbo.api.B) ExtensionLoader.getExtensionLoader(com.apache.dubbo.api.B.class).getExtension(extName);
        return extension.b(arg0);
    }
}
```

###### 获取Adaptive类
这是一个很好的例子
```java
objectFactory = (type == ExtensionFactory.class ? null : ExtensionLoader.getExtensionLoader(ExtensionFactory.class).getAdaptiveExtension());
```
![ExtensionFactory](/images/dubbo-3-2.png)


#### AOP
```java
wrapperClass.getConstructor(type).newInstance(instance)
```
获取参数的构造函数，然后获取对应Wrapper类，aop就是这样实现

## github
附源码备注：https://github.com/tahw/dubbo
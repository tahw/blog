---
title: dubbo应用
date: 2021-11-18 09:56:21
tags:
    - dubbo
categories:
    - dubbo
---

# Dubbo应用
&nbsp;&nbsp;&nbsp;&nbsp;Dubbo应用介绍一下dubbo里面的一些特性，相对比较泛泛，详细可以看官网。只是为后续读取dubbo源码来打下基础。这里介绍的版本2.7.*，最新版本的3.0丰富了很多特性，可以借鉴官网看下

## dubbo如何使用
通过一个简单的api，来说明dubbo使用
### demo api
```java
public interface HelloService {

    public String hello(String name);

    /**
     * 回调
     * @param name
     * @param key
     * @param callBack
     * @return
     */
    default String hello(String name, String key, HelloServiceCallBack callBack) {
        return null;
    }

    /**
     * 异步调用
     * @param name
     * @return
     */
    default CompletableFuture<String> helloAsync(String name){
        return null;
    }
}

```

<!-- more -->

### demo consumer

```java

@EnableAutoConfiguration
public class DefaultHelloServiceConsumer {


    @DubboReference(version = "default")
    private HelloService helloService;

    public static void main(String[] args) {
        ConfigurableApplicationContext context = SpringApplication.run(DefaultHelloServiceConsumer.class, args);
        HelloService helloService = context.getBean(HelloService.class);
        System.out.println(helloService.hello("jianghe"));

    }
}
```
```yml
# 配置
server:
  port: 8081
spring:
  application:
    name: jianghe-consumer
dubbo:
  registry:
    address: zookeeper://127.0.0.1:2181
```
消费者的配置，我们发现只需要注册中心，然后还有服务的名称，其他都是不需要的。

### demo provider

```java
@DubboService(version = "default")
public class DefaultHelloServiceImpl implements HelloService {

    @Override
    public String hello(String name) {
        return name + " default";
    }
}
```

```yml
# 配置
spring:
  application:
    name: jianghe-provider
dubbo:
  registry:
    address: zookeeper://127.0.0.1:2181
#  protocol:
#    name: dubbo
#    port: 20880
  scan:
    base-packages: com.jianghe.dubbo.provider
  protocols:
    p1:
      name: dubbo
      id: dubbo1
      port: 20881
    p2:
      name: dubbo
      id: dubbo2
      port: 20882
    p3:
      name: dubbo
      id: dubbo3
      port: 20883
server:
  port: 8082
```
提供者的配置，除了注册中心、服务名称，还需要定义支持什么协议protocol，协议之间互相通信的端口，协议是提供者来说明的。
上面的protocols配置使提供者支持的协议有三个，都是dubbo，但是dubbo底层使用的协议端口是不一样的

## 负载均衡

官网地址：https://dubbo.apache.org/zh/docsv2.7/user/examples/loadbalance/

### consumer
```java
@EnableAutoConfiguration
public class LoadBalanceHelloServiceConsumer {

    @DubboReference(version = "loadbalance",loadbalance = "leastactive")
    private HelloService helloService;

    public static void main(String[] args) {
        ConfigurableApplicationContext context = SpringApplication.run(LoadBalanceHelloServiceConsumer.class, args);
        HelloService helloService = context.getBean(HelloService.class);
        for (int i = 0; i <20; i++) {
            System.out.println(helloService.hello("jianghe"));
        }
    }
}
```

### provider
```java

@DubboService(version = "loadbalance")
public class LoadBalanceHelloServiceImpl implements HelloService {

    @Override
    public String hello(String name) {
        RpcContext context = RpcContext.getContext();
        URL url = context.getUrl();

        return name + " loadbalance ,host "+url.getHost()+",port "+url.getPort();
    }
}
```
这里说明下`leastactive`，这个是最少活跃调用数，这里的统计数据是在消费者端统计的。那这里会有个疑问，听到最少活动调用数应该放在提供者里面统计的，这里其实仔细想下，如果提供者来统计，那提供者之前还需要相互通信，是非常繁琐的，如果是消费者统计则不用，是非常方便的。
而对一个消费者调用来说，调用，统计次数+1，如果调用完成，调用次数-1。

## 超时

这里要介绍一下，超时分为服务超时和消费超时，消费是调用的那一块开始，经过网络，然后服务执行，再通过网络返回到消费者端，就是消费时间
消费超时：就是消费时间超过自己设置的时间，就会报出异常。
服务超时：就是服务执行的时间超过自己设置的超时时间，其实是不会抛出异常的，会正常执行完，但是会打印一个警告日志。

### consumer
```java
@EnableAutoConfiguration
public class TimeoutHelloServiceConsumer {


    @DubboReference(version = "timeout",timeout = 10000)
    private HelloService helloService;

    public static void main(String[] args) {
        ConfigurableApplicationContext context = SpringApplication.run(TimeoutHelloServiceConsumer.class, args);
        HelloService helloService = context.getBean(HelloService.class);
        System.out.println(helloService.hello("jianghe"));
    }
}
```

### provider

```java
@DubboService(version = "timeout", timeout = 5000)
public class TimeoutHelloServiceImpl implements HelloService {

    private Logger logger = LoggerFactory.getLogger(TimeoutHelloServiceImpl.class);

    @Override
    public String hello(String name) {
        RpcContext context = RpcContext.getContext();
        URL url = context.getUrl();
        try {
            Thread.sleep(6000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        logger.info(name + " timeout ,host "+url.getHost()+",port "+url.getPort());
        return name + " timeout ,host "+url.getHost()+",port "+url.getPort();
    }
}
```
```
* 服务执行6s
* 服务超时时间设置5s
* 消费者超时时间设置10s
消费者端是可以正常返回的。
```

## 集群容错
<b><font color='red'>集群容错是结合重试次数来使用</font></b>

官网地址：https://dubbo.apache.org/zh/docs/v2.7/user/examples/fault-tolerent-strategy/

### consumer
```java
# retries重试，cluster集群模式
@EnableAutoConfiguration
public class RetriesHelloServiceConsumer {

    @DubboReference(version = "retries",timeout = 3000,retries = 1,cluster = "failfast")
    private HelloService helloService;

    public static void main(String[] args) {
        ConfigurableApplicationContext context = SpringApplication.run(RetriesHelloServiceConsumer.class, args);
        HelloService helloService = context.getBean(HelloService.class);
        System.out.println(helloService.hello("jianghe"));
    }
}
```

### provider
```java

@DubboService(version = "retries")
public class RetriesHelloServiceImpl implements HelloService {

    private Logger logger = LoggerFactory.getLogger(RetriesHelloServiceImpl.class);

    @Override
    public String hello(String name) {
        RpcContext context = RpcContext.getContext();
        URL url = context.getUrl();
        try {
            Thread.sleep(5000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        logger.info(name + " retries ,host "+url.getHost()+",port "+url.getPort());
        return name + " retries ,host "+url.getHost()+",port "+url.getPort();
    }
}
```
## 服务降级

官网地址：https://dubbo.apache.org/zh/docs/v2.7/user/examples/service-downgrade/
其实是采用了`mock`，那mock这里也是本地伪装的使用，直接在本地伪装来介绍

## 本地伪装

官网地址：https://dubbo.apache.org/zh/docs/v2.7/user/examples/local-mock/

### consumer
```java

@EnableAutoConfiguration
public class MockHelloServiceConsumer {

    /**
     * 服务降级
     * mock=force:return+null 表示消费方对该服务的方法调用都直接返回 null 值，不发起远程调用。用来屏蔽不重要服务不可用时对调用方的影响。
     * mock=fail:return+null 表示消费方对该服务的方法调用在失败后，再返回 null 值，不抛异常。用来容忍不重要服务不稳定时对调用方的影响。
     */
//    @DubboReference(version = "mock",mock = "force:return mock")
//    private HelloService helloService;

    /**
     * 本地伪装 通常用于服务降级，比如某验权服务，当服务提供方全部挂掉后，客户端不抛出异常
     * 只有当消费者端抛出异常，则会走mock
     */
    @DubboReference(version = "timeout",timeout = 3000, mock = "com.jianghe.dubbo.api.HelloServiceMock")
    private HelloService helloService;

    public static void main(String[] args) {
        ConfigurableApplicationContext context = SpringApplication.run(MockHelloServiceConsumer.class, args);
        HelloService helloService = context.getBean(HelloService.class);
        System.out.println(helloService.hello("jianghe"));

    }
}
```

### provider
```java
@DubboService(version = "mock")
public class MockHelloServiceImpl implements HelloService {

    @Override
    public String hello(String name) {
        return name + " mock";
    }
}
```

## 本地存根（stub）
本地存根其实也是一种服务降级的策略

官网地址：https://dubbo.apache.org/zh/docs/v2.7/user/examples/local-stub/

### consumer
```java
/**
 * 这里的本地存根
 * stub -> HelloServiceStub
 */
@EnableAutoConfiguration
public class StubHelloServiceConsumer {

    /**
    * stub = "true"，默认的就是接口名+Stub
    * 网关应用，这个是放在消费者端
    */ 
    @DubboReference(version = "timeout",timeout = 3000,stub = "true")
    private HelloService helloService;

    public static void main(String[] args) {
        ConfigurableApplicationContext context = SpringApplication.run(StubHelloServiceConsumer.class, args);
        HelloService helloService = context.getBean(HelloService.class);
        System.out.println(helloService.hello("jianghe"));
    }
}

```

### stub类

```java
/**
* 接口名+Stub 需要实现改接口，然后还有一个接口构造方法
* 这个就是dubbo里面适配器模式
*/ 
public class HelloServiceStub implements HelloService {

    private HelloService helloService;

    public HelloServiceStub(HelloService helloService) {
        this.helloService = helloService;
    }

    @Override
    public String hello(String name) {
        try {
            helloService.hello(name);
        } catch (Exception e) {
            return "HelloServiceStub";
        }
        return null;
    }
}

```
## 回调

官方地址：https://dubbo.apache.org/zh/docs/v2.7/user/examples/callback-parameter/

### consumer
```java
@EnableAutoConfiguration
public class CallBackHelloServiceConsumer {


    @DubboReference(version = "callback")
    private HelloService helloService;
 
    public static void main(String[] args) {
        ConfigurableApplicationContext context = SpringApplication.run(CallBackHelloServiceConsumer.class, args);
        HelloService helloService = context.getBean(HelloService.class);
        CountDownLatch countDownLatch = new CountDownLatch(1);
        new Thread(new Task(countDownLatch, "jianghe", "1", helloService)).start();
        new Thread(new Task(countDownLatch, "jianghe", "2", helloService)).start();
        new Thread(new Task(countDownLatch, "jianghe", "3", helloService)).start();
        countDownLatch.countDown();
    }
    
    /**
    * 并发请求，同时请求没有问题
    */
    public static class Task implements Runnable {

        private CountDownLatch countDownLatch;
        private String name;
        private String key;
        private HelloService helloService;

        public Task(CountDownLatch countDownLatch, String name, String key, HelloService helloService) {
            this.countDownLatch = countDownLatch;
            this.name = name;
            this.key = key;
            this.helloService = helloService;
        }
        @Override
        public void run() {
            try {
                countDownLatch.await();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            helloService.hello(name, key, msg -> System.out.println(msg));
        }
    }
}
```

### provider

```java
/**
* 
* 提供者需要定义出来那个方法是是回调方法
* callbacks是支持1个并发
*/
@DubboService(version = "callback",methods = {@Method(name = "hello", arguments = {@Argument(index = 2, callback = true)})},callbacks = 1)
public class CallBackHelloServiceImpl implements HelloService {

    @Override
    public String hello(String name) {
        return name + " callback";
    }

    @Override
    public String hello(String name, String key, HelloServiceCallBack callBack) {
        callBack.changed(key);
        return name + " callback";
    }
}
```
这里说明下，provider里的HelloServiceCallBack是consumer里的`msg -> System.out.println(msg)`吗？答案肯定不是的，两个jvm，没办法直接传的，provider接受到是HelloServiceCallBack代理对象，其中HelloServiceCallBack也是接口

### CallBack接口

```java
public interface HelloServiceCallBack {

    public void changed(String msg);

}
```

## 异步调用
一般业务dubbo调用都是同步调用，必须等待提供者返回才执行其他业务，dubbo也可以支持异步的方式。异步调用方式多种多样，下面介绍相对于开发同学方便的方式来说明。接口中返回Completablefuture，其他方式可自行实践

官方地址：https://dubbo.apache.org/zh/docs/v2.7/user/examples/async-call/

### consumer
```java
@EnableAutoConfiguration
public class CompletableFutureHelloServiceConsumer {

    @DubboReference(version = "completablefuture",timeout = 5000)
    private HelloService helloService;

    public static void main(String[] args) {
        ConfigurableApplicationContext context = SpringApplication.run(CompletableFutureHelloServiceConsumer.class, args);
        HelloService helloService = context.getBean(HelloService.class);
        CompletableFuture<String> completableFuture = helloService.helloAsync("jianghe");

        completableFuture.whenComplete((v,t) ->{
            if (t != null) {
                t.printStackTrace();
            } else {
                System.out.println("Response: " + v);
            }
        });
        System.out.println("1");
    }
}
```

### provider

```java
@DubboService(version = "completablefuture")
public class CompletableFutureHelloServiceImpl implements HelloService {

    public String hello(String name) {
        return name + " completableFuture";
    }

    @Override
    public CompletableFuture<String> helloAsync(String name) {
        System.out.println("completableFuture");
        return CompletableFuture.supplyAsync(() -> hello(name));
    }
}
```

## 泛化（generic）

官方地址：https://dubbo.apache.org/zh/docs/v2.7/user/examples/generic-reference/

1. 泛化调用
    * 泛化调用好处就是不依赖api版本，只要连接到zk地址，就能调用到服务
    * 可以通过泛化来实现一个平台，来解决服务治理，可用、降级、熔断等（目前所在公司营销网关平台就是基于dubbo泛化来做的）
2. 泛化服务
    * 同样可以包装成一个dubbo服务

### consumer
```java
/**
 * 泛化调用
 * 好处：不依赖api版本，直接调用
 */
@EnableAutoConfiguration
public class GenericHelloServiceConsumer {

    /**
    * generic需要设置成true
    */
    @DubboReference(version = "generic", interfaceName = "com.jianghe.dubbo.api.HelloService", generic = true)
    private GenericService genericService;

    public static void main(String[] args) {
        ConfigurableApplicationContext context = SpringApplication.run(GenericHelloServiceConsumer.class, args);
        GenericService genericService = context.getBean(GenericService.class);
        String result = (String) genericService.$invoke("hello", new String[]{"java.lang.String"}, new Object[]{"jianghe"});
        System.out.println(result);
    }
}
```

### provider

```java
@DubboService(version = "generic")
public class GenericHelloServiceImpl implements HelloService {

    @Override
    public String hello(String name) {
        return name + " generic";
    }
}
```

## REST
dubbo也是rest协议，可以直接通过http来访问。依个人而言，dubbo支持rest，更加丰富dubbo协议，但是rest风格springmvc已经很完善，dubbo的rest应用不是很高，这里就不介绍了。具体可以看官网

官方地址：https://dubbo.apache.org/zh/docs/v2.7/user/rest/



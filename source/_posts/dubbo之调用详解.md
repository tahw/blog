---
title: dubbo之调用详解
date: 2022-02-11 17:08:33
tags:
    - dubbo
categories:
    - dubbo
---

# dubbo之调用详解
前言：分析的dubbo源码是2.7.7版本

# 入口
代理类方法调用，走的是InvokerInvocationHandler.invoke方法
```java
/**
 * InvokerHandler
 */
public class InvokerInvocationHandler implements InvocationHandler {
    private static final Logger logger = LoggerFactory.getLogger(InvokerInvocationHandler.class);
    private final Invoker<?> invoker;
    private ConsumerModel consumerModel;

    public InvokerInvocationHandler(Invoker<?> handler) {
        this.invoker = handler;
        String serviceKey = invoker.getUrl().getServiceKey();
        if (serviceKey != null) {
            this.consumerModel = ApplicationModel.getConsumerModel(serviceKey);
        }
    }

    /**
     * 这里的Invoker就是
     * MockClusterInvoker ->
     *   AbstractCluster$Interceptor
     *     FailoverClusterInvoker ->
     *        RegistryDirectory ->
     *          路由
     *          List<RegistryDirectory$InvokerDelegate>
     *          负载
     *               RegistryDirectory$InvokerDelegate
     *                  ListenerInvokerWrapper ->
     *                     Invoker$0(ConsumerContextFilter)
     *                     Invoker$1(FutureFilter)
     *                     Invoker$2(MonitorFilter) ->
     *                         AsyncToSyncInvoker ->
     *                          DubboInvoker
     * @param proxy
     * @param method
     * @param args
     * @return
     * @throws Throwable
     */
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        if (method.getDeclaringClass() == Object.class) {
            return method.invoke(invoker, args);
        }
        String methodName = method.getName();
        Class<?>[] parameterTypes = method.getParameterTypes();
        if (parameterTypes.length == 0) {
            if ("toString".equals(methodName)) {
                return invoker.toString();
            } else if ("$destroy".equals(methodName)) {
                invoker.destroy();
                return null;
            } else if ("hashCode".equals(methodName)) {
                return invoker.hashCode();
            }
        } else if (parameterTypes.length == 1 && "equals".equals(methodName)) {
            return invoker.equals(args[0]);
        }
        RpcInvocation rpcInvocation = new RpcInvocation(method, invoker.getInterface().getName(), args);
        String serviceKey = invoker.getUrl().getServiceKey();
        rpcInvocation.setTargetServiceUniqueName(serviceKey);
      
        if (consumerModel != null) {
            rpcInvocation.put(Constants.CONSUMER_MODEL, consumerModel);
            rpcInvocation.put(Constants.METHOD_MODEL, consumerModel.getMethodModel(method));
        }

        return invoker.invoke(rpcInvocation).recreate(); // 构造成RpcInvocation
    }
}
```
<!-- more -->

# 服务调用Invoker链
```java
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
                                    DubboInvoker（调用client） -> 
                                        ReferenceCountExchangeClient
                                            -> HeaderExchangeClinet
                                                -> NettyClient
```

# 服务调用响应handler链
```java
NettyClientHandler -> 
    NettyClient -> 
        MultiMessageHandler -> 
            HeartbeatHandler -> 
                AllChannelHandler -> 
                    DecodeHandler -> 
                        HeaderExchangeHandler
                            -> DefaultFuture
```

# 服务暴露Handler链
```java
NettyServerHandler->
    NettyServer->
        MultiMessageHandler->
            HeartbeatHandler->
                AllChannelHandler->
                    DecodeHandler->
                        HeaderExchangeHandler->
                            ExchangeHandlerAdapter
                                (调用真正的Invoker.invoke)
                                ProtocolFilterWrapper$Invoker(EchoFilter -> ClassLoaderFilter -> GenericFilter -> ContextFilter -> TraceFilter -> TimeoutFilter -> MonitorFilter -> ExceptionFilter)->
                                    RegistryProtocol$InvokerDelegate->
                                        DelegateProviderMetaDataInvoker->
                                            AbstractProxyInvoker ->
                                                执行真正的业务方法
```

# 整体流程图
![整体流程图](/images/dubbo-7-1.png)
<b>ps：流程图很重要，建议下载仔细看</b>

# 服务消费端执行逻辑

1. InvokerInvocationHandler判断如果是对象内部方法toString、$destroy、hashCode，直接处理返回，否则构造RpcInvocation对象，调用下一层
2. MockClusterInvoker：mock逻辑
3. AbstractCluster$Interceptor处理逻辑是AbstractClusterInvoker，这个类是把RpcContext中设置的attachments放入到RpcInvocation，调用服务目录路由链列举Invokers，根据Invokers获取负载均衡策略（多个Invokers选用第一个）
4. FailoverClusterInvoker.doInvoke()根据负载策略选出一个Invoker，然后执行，这里会有容错的逻辑，重试，默认执行3次，重试两次
5. RegistryDirectory$InvokerDelegate的父类InvokerWrapper调用invoke，这里没做什么
6. 开始执行Filter链，当Filter执行完返回结果后，就会执行Result#whenCompleteWithContext
7. ConsumerContextFilter.invoke，设置RpcContext中LocalAddress、RemoteAddress、RemoteApplicationName、remote.application属性
8. FutureFilter.invoke
9. MonitorFilter.invoke，设置monitor_filter_start_time属性，方法的执行次数+1
10. ListenerInvokerWrapper.invoke没做什么
11. <font color='red'><b>AsyncToSyncInvoker.invoke，异步转同步，先调用下层Invoker异步执行，如果是同步方法就会阻塞Integer.MAX_VALUE毫秒获取结果</b></font>
12. AbstractInvoker.invoke：先调用DubboInvoker.doInvoke方法，如果出现异常，转换为AsyncRpcResult返回
13. DubboInvoker.doInvoke，从clients选择一个client进行发生数据，如果不用返回结果，则调用client.send，如果需要返回结果，则调用client.request，得到结果后，封装成AsyncRpcResult对象返回结果
14. 这里调用的ReferenceCountExchangeClient.request，没有做什么，调用下一层
15. HeaderExchangeClient.request，这里没有做什么，调用下一层
16. <font color='red'><b>HeaderExchangeChannel.request，这里构建Request，并且会构造一个DefaultFuture对象来返回，其中DefaultFuture会阻塞timeout（用户配置，默认1s）的时间来等待结果，DefaultFuture也是实现CompletableFuture，在构造DefaultFuture的时候，会把requestId和DefaultFuture放进去FUTURES map里，当提供者返回结果，就会调用HeaderExchangeHandler.handleResponse的方法，这时候会调用DefaultFuture.received方法，根据FUTURES map来获取DefaultFuture，然后执行DefaultFuture.doReceived方法，返回Response</b></font>
17. AbstractPeer.send方法，没有做什么
18. AbstractClient.send
19. 调用NettyChannel.send方法，调用NioSocketChannel.writeAndFlush方法，最底层是Netty非阻塞式发送数据

## 消费端异常处理
我们知道服务提供者如果出现异常，其实是包装成AppResponse正常信息返回，那消费者需要抛出这个异常信息，那这块是在哪里做的？在InvokerInvocationHandler.invoke返回值的recreate方法，这个最终会调用到AppResponse#recreate，里面包含异常信息就throw抛出
```java
public class InvokerInvocationHandler implements InvocationHandler {
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        // ...
        // AsyncRpcResult.recreate -> AppResponse.recreate
        return invoker.invoke(rpcInvocation).recreate();
    }

}
```
```java
public class AppResponse implements Result {
    @Override
    public Object recreate() throws Throwable {
        if (exception != null) {
            // fix issue#619
            try {
                // get Throwable class
                Class clazz = exception.getClass();
                while (!clazz.getName().equals(Throwable.class.getName())) {
                    clazz = clazz.getSuperclass();
                }
                // get stackTrace value
                Field stackTraceField = clazz.getDeclaredField("stackTrace");
                stackTraceField.setAccessible(true);
                Object stackTrace = stackTraceField.get(exception);
                if (stackTrace == null) {
                    exception.setStackTrace(new StackTraceElement[0]);
                }
            } catch (Exception e) {
                // ignore
            }
            throw exception;
        }
        return result;
    }
}
```
ps:这里在说明一点，是什么样的错误dubbo才会容错呢？如果是业务异常，正常抛出，也不会mock，也不会容错，如果是系统的异常RpcException（超时异常、系统异常）则会mock容错


# 服务提供者执行逻辑

1. NettyServerHandler.channelRead读取数据，交给下一层
2. AbstractPeer.received没有做什么，直接调用下一层
3. MultiMessageHandler.received，判断msg是MultiMessageHandler，如果是的话获取单个message，调用下一层
4. HeartbeatHandler.received，判断msg是心跳请求，是的话返回response，如果是心跳响应，不做什么，最后则调用下一层
5. AllChannelHandler.received，把msg构建成ChannelEventRunnable，放到线程池里面，然后在ChannelEventRunnable里的run方法调用下一层
6. DecodeHandler.received，反编译，设置RpcInvocation methodName、parameterTypes、arguments、attachments等属性，然后调用下一层
7. <font color='red'><b>HeaderExchangeHandler.handleRequest方法，先构建Response对象，调用下层的ExchangeHandlerAdapter.reply方法返回CompletionStage对象，CompletionStage是CompletableFuture父类，然后CompletionStage.whenComplete方法绑定回调函数，当ExchangeHandlerAdapter.reply返回结果，就会设置Response对象，然后返回到消费者</b></font>
8. ExchangeHandlerAdapter.reply，从exporter获取Invoker，然后执行Invoker.invoke
9. 这里的就会执行ProtocolFilterWrapper$Invoker的Filter链，当Filter执行完返回结果后，就会执行Result#whenCompleteWithContext
10. EchoFilter.invoke，判断当前请求是不是一个回声测试，如果是，则不后续调用了，否则调用下一层
11. ClassLoaderFilter.invoke，设置设置当前线程类加载器为执行的服务接口的类加载器，调用下一层
12. GenericFilter.invoke，把泛化信息包装成RpcInvocation对象，调用下一层
13. ContextFilter.invoke，设置RpcContext里面context属性，调用下一层
14. TraceFilter.invoke，先执行下一层，然后记录信息
15. TimeoutFilter.invoke，设置invocation的timeout_filter_start_time值，调用下一层，但是当后面内容调用完成后，会回调TimeoutFilter.onResponse方法，里面有验证超时，但是只是打印了warn日志
16. MonitorFilter.invoke，设置monitor_filter_start_time属性，方法的执行次数+1
17. ExceptionFilter.invoke，没有做什么，直接调用下一层，但是后面执行完成，会回调ExceptionFilter.onResponse方法，对不同的异常做处理
18. InvokerWrapper.invoke 没有做什么，调用下一层
19. AbstractProxyInvoker.invoke，没有做什么，直接调用下一层
20. <font color='red'><b>AbstractProxyInvoker.invoke，先调用JavassistProxyFactory.getInvoker里面的wrapper.invokeMethod方法，调用真正的方法返回，包装成CompletableFuture，这里然后执行方法CompletableFuture.handle，等待CompletableFuture执行完再包装成AsyncRpcResult返回</b></font>

## 服务端异常处理
ExceptionFilter是服务端过滤器，当结果返回后，会执行onResponse方法，此方法来判断异常信息
1. 异常信息是Exception，不用做处理
2. 接口信息中定义异常信息，不用做处理
3. 如果异常和接口方法的jar是一个，不用做处理
4. 异常类名以java打头或者javax打头的，不用做处理
5. 异常如果是RpcException，不用做处理
6. 其他识别不了的直接转换成RuntimeException

```java
/*
 * Licensed to the Apache Software Foundation (ASF) under one or more
 * contributor license agreements.  See the NOTICE file distributed with
 * this work for additional information regarding copyright ownership.
 * The ASF licenses this file to You under the Apache License, Version 2.0
 * (the "License"); you may not use this file except in compliance with
 * the License.  You may obtain a copy of the License at
 *
 *     http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */
package org.apache.dubbo.rpc.filter;

import org.apache.dubbo.common.constants.CommonConstants;
import org.apache.dubbo.common.extension.Activate;
import org.apache.dubbo.common.logger.Logger;
import org.apache.dubbo.common.logger.LoggerFactory;
import org.apache.dubbo.common.utils.ReflectUtils;
import org.apache.dubbo.common.utils.StringUtils;
import org.apache.dubbo.rpc.Filter;
import org.apache.dubbo.rpc.Invocation;
import org.apache.dubbo.rpc.Invoker;
import org.apache.dubbo.rpc.Result;
import org.apache.dubbo.rpc.RpcContext;
import org.apache.dubbo.rpc.RpcException;
import org.apache.dubbo.rpc.service.GenericService;

import java.lang.reflect.Method;


/**
 * ExceptionInvokerFilter
 * <p>
 * Functions:
 * <ol>
 * <li>unexpected exception will be logged in ERROR level on provider side. Unexpected exception are unchecked
 * exception not declared on the interface</li>
 * <li>Wrap the exception not introduced in API package into RuntimeException. Framework will serialize the outer exception but stringnize its cause in order to avoid of possible serialization problem on client side</li>
 * </ol>
 */
@Activate(group = CommonConstants.PROVIDER)
public class ExceptionFilter implements Filter, Filter.Listener {
    private Logger logger = LoggerFactory.getLogger(ExceptionFilter.class);

    @Override
    public Result invoke(Invoker<?> invoker, Invocation invocation) throws RpcException {
        return invoker.invoke(invocation);
    }

    /**
     * 异常处理
     * @param appResponse
     * @param invoker
     * @param invocation
     */
    @Override
    public void onResponse(Result appResponse, Invoker<?> invoker, Invocation invocation) {
        if (appResponse.hasException() && GenericService.class != invoker.getInterface()) { // 存在异常
            try {
                Throwable exception = appResponse.getException();

                // directly throw if it's checked exception
                if (!(exception instanceof RuntimeException) && (exception instanceof Exception)) { // java可以识别异常，可以直接返回
                    return;
                }
                // directly throw if the exception appears in the signature
                try {
                    Method method = invoker.getInterface().getMethod(invocation.getMethodName(), invocation.getParameterTypes());
                    Class<?>[] exceptionClassses = method.getExceptionTypes();
                    for (Class<?> exceptionClass : exceptionClassses) {
                        if (exception.getClass().equals(exceptionClass)) { // 接口方法定义异常信息直接返回
                            return;
                        }
                    }
                } catch (NoSuchMethodException e) {
                    return;
                }

                // for the exception not found in method's signature, print ERROR message in server's log.
                logger.error("Got unchecked and undeclared exception which called by " + RpcContext.getContext().getRemoteHost() + ". service: " + invoker.getInterface().getName() + ", method: " + invocation.getMethodName() + ", exception: " + exception.getClass().getName() + ": " + exception.getMessage(), exception);

                // directly throw if exception class and interface class are in the same jar file.
                String serviceFile = ReflectUtils.getCodeBase(invoker.getInterface()); // 接口方法的文件路径
                String exceptionFile = ReflectUtils.getCodeBase(exception.getClass()); // 异常的文件路径
                if (serviceFile == null || exceptionFile == null || serviceFile.equals(exceptionFile)) { // 异常的文件路径和接口方法路径相同
                    return;
                }
                // directly throw if it's JDK exception
                String className = exception.getClass().getName();
                if (className.startsWith("java.") || className.startsWith("javax.")) { // jdk自带的
                    return;
                }
                // directly throw if it's dubbo exception
                if (exception instanceof RpcException) { // dubbo自定义的异常
                    return;
                }

                // otherwise, wrap with RuntimeException and throw back to the client
                appResponse.setException(new RuntimeException(StringUtils.toString(exception))); // 识别不了的异常转换成RuntimeException异常处理
            } catch (Throwable e) {
                logger.warn("Fail to ExceptionFilter when called by " + RpcContext.getContext().getRemoteHost() + ". service: " + invoker.getInterface().getName() + ", method: " + invocation.getMethodName() + ", exception: " + e.getClass().getName() + ": " + e.getMessage(), e);
            }
        }
    }

    @Override
    public void onError(Throwable e, Invoker<?> invoker, Invocation invocation) {
        logger.error("Got unchecked and undeclared exception which called by " + RpcContext.getContext().getRemoteHost() + ". service: " + invoker.getInterface().getName() + ", method: " + invocation.getMethodName() + ", exception: " + e.getClass().getName() + ": " + e.getMessage(), e);
    }

    // For test purpose
    public void setLogger(Logger logger) {
        this.logger = logger;
    }
}
```

# 线程模型
![线程模型](/images/dubbo-7-2.png)
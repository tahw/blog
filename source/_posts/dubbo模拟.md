---
title: dubbo模拟
date: 2021-11-15 17:33:08
tags:
    - dubbo
categories:
    - dubbo
---

# dubbo介绍
&nbsp;&nbsp;&nbsp;&nbsp;什么是dubbo？dubbo官网介绍了，Apache Dubbo 是一款高性能、轻量级的开源服务框架，但是英文版的还是一个rpc框架`Apache Dubbo is a high-performance, java based open source RPC framework.`
>https://dubbo.apache.org/zh/

# 什么是rpc
&nbsp;&nbsp;&nbsp;&nbsp;rpc，就是远程方法调用，经过网络远程调用，就是rpc。经过网络调用就会出现
> rpc over http： rpc基于http调用
> rpc over tcp：rpc基于tcp调用

<!--more -->

根据接口、协议、参数就会产生不同的框架，而dubbo就是组合支持这些，然后形成框架。而现在的dubbo已经不仅仅做rpc 框架，正在做服务。

![rpc](/images/pasted-100.png)

# 模拟dubbo
&nbsp;&nbsp;&nbsp;&nbsp;为什么要模拟dubbo呢？其实是为读dubbo源码做准备，那当我们去模拟dubbo的时候，就会设身处地的思考dubbo开发者出现的问题和思路，也会更加理解dubbo源码

## 分析
![dubbo调用](/images/pasted-101.png)
1. 服务提供者
    1. 提供服务
    2. 服务监听
2. 服务消费者
    1. 消费服务
    2. 提供者地址
    3. 负载、熔断、降级、mock
3. 注册中心
    1. 提供者地址端口
4. 网络
5. 监控中心
    1. 监控


### dubbo api
接口
```java
public interface HelloService {

    public String say(String name);

}
```

`Invocation`就是dubbo底层调用传参的实体，非常重要的存在，这里因为是dubbo provider和dubbo consumer共享的，就放在这里。下面会介绍。
```java
/**
 * 反射基本信息
 */
@Data
@Accessors(chain = true)
public class Invocation implements Serializable {

    // 接口类名
    private String className;

    // 方法名
    private String methodName;

    // 方法参数类型
    private Class[] paramTypes;

    // 参数
    private Object[] params;

    // 版本
    private String  version;
}
```

### dubbo provider
#### 接口实现
```java
public class HelloServiceImpl implements HelloService {

    @Override
    public String say(String name) {
        return name+" , rpc is ok";
    }
}
```
#### 本地注册
```java
/**
 * 本地注册
 */
public class LocalRegister {

    private static Map<String,Class> map = new HashMap<>();

    public static void register(String interfaceName,Class clazz){
        map.put(interfaceName,clazz);
    }

    public static Class get(String interfaceName){
        return map.get(interfaceName);
    }
}
```
<font color='red'><b>为什么需要本地注册？</b></font>，其实答案需要思考下，可以思考下，dubbo consumer调用接口，然后传的就是个接口，那当dubbo provider接受服务的时候，参数就是个接口，那dubbo provider就会根据接口找到对应的实现类，而这里可以延伸下，如果是在spring环境下，根据接口来获取spring bean直接使用，那谁放进去spring context里呢？就是dubbo放的，而放的应该是接口实现类的代理类。

#### 启动服务
```java
Url url = new Url();
url.setHostName("127.0.0.1");
url.setPort(8080);
// 远程注册中心，后面会介绍
RemoteRegister.register(HelloService.class.getName(),url);

// 本地注册中心
LocalRegister.register(HelloService.class.getName(), HelloServiceImpl.class);

ServerProtocol protocol = ServerProtocolFactory.getProtocol();
protocol.start(url.getHostName(),url.getPort());
```
工厂模式，根据系统参数来使用什么协议来启动服务，那这里也说明下，`over http`可以使用`httpClient`来调用，`over tcp`可以使用`netty`来调用，其实dubbo是采用spi的方式来实现，这里简化直接使用-D配置来解决
```java
/**
 * 工厂模式
 */
public class ServerProtocolFactory {

    /**
     * 获取Protocol 协议
     * @return
     */
    public static ServerProtocol getProtocol(){
        String protocol = System.getProperty("protocol");
        if(StringUtils.isEmpty(protocol)){
            return new HttpServer();
        }
        switch (protocol){
            case "http":
                return new HttpServer();
            case "dubbo":
                return new NettyServer();
            default:

        }
        return new HttpServer();
    }
}
```
##### http服务
使用http来实现，可以使用spring boot内嵌的tomcat
```java
public class HttpServer implements ServerProtocol {

    /**
     * 启动服务
     * @param hostName
     * @param port
     */
    public void start(String hostName,Integer port){
        Tomcat tomcat = new Tomcat();

        Server server = tomcat.getServer();
        Service service = server.findService("Tomcat");

        Connector connector = new Connector();
        connector.setPort(port);

        Engine engine = new StandardEngine();
        engine.setDefaultHost(hostName);

        Host host = new StandardHost();
        host.setName(hostName);

        String contextPath = "";
        Context context = new StandardContext();
        context.setPath(contextPath);
        context.addLifecycleListener(new Tomcat.FixContextListener());

        host.addChild(context);
        engine.addChild(host);

        service.setContainer(engine);
        service.addConnector(connector);

        tomcat.addServlet(contextPath, "dispatcher", new HttpServerDispatcherServlet());
        context.addServletMappingDecoded("/*", "dispatcher");

        try {
            tomcat.start();
            tomcat.getServer().await();
        } catch (LifecycleException e) {
            e.printStackTrace();
        }
    }
}
```
```java
public class HttpServerDispatcherServlet extends HttpServlet {

    @Override
    protected void service(HttpServletRequest req, HttpServletResponse resp) {
        new HttpServerHandler().handler(req,resp);
    }
}
```
真正处理的逻辑，就是handler
```java
/**
 * 处理器
 */
public class HttpServerHandler {

    public void handler(HttpServletRequest req, HttpServletResponse resp)  {
        InputStream is = null;
        OutputStream os = null;
        BufferedReader br = null;
        String result = null;
        ObjectInputStream ois = null;
        try {
            is = req.getInputStream();
            // 对输入流对象进行包装:charset根据工作项目组的要求来设置
            br = new BufferedReader(new InputStreamReader(is, "UTF-8"));
            StringBuffer sbf = new StringBuffer();
            String temp = null;
            // 循环遍历一行一行读取数据
            while ((temp = br.readLine()) != null) {
                sbf.append(temp);
                sbf.append("\r\n");
            }
            result = sbf.toString();
            // 1. 接口结果返回Invocation
            Invocation invocation = JSONObject.parseObject(result,Invocation.class);
            // 2. 本地注册器，根据接口名来获取实现类
            Class clazz = LocalRegister.get(invocation.getClassName()); 
            // 3. 根据类接口，接口参数类型，获取Method
            Method method = clazz.getMethod(invocation.getMethodName(), invocation.getParamTypes());
            // 4. 反射调用，就能调用到接口，返回数据
            String respResult = (String) method.invoke(clazz.newInstance(), invocation.getParams());
            // 5. 返回给调用者
            resp.getOutputStream().write(respResult.getBytes(StandardCharsets.UTF_8)); // 输出
        } catch (IOException e) {
            e.printStackTrace();
        } catch (NoSuchMethodException e) {
            e.printStackTrace();
        } catch (InvocationTargetException e) {
            e.printStackTrace();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        } catch (InstantiationException e) {
            e.printStackTrace();
        }
    }
}
```
##### tcp服务
```java
public class NettyServer implements ServerProtocol {

    /**
     * 启动服务
     * @param hostName
     * @param port
     */
    public void start(String hostName,Integer port){
        try {
            final ServerBootstrap bootstrap = new ServerBootstrap();
            NioEventLoopGroup eventLoopGroup = new NioEventLoopGroup();
            bootstrap.group(eventLoopGroup)
                    .channel(NioServerSocketChannel.class)
                    .childHandler(new ChannelInitializer<SocketChannel>() {

                        protected void initChannel(SocketChannel socketChannel) throws Exception {
                            ChannelPipeline pipeline = socketChannel.pipeline();
                            pipeline.addLast("decoder", new ObjectDecoder(ClassResolvers
                                    .weakCachingConcurrentResolver(this.getClass()
                                            .getClassLoader())));
                            pipeline.addLast("encoder", new ObjectEncoder());
                            pipeline.addLast("handler", new NettyServerHandler());
                        }
                    });
            bootstrap.bind(hostName, port).sync();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```
```java
public class NettyServerHandler extends ChannelInboundHandlerAdapter {

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        // 1. 接口结果返回Invocation
        Invocation invocation = (Invocation) msg;
        // 2. 本地注册器，根据接口名来获取实现类
        Class clazz = LocalRegister.get(invocation.getClassName());
        // 3. 根据类接口，接口参数类型，获取Method
        Method method = clazz.getMethod(invocation.getMethodName(), invocation.getParamTypes());
        // 4. 反射调用，就能调用到接口，返回数据
        String result = (String) method.invoke(clazz.newInstance(), invocation.getParams());
        // 5. 返回给调用者
        ctx.writeAndFlush("Netty: " + result);
    }
}
```
### dubbo consumer

#### 启动服务
<font color='red'><b>这里要注意一下，JdkProxy</b></font>，那这个为什么要注意一下，作为消费者，直接使用这个接口，传参然后调用接口方法，这样无痕的调用方法，然后就直接能调到dubbo provider，然后就直接返回数据。那这里的接口肯定是接口代理类，这里就是使用jdk 动态代理来实现。

```java
//创建httpclient对象
HelloService helloService = JdkProxy.getProxy(HelloService.class);
String result = helloService.say("jianghe");
System.out.println(result);
```

#### 接口代理

jdk动态代理`Proxy.newProxyInstance`，然后里面可以通过传入的interfaceClass，调用的方法Method，可以直接构建出`Invocation`对象，然后通过远程注册器可以获取到远程服务端的地址

```java
/**
 * 接口代理
 * jdk proxy
 */
public class JdkProxy {

    @SuppressWarnings("unchecked")
    public static <T> T getProxy(final Class interfaceClass){
        return (T) Proxy.newProxyInstance(interfaceClass.getClassLoader(), new Class[]{interfaceClass}, new InvocationHandler() {
            @Override
            public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                try {
                    // mock方法
                    String mock = System.getProperty("mock");
                    if(StringUtils.isNotEmpty(mock)){ // mock 方法
                        return "mock";
                    }
                    Invocation invocation = new Invocation();
                    invocation.setClassName(interfaceClass.getName());
                    invocation.setMethodName(method.getName());
                    invocation.setParamTypes(method.getParameterTypes());
                    invocation.setParams(args);
                    // 客户端协议，工厂模式获取
                    ClientProtocol protocol = ClientProtocolFactory.getProtocol();
                    // 这里有远程方法注册器，这里就是当注册中心使用了
                    List<Url> urls = RemoteRegister.get(interfaceClass.getName());
                    // 负载均衡
                    Url url = Balance.getUrl(urls); // load balance
                    String result = protocol.send(url.getHostName(), url.getPort(), invocation);
                    return result;
                } catch (Exception e){ // 容错方法
                    // 容错
                    return "容错方法";
                }
            }
        });
    }
}
```


```java
public interface ClientProtocol {

    public String send(String hostName, Integer port, Invocation invocation);

}

/**
 * 工厂模式
 */
public class ClientProtocolFactory {
    
    /**
     * 获取Protocol 协议
     * @return
     */
    public static ClientProtocol getProtocol(){
        String protocol = System.getProperty("protocol");
        if(StringUtils.isEmpty(protocol)){
            return new HttpClient();
        }
        switch (protocol){
            case "http":
                return new HttpClient();
            case "dubbo":
                return new NettyClient();
            default:
        }
        return new HttpClient();
    }
}

```

#### Http接口代理

```java
public class HttpClient implements ClientProtocol {

    /**
     * http://localhost:8080/
     * @param hostName
     * @param port
     * @param invocation
     * @return
     */
    public String send(String hostName, Integer port, Invocation invocation){
        return HttpClientUtil.doPost("http://"+hostName+":"+port+"/", JSON.toJSONString(invocation));
    }
}

```
```java
/**
 * http util
 */
public class HttpClientUtil {

    public static String doGet(String httpurl) {
        HttpURLConnection connection = null;
        InputStream is = null;
        BufferedReader br = null;
        String result = null;// 返回结果字符串
        try {
            // 创建远程url连接对象
            URL url = new URL(httpurl);
            // 通过远程url连接对象打开一个连接，强转成httpURLConnection类
            connection = (HttpURLConnection) url.openConnection();
            // 设置连接方式：get
            connection.setRequestMethod("GET");
            // 设置连接主机服务器的超时时间：15000毫秒
            connection.setConnectTimeout(15000);
            // 设置读取远程返回的数据时间：60000毫秒
            connection.setReadTimeout(60000);
            // 发送请求
            connection.connect();
            // 通过connection连接，获取输入流
            if (connection.getResponseCode() == 200) {
                is = connection.getInputStream();
                // 封装输入流is，并指定字符集
                br = new BufferedReader(new InputStreamReader(is, "UTF-8"));
                // 存放数据
                StringBuffer sbf = new StringBuffer();
                String temp = null;
                while ((temp = br.readLine()) != null) {
                    sbf.append(temp);
                    sbf.append("\r\n");
                }
                result = sbf.toString();
            }
        } catch (MalformedURLException e) {
            e.printStackTrace();
        } catch (IOException e) {
            e.printStackTrace();
        } finally {
            // 关闭资源
            if (null != br) {
                try {
                    br.close();
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
            if (null != is) {
                try {
                    is.close();
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
            connection.disconnect();// 关闭远程连接
        }
        return result;
    }
    public static String doPost(String httpUrl, String param) {
        HttpURLConnection connection = null;
        InputStream is = null;
        OutputStream os = null;
        BufferedReader br = null;
        String result = null;
        try {
            URL url = new URL(httpUrl);
            // 通过远程url连接对象打开连接
            connection = (HttpURLConnection) url.openConnection();
            // 设置连接请求方式
            connection.setRequestMethod("POST");
            // 设置连接主机服务器超时时间：15000毫秒
            connection.setConnectTimeout(15000);
            // 设置读取主机服务器返回数据超时时间：60000毫秒
            connection.setReadTimeout(60000);
            // 默认值为：false，当向远程服务器传送数据/写数据时，需要设置为true
            connection.setDoOutput(true);
            // 默认值为：true，当前向远程服务读取数据时，设置为true，该参数可有可无
            connection.setDoInput(true);
            // 设置传入参数的格式:请求参数应该是 name1=value1&name2=value2 的形式。
            connection.setRequestProperty("Content-Type", "application/json");
            // 通过连接对象获取一个输出流
            os = connection.getOutputStream();
            // 通过输出流对象将参数写出去/传输出去,它是通过字节数组写出的
            os.write(param.getBytes());
            // 通过连接对象获取一个输入流，向远程读取
            if (connection.getResponseCode() == 200) {
                is = connection.getInputStream();
                // 对输入流对象进行包装:charset根据工作项目组的要求来设置
                br = new BufferedReader(new InputStreamReader(is, "UTF-8"));
                StringBuffer sbf = new StringBuffer();
                String temp = null;
                // 循环遍历一行一行读取数据
                while ((temp = br.readLine()) != null) {
                    sbf.append(temp);
                    sbf.append("\r\n");
                }
                result = sbf.toString();
            }
        } catch (MalformedURLException e) {
            e.printStackTrace();
        } catch (IOException e) {
            e.printStackTrace();
        } finally {
            // 关闭资源
            if (null != br) {
                try {
                    br.close();
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
            if (null != os) {
                try {
                    os.close();
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
            if (null != is) {
                try {
                    is.close();
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
            // 断开与远程地址url的连接
            connection.disconnect();
        }
        return result;
    }
}
```


#### tcp接口代理

```java
public class NettyClient implements ClientProtocol {

    private static NettyClientHandler nettyClientHandler;

    private static ThreadPoolExecutor executor = new ThreadPoolExecutor(5,10,5, TimeUnit.SECONDS,new ArrayBlockingQueue<>(20), new ThreadPoolExecutor.CallerRunsPolicy());

    public String send(String hostName, Integer port, Invocation invocation){
        if(nettyClientHandler == null){
            start(hostName,port);
        }
        nettyClientHandler.setInvocation(invocation);
        Future future = executor.submit(nettyClientHandler);
        try {
            return (String) future.get();
        } catch (InterruptedException e) {
            e.printStackTrace();
        } catch (ExecutionException e) {
            e.printStackTrace();
        }
        return null;
    }

    private static void start(String hostName, Integer port){
        nettyClientHandler = new NettyClientHandler();
        Bootstrap b = new Bootstrap();
        EventLoopGroup group = new NioEventLoopGroup();
        b.group(group)
                .channel(NioSocketChannel.class)
                .option(ChannelOption.TCP_NODELAY, true)
                .handler(new ChannelInitializer<SocketChannel>() {
                    protected void initChannel(SocketChannel socketChannel) throws Exception {
                        ChannelPipeline pipeline = socketChannel.pipeline();
                        pipeline.addLast("decoder", new ObjectDecoder(ClassResolvers
                                .weakCachingConcurrentResolver(this.getClass()
                                        .getClassLoader())));
                        pipeline.addLast("encoder", new ObjectEncoder());
                        pipeline.addLast("handler", nettyClientHandler);
                    }
                });

        try {
            b.connect(hostName, port).sync();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}

public class NettyClientHandler extends ChannelInboundHandlerAdapter implements Callable<String> {

    private Object object = new Object();

    private Invocation invocation;

    private ChannelHandlerContext context;

    private String result;

    /**
     * 连接成功
     * @param ctx
     * @throws Exception
     */
    @Override
    public void channelActive(ChannelHandlerContext ctx) throws Exception {
        context = ctx;
    }

    /**
     * 返回数据
     * @param ctx
     * @param msg
     * @throws Exception
     */
    @Override
    public synchronized void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        result = (String) msg;
        notify(); // 必须要synchronized 修饰
    }

    public NettyClientHandler(){

    }

    public NettyClientHandler(Invocation invocation) {
        this.invocation = invocation;
    }

    public void setInvocation(Invocation invocation) {
        this.invocation = invocation;
    }
    /**
     * 发送数据
     * @return
     * @throws Exception
     */
    @Override
    public synchronized String call() throws Exception {
        context.writeAndFlush(this.invocation);
        wait(); // 必须要synchronized 修饰
        return result;
    }
}
```

#### 负载均衡

```java
/**
 * 负载均衡
 */
public class Balance {

    public static Url getUrl(List<Url> urls) {
        String loadBalance = System.getProperty("loadBalance");
        if (StringUtils.isEmpty(loadBalance)) {
            return urls.get(0);
        }
        switch (loadBalance){
            case "random":
                return getRandom(urls);
            case "last":
                return getLast(urls);
            default:
        }
        return urls.get(0);
    }

    private static Url getLast(List<Url> urls) {
        return urls.get(urls.size() - 1);
    }

    private static Url getRandom(List<Url> urls) {
        Random random = new Random();
        int index = random.nextInt(urls.size());
        return urls.get(index);
    }

}

```
> 这里要注意下，负载均衡、熔断、降级其实都是在dubbo consumer上操作，这里可以多想想

### register center

<font color='red'><b>远程注册中心，这里需要注意，dubbo provider启动时，需要把自己注册到远程注册中心，也需要把自己注册到本地注册中心</b></font>


这里临时通过保存本地文件的方式来解决远程注册中心的问题，也可以替换成zookeeper、redis
```java
/**
 * 存储文件
 * 远程注册
 */
public class RemoteRegister {

    /**
     * Url 地址
     * String 接口名
     */
    private static Map<String, List<Url>> map = new HashMap<>();

    public static void register(String interfaceName,Url url){
        List<Url> urls = map.get(interfaceName);
        if(CollectionUtils.isEmpty(urls)){ //空的
            map.put(interfaceName, Lists.newArrayList(url));
        }else {
            urls.add(url);
            map.put(interfaceName, urls);
        }
        saveFile();
    }

    /**
     * 根据接口名获取url地址
     * @param interfaceName
     * @return
     */
    public static List<Url> get(String interfaceName){
        Map<String, List<Url>> file = getFile();
        return file.get(interfaceName);
    }

    private static void saveFile() {
        try {
            FileOutputStream fileOutputStream = new FileOutputStream("/Users/lijunlong/temp.txt");
            ObjectOutputStream objectOutputStream = new ObjectOutputStream(fileOutputStream);
            objectOutputStream.writeObject(map);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    private static Map<String, List<Url>> getFile() {
        try {
            FileInputStream fileInputStream = new FileInputStream("/Users/lijunlong/temp.txt");
            ObjectInputStream objectInputStream = new ObjectInputStream(fileInputStream);
            return (Map<String, List<Url>>) objectInputStream.readObject();
        } catch (IOException | ClassNotFoundException e) {
            e.printStackTrace();
        }
        return null;
    }
}

```

# 结语

目前dubbo模拟也就上述差不多，mock、负载均衡、熔断、降级、dubbo over tcp、dubbo over http，动态配置已实现，这里可以结合一步一步的延伸，然后可以仔细体会dubbo是如何设计。
也为后续读dubbo源码有个借鉴作用。


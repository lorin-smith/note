# Dubbo从入门到源码

## Dubbo的基本应用

### 什么是Dubbo

Dubbo是阿里巴巴公司开源的一个高性能、轻量级的 Java RPC 框架

致力于提供高性能和透明化的 RPC 远程服务调用方案，以及 SOA 服务治理方案。

### Dubbo的发展历史

**2011/10/27：** 阿里巴巴巴宣布 Dubbo 开源。

**2012/10/23：** 发布最后一个版本 2.5.3 并停止维护更新。

**2017/07/31：** 起死回生，官方宣布开启重新更新，并会得到重点维护.

**2017/09/07：** 发布起死回生的第一个版本：[dubbo-2.5.4](https://mp.weixin.qq.com/s/DEG-i3Y5u4dajS05gmJboA)。

**2018/01/08：**

1、Dubbo 团队透露 Dubbo 3.0 宣布正式开工

2、发布了 dubbo-2.6.0 版本，主要合并了由当当网开源的 dubbox 项目分支。PS：dubbo停止维护期间，当当网基于 dubbo 开源了dubbox。

**2018/01/22：** Dubbo Spring Boot 版正式发布：[dubbo-spring-boot-starter](https://mp.weixin.qq.com/s/yCcIMcn1MfwItycsPFgfOA) v1.0.0 公测版。

**2018/02/09：** Dubbo 通过投票正式进入 Apache 基金会孵化器，更新了 Apache 官方域名，也不再仅限于 Java 语言。

**2019/05/20：** Apache 软件基金会宣布 Dubbo 正式毕业，成为 Apache 的顶级项目。

2021/7月1日 : Dubbo 3.0版本正式发布

### Dubbo主要特性

* 面向接口代理的高性能RPC调用：提供高性能的基于代理的远程调用能力，服务以接口为粒度，屏蔽了远程调用底层细节。
* 智能负载均衡：内置多种负载均衡策略，智能感知下游节点健康状况，显著减少调用延迟，提高系统吞吐量。
* 服务自动注册与发现：支持多种注册中心服务，服务实例上下线实时感知。
* 高度可扩展能力：遵循微内核+插件的设计原则，所有核心能力如Protocol、Transport、Serialization被设计为扩展点，平等对待内置实现和第三方实现。
* 运行期流量调度：内置条件、脚本等路由策略，通过配置不同的路由规则，轻松实现灰度发布，同机房优先等功能。
* 可视化的服务治理与运维：提供丰富服务治理、运维工具：随时查询服务元数据、服务健康状态及调用统计，实时下发路由策略、调整配置参数。

Spring Cloud  团队

Spring Cloud Alibaba

HTTP     短连接  重新建立TCP连接

RPC     Dubbo Spring Cloud

### Dubbo的基本使用

Dubbo支持的注册中心

consul

zk

Eureka

Redis

ectd

nacos

Dubbo       Spring  Cloud

：

|              | Dubbo         | Spring   Cloud               |
| ------------ | ------------- | ---------------------------- |
| 服务注册中心 | Zookeeper     | Spring Cloud Netfix Eureka   |
| 服务调用方式 | RPC           | REST API                     |
| 服务监控     | Dubbo-monitor | Spring Boot Admin            |
| 熔断器       | 不完善        | Spring Cloud Netflix Hystrix |
| 服务网关     | 无            | Spring Cloud Netflix Zuul    |
| 分布式配置   | 无            | Spring Cloud Config          |
| 服务跟踪     | 无            | Spring Cloud Sleuth          |
| 数据流       | 无            | Spring Cloud Stream          |
| 批量任务     | 无            | Spring Cloud Task            |
| 信息总线     | 无            | Spring Cloud Bus             |

Nacos

#### Dubbo  Spring Cloud

Spring   Cloud

| 功能组件                                             | Spring Cloud                 | Dubbo Spring Cloud                              |
| ---------------------------------------------------- | ---------------------------- | ----------------------------------------------- |
| 分布式配置（Distributed configuration）              | Git、Zookeeper、Consul、JDBC | Spring Cloud 分布式配置 + Dubbo 配置中心        |
| 服务注册与发现（Service registration and discovery） | Eureka、Zookeeper、Consul    | Spring Cloud 原生注册中心 + Dubbo 原生注册中心  |
| 负载均衡（Load balancing）                           | Ribbon（随机、轮询等算法）   | Dubbo 内建实现（随机、轮询等算法 + 权重等特性） |
| 服务熔断（Circuit Breakers）                         | Spring Cloud Hystrix         | Spring Cloud Hystrix + Alibaba Sentinel 等      |
| 服务调用（Service-to-service calls）                 | Open Feign、RestTemplate     | Spring Cloud 服务调用 + Dubbo @Reference        |
| 链路跟踪（Tracing）                                  | Spring Cloud Sleuth + Zipkin | Zipkin、opentracing 等                          |

以上对比表格摘自Dubbo Spring Cloud官方文档。

本质上   都是为业务而生

### Dubbo多注册中心

### Dubbo多协议支持

### Dubbo源码

怎么学？  spring的2.5之后 ，Spring支持自定义Schema扩展XML配置

1.Xml  Schema

```xml
<bean id="dubbo-server" class="com.alibaba.dubbo.config.ApplicationConfig"/>  
    <bean id="registryConfig" class="com.alibaba.dubbo.config.RegistryConfig">  
        <property name="address" value="192.168.88.88:2181"/>  
        <property name="protocol" value="zookeeper"/>  
    </bean>  
    <bean id="dubbo" class="com.alibaba.dubbo.config.ProtocolConfig">  
        <property name="port" value="20880"/>  
    </bean>  
    <bean id="com.msb.dubbo.server.IloginService" class="com.alibaba.dubbo.config.spring.ServiceBean">  
        <property name="interface" value="com.msb.dubbo.server.IloginService"/>  
        <property name="ref" ref="loginService"/>  
    </bean>  
    <bean id="loginService" class="com.msb.dubbo.server.IloginServiceImpl" />  
```

```java
private void doExportUrlsFor1Protocol(ProtocolConfig protocolConfig, List<URL> registryURLs) {

        String name = protocolConfig.getName();
        if (StringUtils.isEmpty(name)) {
            name = DUBBO;
        }
s
        Map<String, String> map = new HashMap<String, String>();
        //（"side":"provide"）
        map.put(SIDE_KEY, PROVIDER_SIDE);

        ServiceConfig.appendRuntimeParameters(map);
        AbstractConfig.appendParameters(map, getMetrics());
        AbstractConfig.appendParameters(map, getApplication());
        AbstractConfig.appendParameters(map, getModule());
        // remove 'default.' prefix for configs from ProviderConfig
        // appendParameters(map, provider, Constants.DEFAULT_KEY);
        AbstractConfig.appendParameters(map, provider);
        AbstractConfig.appendParameters(map, protocolConfig);
        AbstractConfig.appendParameters(map, this);
        MetadataReportConfig metadataReportConfig = getMetadataReportConfig();

        if (metadataReportConfig != null && metadataReportConfig.isValid()) {
            map.putIfAbsent(METADATA_KEY, REMOTE_METADATA_STORAGE_TYPE);
        }

        if (CollectionUtils.isNotEmpty(getMethods())) {
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
        //泛化方式
        if (ProtocolUtils.isGeneric(generic)) {
            map.put(GENERIC_KEY, generic);
            map.put(METHODS_KEY, ANY_VALUE);
        } else {
            String revision = Version.getVersion(interfaceClass, version);
            if (revision != null && revision.length() > 0) {
                map.put(REVISION_KEY, revision);
            }
            //获取我们的包装类
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
        URL url = new URL(name, host, port, getContextPath(protocolConfig).map(p -> p + "/" + path).orElse(path), map);

        // You can customize Configurator to append extra parameters
        if (ExtensionLoader.getExtensionLoader(ConfiguratorFactory.class)
                .hasExtension(url.getProtocol())) {
            url = ExtensionLoader.getExtensionLoader(ConfiguratorFactory.class)
                    .getExtension(url.getProtocol()).getConfigurator(url).configure(url);
        }

        String scope = url.getParameter(SCOPE_KEY);
        // don't export when none is configured
        if (!SCOPE_NONE.equalsIgnoreCase(scope)) {

            // export to local if the config is not remote (export to remote only when config is remote)
            if (!SCOPE_REMOTE.equalsIgnoreCase(scope)) {
                exportLocal(url);
            }
            // export to remote if the config is not local (export to local only when config is local)
            if (!SCOPE_LOCAL.equalsIgnoreCase(scope)) {
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
                        //？
                        Invoker<?> invoker = PROXY_FACTORY.getInvoker(ref, (Class) interfaceClass, registryURL.addParameterAndEncoded(EXPORT_KEY, url.toFullString()));
                        DelegateProviderMetaDataInvoker wrapperInvoker = new DelegateProviderMetaDataInvoker(invoker, this);
                        //？
                        Exporter<?> exporter = PROTOCOL.export(wrapperInvoker);
                        exporters.add(exporter);
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

Dubbo扩展点

### **指定名称的扩展点**

```
ExtensionLoader.getExtensionLoader(Protocol.class).getExtension("name");

```

### 自适应的扩展点

```
ExtensionLoader.getExtensionLoader(Protocol.class).getAdaptiveExtension();
```

### 激活扩展点

```
ExtensionLoader.getExtensionLoader(Protocol.class).getExtension();
```



指定名称的拓展点   

或者说你直接从指定文件中找到我们的name


我们如何判定dubbo中哪些类是拓展点呢？ 


找出拓展点并加载的流程

1.找到路径：在我们的META-INF/dubbo/inertnal下面

2.

```properties
random=org.apache.dubbo.rpc.cluster.loadbalance.RandomLoadBalance
roundrobin=org.apache.dubbo.rpc.cluster.loadbalance.RoundRobinLoadBalance
leastactive=org.apache.dubbo.rpc.cluster.loadbalance.LeastActiveLoadBalance
consistenthash=org.apache.dubbo.rpc.cluster.loadbalance.ConsistentHashLoadBalance
shortestresponse=org.apache.dubbo.rpc.cluster.loadbalance.ShortestResponseLoadBalance

```

3.Package.Class

4.解析文件    properties文件的解析

5.内容放进内存       key（name）  value =org.apache.dubbo.rpc.cluster.loadbalance.RandomLoadBalance 

6.key（name）  value =Class          newInstance


### 自适应拓展点

在运行期间，我们会根据上下文来判断当前你返回哪个拓展点。

```
getAdaptiveExtension()
```

标识位：

@Adaptive

* 声明在我们的方法级别
* 声明在我们的类级别

实现原理：

* 如果修饰在方法级别，那么我们会动态的创建一个代理类 （javassist）
* 如果修饰的是类，那么我们直接返回被修饰的这个类


如果我们当前架子啊的扩展点存在自适应的类，那么我们直接返回，

否则，我们会动态的创建一个字节码，然后进行返回。


### 激活拓展点

相当于Spring当中的Conditional

@ConditionalOnBean（XXX.class）

只要你的@Activate注解中有value，那么这个时候我们当前的Filter就会被激活。

1.判断是否要发布到远程服务，或者说是否需要发布服务，如果为none，那么我们就不需要发布

2.循环遍历注册中心的配置，实际上这里就是对多注册中心支持的最好体现




Protoco$Adaptive  --> export方法


1.进行了Netty服务的启动

2.进行服务注册

protocol  = protocol$Adaptive  


Protocol = Wrapper(（（DubboProtocol））)


Dubbo  -- Netty4

ServerSocket   serverSocket；

20880（默认端口）

RegistryFactoryWrapper（ZookeeperRegistryFactory）


服务注册的总结：

1.生成invoke对象

2.doLocalExport方法发布一个本地服务，实际上就是利用配置去进行一个监听

3.通过Registy去进行服务注册


### invoker对象是什么

对象实例 ---  保存在Map集合中的

收到客户端的请求，（方法名称，接口的全路径，参数，参数类型）

根据接口的全路径做为key，找到实例方法，反射调用他

```
Invoker<?> invoker = PROXY_FACTORY.getInvoker(ref, (Class) interfaceClass, registryURL.addParameterAndEncoded(EXPORT_KEY, url.toFullString()));
```

1.如果假设我们的Url中有配置了proxy，我就按照我们Proxy参数的配置来进行查找

但是，如果没有配置，那么我们采用默认的拓展点

```java


public <T> Invoker<T> getInvoker(T proxy, Class<T> type, URL url) {
        final Wrapper wrapper = Wrapper.getWrapper(proxy.getClass().getName().indexOf(36) < 0 ? proxy.getClass() : type);
        return new AbstractProxyInvoker<T>(proxy, type, url) {

          //当我们的服务端接受到请求之后，我们接下来调用具体的方法
            protected Object doInvoke(T proxy, String methodName, Class<?>[] parameterTypes, Object[] arguments) throws Throwable {
                return wrapper.invokeMethod(proxy, methodName, parameterTypes, arguments);
            }
        };
    }
```


invoker本身就是一个调用器

服务本身会以我们invoker的形态存在

### 服务消费

关键性注解  

1.生成远程服务的代理

2.获取目标服务的Url

2.1 建立与注册中心的动态感知

3.网络连接的建立（启动时创建还是通信是创建？） -->  Dubbo启动时创建   Netty(nio)

4.服务通信

4.1 filter过滤

4.2 负载均衡

4.3 容错



用到Spring的一系列知识

### 服务消费端的操作

#### 服务消费端的对象的注入

* Xml
* 注解的方式

BeanPostProcessor

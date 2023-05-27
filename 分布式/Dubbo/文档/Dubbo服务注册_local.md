* 了解需求
* 先主线, 找到入口
* 技术设计的整理(云笔记)

# Dubbo服务的注册流程

## 服务发布步骤

* 注解

  ```java
  @DubboService(
          loadbalance = "random",
          cluster = "failover",
          retries = 2)
  ```

* 注解扫描

  ```java
  @DubboComponentScan
  ```

## 引发思考

* 扫描注解（解析注解）
* url驱动， url的组装
* 注册到zookeeper（？）
* 启动服务（根据url中配置的协议、端口去发布对应的服务） （？）

## dubbo源码

### dubbo服务发布有两种形式

* xml 形式 <dubbo:service>
* @DubboService/ @Service

# Dubbo注解的解析流程

## DubboComponetScan

无非就是，把哪个bean注册到了Spring IOC容器

> ServiceAnnotationBeanPostProcessor

```java
@Override
public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {

    Set<String> packagesToScan = getPackagesToScan(importingClassMetadata);

    registerServiceAnnotationBeanPostProcessor(packagesToScan, registry);

    // @since 2.7.6 Register the common beans
    registerCommonBeans(registry);
}
```

## ServiceAnnotationBeanPostProcessor

```java
public ServiceAnnotationBeanPostProcessor(Set<String> packagesToScan) {
    super(packagesToScan);
}
```

## BeanDefinitionRegistryPostProcessor

bean装载完成之后，会触发下面这个方法.

```java
public ServiceAnnotationBeanPostProcessor(Set<String> packagesToScan) {
    super(packagesToScan);
}
```



```java
    @Override
    public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException {

        // @since 2.7.5
        registerInfrastructureBean(registry, DubboBootstrapApplicationListener.BEAN_NAME, DubboBootstrapApplicationListener.class);

        Set<String> resolvedPackagesToScan = resolvePackagesToScan(packagesToScan);

        if (!CollectionUtils.isEmpty(resolvedPackagesToScan)) {
            registerServiceBeans(resolvedPackagesToScan, registry);
        } else {
            if (logger.isWarnEnabled()) {
                logger.warn("packagesToScan is empty , ServiceBean registry will be ignored!");
            }
        }

    }
```

### 注册一个DubboBootstrapApplicationListener

这个bean会在spring 容器的上下文装载完成之后，触发监听

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
```

### registerServiceBeans

```java
private void registerServiceBeans(Set<String> packagesToScan, BeanDefinitionRegistry registry) {

    DubboClassPathBeanDefinitionScanner scanner =
        new DubboClassPathBeanDefinitionScanner(registry, environment, resourceLoader);

    BeanNameGenerator beanNameGenerator = resolveBeanNameGenerator(registry);

    scanner.setBeanNameGenerator(beanNameGenerator);

    // refactor @since 2.7.7
    serviceAnnotationTypes.forEach(annotationType -> {
        scanner.addIncludeFilter(new AnnotationTypeFilter(annotationType));
    });

    for (String packageToScan : packagesToScan) {

        // Registers @Service Bean first
        scanner.scan(packageToScan);

        // Finds all BeanDefinitionHolders of @Service whether @ComponentScan scans or not.
        Set<BeanDefinitionHolder> beanDefinitionHolders =
            findServiceBeanDefinitionHolders(scanner, packageToScan, registry, beanNameGenerator);

        if (!CollectionUtils.isEmpty(beanDefinitionHolders)) {

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
```

### registerServiceBean

和服务有关的信息，实际上都在我们刚刚定义的@DubboService  

[DemoService]

* 服务以什么协议发布
* 服务的负载均衡策略
* 服务的容错策略
* 服务发布端口

org.apache.dubbo.config.spring.ServiceBean 

```java
private void registerServiceBean(BeanDefinitionHolder beanDefinitionHolder, BeanDefinitionRegistry registry,
                                     DubboClassPathBeanDefinitionScanner scanner) {

    Class<?> beanClass = resolveClass(beanDefinitionHolder);

    Annotation service = findServiceAnnotation(beanClass);

    /**
         * The {@link AnnotationAttributes} of @Service annotation
         */
    AnnotationAttributes serviceAnnotationAttributes = getAnnotationAttributes(service, false, false);

    Class<?> interfaceClass = resolveServiceInterfaceClass(serviceAnnotationAttributes, beanClass);

    String annotatedServiceBeanName = beanDefinitionHolder.getBeanName();

    AbstractBeanDefinition serviceBeanDefinition =
        buildServiceBeanDefinition(service, serviceAnnotationAttributes, interfaceClass, annotatedServiceBeanName);

    // ServiceBean Bean name
    String beanName = generateServiceBeanName(serviceAnnotationAttributes, interfaceClass);

    if (scanner.checkCandidate(beanName, serviceBeanDefinition)) { // check duplicated candidate bean
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

>  registry.registerBeanDefinition(beanName, serviceBeanDefinition);
>
> 最终通过上述代码，讲一个 dubbo中提供的ServiceBean注入到Spring IOC容器

# ServiceBean的初始化阶段

```java
public ServiceBean() {
    super();
    this.service = null;
}
```

> 当ServiceBean初始化完成之后，会调用下面的方法.

```java
@Override
public void afterPropertiesSet() throws Exception {
    if (StringUtils.isEmpty(getPath())) {
        if (StringUtils.isNotEmpty(beanName)
            && StringUtils.isNotEmpty(getInterface())
            && beanName.startsWith(getInterface())) {
            setPath(beanName);
        }
    }
}
```

## DubboBootstrapApplicationListener

> 启动dubbo服务

```java
private void onContextRefreshedEvent(ContextRefreshedEvent event) {
    dubboBootstrap.start();
}
```

* 元数据/远程配置信息的初始化
* 拼接url（）
* 如果是dubbo协议，则启动netty server
* 服务注册

## start()

```java
public DubboBootstrap start() {
    if (started.compareAndSet(false, true)) {
        ready.set(false);
        initialize();
        if (logger.isInfoEnabled()) {
            logger.info(NAME + " is starting...");
        }
        // 1. export Dubbo Services
        exportServices();

        // Not only provider register
        if (!isOnlyRegisterProvider() || hasExportedServices()) {
            // 2. export MetadataService
            exportMetadataService();
            //3. Register the local ServiceInstance if required
            registerServiceInstance();
        }

        referServices();
        if (asyncExportingFutures.size() > 0) {
            new Thread(() -> {
                try {
                    this.awaitFinish();
                } catch (Exception e) {
                    logger.warn(NAME + " exportAsync occurred an exception.");
                }
                ready.set(true);
                if (logger.isInfoEnabled()) {
                    logger.info(NAME + " is ready.");
                }
            }).start();
        } else {
            ready.set(true);
            if (logger.isInfoEnabled()) {
                logger.info(NAME + " is ready.");
            }
        }
        if (logger.isInfoEnabled()) {
            logger.info(NAME + " has started.");
        }
    }
    return this;
}
```

## exportServices

> 发布dubbo服务

```java
private void exportServices() {
    configManager.getServices().forEach(sc -> {
        // TODO, compatible with ServiceConfig.export()
        ServiceConfig serviceConfig = (ServiceConfig) sc;
        serviceConfig.setBootstrap(this);

        if (exportAsync) {
            ExecutorService executor = executorRepository.getServiceExporterExecutor();
            Future<?> future = executor.submit(() -> {
                sc.export();
                exportedServices.add(sc);
            });
            asyncExportingFutures.add(future);
        } else {
            sc.export();
            exportedServices.add(sc);
        }
    });
}
```

遍历所有dubbo服务，进行服务发布.

```java
<dubbo:service beanName="ServiceBean:com.msbedu.springboot.dubbo.springbootdubbosampleprovider.services.IDemoService" />
<dubbo:service beanName="ServiceBean:com.msbedu.springboot.dubbo.ISayHelloService" />
dubbo://ip:port?com.msbedu.springboot.dubbo.springbootdubbosampleprovider.services.IDemoService&xxx&xxx
dubbo://ip:port?com.msbedu.springboot.dubbo.ISayHelloService&xxx&xxx
```

一个接口需要发布几次？

一个dubbo服务需要发布几次，取决于协议的配置数，如果一个dubbo服务配置了3个协议，rest、webservice、dubbo。

dubbo://

rest://

webservice://

## export

```java
public synchronized void export() {
    if (!shouldExport()) {
        return;
    }

    if (bootstrap == null) {
        bootstrap = DubboBootstrap.getInstance();
        bootstrap.init();
    }

    checkAndUpdateSubConfigs();

    //init serviceMetadata
    serviceMetadata.setVersion(version);
    serviceMetadata.setGroup(group);
    serviceMetadata.setDefaultGroup(group);
    serviceMetadata.setServiceType(getInterfaceClass());
    serviceMetadata.setServiceInterfaceName(getInterface());
    serviceMetadata.setTarget(getRef());

    if (shouldDelay()) {
        DELAY_EXPORT_EXECUTOR.schedule(this::doExport, getDelay(), TimeUnit.MILLISECONDS);
    } else {
        doExport();
    }

    exported();
}
```

## doExportUrls

>  <dubbo:protocol name="dubbo" port="-1" />

 主要流程，根据开发者配置的协议列表，遍历协议列表逐项进行发布。

```java
private void doExportUrls() {
    ServiceRepository repository = ApplicationModel.getServiceRepository();
    ServiceDescriptor serviceDescriptor = repository.registerService(getInterfaceClass());
    repository.registerProvider(
        getUniqueServiceName(),
        ref,
        serviceDescriptor,
        this,
        serviceMetadata
    );

    List<URL> registryURLs = ConfigValidationUtils.loadRegistries(this, true);

    for (ProtocolConfig protocolConfig : protocols) {
        String pathKey = URL.buildKey(getContextPath(protocolConfig)
                                      .map(p -> p + "/" + path)
                                      .orElse(path), group, version);
        // In case user specified path, register service one more time to map it to path.
        repository.registerService(pathKey, interfaceClass);
        // TODO, uncomment this line once service key is unified
        serviceMetadata.setServiceKey(pathKey);
        doExportUrlsFor1Protocol(protocolConfig, registryURLs);
    }
}
```

## doExportUrlsFor1Protocol

* 生成url
* 根据url中配置的协议类型，调用指定协议进行服务的发布
  * 启动服务
  * 注册服务

```java
private void doExportUrlsFor1Protocol(ProtocolConfig protocolConfig, List<URL> registryURLs) {
    String name = protocolConfig.getName();
    if (StringUtils.isEmpty(name)) {
        name = DUBBO;
    }
    //用来存储所有的配置信息
    * dubbo:service
        dubbo:method
            dubbo:argument
    Map<String, String> map = new HashMap<String, String>();
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

    if (ProtocolUtils.isGeneric(generic)) {
        map.put(GENERIC_KEY, generic);
        map.put(METHODS_KEY, ANY_VALUE);
    } else {
        String revision = Version.getVersion(interfaceClass, version);
        if (revision != null && revision.length() > 0) {
            map.put(REVISION_KEY, revision);
        }

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

                    Invoker<?> invoker = PROXY_FACTORY.getInvoker(ref, (Class) interfaceClass, registryURL.addParameterAndEncoded(EXPORT_KEY, url.toFullString()));
                    DelegateProviderMetaDataInvoker wrapperInvoker = new DelegateProviderMetaDataInvoker(invoker, this);

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



```txt
dubbo://192.168.1.104:20880/com.msbedu.springboot.dubbo.springbootdubbosampleprovider.services.IDemoService?anyhost=true&application=spring-boot-dubbo-sample-provider&bind.ip=192.168.1.104&bind.port=20880&default=true&deprecated=false&dubbo=2.0.2&dynamic=true&generic=false&interface=com.msbedu.springboot.dubbo.springbootdubbosampleprovider.services.IDemoService&methods=getTxt&pid=225512&qos.accept.foreign.ip=false&qos.enable=true&qos.port=8888&release=2.7.7&side=provider&timestamp=1597498654297 
& loadbalance=msbloadbalance

consumer://ip:port....&
```



# dubbo中的扩展点

```java
ExtensionLoader.getExtensionLoader(ConfiguratorFactory.class).hasExtension(url.getProtocol())
```

* Java中的spi
* SpringFactoriesLoader
* Dubbo SPI

## Dubbo中的扩展点

## 指定名称的扩展点

```java
ExtensionLoader.getExtensionLoader(Protocol.class).getExtension("name");
```

* 找到Protocol的全路径名称， 在/META-INF/dubbo/intenal
* 在指定文件中找到“name”对应的实现类.

### 自定义扩展点

* 在resource/META-INF/dubbo/ org.apache.dubbo.rpc.cluster.LoadBalance

* 实现扩展点

  ```java
  public class msbDefineLoadBalance extends AbstractLoadBalance{
      @Override
      protected <T> Invoker<T> doSelect(List<Invoker<T>> invokers, URL url, Invocation invocation) {
  
          return null;
      }
  }
  ```

* 测试

  ```java
    //解析URL
  String loadbalance="random";
  //URL loadbalance="msbloadbalance"
  //loadlance=msbloadbalance
  LoadBalance loadBalance=ExtensionLoader.getExtensionLoader(LoadBalance.class).getExtension(loadbalance);
  System.out.println(loadBalance);
  ```

### 扩展点的特征

在类级别标准`@SPI(RandomLoadBalance.NAME)`.

其中，括号内的数据，表示当前扩展点的默认扩展点。

### 指定名称的扩展点的原理

* 思考实现步骤的拆解
* 

## 自适应扩展点

```java
ExtensionLoader.getExtensionLoader(Protocol.class).getAdaptiveExtension();
```

## 激活扩展点

```java
ExtensionLoader.getExtensionLoader(Protocol.class).getActiveExtension
```
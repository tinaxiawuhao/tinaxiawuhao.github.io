---
title: 'Nacos核心源码剖析——配置中心'
date: 2022-07-04 20:33:38
tags: [springCloud]
published: true
hideInList: false
feature: /post-images/bp8DT_bHg.png
isTop: false
---
Nacos官方文档：https://nacos.io/zh-cn/docs/quick-start.html

服务端对外暴露的API：https://nacos.io/zh-cn/docs/open-api.html

Nacos的Server端其实就是一个Web服务，对外提供了Http服务接口，所有的客户端与服务端的通讯都通过Http调用完成（短链接）。

> **Nacos注册服务核心类：NacosNamingService**
>
> **Nacos配置中心核心类：NacosConfigService**

Nacos配置中心的nameSpace/Group和注册中心类似，但是没有集群Cluster的概念！

配置文件的核心主键是DataId（与注册中心不一样，注册中心为ServiceName）

```yaml
spring:
  application:
    name: nacos-config-client
  cloud:
    nacos:
      config:
        server-addr: 10.206.73.156:8848
        namespace: haier-iot
        group: dev
        file-extension: yaml
# 上面的配置，拼装成的最高优先级配置文件为 haier-iot/dev/ nacos-config-client-dev.yaml
```

**Nacos还支持扩展配置：extension-configs，和共享配置：shared-configs**，以支持各种复杂的应用场景！

官方github wiki地址：https://github.com/alibaba/spring-cloud-alibaba/wiki/Nacos-config

**Client客户端核心知识点：**

- **当需要获取配置时，先尝试从本地配置文件获取，获取不到时，再去远端Server获取；从远端获取成功后，保存到本地配置文件，后面通过ClientWorker中的长轮询完成配置的实时更新!**

ClientWorker中有两个线程executor和executorService：

![](https://tianxiawuhao.github.io/post-images/1658666099885.png)

- **单线程executor定时任务**：每10ms执行checkConfigInfo()方法，看看本地配置信息是否有变化，以3000个为一组，判断是否要增加新的LongPollingRunnable长轮询任务；
- **多线程executorService执行LongPollingRunnable长轮询任务核心逻辑**：根据本地配置项的dataId，group，Md5值，tenant拼接字符串，调用服务端“监听配置——长轮询接口”看看这批次的配置是否有变化，有变化的话，就遍历调用“获取配置详情接口”获取最新配置值，没变化的不用动，任务最后，再次执行this，循环往复执行此长轮询任务！

  当本地的配置文件发生改变时，会回调注册在这些配置上的监听器的回调方法，从而完成应用程序的配置更新！（refresh(context)后完成Nacos监听器的注册）

**服务端核心知识点：**

![](https://tianxiawuhao.github.io/post-images/1658666125421.png)

- **服务端即使配置了mysql，每次请求也不是直接去查询mysql的，而是借助 本地内存缓存中的元数据 + 本地磁盘中的配置文件；**
- **Mysql主要用于集群节点启动时的数据加载（全量加载、增量加载） 和 数据变动时的刷新同步；**
- **某节点处理配置发布请求时，不是着急更新自己的状态，而是会先写入mysql数据库，之后通过ConfigDataChangeEvent事件实现异步处理，通知所有节点更新自己的内存缓存和本地磁盘文件；（包括自己）**
- AP集群，存在数据的短时间不一致，但是可以保证最终一致性，对客户端的影响也就是可能配置更新慢那么一点。

------

### 一 nacos客户端加载配置的核心逻辑

#### **1 nacos核心配置NacosPropertySourceLocator类的定位：**

如果要弄清楚Nacos配置文件加载到Spring容器中的流程，还需要熟悉Springcloud的源码流程，了解Springcloud中的配置，还有重要的Bean是如果装载到Spring容器中的；

这里只能大概聊一下：

- Springboot项目启动时，在prepareEnvironment()阶段，会通过spring.factories文件中的BootstrapConfiguration类找到NacosConfigBootstrapConfiguration配置类，并完成注入：

![](https://tianxiawuhao.github.io/post-images/1658666155462.png)

- NacosConfigBootstrapConfiguration配置类，会向Spring容器中注入几个Nacos配置读取重要的类

```java
public class NacosConfigBootstrapConfiguration {
    public NacosConfigBootstrapConfiguration() {
    }
 
    @Bean
    @ConditionalOnMissingBean
    public NacosConfigProperties nacosConfigProperties() {
        return new NacosConfigProperties();
    }
 
    @Bean
    @ConditionalOnMissingBean
    public NacosConfigManager nacosConfigManager(NacosConfigProperties nacosConfigProperties) {
        return new NacosConfigManager(nacosConfigProperties);
    }
 
    // NacosPropertySourceLocator能够一步步地把把Nacos的配置文件都找到
    @Bean
    public NacosPropertySourceLocator nacosPropertySourceLocator(NacosConfigManager nacosConfigManager) {
        return new NacosPropertySourceLocator(nacosConfigManager);  
    }
}
```

- 然后在prepareContext()上下文的时候，会通过之前从spring.factories中读取到的ApplicationInitializer初始化器，这里遍历执行时，就会执行到springcloud的PropertySpurceBootstrapConfiguration类的initialize()方法：

![](https://tianxiawuhao.github.io/post-images/1658666184206.png)

  可以看到PropertySpurceBootstrapConfiguration.initialize()方法中需要用到PropertySourceLocator接口的实现类，而我们配置的Nacos正好为这个接口提供了实现类NacosPropertySourceLocator

![](https://tianxiawuhao.github.io/post-images/1658666209114.png)

#### **2 通过NacosPropertySourceLocator类，理清Nacos各中配置文件的优先级：**

```java
// com.alibaba.cloud.nacos.client.NacosPropertySourceLocator#locate
public PropertySource<?> locate(Environment env) {
    this.nacosConfigProperties.setEnvironment(env);
    ConfigService configService = this.nacosConfigManager.getConfigService();
    if (null == configService) {
        log.warn("no instance of config service found, can't load config from nacos");
        return null;
    } else {
        long timeout = (long)this.nacosConfigProperties.getTimeout();
        this.nacosPropertySourceBuilder = new NacosPropertySourceBuilder(configService, timeout);
        String name = this.nacosConfigProperties.getName();
        String dataIdPrefix = this.nacosConfigProperties.getPrefix();
        if (StringUtils.isEmpty(dataIdPrefix)) {
            dataIdPrefix = name;
        }
 
        if (StringUtils.isEmpty(dataIdPrefix)) {
            dataIdPrefix = env.getProperty("spring.application.name");
        }
 
        CompositePropertySource composite = new CompositePropertySource("NACOS");
        // 1. 先加载共享配置文件
        this.loadSharedConfiguration(composite); 
        // 2. 再加载扩展配置文件 
        this.loadExtConfiguration(composite);  
        // 3. 最后才加载本应用自己的配置文件
        this.loadApplicationConfiguration(composite, dataIdPrefix, this.nacosConfigProperties, env); 
        return composite;
    }
}
```

本应用自己的配置文件也是可以存在多个的，也是有优先级的：

```java
// 入参中的dataIdPrefix，理解为就是spring.application.name
private void loadApplicationConfiguration(CompositePropertySource compositePropertySource, String dataIdPrefix, NacosConfigProperties properties, Environment environment) {
    String fileExtension = properties.getFileExtension();
    String nacosGroup = properties.getGroup();
    // 1. 先尝试加载：纯微服务名称对应的配置文件，如：order-config
    this.loadNacosDataIfPresent(compositePropertySource, dataIdPrefix, nacosGroup, fileExtension, true);
    // 2. 再尝试加载：微服务名称 + "." + 文件扩展名的文件，如：order-config.yaml
    this.loadNacosDataIfPresent(compositePropertySource, dataIdPrefix + "." + fileExtension, nacosGroup, fileExtension, true);
    String[] var7 = environment.getActiveProfiles();
    int var8 = var7.length;
 
    // 3. 最后再尝试加载：微服务名称 + "-" + profile + "." + 文件扩展名的文件，如：order-config-dev.yaml
    for(int var9 = 0; var9 < var8; ++var9) {
        String profile = var7[var9];
        String dataId = dataIdPrefix + "-" + profile + "." + fileExtension;
        this.loadNacosDataIfPresent(compositePropertySource, dataId, nacosGroup, fileExtension, true);
    }
 
}
```

> 根据后加载的覆盖先加载的原则，最后我们很容易就可以知道整个服务的配置文件的优先级为：
>
> **order-config-dev.yaml > order-config.yaml > order-config > extension-configs > shared-configs**

------

### 二 Nacos配置中心的核心类NacosConfigService的引入

![](https://tianxiawuhao.github.io/post-images/1658666249886.png)

#### **1 客户端是何时从远程配置Nacos服务端拉取配置的**

```java
// 还记得刚开始时候，通过spring.factories注入了一个配置类NacosConfigBootstrapConfiguration 
// 该配置类，会向Spring容器中注入一个Bean：NacosConfigManager
@Bean
@ConditionalOnMissingBean
public NacosConfigManager nacosConfigManager(NacosConfigProperties nacosConfigProperties) {
    return new NacosConfigManager(nacosConfigProperties);
}
 
//而NacosConfigManager的构造方法中，就会“写死”为我们create一个ConfigService，赋值给静态变量service
private static ConfigService service = createConfigService(nacosConfigProperties);
|
public static ConfigService createConfigService(Properties properties) throws NacosException {
    try {
        Class<?> driverImplClass = Class.forName("com.alibaba.nacos.client.config.NacosConfigService");
        Constructor constructor = driverImplClass.getConstructor(Properties.class);
        ConfigService vendorImpl = (ConfigService)constructor.newInstance(properties);
        return vendorImpl;
    } catch (Throwable var4) {
        throw new NacosException(-400, var4);
    }
}
```

此时，NacosConfigService闪亮登场！

#### **2 NacosConfigService类的构造方法中会创建重要的两个属性agent和worker：**

```java
public NacosConfigService(Properties properties) throws NacosException {
    ValidatorUtils.checkInitParam(properties);
    String encodeTmp = properties.getProperty(PropertyKeyConst.ENCODE);
    if (StringUtils.isBlank(encodeTmp)) {
        this.encode = Constants.ENCODE;
    } else {
        this.encode = encodeTmp.trim();
    }
    initNamespace(properties);
     
    // agent是一个http代理，如果需要登录验证等操作，ServerHttpAgent构造时会完成验证
    this.agent = new MetricsHttpAgent(new ServerHttpAgent(properties));
    this.agent.start();
     
    // 客户端的实际工作者ClientWorker，其中的agent也就是上面创建的agent
    this.worker = new ClientWorker(this.agent, this.configFilterChainManager, properties);
}
```

#### **3 NacosConfigService中的核心获取配置方法getConfig() ——> getConfigInner()：**

- 客户端需要使用配置文件时，**不是直接去调用远端Server获取，而是先尝试从本地Failover文件获取**；
- 如果**本地文件不存在，则才会从远端Server获取**；
- 从**远端Server成功获取配置后，会向本地文件保存快照**，以备后用；（本地文件的更新由后面的长轮询完成）

![](https://tianxiawuhao.github.io/post-images/1658666274991.png)

```java
private String getConfigInner(String tenant, String dataId, String group, long timeoutMs) throws NacosException {
    group = null2defaultGroup(group);
    ParamUtils.checkKeyParam(dataId, group);
    ConfigResponse cr = new ConfigResponse();
     
    cr.setDataId(dataId);
    cr.setTenant(tenant);
    cr.setGroup(group);
     
    // 优先使用本地配置
    String content = LocalConfigInfoProcessor.getFailover(agent.getName(), dataId, group, tenant);
    if (content != null) {
        cr.setContent(content);
        configFilterChainManager.doFilter(null, cr);
        content = cr.getContent();
        return content;
    }
     
    try {
        // 本地没有后，就尝试从远端获取配置
        String[] ct = worker.getServerConfig(dataId, group, tenant, timeoutMs);
        cr.setContent(ct[0]);
         
        configFilterChainManager.doFilter(null, cr);
        content = cr.getContent();
         
        return content;
    } catch (NacosException ioe) {
        ......
    }
     
    ......
    return content;
}
```

> 所以Nacos配置中心与注册中心类似，都是先尝试从本地获取配置，只不过注册中心比配置中心多了一份内存注册表！
>
> - 注册中心：本地Failover故障转移文件 ——> 本地内存注册表 ——> 远端请求服务列表
> - 配置中心：本地Failover故障转移文件（也就是本地配置快照文件）——> 远端请求配置文件 

worker.getServerConfig()远端配置请求成功后，还会往本地配置文件存一份Snapshot：

```java
// worker.getServerConfig()
public String[] getServerConfig(String dataId, String group, String tenant, long readTimeout){
    // 从远端Server获取配置，这里的agent就是NacosConfigService构造方法中创建的agent代理
    result = agent.httpGet(Constants.CONFIG_CONTROLLER_PATH, null, params, agent.getEncode(), readTimeout);
     
    switch (result.getCode()) {
        case HttpURLConnection.HTTP_OK:
            // 往本地配置快照文件存一份
            LocalConfigInfoProcessor.saveSnapshot(agent.getName(), dataId, group, tenant, result.getData());
            ct[0] = result.getData();
            if (result.getHeader().getValue(CONFIG_TYPE) != null) {
                ct[1] = result.getHeader().getValue(CONFIG_TYPE);
            } else {
                ct[1] = ConfigType.TEXT.getType();
            }
            return ct;
        case HttpURLConnection.HTTP_NOT_FOUND:
            LocalConfigInfoProcessor.saveSnapshot(agent.getName(), dataId, group, tenant, null);
            return ct;
        case HttpURLConnection.HTTP_CONFLICT: {
            ...
        }
        case HttpURLConnection.HTTP_FORBIDDEN: {
            ...
        }
        default: {
            ...
        }
    }
}
```

#### **4 Client第一次获取到Server端配置后，之后如何进行定时更新？**

```java
// ClientWorker构造时，会创建2个线程池
// 1. executor(单线程) ：定时每10毫秒执行checkConfigInfo()方法
// 2. executorService(1~核数/2): 具体的执行长轮询的线程
public ClientWorker(final HttpAgent agent, final ConfigFilterChainManager configFilterChainManager,final Properties properties) {
    this.agent = agent;
    this.configFilterChainManager = configFilterChainManager;
     
    // Initialize the timeout parameter
     
    init(properties);
     
    this.executor = Executors.newScheduledThreadPool(1, new ThreadFactory() {
        @Override
        public Thread newThread(Runnable r) {
            Thread t = new Thread(r);
            t.setName("com.alibaba.nacos.client.Worker." + agent.getName());
            t.setDaemon(true);
            return t;
        }
    });
     
    this.executorService = Executors
            .newScheduledThreadPool(Runtime.getRuntime().availableProcessors(), new ThreadFactory() {
                @Override
                public Thread newThread(Runnable r) {
                    Thread t = new Thread(r);
                    t.setName("com.alibaba.nacos.client.Worker.longPolling." + agent.getName());
                    t.setDaemon(true);
                    return t;
                }
            });
     
    this.executor.scheduleWithFixedDelay(new Runnable() {
        @Override
        public void run() {
            try {
                checkConfigInfo();
            } catch (Throwable e) {
                LOGGER.error("[" + agent.getName() + "] [sub-check] rotate check error", e);
            }
        }
    }, 1L, 10L, TimeUnit.MILLISECONDS);
}
 
public void checkConfigInfo() {
    // 将总的需要监听的配置数，以3000个为一组，创建长轮询LongPollingRunnable任务，监听配置更新！
    int listenerSize = cacheMap.get().size();
    int longingTaskCount = (int) Math.ceil(listenerSize / ParamUtil.getPerTaskConfigSize()); 
    if (longingTaskCount > currentLongingTaskCount) {
        for (int i = (int) currentLongingTaskCount; i < longingTaskCount; i++) {
            // 实际的干活线程，从远端拉取最新的config，与本地的config对比MD5值，看是否发生变化
            executorService.execute(new LongPollingRunnable(i));
        }
        currentLongingTaskCount = longingTaskCount;
    }
}
```

> **总结：ClientWorker实例化时，会创建两个线程：executor和executorService**
>
> - **executor（单线程）：每隔10ms检查本地的配置数是否发生改变，以3000为一批次创建长轮询任务，不足的话，不另外创建长轮询任务；**
> - **executorService（多线程1~核数/2）:具体执行长轮询LongPolling任务的工作线程；**

#### **5 Nacos客户端长轮询LongPolling任务的核心逻辑（md5比对）：**

监听配置的长轮询接口API：https://nacos.io/zh-cn/docs/open-api.html

![](https://tianxiawuhao.github.io/post-images/1658666302821.png)

```java
// LongPollingRunnable.run()
public void run() {
     
    List<CacheData> cacheDatas = new ArrayList<CacheData>();
    List<String> inInitializingCacheList = new ArrayList<String>();
    try {
        // 检查本地的配置文件
        for (CacheData cacheData : cacheMap.get().values()) {
            if (cacheData.getTaskId() == taskId) {
                cacheDatas.add(cacheData);
                try {
                    checkLocalConfig(cacheData);
                    if (cacheData.isUseLocalConfigInfo()) {
                        cacheData.checkListenerMd5();
                    }
                } catch (Exception e) {
                    LOGGER.error("get local config info error", e);
                }
            }
        }
         
        // 会根据上面得到的cacheDatas，组装参数，调用服务端的监听配置的长轮询接口/nacos/v1/cs/configs/listener
        // 长轮询的返回值是dataId^2group^2tenant^1，空串代表无变化
        List<String> changedGroupKeys = checkUpdateDataIds(cacheDatas, inInitializingCacheList);
        if (!CollectionUtils.isEmpty(changedGroupKeys)) {
            LOGGER.info("get changedGroupKeys:" + changedGroupKeys);
        }
         
        for (String groupKey : changedGroupKeys) {
            String[] key = GroupKey.parseKey(groupKey);
            String dataId = key[0];
            String group = key[1];
            String tenant = null;
            if (key.length == 3) {
                tenant = key[2];
            }
            try {
                // 会根据返回值中有变化的dataId配置项，单独去获取最新的配置值回本地
                String[] ct = getServerConfig(dataId, group, tenant, 3000L);
                CacheData cache = cacheMap.get().get(GroupKey.getKeyTenant(dataId, group, tenant));
                cache.setContent(ct[0]);
                if (null != ct[1]) {
                    cache.setType(ct[1]);
                }
                LOGGER.info("[{}] [data-received] dataId={}, group={}, tenant={}, md5={}, content={}, type={}",
                        agent.getName(), dataId, group, tenant, cache.getMd5(),
                        ContentUtils.truncateContent(ct[0]), ct[1]);
            } catch (NacosException ioe) {
                String message = String
                        .format("[%s] [get-update] get changed config exception. dataId=%s, group=%s, tenant=%s",
                                agent.getName(), dataId, group, tenant);
                LOGGER.error(message, ioe);
            }
        }
        for (CacheData cacheData : cacheDatas) {
            if (!cacheData.isInitializing() || inInitializingCacheList
                    .contains(GroupKey.getKeyTenant(cacheData.dataId, cacheData.group, cacheData.tenant))) {
                cacheData.checkListenerMd5();
                cacheData.setInitializing(false);
            }
        }
        inInitializingCacheList.clear();
         
        // 再次执行该方法，不断循环，不停监听配置文件的更新
        executorService.execute(this);
         
    } catch (Throwable e) {
         
        // If the rotation training task is abnormal, the next execution time of the task will be punished
        LOGGER.error("longPolling error : ", e);
        executorService.schedule(this, taskPenaltyTime, TimeUnit.MILLISECONDS);
    }
}
```

checkUpdateDataIds(cacheDatas, inInitializingCacheList)核心逻辑：

```java
// 拼接本地所有的dataId为一个长字符串
List<String> checkUpdateDataIds(List<CacheData> cacheDatas, List<String> inInitializingCacheList) throws Exception {
    StringBuilder sb = new StringBuilder();
    for (CacheData cacheData : cacheDatas) {
        if (!cacheData.isUseLocalConfigInfo()) {
            sb.append(cacheData.dataId).append(WORD_SEPARATOR);
            sb.append(cacheData.group).append(WORD_SEPARATOR);
            if (StringUtils.isBlank(cacheData.tenant)) {
                sb.append(cacheData.getMd5()).append(LINE_SEPARATOR);
            } else {
                sb.append(cacheData.getMd5()).append(WORD_SEPARATOR);
                sb.append(cacheData.getTenant()).append(LINE_SEPARATOR);
            }
            if (cacheData.isInitializing()) {
                // It updates when cacheData occours in cacheMap by first time.
                inInitializingCacheList
                        .add(GroupKey.getKeyTenant(cacheData.dataId, cacheData.group, cacheData.tenant));
            }
        }
    }
    boolean isInitializingCacheList = !inInitializingCacheList.isEmpty();
    return checkUpdateConfigStr(sb.toString(), isInitializingCacheList);
}

// 向服务端发起长轮询的查询逻辑，超时为30秒，不要挂起我
List<String> checkUpdateConfigStr(String probeUpdateString, boolean isInitializingCacheList) {
     
    Map<String, String> params = new HashMap<String, String>(2);
    params.put(Constants.PROBE_MODIFY_REQUEST, probeUpdateString);
    Map<String, String> headers = new HashMap<String, String>(2);
    headers.put("Long-Pulling-Timeout", "" + timeout);
     
    if (isInitializingCacheList) {
        headers.put("Long-Pulling-Timeout-No-Hangup", "true"); 
    }
     
    if (StringUtils.isBlank(probeUpdateString)) {
        return Collections.emptyList();
    }
     
    try {
        // In order to prevent the server from handling the delay of the client's long task,
        // increase the client's read timeout to avoid this problem.
         
        long readTimeoutMs = timeout + (long) Math.round(timeout >> 1);
        HttpRestResult<String> result = agent
                .httpPost(Constants.CONFIG_CONTROLLER_PATH + "/listener", headers, params, agent.getEncode(),
                        readTimeoutMs);
         
        if (result.ok()) {
            setHealthServer(true);
            return parseUpdateDataIdResponse(result.getData());
        } else {
            setHealthServer(false);
            LOGGER.error("[{}] [check-update] get changed dataId error, code: {}", agent.getName(),
                    result.getCode());
        }
    } catch (Exception e) {
        setHealthServer(false);
        LOGGER.error("[" + agent.getName() + "] [check-update] get changed dataId exception", e);
        throw e;
    }
    return Collections.emptyList();
}
```

> 总结：长轮询LongPollingRunnable长轮询任务的核心逻辑就是：
>
> - **先查询本地配置文件Failover文件**（本地配置快照Snapshot文件）；
> - 根据本地配置文件，得到**所有配置项，拼接请求参数（dataId/group/Md5/tenant），向服务端提供的长轮询接口（监听配置）：POST：/nacos/v1/cs/configs/listener 发起长轮询调用**；
> - 如果**本批次的配置项有变动，服务端就会返回有变动的配置项字符串数组**，如果没有变动，就返回空串；
> - 如果返回不为空，说明有变动，就**遍历这些变动项，然后通过具体的获取配置接口：GET：/nacos/v1/cs/configs 获取该配置项的最新配置**；
> - 最后，**再次执行本次的 LongPollingRunnable 任务，循环往复，完成配置的实时监听更新**！

#### **6 经过长轮询后，如果本地配置文件更新后，该如何通知到我们的应用程序（注册监听器）？**

在Springboot项目完成refresh(context)后，会调用 listeners.running(context) 方法，这个方法会向系统发出ApplicationReadyEvent事件，其它的监听了这个事件的Listener就可以做对应的工作了，而**NacosContextRefresher**就是其中之一！

```java
public class NacosContextRefresher implements ApplicationListener<ApplicationReadyEvent>, ApplicationContextAware {
 
    // 监听到ApplicationReadyEvent事件后，开始注册Nacos的监听器
    public void onApplicationEvent(ApplicationReadyEvent event) {
        if (this.ready.compareAndSet(false, true)) {
            this.registerNacosListenersForApplications();
        }
    }
     
     
  private void registerNacosListenersForApplications() {
        if (this.isRefreshEnabled()) {
            Iterator var1 = NacosPropertySourceRepository.getAll().iterator();
     
            while(var1.hasNext()) {
                NacosPropertySource propertySource = (NacosPropertySource)var1.next();
                if (propertySource.isRefreshable()) {  // 默认使开启的
                    String dataId = propertySource.getDataId();
                    this.registerNacosListener(propertySource.getGroup(), dataId);
                }
            }
        }
     
    }
}

```

![](https://tianxiawuhao.github.io/post-images/1658666327231.png)

> **有了这一系列的监听器，当客户端知道配置发生改变时，就会回调对应的监听器的回调方法，通知应用程序更新对应的Bean（从IOC容器中删除旧Bean，放入新的Bean）。**

------

### 三 AP模式nacos集群，服务端如何工作

首先一定要清楚，即使集群情况下，配置了mysql，当客户端查询配置时，也不是直接从mysql获取的，而是从每个节点的本地文件读取；

- **各节点本地文件：具体的配置；**
- **各节点内存缓存：存储配置的元信息，如Md5值等；**
- **Mysql：具体的配置的最新值和历史记录，方便新节点启动时加载，以及节点之间的数据同步；**

![](https://tianxiawuhao.github.io/post-images/1658666346288.png)

#### **1 配置文件的Dump加载：DumpService**

DumpService由两个实现类：EmbeddedDumpService（derby） 和 ExternalDumpService（mysql）；

当新节点启动时，需要从mysql中加载配置数据，如果最后心跳时间>6h，则从mysql加载全量数据；如果最后心跳时间<6h，则从mysql加载增量数据；

- 全量加载：删除本地的配置文件，全部从mysql加载配置数据；（每次捞取1000条）
- 增量加载：捞取最近6小时的新增配置，更新本地和内存元数据后，与mysql数据库中的配置进行比对，如果不一致，则再同步一次！

#### **2 新的配置被发布后，如何在集群间进行同步？**

- AP集群，每个节点地位平等，发布配置后，根据轮询机制，会**由某一台Server节点处理本地请求，该节点首先会将配置写入到Mysql数据库中**；
- 该节点会**发布一个ConfigDataChangeEvent事件**，该事件会**被自己的监听器处理**，处理时，会**通过HTTP调用集群的所有节点，告知配置发生改变（包括自己**，因为上面只是写mysql，自己的本地文件和内存文件也都没有被修改）
- **所有节点接收到本地配置修改的通知后，会到Mysql中同步最新的配置**，刷新内存缓存 和 本地磁盘文件；

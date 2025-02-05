在开始学习 dubbo-admin 之前，我们先了解 dubbo 应用（指集成 dubbo 框架用来远程调用的微服务项目）在注册中心（为了简化示例，此处使用注册中心作为配置中心和元数据中心，使用的注册中心为 zookeeper）使用到的路径。

## dubbo 相关的路径

在 dubbo 应用中，如果未对 namespace、group 等进行配置，则其在注册中心的路径如下：
```
- dubbo
	- config #相关配置
		- dubbo
	- metadata #所有服务元数据
		- org.apache.dubbo.samples.governance.api.DemoService #指定服务元数据
			- consumer #服务的消费方
				- governance-conditionrouter-consumer
			- provider #服务的提供方
				- governance-conditionrouter-provider
	- mapping #服务和应用的映射关系
		- org.apache.dubbo.samples.governance.api.DemoService #指定服务和应用的映射关系
- services #所有服务提供方实例
	- governance-conditionrouter-provider
```

Dubbo Admin 项目并非标准的 Dubbo 应用，它在启动时不会进行服务的注册和订阅。实际上，它是一个能够对 Dubbo 应用发布的服务进行查找、测试和修改的管理平台。

那么 Dubbo Admin 是如何感知到其他 Dubbo 应用发布的服务呢？又是如何对这些 Dubbo 服务进行测试和治理的呢？

我们可以观察到，Dubbo Admin 在启动时会初始化若干个 Bean：

1. **注册中心（Registry）**：负责服务的注册与发现，Dubbo Admin 通过注册中心获取所有已注册的服务信息。
2. **配置中心（GovernanceConfiguration）**：用于管理和配置服务的治理规则，如负载均衡策略、路由规则等。
3. **服务发现组件（ServiceDiscovery）**：用于动态发现和监控服务实例的状态和变化。
4. **服务映射组件（ServiceMapping）**：负责将服务名称映射到具体的服务实例，便于管理和调用。

结合上面介绍的路径，dubbo admin 创建的 bean 监听的路径如下图所示：

![](assets/dubbo-path.excalidraw.png)

# 源码分析

接下来我们通过分析源代码，进一步了解相关 bean 的注册过程和功能：
![config-center-uml.png](config-center-uml.png)

**GovernanceConfiguration**
后续 dubbo admin 的相关配置读取、更新由该组件处理
```java
@Bean("governanceConfiguration")  
GovernanceConfiguration getDynamicConfiguration() {  
    GovernanceConfiguration dynamicConfiguration = null;  

	// 根据配置初始化 configCenter
    if (StringUtils.isNotEmpty(configCenter)) {  
        configCenterUrl = formUrl(configCenter, configCenterGroup, configCenterGroupNameSpace, username, password);  
        // ....
        // 从 configCenter 中获取 dubbo 的其他配置（如：注册中心地址、元数据中心地址）
        String config = dynamicConfiguration.getConfig(Constants.DUBBO_PROPERTY);  

		// 如果存在以上配置，则创建注册中心、元数据中心
        if (StringUtils.isNotEmpty(config)) {  
            // ....
        }  
    }  

	// 如果没有配置 configCenter 地址，则使用注册中心作为配置中心
    if (dynamicConfiguration == null) {  
        if (StringUtils.isNotEmpty(registryAddress)) {  
            registryUrl = formUrl(registryAddress, registryGroup, registryNameSpace, username, password);  
            // ....
        } else {  
            throw new ConfigurationException("Either config center or registry address is needed, please refer to https://github.com/apache/dubbo-admin/wiki/Dubbo-Admin-configuration");  
            //throw exception  
        }  
    }  
    return dynamicConfiguration;  
}
```

**Registry**
dubbo 应用使用的注册中心，用于帮助 dubbo admin 发现、监听 dubbo 服务等信息
```java
@Bean("dubboRegistry")  
@DependsOn("governanceConfiguration")  
Registry getRegistry() {  
    Registry registry = null;  
    // 根据项目中的配置创建注册中心
    if (registryUrl == null) {  
        // ....
        registryUrl = formUrl(registryAddress, registryGroup, registryNameSpace, username, password);  
    }  
  
    logger.info("Admin using registry address: " + registryUrl);  
  
    RegistryFactory registryFactory = ApplicationModel.defaultModel().getExtensionLoader(RegistryFactory.class).getAdaptiveExtension();  
    registry = registryFactory.getRegistry(registryUrl.addParameter(ENABLE_EMPTY_PROTECTION_KEY, String.valueOf(false)));  
    return registry;  
}
```

**ServiceDiscovery**
服务发现组件，帮助 dubbo admin 发现、监听 dubbo 应用实例
```java
@Bean(destroyMethod = "destroy")  
@DependsOn("dubboRegistry")  
ServiceDiscovery getServiceDiscoveryRegistry() throws Exception {  
	// 根据注册中心地址初始化服务发现组件
    URL registryURL = registryUrl.setPath(RegistryService.class.getName())  
            .addParameter(ENABLE_EMPTY_PROTECTION_KEY, String.valueOf(false));  
    ServiceDiscoveryFactory factory = getExtension(registryURL);  
    return factory.getServiceDiscovery(registryURL);  
}
```

**MetaDataCollector**
获取相关元信息
```java
@Bean("metaDataCollector")  
@DependsOn("governanceConfiguration")  
MetaDataCollector getMetadataCollector() {  
    MetaDataCollector metaDataCollector = new NoOpMetadataCollector();  
    if (metadataUrl == null) {  
        if (StringUtils.isNotEmpty(metadataAddress)) {  
            metadataUrl = formUrl(metadataAddress, metadataGroup, metadataGroupNameSpace, username, password);  
            metadataUrl = metadataUrl.addParameter(CLUSTER_KEY, cluster);  
        }  
        logger.info("Admin using metadata address: " + metadataUrl);  
    }  
    if (metadataUrl != null) {  
        metaDataCollector = ApplicationModel.defaultModel().getExtensionLoader(MetaDataCollector.class).getExtension(metadataUrl.getProtocol());  
        metaDataCollector.setUrl(metadataUrl);  
        metaDataCollector.init();  
    } else {  
        logger.warn("you are using dubbo.registry.address, which is not recommend, please refer to: https://github.com/apache/dubbo-admin/wiki/Dubbo-Admin-configuration");  
        ApplicationModel applicationModel = ApplicationModel.defaultModel();  
        ScopeBeanFactory beanFactory = applicationModel.getBeanFactory();  
        MetadataReportInstance metadataReportInstance = beanFactory.registerBean(MetadataReportInstance.class);  
  
        Optional<MetadataReport> metadataReport = metadataReportInstance.getMetadataReports(true)  
                .values().stream().findAny();  
  
        metaDataCollector = new DubboDelegateMetadataCollector(metadataReport.get());  
        metaDataCollector.setUrl(registryUrl);  
    }  
    return metaDataCollector;  
}
```

**ServiceMapping**
服务应用映射组件，用于帮助 dubbo admin 查找服务与应用的对应关系
```java
@Bean  
@DependsOn("metaDataCollector")  
ServiceMapping getServiceMapping(ServiceDiscovery serviceDiscovery, InstanceRegistryCache instanceRegistryCache) {  
    ServiceMapping serviceMapping = new NoOpServiceMapping(); 
    // 如果配置 metadataurl，则无法感知服务与应用的映射关系 
    if (metadataUrl == null) {  
        return serviceMapping;  
    }  
    // dubbo admin 自行实现的 mappingListener，用于在本地维护服务应用映射数据的缓存
    MappingListener mappingListener = new AdminMappingListener(serviceDiscovery, instanceRegistryCache);  
    serviceMapping = ExtensionLoader.getExtensionLoader(ServiceMapping.class).getExtension(metadataUrl.getProtocol());  
    serviceMapping.addMappingListener(mappingListener);  
    // 初始化 serviceMapping，即初始拉取 dubbo 应用地址及其对应服务内容
    if (serviceMapping instanceof NacosServiceMapping) {  
        serviceMapping.init(registryUrl);  
    } else {  
        serviceMapping.init(metadataUrl);  
    }  
    return serviceMapping;  
}
```

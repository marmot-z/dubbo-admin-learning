在 dubbo 3.x 版本中，建议使用应用级别的注册模式（register-mode=instance），服务与应用的对应关系保存到 /dubbo/mapping 路径下了。那么 dubbo-admin 如何感知、监听到 dubbo 应用实例呢？

dubbo 通过 ServiceMapping、AdminMappingListener 来发现、监听应用实例。我们以 zookeeper 作为元数据中心，则 ServiceMapping 实现为 ZookeeperServiceMapping。

// TODO 补充示意图
![](./asset/instance-registry-cache.png)

```java
public class ZookeeperServiceMapping implements ServiceMapping {  
  
    @Override  
    public void init(URL url) {  
        ZookeeperTransporter zookeeperTransporter = ZookeeperTransporter.getExtension(ApplicationModel.defaultModel());  
        // url 为 registryUrl（注册中心的 url）
        zkClient = zookeeperTransporter.connect(url);  
        listenerAll();  
    }  
  
    @Override  
    public void listenerAll() {  
	    // MAPPING_PATH 固定为 /dubbo/mapping	    
        zkClient.create(MAPPING_PATH, false, false);  
        // 监听 /dubbo/mapping 下所有服务路径
        List<String> services = zkClient.addChildListener(MAPPING_PATH, (path, currentChildList) -> {  
            for (String child : currentChildList) {  
                if (anyServices.add(child)) {  
	                // 服务路径内容发生变化，则触发映射变更事件	                
                    notifyMappingChangedEvent(child);  
                }  
            }  
        });  
        // 初始化全量触发映射变更事件，用于构建本地服务实例缓存
        if (CollectionUtils.isNotEmpty(services)) {  
            for (String service : services) {  
                if (anyServices.add(service)) {  
                    notifyMappingChangedEvent(service);  
                }  
            }  
        }  
    }  
  
    private void notifyMappingChangedEvent(String service) {  
	    // 忽略 configurations、routers、consumers、providers 路径
	    // 仅对服务路径（如：org.apache.dubbo.samples.governance.api.DemoService）内容发生变化进行对应处理
        if (service.equals(Constants.CONFIGURATORS_CATEGORY) || service.equals(Constants.CONSUMERS_CATEGORY)  
                || service.equals(Constants.PROVIDERS_CATEGORY) || service.equals(Constants.ROUTERS_CATEGORY)) {  
            return;  
        }  
        String servicePath = MAPPING_PATH + Constants.PATH_SEPARATOR + service;  
        // 获取服务对应的应用名称（如：governance-conditionrouter-provider）
        String content = zkClient.getContent(servicePath);  
        if (content != null) {  
            Set<String> apps = getAppNames(content);  
            MappingChangedEvent event = new MappingChangedEvent(service, apps);  
            for (MappingListener listener : listeners) {  	   
	            // AdminMappingListener 处理映射变更事件     
                listener.onEvent(event);  
            }  
        }  
    }
}
```

AdminMappingListener

```java
public class AdminMappingListener implements MappingListener {  
    @Override  
    public void onEvent(MappingChangedEvent event) {  
        Set<String> apps = event.getApps();  
        if (CollectionUtils.isEmpty(apps)) {  
            return;  
        }  
        for (String serviceName : apps) {  
	        // 获取应用实例变更监听器，如果没有则创建
            ServiceInstancesChangedListener serviceInstancesChangedListener = serviceListeners.get(serviceName);              
            if (serviceInstancesChangedListener == null) {  
                synchronized (this) {  
                    serviceInstancesChangedListener = serviceListeners.get(serviceName);  
                    // 加锁后的双重检查
                    if (serviceInstancesChangedListener == null) {  
                        AddressChangeListener addressChangeListener = new DefaultAddressChangeListener(serviceName, instanceRegistryCache);  
                        serviceInstancesChangedListener = new AdminServiceInstancesChangedListener(Sets.newHashSet(serviceName), serviceDiscovery, addressChangeListener);    
                        List<ServiceInstance> allInstances = new ArrayList<>();  
                        // 根据应用名称（如：governance-conditionrouter-provider）从 serviceDiscovery 中获取应用实例信息
                        List<ServiceInstance> serviceInstances = serviceDiscovery.getInstances(serviceName);  
                        if (serviceDiscovery instanceof NacosServiceDiscovery) {  
                            List<ServiceInstance> consumerInstances = convertToInstance(NacosOpenapiUtil.getSubscribeAddressesWithHttpEndpoint(serviceDiscovery.getUrl(), serviceName));  
                            allInstances.addAll(consumerInstances);  
                        }  
                        if (CollectionUtils.isNotEmpty(serviceInstances)) {  
                            allInstances.addAll(serviceInstances);  
                            // 触发应用实例变更的事件
                            serviceInstancesChangedListener.onEvent(new ServiceInstancesChangedEvent(serviceName, allInstances));  
                        }  
                        serviceListeners.put(serviceName, serviceInstancesChangedListener);  
//                        serviceInstancesChangedListener.setUrl(CONSUMER_URL);  
                        serviceDiscovery.addServiceInstancesChangedListener(serviceInstancesChangedListener);  
                    }  
                }  
            }  
        }  
    }
}
```

DefaultAddressChangeListener

```java
private static class DefaultAddressChangeListener implements AddressChangeListener {  
  
    @Override  
    public void notifyAddressChanged(String protocolServiceKey, List<URL> urls) {  
        String serviceKey = removeProtocol(protocolServiceKey);  
        ConcurrentMap<String, Map<String, List<InstanceAddressURL>>> appServiceMap = instanceRegistryCache.computeIfAbsent(Constants.PROVIDERS_CATEGORY, key -> new ConcurrentHashMap<>());  
		// 从本地缓存中后去提供方和消费方内容
        Map<String, List<InstanceAddressURL>> serviceMap = appServiceMap.computeIfAbsent(serviceName, key -> new ConcurrentHashMap<>());  
        Map<String, List<URL>> consumerServiceMap = instanceRegistryCache.getSubscribedCache().computeIfAbsent(serviceName, key -> new ConcurrentHashMap<>());  

		// urls 为空，代表应用下线，本地缓存清除相关数据
        if (CollectionUtils.isEmpty(urls)) {  
            serviceMap.remove(serviceKey);  
            consumerServiceMap.remove(serviceKey);  
        } else {  
	        // 服务上线，添加消费方、提供方与应用实例的映射关系
            if ("consumer".equals(urls.get(0).getParameter(CATEGORY_KEY))) {  
                consumerServiceMap.put(serviceKey, urls);  
            } else {  
                List<InstanceAddressURL> instanceAddressUrls = urls.stream().map(url -> (InstanceAddressURL) url).collect(Collectors.toList());  
                serviceMap.put(serviceKey, instanceAddressUrls);  
            }  
        }  
        instanceRegistryCache.refreshConsumer(false);  
    }
}
```


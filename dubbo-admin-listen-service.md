前文介绍了 dubbo admin 创建了哪些组件，用于发现 dubbo 服务等用途。那么 dubbo admin 如何发现感知到 dubbo 服务的呢？

其是通过 RegistryServerSync 类订阅注册中心路径来发现服务，并更新本地缓存的。

```java
public class RegistryServerSync implements DisposableBean, NotifyListener {  
  
    private static final URL SUBSCRIBE = new URL(Constants.ADMIN_PROTOCOL, NetUtils.getLocalHost(), 0, "",  
            INTERFACE_KEY, Constants.ANY_VALUE,  
            Constants.GROUP_KEY, Constants.ANY_VALUE,  
            Constants.VERSION_KEY, Constants.ANY_VALUE,  
            Constants.CLASSIFIER_KEY, Constants.ANY_VALUE,  
            Constants.CATEGORY_KEY, Constants.PROVIDERS_CATEGORY + ","  
            + Constants.CONSUMERS_CATEGORY + ","  
            + Constants.ROUTERS_CATEGORY + ","  
            + Constants.CONFIGURATORS_CATEGORY,  
            Constants.ENABLED_KEY, Constants.ANY_VALUE,  
	            Constants.CHECK_KEY, String.valueOf(false));  
	// ....
  
    @EventListener(classes = ApplicationReadyEvent.class)  
    public void startSubscribe() {  
        logger.info("Init Dubbo Admin Sync Cache...");  
        // 应用就绪时，订阅服务中心对应地址
        registry.subscribe(SUBSCRIBE, this);  
    }  
  
    @Override  
    public void destroy() throws Exception {  
	    // 应用关闭时，取消订阅
        registry.unsubscribe(SUBSCRIBE, this);  
    }
}
```

我们可以看到在 dubbo admin 启动后，RegistryServerSync 会通过 Registry 来订阅对应路径，其会订阅以下路径的 consumers、providers、configurators、routers 子路径（如果没有对应子路径则创建）：

```
- dubbo
	- config
		- dubbo
	- metadata
		- org.apache.dubbo.samples.governance.api.DemoService
			- consumer
				- governance-conditionrouter-consumer
			- provider
				- governance-conditionrouter-provider
	- mapping
		- org.apache.dubbo.samples.governance.api.DemoService
```

当对应路径上的内容发生变化时，则会通知到 RegistryServerSync#notify 方法，更新本地服务缓存：

```java
@Override  
public void notify(List<URL> urls) {  
    if (urls == null || urls.isEmpty()) {  
        return;  
    }  
    // Map<category, Map<servicename, Map<Long, URL>>>  
    final Map<String, Map<String, Map<String, URL>>> categories = new HashMap<>();  
    String interfaceName = null;  
    for (URL url : urls) {  
        // ....

		// 根据 url 获取 category 参数
        String category = url.getUrlParam().getParameter(Constants.CATEGORY_KEY);  
        if (category == null) {  
            // .... 
        }  

		// 如果是 empty 协议，则清除本地缓存中的对应服务内容
        if (Constants.EMPTY_PROTOCOL.equalsIgnoreCase(url.getProtocol())) {  
            // ....
        } 
        // 否则，则提取服务内容信息
        else {  
            // ....
        }  
    }  
    if (categories.size() == 0) {  
        return;  
    }  

	// 将新发现的服务信息添加到本地缓存中（interfaceRegistryCache）
    for (Map.Entry<String, Map<String, Map<String, URL>>> categoryEntry : categories.entrySet()) {  
        String category = categoryEntry.getKey();  
        ConcurrentMap<String, Map<String, URL>> services = interfaceRegistryCache.get(category);  
        if (services == null) {  
            services = new ConcurrentHashMap<>();  
            interfaceRegistryCache.put(category, services);  
        } else {// Fix map can not be cleared when service is unregistered: when a unique “group/service:version” service is unregistered, but we still have the same services with different version or group, so empty protocols can not be invoked.  
            Set<String> keys = new HashSet<>(services.keySet());  
            for (String key : keys) {  
                if (Tool.getInterface(key).equals(interfaceName) && !categoryEntry.getValue().containsKey(key)) {  
                    services.remove(key);  
                }  
            }  
        }  
        services.putAll(categoryEntry.getValue());  
    }  
}
```
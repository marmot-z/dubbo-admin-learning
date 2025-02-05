dubbo admin 中可以对 dubbo 服务进行治理，其中一项治理能力就是为 dubbo 消费方设置条件路由。dubbo admin 中通过 `/api/{env}/rules/route/condition/` 接口创建条件路由。

```java
@RequestMapping(method = RequestMethod.POST)  
@ResponseStatus(HttpStatus.CREATED)  
public boolean createRule(@RequestBody ConditionRouteDTO routeDTO, @PathVariable String env) {  
    String serviceName = routeDTO.getService();  
    String app = routeDTO.getApplication();  
    if (StringUtils.isEmpty(serviceName) && StringUtils.isEmpty(app)) {  
        throw new ParamValidationException("serviceName and app is Empty!");  
    }  
    // 如果是为应用创建条件路由，则先查找是否有该 consumer，且其 dubbo version 是否大于 2.6
    if (StringUtils.isNotEmpty(app) && consumerService.findVersionInApplication(app).equals("2.6")) {  
        throw new VersionValidationException("dubbo 2.6 does not support application scope routing rule");  
    }  
    // 创建条件路由
    routeService.createConditionRoute(routeDTO);  
    return true;  
}
```

查找是否有对应 consumer 应用，此处是通过监听服务的 consumer 路径来获取服务消费方应用的，具体逻辑见 `RegistryServerSync` 类：

```java
@Override  
public String findVersionInApplication(String application) {  
    Map<String, String> filter = new HashMap<>();  
    filter.put(Constants.CATEGORY_KEY, Constants.CONSUMERS_CATEGORY);  
    filter.put(Constants.APPLICATION_KEY, application);  
    // 查找本地缓存中消费方为 ${application} 的服务 url
    Map<String, URL> stringURLMap = SyncUtils.filterFromCategory(getInterfaceRegistryCache(), filter);  
    if (stringURLMap == null || stringURLMap.isEmpty()) {  
        throw new ParamValidationException("there is no consumer for application: " + application);  
    }  
    String defaultVersion = "2.6";  
    URL url = stringURLMap.values().iterator().next();  
    // 获取消费方 dubbo 的版本，默认是 2.6
    String version = url.getParameter(Constants.SPECIFICATION_VERSION_KEY);  
    return StringUtils.isBlank(version) ? defaultVersion : version;  
}
```

之后则从配置中心中拉取对应的条件路由内容，有则更新，无则新增

```java
@Override  
public void createConditionRoute(ConditionRouteDTO conditionRoute) {  
    String id = ConvertUtil.getIdFromDTO(conditionRoute);  
    String path = getPath(id, Constants.CONDITION_ROUTE);  
    // 从配置中心拉取服务或应用的条件路由
    String existConfig = dynamicConfiguration.getConfig(path);  
    RoutingRule existRule = null;  
    // 有则拉取旧路由内容，并与新路由进行合并
    if (existConfig != null) {  
        existRule = YamlParser.loadObject(existConfig, RoutingRule.class);  
    }  
    existRule = RouteUtils.insertConditionRule(existRule, conditionRoute);  
    //register2.7  
    dynamicConfiguration.setConfig(path, YamlParser.dumpObject(existRule));  
  
    //register2.6  
    if (StringUtils.isNotEmpty(conditionRoute.getService())) {  
        for (Route old : convertRouteToOldRoute(conditionRoute)) {  
            registry.register(old.toUrl().addParameter(Constants.COMPATIBLE_CONFIG, true));  
        }  
    }  
}
```


 dubbo admin 如何将条件路由内容下发给 dubbo 应用并生效呢？

当 dubbo 应用启动时会监听配置中心对应的路径（该路径内容即为条件路由内容），如果路径内容发生变化，则加载到本地并生效。

```java
public class RegistryProtocol implements Protocol, ScopeModelAware {
	protected <T> ClusterInvoker<T> doCreateInvoker(DynamicDirectory<T> directory, Cluster cluster, Registry registry, Class<T> type) {  
	    // ....
	    // 为 invoker 创建路由链时会初始化对应路由器
	    directory.buildRouterChain(urlToRegistry);  
	    directory.subscribe(toSubscribeUrl(urlToRegistry));  
	  
	    return (ClusterInvoker<T>) cluster.join(directory, true);  
	}
}
```

部分路由器继承于 `ListenableStateRouter`，使得其具有监听路由内容的能力

```java
public abstract class ListenableStateRouter<T> extends AbstractStateRouter<T> implements ConfigurationListener {  
    public ListenableStateRouter(URL url, String ruleKey) {  
        super(url);  
        this.setForce(false);  
        // 初始化监听逻辑
        this.init(ruleKey);  
        this.ruleKey = ruleKey;  
    }  
  
    @Override  
    public synchronized void process(ConfigChangedEvent event) {  
        if (logger.isInfoEnabled()) {  
            logger.info("Notification of condition rule, change type is: " + event.getChangeType() +  
                    ", raw rule is:\n " + event.getContent());  
        }  

		// 如果路由被删除，则清空本地条件路由
        if (event.getChangeType().equals(ConfigChangeType.DELETED)) {  
            routerRule = null;  
            conditionRouters = Collections.emptyList();  
        } 
        // 新增时，则解析路由内容，创建对应的条件路由
        else {  
            try {  
                routerRule = ConditionRuleParser.parse(event.getContent());  
                generateConditions(routerRule);  
            } catch (Exception e) {  
                logger.error(CLUSTER_FAILED_RULE_PARSING,"Failed to parse the raw condition rule","","Failed to parse the raw condition rule and it will not take effect, please check " +  
                    "if the condition rule matches with the template, the raw rule is:\n " + event.getContent(),e);  
            }  
        }  
    }  
  
    private synchronized void init(String ruleKey) {  
        if (StringUtils.isEmpty(ruleKey)) {  
            return;  
        }  
        String routerKey = ruleKey + RULE_SUFFIX; 
        // 通过 ruleRepository 监听对应路由路径
        // 如果没有配置 dynamicConfiguration（配置中心配置），则实际不会进行监听 
        this.getRuleRepository().addListener(routerKey, this);  
        String rule = this.getRuleRepository().getRule(routerKey, DynamicConfiguration.DEFAULT_GROUP);  
        // 初始化加载路由
        if (StringUtils.isNotEmpty(rule)) {  
            this.process(new ConfigChangedEvent(routerKey, DynamicConfiguration.DEFAULT_GROUP, rule)); 
        }  
    }  
}
```
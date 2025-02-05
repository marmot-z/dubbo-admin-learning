一般的我们认为通过 dubbo 进行 rpc 调用时，调用方必须持有提供方相关的接口包（该包中会包含服务类、方法、实体类等 class）；且 dubbo 框架一般结合其他框架（如：Spring）一起使用，所以其生命周期一般与应用的生命周期一致，即应用启动后 dubbo 相关代码开始运行，应用销毁后 dubbo 相关代码才结束运行。

但 dubbo 还提供另一种调用方式，使得不需要预先加载服务提供方的 client 包（Generic Call）；且其支持最普通的 API 调用，即可在普通方法内完成一次 rpc 调用，而不需要提前初始化相关 dubbo 配置。具体我们可以参考官方给出的示例：
https://cn.dubbo.apache.org/en/blog/2018/08/14/generic-invoke-of-dubbo/

这就为在 dubbo admin 中对任意未知的服务进行 rpc 调用提供了技术支持，GenericServiceImpl 类中实现了 dubbo admin 对其他 dubbo 应用进行 rpc 调用的逻辑：

```java
@Component  
public class GenericServiceImpl {  
    private ApplicationConfig applicationConfig;  
    private final Registry registry;  
  
    public GenericServiceImpl(Registry registry) {  
        this.registry = registry;  
    }  
  
    @PostConstruct  
    public void init() {  
	    // 根据配置中心构建配置中心相关配置内容
        RegistryConfig registryConfig = buildRegistryConfig(registry);  

		// 构建应用配置
        applicationConfig = new ApplicationConfig();  
        applicationConfig.setName("dubbo-admin");  
        applicationConfig.setRegistry(registryConfig);  
    }  
  
    private RegistryConfig buildRegistryConfig(Registry registry) {  
        URL fromUrl = registry.getUrl();  
  
        RegistryConfig config = new RegistryConfig();  
        config.setGroup(fromUrl.getParameter("group"));  
  
        URL address = URL.valueOf(fromUrl.getProtocol() + "://" + fromUrl.getAddress());  
        if (fromUrl.hasParameter(Constants.NAMESPACE_KEY)) {  
            address = address.addParameter(Constants.NAMESPACE_KEY, fromUrl.getParameter(Constants.NAMESPACE_KEY));  
        }  
  
        config.setAddress(address.toString());  
        return config;  
    }  
  
    public Object invoke(String service, String method, String[] parameterTypes, Object[] params) {  
		// 使用 dubbo api 构建远程服务配置
        ReferenceConfig<GenericService> reference = new ReferenceConfig<>();  
        String group = Tool.getGroup(service);  
        String version = Tool.getVersion(service);  
        String intf = Tool.getInterface(service);  
        reference.setGeneric(true);  
        reference.setApplication(applicationConfig);  
        reference.setInterface(intf);  
        reference.setVersion(version);  
        reference.setGroup(group);  
        //Keep it consistent with the ConfigManager cache  
        reference.setSticky(false);  
        try {  
            removeGenericSymbol(parameterTypes);  
            // 获取远程服务的本地通用代理类（GenericService）
            GenericService genericService = reference.get();  
            // 发起 rpc 调用
            return genericService.$invoke(method, parameterTypes, params);  
        } finally {  
	        // 销毁相关远程服务的配置
            reference.destroy();  
        }  
    }  
}
```
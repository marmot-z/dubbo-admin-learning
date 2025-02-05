本项目主要介绍 dubbo-admin 项目的一些实现原理。

以下是我们后续学习需要用到的示例项目：
- [dubbo-admin](https://github.com/apache/dubbo-admin)
- [dubbo-samples-configconditionrouter:](https://github.com/apache/dubbo-samples/tree/master/4-governance/dubbo-samples-configconditionrouter)
使用的项目版本为：
- dubbo version：3.2.6
- dubbo-admin version: 0.6.0

注意：
dubbo 应用的注册方式为 instance `register-mode=instance`

目录：
- [介绍 dubbo 的一些路径以及 dubbo admin 如何监听该路径](dubbo-admin-path.md)
- [介绍 dubbo admin 如何监听服务的路由](dubbo-admin-listen-service.md)
- [介绍 dubbo admin 如何监听到服务实例的变化](dubbo-admin-service-discovery.md)
- 介绍 dubbo admin 服务治理
	- [如何增加路由，以及下发到 dubbo 服务中](dubbo-admin-create-condition-router.md)
- [介绍 dubbo admin 如何测试 dubbo 服务](dubbo-admin-test.md)

